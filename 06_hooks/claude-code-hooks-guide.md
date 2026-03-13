# Claude Code Hooks 시스템 가이드

## 학습 목표

Claude Code의 **Hooks 시스템**을 이해하고, 이벤트 기반 자동화 파이프라인을 구성하여 코드 품질 관리, 보안 검사, 알림 등을 자동으로 처리할 수 있다.

> **다른 도구와의 대응**: Antigravity의 **Workflows**, Cursor의 **Automations**, VS Code의 **Agent Hooks**에 해당하는 Claude Code 고유 기능이다.

## 사전 준비

- Claude Code 기본 사용법 숙지 (`01_agentic_coding_tool/01_cli_driven/claude-code-guide.md` 참고)
- JSON 형식에 대한 기본 이해
- 셸 스크립트 기초 지식

---

## 전체 흐름 한눈에 보기

Claude Code가 도구를 실행하는 모든 순간마다 사용자 정의 훅이 자동으로 개입한다. 파일을 수정할 때마다 포매터를 실행하거나, Bash 명령 실행 전에 보안 검사를 수행하거나, 작업 완료 시 슬랙으로 알림을 보내는 등의 자동화가 설정 파일 한 곳에서 관리된다.

1. **개념 이해** — Hooks 시스템이 해결하는 문제
2. **기본 Hook 작성** — PreToolUse / PostToolUse 설정
3. **고급 자동화** — 이벤트 조합, HTTP 훅, CI/CD 연동
4. **실전 예시** — 보안 검사, 자동 포매팅, 리포트 생성

---

## Phase 1: Hooks 시스템이란?

### 목표

Claude Code Hooks가 해결하는 문제를 이해하고 네 가지 이벤트 유형과 세 가지 훅 타입을 파악한다.

### 단계별 구현

#### Step 1.1 — Hooks가 해결하는 문제

> **💡 개념 설명: 왜 Hooks인가?**
>
> Claude Code가 파일을 수정할 때마다 수동으로 `prettier`를 실행하거나, `rm -rf /` 같은 위험한 명령이 실행되기 전에 경고를 받거나, 에이전트가 작업을 마칠 때 자동으로 보고서를 생성하려면 어떻게 해야 할까? Hooks는 이런 반복 작업을 이벤트에 연결된 자동화로 처리한다.
>
> **핵심 한 줄:** Hooks = Claude Code의 모든 행동에 자동으로 끼어드는 사용자 정의 파이프라인

#### Step 1.2 — 열여덟 가지 Hook 이벤트

**세션 라이프사이클**

| 이벤트 | 발생 시점 | 차단 가능 | 주요 활용 |
|--------|-----------|-----------|-----------|
| `SessionStart` | 새 세션 시작·재개·`/clear` 시 | No | 환경 초기화, 세션 로깅 |
| `InstructionsLoaded` | CLAUDE.md 또는 `.claude/rules/*.md` 로드 시 (비동기) | No | 규칙 감사 (종료 코드 무시) |
| `ConfigChange` | 설정 파일 변경 시 | Yes | 설정 변경 감지·차단 |
| `PreCompact` | 컨텍스트 압축 직전 | No | 압축 전 상태 스냅샷 |
| `SessionEnd` | 세션 종료 시 | No | 최종 정리 작업 |

**프롬프트 처리**

| 이벤트 | 발생 시점 | 차단 가능 | 주요 활용 |
|--------|-----------|-----------|-----------|
| `UserPromptSubmit` | 사용자 프롬프트 제출 직후 | Yes | 입력 검증, 컨텍스트 주입, 프롬프트 로깅 |

**도구 실행**

| 이벤트 | 발생 시점 | 차단 가능 | 주요 활용 |
|--------|-----------|-----------|-----------|
| `PermissionRequest` | 권한 승인 다이얼로그 표시 직전 | Yes | 자동 허용/거부, 승인 정책 |
| `PreToolUse` | 도구 실행 직전 | Yes | 보안 검사, 입력 검증 |
| `PostToolUse` | 도구 실행 성공 직후 | No* | 포매팅, 린트, 로깅 |
| `PostToolUseFailure` | 도구 실행 실패 시 | No | 에러 알림, 재시도 로직 |

> *`PostToolUse`는 실행 자체를 되돌릴 수 없지만, JSON으로 Claude에게 피드백을 전달할 수 있다.

**에이전트 제어**

| 이벤트 | 발생 시점 | 차단 가능 | 주요 활용 |
|--------|-----------|-----------|-----------|
| `SubagentStart` | 서브에이전트(Task) 생성 시 | No | 서브태스크 감사, 컨텍스트 주입 |
| `SubagentStop` | 서브에이전트 완료 시 | Yes | 결과 검증, 중단 제어 |
| `Stop` | 메인 에이전트 응답 완료 시 | Yes | 정리 작업, 보고서 생성, 추가 작업 지시 |
| `Notification` | 에이전트 알림 발생 시 | No | Slack 알림, 데스크톱 알림 |

**워크트리**

| 이벤트 | 발생 시점 | 차단 가능 | 주요 활용 |
|--------|-----------|-----------|-----------|
| `WorktreeCreate` | 워크트리 생성 시 (기본 동작 교체) | Yes | 커스텀 워크트리 생성 로직 |
| `WorktreeRemove` | 워크트리 제거 시 | No | 정리 작업 |

**멀티에이전트 팀**

| 이벤트 | 발생 시점 | 차단 가능 | 주요 활용 |
|--------|-----------|-----------|-----------|
| `TaskCompleted` | Task가 완료 상태로 마킹될 때 | Yes | 완료 조건 검증, 추가 작업 지시 |
| `TeammateIdle` | 팀 내 teammate가 idle 상태로 전환 직전 | Yes | 팀 오케스트레이션, 강제 계속 실행 |

#### Step 1.3 — 네 가지 Hook 타입

| 타입 | 설명 | 지원 이벤트 |
|------|------|-------------|
| `command` | 셸 명령 실행 | 모든 이벤트 |
| `http` | HTTP 요청 전송 | 모든 이벤트 |
| `prompt` | 추가 프롬프트 주입 | 모든 이벤트 |
| `agent` | 별도 에이전트 인스턴스 실행 | `PermissionRequest`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `Stop`, `SubagentStop`, `TaskCompleted`, `UserPromptSubmit` |

> **`agent` 타입**: 훅 자체가 Claude 에이전트로 실행되어 복잡한 판단이 필요한 자동화에 활용된다.

#### Step 1.4 — 설정 파일 위치

```
전역 설정:    ~/.claude/settings.json
프로젝트 설정: <project-root>/.claude/settings.json
```

우선순위: `프로젝트 > 전역`

#### Step 1.5 — 주요 환경 변수

| 변수 | 설명 |
|------|------|
| `$CLAUDE_FILE_PATH` | 현재 처리 중인 파일의 절대 경로 |
| `$CLAUDE_TOOL_NAME` | 실행된 도구 이름 (Edit, Write, Bash 등) |
| `$CLAUDE_SESSION_ID` | 현재 세션의 고유 ID |
| `$CLAUDE_TOOL_INPUT` | 도구에 전달된 입력 (JSON) |

### 체크포인트

열여덟 가지 Hook 이벤트와 네 가지 훅 타입(`command`, `http`, `prompt`, `agent`)을 설명할 수 있는가? 특히 차단 가능 여부가 이벤트마다 다름을 이해했는가?

---

## Phase 2: 기본 Hook 작성

### 목표

PostToolUse로 자동 포매팅 훅을 작성하고, PreToolUse로 보안 검사 훅을 구성한다.

### 단계별 구현

#### Step 2.1 — 설정 파일 기본 구조

`.claude/settings.json` 파일을 생성한다:

```json
{
  "hooks": {
    "PostToolUse": [],
    "PreToolUse": [],
    "Notification": [],
    "Stop": []
  }
}
```

#### Step 2.2 — PostToolUse: 자동 포매팅

파일 수정 후 자동으로 Prettier를 실행하는 훅이다:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

> **💡 개념 설명: matcher 패턴**
>
> `matcher`는 정규식 패턴으로 도구 이름을 필터링한다. `"Edit|Write"`는 Edit 또는 Write 도구 실행 시만 이 훅이 동작함을 의미한다. `".*"`는 모든 도구에 적용된다.
>
> **핵심 한 줄:** matcher = 특정 도구에만 훅을 적용하는 필터

#### Step 2.3 — PostToolUse: 린트 + 포매팅 조합

여러 훅을 배열로 등록하면 순서대로 실행된다:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH 2>/dev/null || true"
          },
          {
            "type": "command",
            "command": "npx eslint --fix $CLAUDE_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

#### Step 2.4 — PreToolUse: 보안 검사

Bash 명령 실행 전에 위험한 명령어를 차단하는 스크립트를 연결한다:

```bash
#!/bin/bash
# ~/.claude/hooks/security-check.sh

TOOL_INPUT="${CLAUDE_TOOL_INPUT:-}"
DANGEROUS_PATTERNS=("rm -rf /" "mkfs" "dd if=" "> /dev/sd" "chmod 777 /")

for pattern in "${DANGEROUS_PATTERNS[@]}"; do
  if echo "$TOOL_INPUT" | grep -q "$pattern"; then
    echo "보안 경고: 위험한 명령이 감지되었습니다: $pattern" >&2
    exit 1  # 비 0 종료 코드로 도구 실행 차단
  fi
done

exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

> **💡 개념 설명: 훅의 종료 코드**
>
> PreToolUse 훅이 비 0 종료 코드로 종료되면 Claude Code는 해당 도구 실행을 **중단**한다. 이를 이용해 보안 검사나 승인 프로세스를 구현할 수 있다.
>
> **핵심 한 줄:** 종료 코드 0 = 허용, 비 0 = 차단

### 체크포인트

파일 수정 시 Prettier가 자동 실행되고, 위험한 Bash 명령이 차단되면 성공이다.

---

## Phase 3: 고급 자동화

### 목표

HTTP 훅으로 외부 서비스에 알림을 보내고, Stop 훅으로 작업 완료 보고서를 생성하며, GitHub Actions와 연동하는 방법을 익힌다.

### 단계별 구현

#### Step 3.1 — Notification 훅: Slack 알림

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
            "method": "POST",
            "headers": {
              "Content-Type": "application/json"
            },
            "body": {
              "text": "Claude Code 알림: 세션 $CLAUDE_SESSION_ID에서 알림 발생"
            }
          }
        ]
      }
    ]
  }
}
```

#### Step 3.2 — Notification 훅: 데스크톱 알림

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' '작업이 완료되었습니다' --icon=dialog-information 2>/dev/null || osascript -e 'display notification \"작업 완료\" with title \"Claude Code\"' 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

#### Step 3.3 — Stop 훅: 작업 완료 보고서 생성

에이전트가 중단될 때 변경된 파일 목록과 요약 보고서를 자동 생성한다:

```bash
#!/bin/bash
# ~/.claude/hooks/generate-report.sh

REPORT_FILE=".claude/session-report-$(date +%Y%m%d-%H%M%S).md"
mkdir -p .claude

cat > "$REPORT_FILE" << EOF
# Claude Code 세션 보고서

**세션 ID**: ${CLAUDE_SESSION_ID}
**완료 시각**: $(date '+%Y-%m-%d %H:%M:%S')

## 변경된 파일

$(git diff --name-only 2>/dev/null || echo "Git 저장소 없음")

## 커밋되지 않은 변경사항

$(git status --short 2>/dev/null || echo "")
EOF

echo "보고서 생성 완료: $REPORT_FILE"
```

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/generate-report.sh"
          }
        ]
      }
    ]
  }
}
```

#### Step 3.4 — prompt 타입 훅: 추가 컨텍스트 주입

특정 도구 실행 후 에이전트에게 추가 지시를 자동으로 전달한다:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "파일을 작성한 후에는 반드시 해당 파일에 대한 단위 테스트가 있는지 확인하고, 없다면 테스트 파일도 생성해줘."
          }
        ]
      }
    ]
  }
}
```

#### Step 3.5 — GitHub Actions 통합

CI 파이프라인에서 Claude Code를 비대화형으로 실행한다:

```yaml
# .github/workflows/claude-review.yml
name: Claude Code 자동 리뷰

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Claude Code 설치
        run: npm install -g @anthropic-ai/claude-code

      - name: 변경 파일 보안 검토
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          CHANGED_FILES=$(git diff --name-only origin/main...HEAD)
          claude -p "다음 파일들의 보안 이슈를 검토하고 심각도별로 분류해줘: $CHANGED_FILES" \
            --permission-mode bypassPermissions \
            --output-format json > review-result.json

      - name: PR에 리뷰 댓글 작성
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          REVIEW=$(cat review-result.json | jq -r '.result')
          gh pr comment ${{ github.event.pull_request.number }} --body "$REVIEW"
```

#### Step 3.6 — 조건부 실행: 파일 확장자 기반 필터링

파일 확장자에 따라 서로 다른 포매터를 실행한다:

```bash
#!/bin/bash
# ~/.claude/hooks/smart-format.sh

FILE="$CLAUDE_FILE_PATH"
EXT="${FILE##*.}"

case "$EXT" in
  js|jsx|ts|tsx)
    npx prettier --write "$FILE" 2>/dev/null
    npx eslint --fix "$FILE" 2>/dev/null
    ;;
  py)
    black "$FILE" 2>/dev/null
    isort "$FILE" 2>/dev/null
    ;;
  go)
    gofmt -w "$FILE" 2>/dev/null
    ;;
  rs)
    rustfmt "$FILE" 2>/dev/null
    ;;
  *)
    # 알 수 없는 파일 형식 — 아무것도 하지 않음
    ;;
esac
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/smart-format.sh"
          }
        ]
      }
    ]
  }
}
```

### 체크포인트

작업 완료 시 보고서 파일이 생성되고, GitHub Actions 워크플로우가 PR에 자동으로 리뷰 댓글을 작성하면 성공이다.

---

## Phase 4: 실전 예시

### 목표

실제 개발 팀에서 활용할 수 있는 완성된 훅 구성을 구축한다.

### 단계별 구현

#### Step 4.1 — 완성된 프로젝트 설정 예시

```json
// .claude/settings.json (Node.js 프로젝트 기준)
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/security-check.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/smart-format.sh"
          },
          {
            "type": "command",
            "command": "echo \"[$(date '+%H:%M:%S')] 수정됨: $CLAUDE_FILE_PATH\" >> .claude/activity.log"
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/desktop-notify.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/generate-report.sh"
          }
        ]
      }
    ]
  }
}
```

#### Step 4.2 — 팀 공유 훅 vs 개인 훅 분리

```
프로젝트 공유 (.claude/settings.json):
  - 코드 품질 훅 (포매팅, 린트)
  - 보안 검사 훅
  - 로깅 훅

개인 전용 (~/.claude/settings.json):
  - 데스크톱 알림
  - 개인 Slack 채널 알림
  - 개인 보고서 설정
```

#### Step 4.3 — 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 훅이 실행되지 않음 | JSON 문법 오류 | `cat .claude/settings.json \| python3 -m json.tool`로 검증 |
| 포매팅 도구가 없음 | 패키지 미설치 | `|| true`로 오류 무시 처리 |
| 훅이 무한 루프 | PostToolUse가 다시 파일 수정 | 훅 내에서 파일 수정 금지 |
| 보안 훅이 정상 명령도 차단 | 패턴이 너무 광범위 | 정규식 패턴 범위 축소 |

### 체크포인트

완성된 설정으로 파일 수정 시 자동 포매팅, Bash 실행 시 보안 검사, 작업 완료 시 보고서 생성이 모두 동작하면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- Hooks 시스템 = 이벤트 기반 자동화의 핵심
- 열여덟 가지 이벤트: 세션(SessionStart·InstructionsLoaded·ConfigChange·PreCompact·SessionEnd), 프롬프트(UserPromptSubmit), 도구(PermissionRequest·PreToolUse·PostToolUse·PostToolUseFailure), 에이전트(SubagentStart·SubagentStop·Stop·Notification), 워크트리(WorktreeCreate·WorktreeRemove), 멀티에이전트(TaskCompleted·TeammateIdle)
- 네 가지 훅 타입: command, http, prompt, agent
- 프로젝트 설정과 전역 설정으로 팀/개인 훅 분리

### 실용적인 훅 아이디어

| 훅 | 이벤트 | 설명 |
|----|--------|------|
| 프롬프트 로깅 | UserPromptSubmit | 모든 사용자 입력 기록 |
| 컨텍스트 자동 주입 | UserPromptSubmit | 현재 git 브랜치/상태를 프롬프트에 추가 |
| 자동 포매팅 | PostToolUse/Edit | Prettier, Black, gofmt 자동 실행 |
| 보안 검사 | PreToolUse/Bash | 위험 명령 패턴 차단 |
| 테스트 자동화 | PostToolUse/Write | 새 파일 생성 시 테스트 실행 |
| Slack 알림 | Notification | 장시간 작업 완료 알림 |
| 세션 보고서 | Stop | 변경 파일 목록 및 요약 생성 |
| 활동 로그 | PostToolUse | 모든 파일 수정 이력 기록 |

---

## 다른 도구와의 비교표

| 기능 | Claude Code Hooks | Antigravity Workflows | Cursor Automations | VS Code Agent Hooks |
|------|-------------------|-----------------------|--------------------|---------------------|
| 이벤트 트리거 | 도구 실행 전/후, 알림, 중단 | 채팅 명령어 호출 | Schedule, GitHub, Slack, Webhook | 에디터 이벤트 |
| 설정 방식 | JSON | YAML 프론트매터 | GUI 설정 | JSON settings |
| 실행 위치 | 로컬 셸 | 로컬 셸 | 로컬/원격 VM | 로컬 셸 |
| CI/CD 통합 | GitHub Actions 직접 지원 | 제한적 | GitHub 이벤트 트리거 | GitHub Actions |
| HTTP 훅 | 지원 | 미지원 | Webhook 수신만 | 미지원 |
| 코드 차단 기능 | PreToolUse로 차단 가능 | 미지원 | 제한적 | 미지원 |
| 팀 공유 | 프로젝트 커밋 가능 | 프로젝트 커밋 가능 | 팀 공유 가능 | 프로젝트 커밋 가능 |
