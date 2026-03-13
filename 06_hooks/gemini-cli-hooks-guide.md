# Gemini CLI Hooks 시스템 가이드

## 학습 목표

Gemini CLI의 **Hooks 시스템**을 이해하고, Claude Code hooks와의 차이점을 파악한 뒤, BeforeTool/AfterTool 이벤트를 활용한 자동화 파이프라인을 직접 구성할 수 있다.

> **Claude Code와의 대응**: Gemini CLI의 `BeforeTool`/`AfterTool` ↔ Claude Code의 `PreToolUse`/`PostToolUse`에 해당한다. 두 시스템 모두 풍부한 이벤트 체계를 갖추고 있으나 특화 방향이 다르다. Gemini CLI는 **LLM 수준 인터셉트**와 **도구 목록 동적 필터링**이 고유 강점이고, Claude Code는 **멀티에이전트 제어**(TaskCompleted·TeammateIdle), **워크트리 이벤트**, **agent 훅 타입**(훅 자체를 AI 에이전트로 실행)이 고유 강점이다.

## 사전 준비

- Gemini CLI v0.26.0 이상 설치 (`npm install -g @google/gemini-cli`)
- `.gemini/settings.json` 파일 편집 권한
- 셸 스크립트 기초 지식
- JSON 형식에 대한 기본 이해

> **버전 주의**: Hooks 시스템은 v0.26.0(2026년 1월 26일)에 기본 활성화되었다. 이전 버전에서는 사용할 수 없다.

---

## 전체 흐름 한눈에 보기

Gemini CLI가 도구를 실행하는 모든 순간마다 사용자 정의 훅이 자동으로 개입한다. Claude Code(18가지 이벤트)보다 이벤트 수는 적지만, Gemini CLI는 **LLM 요청/응답 수준의 인터셉트**(`BeforeModel`/`AfterModel`)와 도구 목록 자체를 동적으로 필터링하는 `BeforeToolSelection`이라는 고유 기능을 제공한다. 훅은 stdin/stdout JSON으로 입출력하며, 종료 코드 2로 도구 실행을 강제 차단한다.

1. **개념 이해** — Gemini CLI Hooks가 Claude Code와 어떻게 다른가
2. **기본 설정** — `.gemini/settings.json`에 훅 등록하기
3. **도구 제어** — BeforeTool로 차단, AfterTool로 사후 처리
4. **고급 기능** — BeforeModel/AfterModel, BeforeToolSelection

---

## Phase 1: Gemini CLI Hooks 개념

### 목표

Gemini CLI Hooks의 이벤트 체계를 파악하고, Claude Code와의 차이점을 명확히 이해한다.

### 단계별 구현

#### Step 1.1 — 이벤트 체계 비교

> **💡 개념 설명: Gemini CLI Hooks는 어떻게 다른가?**
>
> Claude Code는 18가지 이벤트와 4가지 훅 타입을 지원하는 풍부한 시스템을 갖추고 있다. Gemini CLI는 이벤트 수(11가지)는 적지만, **LLM 요청/응답 수준의 인터셉트**(`BeforeModel`/`AfterModel`)와 **도구 목록 자체를 동적으로 필터링**하는 `BeforeToolSelection`이라는 고유 기능을 제공한다. 통신 방식도 환경 변수 방식과 달리 stdin/stdout JSON 프로토콜을 사용한다.
>
> **핵심 한 줄:** Gemini CLI Hooks의 차별점 = LLM 수준 인터셉트 + 도구 목록 필터링

| Gemini CLI 이벤트 | Claude Code 대응 | 트리거 시점 | 차단 가능 |
|-------------------|------------------|-------------|-----------|
| `BeforeTool` | `PreToolUse` | 도구 실행 직전 | Yes |
| `AfterTool` | `PostToolUse` | 도구 실행 직후 | Yes (결과 숨김) |
| `BeforeAgent` | `UserPromptSubmit` | 프롬프트 제출 후, 계획 수립 전 | Yes |
| `AfterAgent` | (없음) | 에이전트 루프 완료 후 | Yes (재시도 강제) |
| `BeforeModel` | (없음) | LLM 요청 전송 전 | Yes (목 응답 가능) |
| `AfterModel` | (없음) | LLM 응답 수신 후 (스트리밍 청크 단위) | Yes (청크 교체) |
| `BeforeToolSelection` | (없음) | 도구 선택 전 | 부분 (목록 필터링만) |
| `SessionStart` | `SessionStart` | 세션 시작·재개·`/clear` 시 | No |
| `SessionEnd` | `SessionEnd` | 세션 종료 시 | No |
| `Notification` | `Notification` | 시스템 알림 발생 시 | No |
| `PreCompress` | `PreCompact` | 컨텍스트 압축 전 | No |

#### Step 1.2 — 설정 파일 위치

```
전역 설정:    ~/.gemini/settings.json
프로젝트 설정: <project-root>/.gemini/settings.json
```

우선순위: `프로젝트 > 전역`

#### Step 1.3 — 훅 통신 방식

> **💡 개념 설명: JSON stdin/stdout 프로토콜**
>
> Claude Code와 Gemini CLI 모두 훅 응답에 stdout JSON을 사용하지만, **컨텍스트 전달 방식**이 다르다. Claude Code는 환경 변수(`$CLAUDE_FILE_PATH`, `$CLAUDE_TOOL_INPUT` 등)로 컨텍스트를 전달하는 반면, Gemini CLI는 **stdin으로 JSON을 수신**한다. Gemini CLI에서 stderr는 로그 전용이며, 절대 JSON을 stderr에 쓰면 안 된다.
>
> **핵심 한 줄:** (입력) Claude Code = 환경 변수, Gemini CLI = stdin JSON / (출력) 둘 다 stdout JSON + 종료 코드

| 항목 | Claude Code | Gemini CLI |
|------|-------------|------------|
| 컨텍스트 전달 | 환경 변수 (`$CLAUDE_FILE_PATH` 등) | stdin JSON (`session_id`, `tool_name` 등) |
| 응답 반환 | exit 코드 + stdout JSON (이벤트별 상이) | stdout JSON + 종료 코드 |
| 차단 방법 | exit 2 또는 JSON `{"decision":"block","reason":"..."}` | JSON `{"decision":"deny","reason":"..."}` 또는 exit 2 |
| 허용 방법 | exit 0 | `{"decision":"allow"}` 또는 exit 0 |
| 훅 타입 | command, http, prompt, agent | command만 지원 |

### 체크포인트

11가지 이벤트 유형과 Claude Code와의 대응 관계, stdin/stdout 기반 통신 방식을 설명할 수 있는가?

---

## Phase 2: 기본 훅 작성

### 목표

`.gemini/settings.json`에 BeforeTool 보안 검사 훅과 AfterTool 포매팅 훅을 등록하고 동작을 확인한다.

### 단계별 구현

#### Step 2.1 — 설정 파일 기본 구조

```json
// .gemini/settings.json

{
  "hooks": {
    "BeforeTool": [],
    "AfterTool": [],
    "BeforeAgent": [],
    "AfterAgent": [],
    "SessionStart": [],
    "SessionEnd": []
  }
}
```

#### Step 2.2 — BeforeTool: 보안 검사 훅

위험한 파일 쓰기를 차단하는 훅이다:

```bash
# ~/.gemini/hooks/security-check.sh

#!/bin/bash

# stdin에서 JSON 컨텍스트 읽기
INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('tool_name',''))")
TOOL_ARGS=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d.get('tool_args','')))")

# 위험한 경로 패턴 검사
DANGEROUS_PATHS=("/etc/" "/usr/bin/" "/usr/local/bin/" "~/.ssh/")

for path in "${DANGEROUS_PATHS[@]}"; do
  if echo "$TOOL_ARGS" | grep -q "$path"; then
    echo '{"decision":"deny","reason":"시스템 경로에 대한 쓰기 작업이 차단되었습니다."}'
    exit 0
  fi
done

echo '{"decision":"allow"}'
exit 0
```

```json
// .gemini/settings.json

{
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "write_file|create_file|replace",
        "hooks": [
          {
            "name": "security-check",
            "type": "command",
            "command": "bash ~/.gemini/hooks/security-check.sh",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```

> **💡 개념 설명: matcher 정규식**
>
> Gemini CLI와 Claude Code 모두 matcher에 정규식을 사용한다. Gemini CLI는 Python `re` 모듈 기준이며, `"write_file|create_file"` 패턴은 두 도구 이름에 매칭된다. `"write_.*"` 처럼 와일드카드 패턴도 사용 가능하다. 매처를 생략하면 모든 도구에 적용된다.
>
> **핵심 한 줄:** matcher = Python 정규식 패턴으로 도구 이름 필터링

#### Step 2.3 — AfterTool: 자동 포매팅 훅

```bash
# ~/.gemini/hooks/auto-format.sh

#!/bin/bash

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); args=d.get('tool_args',{}); print(args.get('path','') or args.get('file_path',''))" 2>/dev/null)

if [ -z "$FILE_PATH" ] || [ ! -f "$FILE_PATH" ]; then
  echo '{"decision":"allow"}'
  exit 0
fi

EXT="${FILE_PATH##*.}"
case "$EXT" in
  js|jsx|ts|tsx)
    npx prettier --write "$FILE_PATH" 2>/dev/null
    ;;
  py)
    black "$FILE_PATH" 2>/dev/null
    ;;
  go)
    gofmt -w "$FILE_PATH" 2>/dev/null
    ;;
esac

echo '{"decision":"allow"}'
exit 0
```

```json
// .gemini/settings.json (일부 발췌)

{
  "hooks": {
    "AfterTool": [
      {
        "matcher": "write_file|replace|edit_file",
        "hooks": [
          {
            "name": "auto-format",
            "type": "command",
            "command": "bash ~/.gemini/hooks/auto-format.sh",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

#### Step 2.4 — 종료 코드 정리

| 종료 코드 | 의미 | 결과 |
|-----------|------|------|
| `0` | 성공 | stdout의 JSON 파싱하여 처리 |
| `2` | 강제 차단 | 도구/턴 즉시 중단, stderr가 에러 메시지로 전달 |
| 기타 | 비치명적 경고 | 경고 표시 후 계속 진행 |

### 체크포인트

파일 쓰기 시 보안 검사가 실행되고, 코드 파일 수정 후 자동 포매팅이 동작하면 성공이다.

---

## Phase 3: 고급 기능

### 목표

BeforeToolSelection으로 도구 목록을 동적으로 제한하고, BeforeModel/AfterModel로 LLM 수준 인터셉트를 구현한다.

### 단계별 구현

#### Step 3.1 — BeforeToolSelection: 도구 목록 동적 필터링

> **💡 개념 설명: BeforeToolSelection**
>
> 이 이벤트는 Claude Code에 없는 Gemini CLI 고유 기능이다. 에이전트가 도구를 선택하기 전에 사용 가능한 도구 목록 자체를 제한할 수 있다. "읽기 전용 세션"을 만들거나 "특정 파일 형식만 접근 허용" 같은 시나리오에 유용하다.
>
> 여러 훅이 있을 경우 허용 도구 목록은 **합집합(union)** 으로 처리되며, `mode: "NONE"`만 다른 훅을 오버라이드할 수 있다.
>
> **핵심 한 줄:** BeforeToolSelection = 에이전트가 선택할 수 있는 도구 팔레트 자체를 제한

```bash
# ~/.gemini/hooks/restrict-to-readonly.sh

#!/bin/bash

# 읽기 전용 도구만 허용
echo '{
  "hookSpecificOutput": {
    "hookEventName": "BeforeToolSelection",
    "toolConfig": {
      "mode": "ANY",
      "allowedFunctionNames": ["read_file", "list_directory", "search_files", "grep"]
    }
  }
}'
exit 0
```

```json
// .gemini/settings.json (일부 발췌)

{
  "hooks": {
    "BeforeToolSelection": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "name": "readonly-mode",
            "type": "command",
            "command": "bash ~/.gemini/hooks/restrict-to-readonly.sh",
            "timeout": 3000
          }
        ]
      }
    ]
  }
}
```

#### Step 3.2 — BeforeModel: LLM 요청 수정

```bash
# ~/.gemini/hooks/inject-context.sh

#!/bin/bash

# LLM 요청에 추가 컨텍스트 주입 (예시)
# 실제 사용 시 stdin에서 현재 요청을 읽고 수정하여 반환
INPUT=$(cat)

# 그대로 통과 (수정 없이)
echo '{"decision":"allow"}'
exit 0
```

> **주의**: BeforeModel/AfterModel은 강력하지만, LLM 요청/응답을 직접 수정하면 예기치 않은 동작을 유발할 수 있다. 프로덕션 환경에서는 신중하게 사용한다.

#### Step 3.3 — SessionEnd: 세션 완료 보고서

```bash
# ~/.gemini/hooks/session-report.sh

#!/bin/bash

INPUT=$(cat)
SESSION_ID=$(echo "$INPUT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('session_id','unknown'))" 2>/dev/null)

REPORT_FILE=".gemini/session-report-$(date +%Y%m%d-%H%M%S).md"
mkdir -p .gemini

cat > "$REPORT_FILE" << EOF
# Gemini CLI 세션 보고서

**세션 ID**: ${SESSION_ID}
**완료 시각**: $(date '+%Y-%m-%d %H:%M:%S')

## 변경된 파일

$(git diff --name-only 2>/dev/null || echo "Git 저장소 없음")
EOF

echo "보고서 생성: $REPORT_FILE" >&2
echo '{"decision":"allow"}'
exit 0
```

```json
// .gemini/settings.json (일부 발췌)

{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "name": "session-report",
            "type": "command",
            "command": "bash ~/.gemini/hooks/session-report.sh"
          }
        ]
      }
    ]
  }
}
```

### 체크포인트

BeforeToolSelection으로 읽기 전용 모드가 동작하고, 세션 종료 시 보고서 파일이 생성되면 성공이다.

---

## Phase 4: 완성된 설정 예시

### 목표

실제 프로젝트에서 사용할 수 있는 완성된 훅 구성을 조합한다.

### 단계별 구현

#### Step 4.1 — 완성된 프로젝트 설정

```json
// .gemini/settings.json

{
  "hooks": {
    "BeforeTool": [
      {
        "matcher": "write_file|create_file|replace|edit_file",
        "hooks": [
          {
            "name": "security-check",
            "type": "command",
            "command": "bash ~/.gemini/hooks/security-check.sh",
            "timeout": 5000
          }
        ]
      }
    ],
    "AfterTool": [
      {
        "matcher": "write_file|replace|edit_file",
        "hooks": [
          {
            "name": "auto-format",
            "type": "command",
            "command": "bash ~/.gemini/hooks/auto-format.sh",
            "timeout": 10000
          },
          {
            "name": "activity-log",
            "type": "command",
            "command": "bash -c 'echo \"[$(date +%H:%M:%S)] AfterTool: $(cat | python3 -c \\\"import sys,json; d=json.load(sys.stdin); print(d.get(\\\"tool_name\\\",\\\"\\\"))\\\" 2>/dev/null)\" >> .gemini/activity.log; echo {\\\"decision\\\":\\\"allow\\\"}'",
            "timeout": 3000
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "name": "session-report",
            "type": "command",
            "command": "bash ~/.gemini/hooks/session-report.sh"
          }
        ]
      }
    ]
  }
}
```

#### Step 4.2 — 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 훅이 실행되지 않음 | v0.26.0 미만 버전 | `gemini --version` 확인 후 업그레이드 |
| JSON 파싱 오류 | stderr에 JSON 출력 | stderr는 로그 전용 — stdout에만 JSON 출력 |
| 훅이 항상 차단 | exit 2 잘못 사용 | exit 2는 치명적 오류 전용, 일반 차단은 JSON으로 |
| matcher가 매칭 안 됨 | 정규식 오류 | Python re 모듈 기준으로 패턴 테스트 |
| timeout 초과 | 훅 실행 시간 과다 | timeout 값 조정 또는 훅 최적화 |

---

## 마무리

### 이 가이드에서 배운 것

- Gemini CLI v0.26.0에서 공식 hooks 시스템이 도입됨 (11가지 이벤트)
- `BeforeTool`/`AfterTool` ↔ Claude Code `PreToolUse`/`PostToolUse` 대응
- **Gemini CLI 고유**: `BeforeModel`/`AfterModel` (LLM 수준 인터셉트), `BeforeToolSelection` (도구 목록 필터링), `AfterAgent` (응답 재시도 강제)
- **Claude Code 고유**: `PermissionRequest`, `SubagentStart/Stop`, `TaskCompleted`, `TeammateIdle`, `WorktreeCreate/Remove`, `agent` 훅 타입
- 컨텍스트 입력: Gemini CLI = stdin JSON / Claude Code = 환경 변수
- 훅 응답 출력: 둘 다 stdout JSON + 종료 코드 (방식은 동일)
- Gemini CLI 훅 타입은 `command`만 지원 (Claude Code는 command·http·prompt·agent 4가지)

### Claude Code vs Gemini CLI Hooks 최종 비교

**공통 기능**

| 항목 | Claude Code | Gemini CLI |
|------|-------------|------------|
| 이벤트 수 | 18가지 | 11가지 |
| 도구 실행 전 차단 | `PreToolUse` | `BeforeTool` |
| 도구 실행 후 처리 | `PostToolUse` | `AfterTool` |
| 프롬프트 인터셉트 | `UserPromptSubmit` | `BeforeAgent` |
| 세션 시작/종료 | `SessionStart` / `SessionEnd` | `SessionStart` / `SessionEnd` |
| 컨텍스트 압축 전 | `PreCompact` | `PreCompress` |
| 알림 | `Notification` | `Notification` |
| 설정 파일 | `.claude/settings.json` | `.gemini/settings.json` |

**Gemini CLI 고유 강점**

| 항목 | Claude Code | Gemini CLI |
|------|-------------|------------|
| LLM 수준 인터셉트 | 없음 | `BeforeModel` / `AfterModel` |
| 도구 목록 동적 필터링 | 없음 | `BeforeToolSelection` |
| 에이전트 응답 재시도 강제 | 없음 | `AfterAgent` (deny 시 자동 재시도) |
| 컨텍스트 입력 방식 | 환경 변수 | stdin JSON (구조화된 접근 용이) |

**Claude Code 고유 강점**

| 항목 | Claude Code | Gemini CLI |
|------|-------------|------------|
| 권한 승인 제어 | `PermissionRequest` | 없음 |
| 서브에이전트 제어 | `SubagentStart` / `SubagentStop` | 없음 |
| 멀티에이전트 팀 | `TaskCompleted` / `TeammateIdle` | 없음 |
| 워크트리 이벤트 | `WorktreeCreate` / `WorktreeRemove` | 없음 |
| 훅 타입 다양성 | command · http · prompt · **agent** | command만 지원 |
| agent 훅 타입 | 훅 자체를 Claude 에이전트로 실행 | 없음 |
| 안정화 시점 | 2024년 (초기) | 2026년 1월 (v0.26.0) |
