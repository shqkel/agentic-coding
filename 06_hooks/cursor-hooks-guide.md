# Cursor Hooks 시스템 가이드

## 학습 목표

Cursor v1.7에서 도입된 **Hooks 시스템**을 이해하고, `.cursor/hooks.json`을 작성하여 에이전트 라이프사이클 이벤트를 자동화할 수 있다. 또한 Claude Code hooks와의 관계 및 Automations 기능과의 차이를 파악한다.

> **Claude Code와의 대응**: Cursor의 `preToolUse`/`postToolUse` ↔ Claude Code의 `PreToolUse`/`PostToolUse`. Cursor는 MCP, 셸, 파일, 인라인 편집(Tab) 이벤트까지 추가로 지원한다.

## 사전 준비

- Cursor v1.7 이상 설치
- Claude Code 기본 hooks 개념 이해 (`claude-code-hooks-guide.md` 참고)
- JSON 형식에 대한 기본 이해

> **버전 주의**: Hooks는 Cursor 1.7(2025년 9월 29일)에서 베타로 도입되었다. 이전 버전에서는 사용 불가.

---

## 전체 흐름 한눈에 보기

Cursor의 hooks는 에이전트 루프의 세밀한 지점에 개입한다. `.cursor/hooks.json`에 이벤트를 등록하면 도구 실행 전후, MCP 호출, 파일 읽기/편집, 셸 실행 등 다양한 순간에 자동화가 동작한다. Claude Code hooks와 개념은 동일하지만 이벤트 명칭과 설정 파일이 다르다.

1. **개념 이해** — Cursor Hooks 이벤트 체계
2. **기본 훅 작성** — preToolUse/postToolUse 설정
3. **고급 이벤트** — MCP, 셸, 파일 이벤트 활용
4. **Automations** — hooks와의 차이점 이해

---

## Phase 1: Cursor Hooks 개념

### 목표

Cursor Hooks의 이벤트 체계와 Claude Code hooks와의 차이점을 파악한다.

### 단계별 구현

#### Step 1.1 — 전체 이벤트 목록

**에이전트 이벤트 (18가지)**

| 카테고리 | 이벤트 | 차단 가능 | Claude Code 대응 |
|----------|--------|-----------|------------------|
| 세션 | `sessionStart` | No | `SessionStart` |
| 세션 | `sessionEnd` | No | `SessionEnd` |
| 프롬프트 | `beforeSubmitPrompt` | Yes | `UserPromptSubmit` |
| 도구 | `preToolUse` | Yes | `PreToolUse` |
| 도구 | `postToolUse` | No | `PostToolUse` |
| 도구 | `postToolUseFailure` | No | `PostToolUseFailure` |
| 셸 | `beforeShellExecution` | Yes | `PreToolUse` + Bash 매처 |
| 셸 | `afterShellExecution` | No | `PostToolUse` + Bash 매처 |
| MCP | `beforeMCPExecution` | Yes | (없음) |
| MCP | `afterMCPExecution` | No | (없음) |
| 파일 | `beforeReadFile` | Yes | (없음) |
| 파일 | `afterFileEdit` | No | (없음) |
| 서브에이전트 | `subagentStart` | Yes | `SubagentStart` |
| 서브에이전트 | `subagentStop` | No | `SubagentStop` |
| 응답 | `afterAgentResponse` | No | (없음) |
| 응답 | `afterAgentThought` | No | (없음, thinking 블록) |
| 컨텍스트 | `preCompact` | No | `PreCompact` |
| 완료 | `stop` | No | `Stop` |

**Tab 이벤트 (인라인 편집, 2가지)**

| 이벤트 | 차단 가능 | 설명 |
|--------|-----------|------|
| `beforeTabFileRead` | Yes | Tab 완성 전 파일 읽기 시 |
| `afterTabFileEdit` | No | Tab 완성으로 파일 편집 후 |

#### Step 1.2 — 설정 파일 위치

```
프로젝트 설정: .cursor/hooks.json
사용자 설정:  ~/.cursor/hooks.json
```

우선순위: `프로젝트 > 사용자`

#### Step 1.3 — Claude Code hooks와의 관계

> **💡 개념 설명: Cursor의 서드파티 훅 지원**
>
> Cursor는 공식적으로 Claude Code의 `~/.claude/settings.json`에 정의된 훅을 읽어들이는 "Third-Party Hooks" 기능을 제공한다. 그러나 2026년 3월 기준으로 로딩 버그가 보고되어 있어, Claude hooks는 `.cursor/hooks.json` 형식으로 변환해 사용하는 것이 권장된다.
>
> **핵심 한 줄:** Claude hooks ≈ Cursor hooks, 하지만 파일 위치와 이벤트 명칭이 다르다

| 항목 | Claude Code | Cursor |
|------|-------------|--------|
| 설정 파일 | `.claude/settings.json` | `.cursor/hooks.json` |
| 이벤트: 도구 실행 전 | `PreToolUse` | `preToolUse` |
| 이벤트: 도구 실행 후 | `PostToolUse` | `postToolUse` |
| 이벤트: 세션 종료 | `Stop` | `sessionEnd`, `stop` |
| 매처 방식 | 정규식 문자열 | 정규식 문자열 |
| 훅 타입 | command, http, prompt | command, prompt |
| 차단 방법 | exit 2 | exit 2 |

### 체크포인트

Cursor의 20가지 이벤트(에이전트 18 + Tab 2)와 Claude Code와의 매핑을 설명할 수 있는가? 차단 가능 이벤트와 불가능 이벤트의 차이를 이해했는가?

---

## Phase 2: 기본 훅 작성

### 목표

`preToolUse`로 보안 검사를, `postToolUse`로 자동 포매팅을 구현한다.

### 단계별 구현

#### Step 2.1 — 설정 파일 기본 구조

```json
// .cursor/hooks.json

{
  "hooks": {
    "preToolUse": [],
    "postToolUse": [],
    "sessionStart": [],
    "sessionEnd": [],
    "stop": []
  }
}
```

#### Step 2.2 — preToolUse: Bash 명령 보안 검사

```bash
# ~/.cursor/hooks/security-check.sh

#!/bin/bash

TOOL_INPUT="${CURSOR_TOOL_INPUT:-}"
DANGEROUS_PATTERNS=("rm -rf /" "mkfs" "dd if=" "> /dev/sd")

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$TOOL_INPUT" | grep -q "$pattern"; then
    echo "보안 경고: 위험한 명령이 감지되었습니다: $pattern" >&2
    exit 2
  fi
done

exit 0
```

```json
// .cursor/hooks.json

{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "terminal|shell|bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.cursor/hooks/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

> **💡 개념 설명: 종료 코드**
>
> `preToolUse` 훅이 exit 2로 종료되면 Cursor는 해당 도구 실행을 **차단**한다. exit 0은 허용, 기타 코드는 경고를 표시하고 계속 진행한다.
>
> **핵심 한 줄:** exit 0 = 허용, exit 2 = 차단, 기타 = 경고 후 계속

#### Step 2.3 — postToolUse: 자동 포매팅

```json
// .cursor/hooks.json (일부 발췌)

{
  "hooks": {
    "postToolUse": [
      {
        "matcher": "edit|write|create",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CURSOR_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

#### Step 2.4 — afterFileEdit: 파일 편집 후 린트

`afterFileEdit`은 Claude Code에 없는 Cursor 고유 이벤트로, 에이전트의 파일 편집 도구뿐만 아니라 Tab(인라인 편집) 완성 후에도 발생한다.

```json
// .cursor/hooks.json (일부 발췌)

{
  "hooks": {
    "afterFileEdit": [
      {
        "matcher": ".*\\.(js|ts|jsx|tsx)$",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix $CURSOR_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### 체크포인트

위험한 셸 명령이 차단되고, 파일 편집 후 자동 포매팅이 동작하면 성공이다.

---

## Phase 3: 고급 이벤트 활용

### 목표

MCP 도구 실행 제어, 프롬프트 인터셉트, 세션 보고서 생성을 구현한다.

### 단계별 구현

#### Step 3.1 — beforeMCPExecution: MCP 도구 제어

> **💡 개념 설명: beforeMCPExecution**
>
> MCP(Model Context Protocol) 서버의 도구 실행 전에 개입할 수 있는 Cursor 고유 이벤트다. Claude Code의 `PreToolUse`가 내장 도구만 대상으로 하는 반면, 이 이벤트는 외부 MCP 서버 도구까지 제어할 수 있다.
>
> **핵심 한 줄:** beforeMCPExecution = 외부 MCP 도구에 대한 게이트키퍼

```json
// .cursor/hooks.json (일부 발췌)

{
  "hooks": {
    "beforeMCPExecution": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.cursor/hooks/mcp-audit.sh"
          }
        ]
      }
    ]
  }
}
```

#### Step 3.2 — beforeSubmitPrompt: 프롬프트 전처리

```json
// .cursor/hooks.json (일부 발췌)

{
  "hooks": {
    "beforeSubmitPrompt": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "항상 한국어로 응답하고, 코드 변경 전에 반드시 현재 파일을 읽어라."
          }
        ]
      }
    ]
  }
}
```

#### Step 3.3 — sessionEnd: 세션 보고서

```bash
# ~/.cursor/hooks/session-report.sh

#!/bin/bash

REPORT_FILE=".cursor/session-report-$(date +%Y%m%d-%H%M%S).md"
mkdir -p .cursor

cat > "$REPORT_FILE" << EOF
# Cursor 세션 보고서

**완료 시각**: $(date '+%Y-%m-%d %H:%M:%S')

## 변경된 파일

$(git diff --name-only 2>/dev/null || echo "Git 저장소 없음")

## 커밋되지 않은 변경사항

$(git status --short 2>/dev/null || echo "")
EOF

echo "보고서 생성: $REPORT_FILE"
```

```json
// .cursor/hooks.json (일부 발췌)

{
  "hooks": {
    "sessionEnd": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.cursor/hooks/session-report.sh"
          }
        ]
      }
    ]
  }
}
```

### 체크포인트

MCP 도구 실행이 감사 로그에 기록되고, 세션 종료 시 보고서가 생성되면 성공이다.

---

## Phase 4: Automations — hooks와 무엇이 다른가?

### 목표

Cursor Automations 기능을 이해하고, hooks와의 차이점을 파악하여 용도에 맞게 선택할 수 있다.

### 단계별 구현

#### Step 4.1 — Automations란?

> **💡 개념 설명: Automations vs Hooks**
>
> **Hooks**는 에이전트가 실행 중일 때 라이프사이클 이벤트에 개입하는 반면, **Automations**는 에이전트 외부의 이벤트(Slack 메시지, GitHub PR, 스케줄 등)로 에이전트를 자동 실행하는 "상시 켜진 에이전트"다. Claude Code에는 직접 대응하는 기능이 없다.
>
> **핵심 한 줄:** Hooks = 에이전트 내부 이벤트 처리 / Automations = 외부 이벤트로 에이전트 자동 실행

| 항목 | Hooks | Automations |
|------|-------|-------------|
| 트리거 | 에이전트 라이프사이클 이벤트 | 외부 서비스 이벤트, 스케줄 |
| 실행 대상 | 현재 실행 중인 에이전트에 개입 | 새 에이전트 세션을 시작 |
| 트리거 소스 | 도구 실행, 파일 편집, MCP 호출 등 | Slack, Linear, GitHub, PagerDuty, Webhook, 스케줄 |
| 용도 | 코드 품질, 보안, 로깅 | 이슈 자동 처리, PR 리뷰, 정기 리포트 |
| 설정 방식 | `.cursor/hooks.json` | Cursor 설정 UI |

#### Step 4.2 — 언제 무엇을 사용하는가?

| 시나리오 | 사용 기능 |
|----------|-----------|
| 파일 수정 시마다 자동 포매팅 | **Hooks** (postToolUse) |
| 위험한 명령 실행 차단 | **Hooks** (preToolUse) |
| Slack 메시지 수신 시 자동 작업 | **Automations** |
| GitHub PR 생성 시 자동 리뷰 | **Automations** |
| 매일 정해진 시간에 리포트 생성 | **Automations** |
| 세션 종료 시 요약 보고서 생성 | **Hooks** (sessionEnd) |

---

## 마무리

### 이 가이드에서 배운 것

- Cursor v1.7에서 자체 Hooks 시스템 도입 (`.cursor/hooks.json`)
- 에이전트 18가지 + Tab 2가지 = 총 20가지 이벤트 (MCP, 파일, 서브에이전트, 응답 관찰 포함)
- `preToolUse`/`postToolUse` = Claude Code `PreToolUse`/`PostToolUse`에 직접 대응
- Automations = hooks와 별개 기능, 외부 이벤트로 에이전트 자동 실행

### 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 훅이 실행되지 않음 | Cursor v1.7 미만 | Cursor 업데이트 필요 |
| Claude hooks가 Cursor에서 미동작 | 로딩 버그 | `.cursor/hooks.json`으로 재작성 |
| matcher가 매칭 안 됨 | 정규식 오류 | 이벤트별 매처 패턴 확인 |
| exit 2가 차단 안 됨 | 이벤트 타입 불일치 | 차단 지원 이벤트 확인 (postToolUse는 차단 불가) |

### Automations 공식 트리거 목록 (2026년 3월 기준)

- Slack 메시지/채널
- Linear 이슈 생성/업데이트
- GitHub PR/이슈 이벤트
- PagerDuty 알림
- 커스텀 Webhook
- 시간 기반 스케줄
