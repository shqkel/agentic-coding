# 06_hooks — 도구별 Hooks 시스템 가이드

AI 코딩 도구의 Hooks(이벤트 기반 자동화) 시스템을 비교하고, 각 도구별 실습 가이드를 제공한다.

---

## 도구별 지원 현황 (2026년 3월 기준)

| 도구 | Hooks 지원 | 이벤트 수 | 훅 타입 | 안정화 시점 |
|------|-----------|-----------|---------|------------|
| **Claude Code** | ✅ 완전 지원 | 18가지 | command·http·prompt·agent | 2024년 (초기) |
| **Gemini CLI** | ✅ 완전 지원 | 11가지 | command | 2026년 1월 (v0.26.0) |
| **Cursor** | ✅ 완전 지원 | 20가지 | command | 2025년 9월 (v1.7) |
| **VSCode** | ✅ CLI 공유 | Claude Code와 동일 | Claude Code와 동일 | Claude Code와 동일 |
| **Codex CLI** | ⚠️ 부분 지원 | 3가지 (실험적) | command | 2026년 3월 (v0.114.0, 실험적) |
| **Antigravity** | ❌ 미지원 | — | — | — |

---

## 이벤트 대응표

Claude Code 이벤트를 기준으로 각 도구의 대응 이벤트를 정리한다.

### 세션 라이프사이클

| Claude Code | 차단 | Gemini CLI | Cursor | Codex CLI |
|-------------|------|------------|--------|-----------|
| `SessionStart` | No | `SessionStart` | `sessionStart` | SessionStart (실험적) |
| `InstructionsLoaded` | No | — | — | — |
| `ConfigChange` | Yes | — | — | — |
| `PreCompact` | No | `PreCompress` | `preCompact` | — |
| `SessionEnd` | No | `SessionEnd` | `sessionEnd` | SessionStop (실험적) |

### 프롬프트 처리

| Claude Code | 차단 | Gemini CLI | Cursor | Codex CLI |
|-------------|------|------------|--------|-----------|
| `UserPromptSubmit` | Yes | `BeforeAgent` | `beforeSubmitPrompt` | — |

### 도구 실행

| Claude Code | 차단 | Gemini CLI | Cursor | Codex CLI |
|-------------|------|------------|--------|-----------|
| `PermissionRequest` | Yes | — | — | — |
| `PreToolUse` | Yes | `BeforeTool` | `preToolUse` | — |
| `PostToolUse` | No* | `AfterTool` | `postToolUse` | — |
| `PostToolUseFailure` | No | — | `postToolUseFailure` | — |

> *Claude Code의 PostToolUse는 실행을 되돌릴 수 없지만, JSON으로 Claude에게 피드백을 전달할 수 있다.

### 에이전트 제어

| Claude Code | 차단 | Gemini CLI | Cursor | Codex CLI |
|-------------|------|------------|--------|-----------|
| `SubagentStart` | No | — | `subagentStart` | — |
| `SubagentStop` | Yes | — | `subagentStop` | — |
| `Stop` | Yes | `SessionEnd` 유사 | `stop` | SessionStop (실험적) |
| `Notification` | No | `Notification` | — | `notify` |

### 멀티에이전트 / 워크트리 (Claude Code 고유)

| Claude Code | 차단 | 설명 |
|-------------|------|------|
| `TaskCompleted` | Yes | Task 완료 마킹 시 |
| `TeammateIdle` | Yes | 팀 내 teammate idle 전환 직전 |
| `WorktreeCreate` | Yes | 워크트리 생성 시 (기본 동작 교체) |
| `WorktreeRemove` | No | 워크트리 제거 시 |

### Gemini CLI 고유 이벤트

| Gemini CLI | 차단 | 설명 |
|------------|------|------|
| `AfterAgent` | Yes (재시도 강제) | 에이전트 루프 완료 후 |
| `BeforeModel` | Yes | LLM 요청 전송 전 (목 응답 주입 가능) |
| `AfterModel` | Yes | LLM 응답 수신 후 (스트리밍 청크 단위) |
| `BeforeToolSelection` | 부분 (목록 필터링) | 도구 선택 전 |

### Cursor 고유 이벤트

| Cursor | 차단 | 설명 |
|--------|------|------|
| `beforeShellExecution` | Yes | 셸 명령 실행 전 |
| `afterShellExecution` | No | 셸 명령 실행 후 |
| `beforeMCPExecution` | Yes | MCP 도구 실행 전 |
| `afterMCPExecution` | No | MCP 도구 실행 후 |
| `beforeReadFile` | Yes | 파일 읽기 전 |
| `afterFileEdit` | No | 파일 편집 후 |
| `afterAgentResponse` | No | 에이전트 메시지 완료 후 |
| `afterAgentThought` | No | thinking 블록 완료 후 |
| `beforeTabFileRead` | Yes | Tab 인라인 완성 전 파일 읽기 시 |
| `afterTabFileEdit` | No | Tab 인라인 완성으로 파일 편집 후 |

---

## 설정 파일 위치

| 도구 | 전역 설정 | 프로젝트 설정 |
|------|-----------|--------------|
| Claude Code / VSCode | `~/.claude/settings.json` | `.claude/settings.json` |
| Gemini CLI | `~/.gemini/settings.json` | `.gemini/settings.json` |
| Cursor | `~/.cursor/hooks.json` | `.cursor/hooks.json` |
| Codex CLI | `~/.codex/config.toml` | `.codex/config.toml` |
| Antigravity | `~/.gemini/GEMINI.md` | `.agent/rules/` |

---

## 가이드 파일

| 파일 | 내용 |
|------|------|
| [`claude-code-hooks-guide.md`](./claude-code-hooks-guide.md) | 18가지 이벤트, 4가지 훅 타입(command·http·prompt·agent) 실습 |
| [`gemini-cli-hooks-guide.md`](./gemini-cli-hooks-guide.md) | 11가지 이벤트, BeforeModel/AfterModel/BeforeToolSelection 고유 기능 |
| [`cursor-hooks-guide.md`](./cursor-hooks-guide.md) | 20가지 이벤트, Automations와의 차이 |
| [`vscode-hooks-guide.md`](./vscode-hooks-guide.md) | CLI 공유 구조, VSCode 전용 설정 조합 |
| [`codex-hooks-guide.md`](./codex-hooks-guide.md) | notify 훅, 승인 정책, 현재 한계 |
| [`antigravity-hooks-guide.md`](./antigravity-hooks-guide.md) | Hooks 부재, Denylist·Rules·Workflows 대체 메커니즘 |

---

## 주요 검증 사항

| 항목 | 원본 조사 | 검증 결과 |
|------|-----------|-----------|
| Claude Code 이벤트 수 | 4가지 | **18가지** (SessionStart·InstructionsLoaded·ConfigChange·UserPromptSubmit·PermissionRequest·PreToolUse·PostToolUse·PostToolUseFailure·PreCompact·Notification·SubagentStart·SubagentStop·Stop·TaskCompleted·TeammateIdle·WorktreeCreate·WorktreeRemove·SessionEnd) |
| Claude Code 훅 타입 | 3가지 | **4가지** (command·http·prompt·**agent** 추가) |
| Gemini CLI 이벤트 수 | "최근 추가됨" | **11가지**, v0.26.0(2026-01-26)에 안정화 |
| Gemini CLI 이슈 번호 | #4641 | **#2779** (원본 오류) |
| Cursor hooks | "Claude hooks 직접 지원" | **자체 시스템**, `.cursor/hooks.json` 별도 작성 필요. Claude hooks 로딩 버그로 미동작 |
| Cursor 이벤트 수 | 16가지 | **20가지** (subagentStart/Stop·afterAgentResponse·afterAgentThought 추가) |
| Codex Automations | "CLI 기능" | **Codex App 전용**. CLI는 notify+실험적 SessionStart/Stop만 존재 |
| Antigravity Denylist | "PreToolUse 유사" | **터미널 명령만** 적용, MCP 도구 차단 불가. Hooks 시스템 없음 |
