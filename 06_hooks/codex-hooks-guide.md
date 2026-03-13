# OpenAI Codex CLI Hooks 가이드

## 학습 목표

OpenAI Codex CLI의 **현재 Hooks 지원 범위**를 정확히 파악하고, 제한된 이벤트 시스템(`notify` 훅, SessionStart/Stop) 안에서 가능한 자동화를 구성할 수 있다. 또한 Codex App의 Automations 기능과의 차이를 이해한다.

> **⚠️ 중요**: Codex CLI의 hooks는 2026년 3월 기준 **실험적 단계**이며, Claude Code의 `PreToolUse`/`PostToolUse`에 해당하는 기능은 아직 존재하지 않는다. 이 가이드는 현재 가능한 것과 불가능한 것을 명확히 구분하여 설명한다.

## 사전 준비

- Codex CLI v0.114.0 이상 설치 (`npm install -g @openai/codex`)
- OpenAI API 키
- TOML 형식에 대한 기본 이해

---

## 전체 흐름 한눈에 보기

Codex CLI의 hooks 시스템은 Claude Code에 비해 매우 제한적이다. 도구 실행 수준의 인터셉트는 없으며, 현재 지원되는 것은 에이전트 턴 완료 알림(`notify`)과 실험적 세션 시작/종료 이벤트뿐이다. 도구 실행을 제어하려면 승인 정책(`approval_policy`)을 통한 정적 설정을 사용해야 한다.

1. **현황 파악** — Codex CLI hooks의 지원 범위
2. **notify 훅** — 에이전트 턴 완료 알림 구현
3. **승인 정책** — 도구 실행 제어 방법
4. **Automations** — Codex App의 외부 이벤트 자동화

---

## Phase 1: Codex CLI Hooks 현황

### 목표

Codex CLI hooks의 실제 지원 범위와 Claude Code와의 차이를 정확히 파악한다.

### 단계별 구현

#### Step 1.1 — 지원 현황 (2026년 3월 기준)

| 기능 | Claude Code | Codex CLI |
|------|-------------|-----------|
| `PreToolUse` (도구 실행 전 차단) | **지원** | **미지원** |
| `PostToolUse` (도구 실행 후 처리) | **지원** | **미지원** |
| 세션 시작 이벤트 | 미지원 | 실험적 지원 (v0.114.0+) |
| 세션 종료 이벤트 | `Stop` | 실험적 지원 (v0.114.0+) |
| 에이전트 턴 완료 알림 | `Notification` | `notify` 훅 (안정) |
| HTTP 훅 | 지원 | 미지원 |
| 프롬프트 주입 훅 | 지원 | 미지원 |

> **배경**: GitHub Issue #2109("Event Hooks")가 489개 이상의 찬성표를 받았으나, OpenAI는 CLI/IDE 확장/웹 전반에 걸친 설계 검토가 필요하다는 이유로 커뮤니티 PR을 거절했다. v0.114.0(2026-03-11)에서야 SessionStart/Stop을 실험적으로 추가했다.

#### Step 1.2 — 설정 파일 위치

```
전역 설정: ~/.codex/config.toml
프로젝트 설정: <project-root>/.codex/config.toml
```

### 체크포인트

Codex CLI hooks가 현재 어떤 이벤트를 지원하는지, 그리고 PreToolUse/PostToolUse가 아직 없다는 것을 설명할 수 있는가?

---

## Phase 2: notify 훅 — 에이전트 턴 완료 알림

### 목표

`notify` 훅으로 에이전트가 턴을 완료할 때마다 알림을 받는 자동화를 구현한다.

### 단계별 구현

#### Step 2.1 — notify 훅이란?

> **💡 개념 설명: notify 훅**
>
> `notify`는 에이전트가 하나의 응답 턴을 완료할 때마다 외부 명령을 실행한다. Claude Code의 `Notification` 훅과 유사하지만, **차단 기능은 없다** — fire-and-forget 방식이다. 훅은 JSON 페이로드를 stdin으로 받는다.
>
> **핵심 한 줄:** notify = 에이전트 턴 완료 시 실행되는 알림 전용 훅 (차단 불가)

```toml
# ~/.codex/config.toml

notify = ["python3", "/path/to/notify-handler.py"]
```

또는 여러 명령을 순서대로 실행하려면:

```toml
# ~/.codex/config.toml

notify = ["bash", "-c", "~/.codex/hooks/on-turn-complete.sh"]
```

#### Step 2.2 — notify 훅 페이로드

훅 스크립트는 stdin으로 다음 JSON을 받는다:

```json
{
  "type": "agent-turn-complete",
  "thread-id": "thread_abc123",
  "last-assistant-message": "파일을 수정했습니다.",
  "input-messages": [...],
  "turn-id": "turn_xyz789",
  "cwd": "/home/user/project"
}
```

#### Step 2.3 — 데스크톱 알림 구현

```bash
# ~/.codex/hooks/on-turn-complete.sh

#!/bin/bash

INPUT=$(cat)
MESSAGE=$(echo "$INPUT" | python3 -c "
import sys, json
d = json.load(sys.stdin)
msg = d.get('last-assistant-message', '작업 완료')
# 긴 메시지는 50자로 자름
print(msg[:50] + ('...' if len(msg) > 50 else ''))
" 2>/dev/null || echo "Codex 작업 완료")

# macOS
osascript -e "display notification \"$MESSAGE\" with title \"Codex CLI\"" 2>/dev/null

# Linux
notify-send "Codex CLI" "$MESSAGE" 2>/dev/null

# 활동 로그 기록
CWD=$(echo "$INPUT" | python3 -c "import sys,json; print(json.load(sys.stdin).get('cwd',''))" 2>/dev/null)
echo "[$(date '+%H:%M:%S')] $MESSAGE" >> "$HOME/.codex/activity.log"
```

```toml
# ~/.codex/config.toml

notify = ["bash", "-c", "bash ~/.codex/hooks/on-turn-complete.sh"]
```

#### Step 2.4 — Slack Webhook 알림

```python
# ~/.codex/hooks/slack-notify.py

#!/usr/bin/env python3
import sys, json, urllib.request, urllib.parse

data = json.load(sys.stdin)
message = data.get("last-assistant-message", "")[:200]
thread_id = data.get("thread-id", "")

payload = json.dumps({
    "text": f"*Codex CLI* 턴 완료\n*Thread*: `{thread_id}`\n{message}"
}).encode()

WEBHOOK_URL = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

try:
    req = urllib.request.Request(
        WEBHOOK_URL,
        data=payload,
        headers={"Content-Type": "application/json"}
    )
    urllib.request.urlopen(req, timeout=5)
except Exception as e:
    print(f"Slack 알림 실패: {e}", file=sys.stderr)
```

```toml
# ~/.codex/config.toml

notify = ["python3", "/home/user/.codex/hooks/slack-notify.py"]
```

### 체크포인트

에이전트가 턴을 완료할 때마다 데스크톱 알림이 표시되고 활동 로그에 기록되면 성공이다.

---

## Phase 3: 승인 정책 — 도구 실행 제어

### 목표

hooks 대신 승인 정책(`approval_policy`)을 사용하여 도구 실행을 제어한다.

### 단계별 구현

#### Step 3.1 — 승인 정책이란?

> **💡 개념 설명: PreToolUse 없이 도구를 제어하는 방법**
>
> Codex CLI는 `PreToolUse` hooks가 없지만, 승인 정책으로 도구 실행 전 사용자 확인을 강제할 수 있다. 이는 동적 스크립트 검사가 아니라 정적 규칙 기반이라는 점에서 Claude Code hooks와 다르다.
>
> **핵심 한 줄:** approval_policy = 정적 규칙 기반 도구 실행 게이트 (동적 스크립트 검사 불가)

| `approval_policy` 값 | 동작 |
|----------------------|------|
| `untrusted` | 모든 도구 실행 전 사용자 확인 요청 |
| `on-request` | 에이전트가 필요하다고 판단할 때만 확인 요청 |
| `never` | 모든 도구를 무조건 실행 (Automations에서 사용) |

#### Step 3.2 — 기본 승인 정책 설정

```toml
# ~/.codex/config.toml

[settings]
approval_policy = "untrusted"

# 특정 도구 비활성화
[apps.default.tools.shell]
enabled = false

[apps.default.tools.filesystem]
enabled = true
```

#### Step 3.3 — 완성된 config.toml 예시

```toml
# ~/.codex/config.toml

[settings]
# 모든 도구 실행 전 확인 (가장 안전)
approval_policy = "untrusted"

# 에이전트 턴 완료 시 알림
notify = ["bash", "-c", "bash ~/.codex/hooks/on-turn-complete.sh"]

# 사용 모델
model = "o4-mini"

# 에이전트 응답 언어 설정
[instructions]
system = "항상 한국어로 응답하라."
```

### 체크포인트

`approval_policy = "untrusted"` 설정으로 모든 도구 실행 전 확인 메시지가 표시되면 성공이다.

---

## Phase 4: Codex App Automations

### 목표

Codex App의 Automations 기능을 이해하고, CLI hooks와의 차이점을 파악한다.

### 단계별 구현

#### Step 4.1 — Automations는 CLI가 아닌 App 기능

> **💡 개념 설명: Codex CLI vs Codex App**
>
> OpenAI Codex 생태계는 두 가지 별도 제품으로 구성된다. **Codex CLI**는 오픈소스 터미널 에이전트이고, **Codex App**은 GUI 데스크톱 애플리케이션이다. Automations는 Codex App 전용 기능으로, CLI에서는 사용할 수 없다.
>
> **핵심 한 줄:** Automations = Codex App 전용, CLI hooks와 별개

| 항목 | Codex CLI | Codex App Automations |
|------|-----------|----------------------|
| 실행 환경 | 터미널 | GUI 데스크톱 |
| 트리거 | 수동 실행 + notify 훅 | 스케줄, Skill 조합 |
| 승인 | approval_policy | 완전 자동 (never) |
| 결과 확인 | 터미널 출력 | "Triage" 인박스 |
| 오픈소스 | Yes | No |

#### Step 4.2 — Automations 동작 방식

```
스케줄 또는 수동 트리거
    ↓
Skill 선택 ($skill-name 구문)
    ↓
전용 Git worktree에서 격리 실행
    ↓
결과 → "Triage" 인박스
    ↓
(보고할 내용 없으면 자동 아카이브)
```

---

## 마무리

### 이 가이드에서 배운 것

- Codex CLI hooks는 2026년 3월 기준 실험적 단계 — `PreToolUse`/`PostToolUse` 없음
- 안정적으로 사용 가능한 것: `notify` 훅 (에이전트 턴 완료 알림)
- 실험적으로 추가된 것: `SessionStart`/`Stop` (v0.114.0+)
- 도구 실행 제어는 `approval_policy`로 정적 관리
- Automations는 CLI가 아닌 Codex App 전용 기능

### Claude Code 대비 현재 제한사항

| 기능 | Claude Code | Codex CLI | 상태 |
|------|-------------|-----------|------|
| 도구 실행 전 차단 | `PreToolUse` | 없음 | Issue #2109 요청 중 |
| 도구 실행 후 처리 | `PostToolUse` | 없음 | 미정 |
| HTTP 훅 | 지원 | 미지원 | 미정 |
| 프롬프트 주입 | 지원 | 미지원 | 미정 |
| 세션 이벤트 | 지원 | 실험적 | v0.114.0+ |
| 에이전트 알림 | `Notification` | `notify` (안정) | 사용 가능 |

### 로드맵 참고

- GitHub Issue #2109 (Event Hooks): 489+ 찬성표, 커뮤니티 강력 요청 중
- Issue #8317 (시간 기반 스케줄링): CLI에 없는 Automations 기능 요청 중
