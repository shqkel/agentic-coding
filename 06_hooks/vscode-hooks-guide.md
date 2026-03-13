# VSCode Claude Code Extension Hooks 가이드

## 학습 목표

VSCode Claude Code 확장에서 **Hooks를 CLI와 동일하게 활용**하는 방법을 이해하고, VSCode 고유 설정과 공유 설정의 차이를 파악하여 IDE 환경에 최적화된 자동화를 구성할 수 있다.

> **핵심**: VSCode Claude Code 확장은 `~/.claude/settings.json`의 Hooks를 CLI와 완전히 공유한다. Claude Code CLI hooks 가이드에서 배운 내용을 그대로 VSCode에서 활용할 수 있다.

## 사전 준비

- VSCode 1.98.0 이상 설치
- Claude Code 확장 설치 (VSCode Marketplace: `anthropic.claude-code`)
- Claude Code CLI hooks 기본 개념 이해 (`claude-code-hooks-guide.md` 참고)

---

## 전체 흐름 한눈에 보기

VSCode Claude Code 확장은 Claude Code CLI와 동일한 `~/.claude/settings.json`을 읽는다. 따라서 한 곳에서 훅을 설정하면 CLI와 VSCode 양쪽에서 동일하게 동작한다. VSCode 고유 설정(UI 동작, 권한 모드 기본값 등)은 VSCode extension settings에서 별도로 관리된다.

1. **설정 구조 이해** — 공유 설정 vs VSCode 전용 설정
2. **훅 설정 확인** — VSCode에서 훅 동작 검증
3. **VSCode 고유 설정** — 확장 설정으로 에이전트 동작 조정

---

## Phase 1: 설정 구조 이해

### 목표

VSCode와 CLI가 공유하는 설정과 VSCode 전용 설정의 차이를 명확히 파악한다.

### 단계별 구현

#### Step 1.1 — 두 가지 설정 영역

> **💡 개념 설명: 설정 분리 원칙**
>
> Anthropic은 설정을 두 영역으로 분리한다. **공유 설정** (`~/.claude/settings.json`)은 CLI와 VSCode 확장 모두에서 동일하게 적용된다. **VSCode 전용 설정**은 IDE UI 동작에 관한 것으로, VSCode extension settings에서만 관리된다. Hooks는 항상 공유 설정에 속한다.
>
> **핵심 한 줄:** Hooks → `~/.claude/settings.json` (CLI와 완전 공유)

| 설정 항목 | 위치 | CLI 적용 | VSCode 적용 |
|-----------|------|----------|-------------|
| Hooks | `~/.claude/settings.json` | Yes | Yes |
| MCP 서버 | `~/.claude/settings.json` | Yes | Yes |
| 허용 명령어 (allowedCommands) | `~/.claude/settings.json` | Yes | Yes |
| 환경 변수 | `~/.claude/settings.json` | Yes | Yes |
| 프로젝트 권한 모드 기본값 | VSCode settings.json | No | Yes |
| 인라인 편집 자동 수락 | VSCode settings.json | No | Yes |
| 확장 UI 테마 | VSCode settings.json | No | Yes |

#### Step 1.2 — 설정 파일 위치 정리

```
공유 전역 설정:    ~/.claude/settings.json          ← Hooks 여기에!
공유 프로젝트 설정: <project-root>/.claude/settings.json  ← 프로젝트 훅 여기에!
VSCode 확장 설정:  VSCode 설정 UI (Ctrl+, → "Claude Code")
```

#### Step 1.3 — VSCode에서 훅 설정 접근

VSCode 명령 팔레트(`Ctrl+Shift+P`)에서 `Claude: Open Settings`를 실행하면 `~/.claude/settings.json`이 열린다. 또는 VSCode 채팅 패널의 `/` 커맨드 메뉴 → "Customize" 섹션에서 직접 접근할 수 있다.

### 체크포인트

`~/.claude/settings.json`에 등록한 훅이 `claude` CLI와 VSCode 확장 양쪽에서 동일하게 동작함을 확인할 수 있는가?

---

## Phase 2: VSCode 환경에서 훅 활용

### 목표

VSCode 환경에 최적화된 훅을 구성한다.

### 단계별 구현

#### Step 2.1 — 기본 훅 확인

Claude Code CLI hooks 가이드의 모든 예시는 VSCode에서 그대로 동작한다. 예시:

```json
// ~/.claude/settings.json

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
    ],
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

> Claude Code CLI hooks 예시 전체는 `claude-code-hooks-guide.md`를 참고한다.

#### Step 2.2 — VSCode 전용 Notification 훅

VSCode 환경에서 특히 유용한 데스크톱 알림 훅이다:

```json
// ~/.claude/settings.json (일부 발췌)

{
  "hooks": {
    "Notification": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Claude Code 작업 완료\" with title \"VSCode\"' 2>/dev/null || notify-send 'Claude Code' '작업 완료' 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

#### Step 2.3 — 프로젝트별 훅 설정

팀 공유 훅은 프로젝트 설정에, 개인 훅은 전역 설정에 분리한다:

```
프로젝트 설정 (.claude/settings.json):  ← Git에 커밋하여 팀 공유
  - 코드 품질 훅 (포매팅, 린트)
  - 보안 검사 훅
  - 로깅 훅

전역 설정 (~/.claude/settings.json):   ← 개인 전용
  - 데스크톱 알림
  - 개인 Slack 채널 알림
  - 개인 보고서 설정
```

---

## Phase 3: VSCode 전용 설정으로 에이전트 동작 조정

### 목표

VSCode extension settings으로 Hooks와 함께 에이전트 동작을 세밀하게 조정한다.

### 단계별 구현

#### Step 3.1 — VSCode 주요 확장 설정

VSCode settings (`Ctrl+,` → "Claude Code" 검색):

| 설정 항목 | 설명 | 권장값 |
|-----------|------|--------|
| `claude-code.initialPermissionMode` | 시작 시 권한 모드 | `default` |
| `claude-code.autoAcceptEdits` | 파일 편집 자동 수락 | `false` (훅 검토 후 수락 권장) |
| `claude-code.enableStatusBar` | 상태 표시줄 표시 | `true` |

#### Step 3.2 — 훅과 권한 모드 조합

```json
// ~/.claude/settings.json (일부 발췌)

{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm *)",
      "Bash(npx prettier *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  },
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

> **💡 개념 설명: permissions vs PreToolUse hooks**
>
> `permissions.deny`는 정적 규칙 기반 차단이고, `PreToolUse` hooks는 동적 스크립트 기반 차단이다. 둘을 조합하면 레이어드 보안이 구성된다: 알려진 위험 명령은 permissions으로 빠르게 차단하고, 복잡한 패턴 검사는 hooks 스크립트에서 처리한다.
>
> **핵심 한 줄:** permissions = 정적 화이트/블랙리스트, hooks = 동적 스크립트 검사

---

## 마무리

### 이 가이드에서 배운 것

- VSCode Claude Code 확장은 `~/.claude/settings.json` hooks를 CLI와 완전히 공유
- CLI hooks 가이드의 모든 예시를 VSCode에서 그대로 사용 가능
- VSCode 전용 설정은 extension settings에서 별도 관리
- permissions + hooks 조합으로 레이어드 보안 구성 가능

### VSCode에서만 유의할 점

| 항목 | 주의 사항 |
|------|-----------|
| 설정 파일 경로 | `~/.claude/settings.json`이 공유 파일임을 기억할 것 |
| 훅 스크립트 경로 | 절대 경로 사용 권장 (`~/.claude/hooks/...`) |
| 알림 훅 | VSCode 알림 API와 별개 — OS 네이티브 알림으로 구현 |
| 프로젝트 훅 | `.claude/settings.json`은 Git에 커밋하여 팀 공유 가능 |

### 공식 문서 참고

- VSCode 통합 가이드: `https://code.claude.com/docs/en/vs-code`
- Hooks 레퍼런스: `https://code.claude.com/docs/en/hooks`
- Hooks 퀵스타트: `https://code.claude.com/docs/en/hooks-guide`
