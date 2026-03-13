# OpenAI Codex CLI 플러그인/확장 학습 가이드

## 개요

Codex CLI는 OpenAI가 공개한 오픈소스 터미널 기반 AI 코딩 에이전트이다. Claude Code의 Plugin이나 Gemini CLI의 Extension처럼 **하나의 패키지로 묶어 배포하는 통합 플러그인 시스템이 존재하지 않는다**. 대신, 서로 독립적인 5가지 확장 포인트를 조합하여 유사한 기능을 구현한다.

이 가이드는 Codex CLI의 확장 체계를 이해하고, 각 확장 포인트를 실전에서 활용하는 방법을 다룬다.

---

## 1. Codex CLI의 확장 철학: "플러그인 시스템 없는 확장"

### 핵심 차이점

Claude Code에는 `plugin.json`이라는 매니페스트 파일이 있다. 이 파일 하나에 명령어, 스킬, 에이전트, MCP 서버, 훅, LSP 서버를 모두 선언하고 하나의 패키지로 배포한다. Gemini CLI도 Extension이라는 단일 패키지 개념을 가진다.

Codex CLI는 이런 "단일 패키지" 개념이 **없다**. 대신 다음 5가지 확장 포인트가 각각 독립적으로 동작한다.

```
Claude Code의 접근       Codex CLI의 접근
────────────────       ────────────────
┌─────────────┐        ┌──────────────┐  ┌──────────────┐
│  Plugin      │        │ MCP 서버      │  │ 슬래시 명령어  │
│  (plugin.json│        │ (config.toml) │  │ (prompts/)   │
│   통합 패키지) │        └──────────────┘  └──────────────┘
│             │        ┌──────────────┐  ┌──────────────┐
│ Commands    │        │ Skills       │  │ AGENTS.md    │
│ Skills      │        │ (skills/)    │  │ (프로젝트 루트) │
│ Agents      │        └──────────────┘  └──────────────┘
│ MCP         │        ┌──────────────┐
│ Hooks       │        │ config.toml  │
│ LSP         │        │ (프로필)      │
└─────────────┘        └──────────────┘
```

왼쪽은 하나의 상자에 모든 것이 들어 있다. 오른쪽은 개별 상자들이 흩어져 있고, 개발자가 직접 조합해야 한다. 이것이 Codex CLI 확장 체계의 가장 중요한 특징이다.

### 이 구조의 장단점

**장점**: 필요한 확장 포인트만 골라 쓸 수 있다. MCP 서버만 추가하고 나머지는 건드리지 않아도 된다.

**단점**: 팀에 공유할 때 "이 config.toml 설정을 복사하고, 이 skills 폴더도 가져가고, 이 prompts 파일도 넣어야 해"라고 일일이 안내해야 한다. 통합 배포 패키지가 없기 때문이다.

---

## 2. 핵심 확장 포인트 5가지

### 2-1. MCP 서버 (주력 확장 메커니즘)

**한 줄 설명**: Codex CLI가 **외부 시스템(API, DB, 서비스)과 연결되는 통로**이며, Codex CLI에서 가장 강력한 확장 메커니즘이다.

**왜 MCP가 주력인가**

Codex CLI의 확장 포인트 중 MCP 서버가 가장 범용적이고 성숙한 메커니즘이다. 커스텀 도구를 정의하고, 외부 서비스와 통신하며, 복잡한 로직을 서버 측에서 처리할 수 있다. Claude Code에서 Plugin이 중심 역할을 한다면, Codex CLI에서는 MCP 서버가 그 역할을 대신한다.

**설정 방법**

`~/.codex/config.toml` 파일에 MCP 서버를 등록한다. STDIO 기반 서버와 Streaming HTTP 서버 두 가지 방식을 지원한다.

```toml
# ~/.codex/config.toml

# STDIO 기반 MCP 서버 (가장 일반적)
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_TOKEN = "ghp_xxx"

# 파일시스템 MCP 서버
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]

# Streaming HTTP 기반 MCP 서버
[mcp_servers.remote_api]
url = "https://my-server.example.com/mcp"
```

**CLI 명령으로 관리**

```bash
# MCP 서버 목록 확인
codex mcp list

# 특정 MCP 서버의 도구 목록 확인
codex mcp tools github

# Codex CLI 자체를 MCP 서버로 실행 (다른 에이전트에서 호출 가능)
codex --mcp
```

`codex --mcp` 명령은 Codex CLI 자체를 MCP 서버로 노출한다. 이렇게 하면 다른 AI 에이전트(Claude Code, Cursor 등)에서 Codex CLI를 도구로 사용할 수 있다. 이것은 에이전트 간 협업의 가능성을 열어준다.

**동작 원리**

세션이 시작되면 Codex CLI는 `config.toml`에 등록된 MCP 서버를 자동으로 실행한다. 서버가 제공하는 도구(tool)들은 Codex의 도구 목록에 추가되며, AI 모델이 필요할 때 호출할 수 있게 된다.

```
사용자 → Codex CLI → AI 모델 → "GitHub 이슈를 확인해야겠다"
                                    ↓
                        MCP 서버 (server-github) 호출
                                    ↓
                        GitHub API → 이슈 목록 반환
                                    ↓
                        AI 모델 → 결과를 사용자에게 설명
```

**언제 쓰는가**: 외부 서비스와의 연동이 필요할 때. GitHub, 데이터베이스, 클라우드 서비스 등 Codex가 기본으로 접근할 수 없는 시스템에 연결할 때 사용한다.

---

### 2-2. 커스텀 슬래시 명령어 (Custom Slash Commands)

**한 줄 설명**: 사용자가 `/명령어`로 직접 실행하는 **재사용 가능한 프롬프트 템플릿**이다.

**어떻게 동작하는가**

`~/.codex/prompts/` 디렉토리에 Markdown 파일을 만들면, 파일명이 곧 명령어 이름이 된다. 파일 내용은 AI에게 전달되는 프롬프트로 동작한다. Claude Code의 커스텀 명령어와 기본 개념은 동일하지만, 구현 방식에 차이가 있다.

```markdown
<!-- ~/.codex/prompts/review.md -->
---
description: "코드 리뷰 수행"
argument_hint: "[파일 경로]"
---

다음 파일의 코드를 리뷰해주세요: $1

리뷰 기준:
- 버그 가능성
- 성능 이슈
- 보안 취약점
- 코드 스타일 일관성

발견된 문제를 심각도 순으로 정리하고, 각각에 대한 수정 제안을 포함해주세요.
```

이 파일을 만들면 Codex CLI 세션에서 `/review src/main.ts`처럼 사용할 수 있다.

**YAML Front Matter**

| 필드 | 필수 | 설명 |
|------|------|------|
| `description` | 권장 | 명령어 설명 (자동완성 시 표시) |
| `argument_hint` | 선택 | 인자 형식 힌트 |

**인자 시스템**

Codex CLI의 슬래시 명령어는 두 가지 인자 방식을 지원한다.

```markdown
<!-- 위치 인자: $1, $2, ... $9 -->
---
description: "두 파일을 비교"
argument_hint: "[파일1] [파일2]"
---

다음 두 파일을 비교 분석해주세요:
- 파일 1: $1
- 파일 2: $2

<!-- 명명 변수: $VARIABLE_NAME -->
---
description: "특정 언어로 문서 번역"
argument_hint: "[파일 경로] [대상 언어]"
---

$1 파일의 내용을 $TARGET_LANGUAGE 로 번역해주세요.
```

**디스커버리**: 세션이 시작되면 Codex CLI가 `~/.codex/prompts/` 디렉토리를 자동으로 스캔한다. 새 파일을 추가하면 다음 세션부터 바로 사용할 수 있다.

**실용 예시: 커밋 메시지 생성기**

```markdown
<!-- ~/.codex/prompts/commit.md -->
---
description: "컨벤셔널 커밋 메시지 생성"
argument_hint: "[커밋 유형: feat|fix|refactor|docs|test]"
---

현재 스테이징된 변경사항을 분석하고,
Conventional Commits 형식으로 커밋 메시지를 생성해주세요.

커밋 유형: $1

규칙:
- 제목은 50자 이내
- 본문은 변경 이유를 설명
- Breaking change가 있으면 BREAKING CHANGE: 접두사 추가
```

**Claude Code Commands와의 차이**

| 구분 | Claude Code | Codex CLI |
|------|-------------|-----------|
| 위치 | `.claude/commands/` | `~/.codex/prompts/` |
| 네임스페이싱 | 플러그인명으로 자동 격리 | 없음 (파일명 = 명령어명) |
| 인자 | `$ARGUMENTS` 단일 변수 | `$1`-`$9` 위치 + `$VAR` 명명 변수 |
| 프로젝트별 | 지원 | 미지원 (글로벌만) |

**언제 쓰는가**: 반복적으로 사용하는 프롬프트를 표준화하고 싶을 때. "매번 같은 말을 타이핑한다"는 느낌이 들면 슬래시 명령어로 만들 때이다.

---

### 2-3. Skills 시스템 (권장되는 확장 방식)

**한 줄 설명**: 특정 상황에서 Codex가 **참조할 수 있는 전문 지식과 지시사항을 담은 디렉토리**이다.

**왜 Skills인가**

Codex CLI에서 커스텀 프롬프트(`prompts/`)는 레거시로 분류되고 있으며, Skills가 이를 대체하는 현대적 메커니즘으로 권장된다. Skills는 텍스트 기반 지시사항에 선택적으로 스크립트를 결합할 수 있어 더 강력하다.

**디렉토리 구조**

```
~/.codex/skills/          ← 글로벌 스킬 (모든 프로젝트에 적용)
  └── code-reviewer/
      ├── SKILL.md        ← 필수: 스킬 정의 파일
      └── review.sh       ← 선택: 보조 스크립트

.codex/skills/            ← 프로젝트 스킬 (해당 프로젝트에만 적용)
  └── deploy-helper/
      └── SKILL.md
```

**SKILL.md 작성법**

```markdown
<!-- ~/.codex/skills/code-reviewer/SKILL.md -->
# 코드 리뷰 전문가

이 스킬이 활성화되면 다음 기준으로 코드를 검토한다:

## 검토 기준
1. **버그 가능성**: null 참조, 경계값 오류, 레이스 컨디션
2. **성능 이슈**: O(n^2) 루프, 불필요한 메모리 할당, N+1 쿼리
3. **보안 취약점**: 인젝션, XSS, 하드코딩된 시크릿
4. **유지보수성**: 매직 넘버, 중복 코드, 과도한 결합

## 출력 형식
발견된 문제를 심각도(Critical/Major/Minor) 순으로 정리하여 제시한다.
각 항목에 수정 제안을 포함한다.
```

**슬래시 명령어와의 차이**

| 구분 | Slash Commands (prompts/) | Skills (skills/) |
|------|---------------------------|-------------------|
| 위치 | `~/.codex/prompts/` | `~/.codex/skills/` 또는 `.codex/skills/` |
| 실행 | 사용자가 `/명령어`로 호출 | Codex가 맥락에 따라 참조 |
| 스코프 | 글로벌만 | 글로벌 + 프로젝트 |
| 보조 파일 | 미지원 | 스크립트 등 포함 가능 |
| 상태 | 레거시 전환 중 | 권장 (현대적) |

**주의**: Skills 시스템은 아직 **실험적(experimental)** 단계이다. CLI에서의 동작이 안정화되지 않은 부분이 있으며, 향후 인터페이스가 변경될 수 있다.

**언제 쓰는가**: "이런 종류의 작업에서는 항상 이 규칙을 따라라"는 전문 지식을 Codex에 내재화하고 싶을 때. 특히 프로젝트별로 다른 규칙을 적용해야 할 때 유용하다.

---

### 2-4. AGENTS.md (프로젝트 컨텍스트)

**한 줄 설명**: 프로젝트의 **코딩 표준, 아키텍처 규칙, 작업 패턴을 정의하는 프로젝트 지시 파일**이다.

**Claude Code의 CLAUDE.md에 대응하는 개념이다.** 프로젝트 루트에 위치하며, Codex CLI 세션이 시작될 때 자동으로 로드된다.

**작성 예시**

```markdown
<!-- AGENTS.md (프로젝트 루트) -->
# 프로젝트 컨텍스트

## 아키텍처
- Next.js 14 App Router 사용
- 서버 컴포넌트 우선, 클라이언트 컴포넌트는 최소화
- 데이터베이스: PostgreSQL + Prisma ORM

## 코딩 표준
- TypeScript strict 모드 필수
- 함수형 컴포넌트만 사용 (클래스 컴포넌트 금지)
- ESLint + Prettier 설정 준수
- 테스트: Vitest + Testing Library

## 커밋 규칙
- Conventional Commits 형식
- PR은 단일 기능 단위로 분리

## 금지 사항
- `any` 타입 사용 금지
- `console.log` 프로덕션 코드에 남기지 않기
- `.env` 파일을 커밋하지 않기
```

**스코프 계층**

AGENTS.md는 프로젝트 내 여러 위치에 배치할 수 있다. Codex CLI는 현재 작업 디렉토리 기준으로 상위 디렉토리를 순회하며 모든 AGENTS.md를 수집한다.

```
project-root/
├── AGENTS.md                 ← 프로젝트 전체 규칙
├── src/
│   ├── AGENTS.md             ← src 하위 공통 규칙
│   ├── frontend/
│   │   └── AGENTS.md         ← 프론트엔드 전용 규칙
│   └── backend/
│       └── AGENTS.md         ← 백엔드 전용 규칙
```

하위 디렉토리의 AGENTS.md가 상위의 것을 보완하며, 충돌 시 더 가까운(하위) 것이 우선한다.

**Claude Code CLAUDE.md와의 비교**

| 구분 | Claude Code | Codex CLI |
|------|-------------|-----------|
| 파일명 | `CLAUDE.md` | `AGENTS.md` |
| 위치 | 프로젝트 루트 + 서브디렉토리 | 프로젝트 루트 + 서브디렉토리 |
| 자동 로드 | 세션 시작 시 | 세션 시작 시 |
| 계층 구조 | 지원 | 지원 |
| 글로벌 설정 | `~/.claude/CLAUDE.md` | `~/.codex/instructions.md` |

**언제 쓰는가**: 프로젝트에 합류하는 모든 개발자(그리고 AI)가 동일한 규칙을 따르게 하고 싶을 때. 팀 컨벤션을 코드화하는 가장 직접적인 방법이다.

---

### 2-5. config.toml 프로필 (설정 레벨 확장)

**한 줄 설명**: 모델 선택, 추론 레벨, 승인 정책 등을 **글로벌 및 프로젝트 레벨로 분리 관리하는 설정 파일**이다.

**설정 파일 위치**

| 레벨 | 경로 | 용도 |
|------|------|------|
| 글로벌 | `~/.codex/config.toml` | 모든 프로젝트의 기본값 |
| 프로젝트 | `./codex.toml` (프로젝트 루트) | 프로젝트별 오버라이드 |

프로젝트 레벨 설정이 글로벌 설정을 덮어쓴다.

**주요 설정 항목**

```toml
# ~/.codex/config.toml

# 모델 선택
model = "o4-mini"              # 기본 모델
# model = "gpt-4.1"           # 더 강력한 모델이 필요할 때
# model = "o3"                # 최고 성능 추론 모델

# 추론 레벨 (o-시리즈 모델 전용)
reasoning_effort = "medium"    # low, medium, high

# 승인 정책: 도구 실행 전 사용자 확인 수준
approval_policy = "unless-allow-listed"
# "suggest"        → 모든 명령에 사용자 승인 필요
# "auto-edit"      → 파일 편집은 자동, 명령 실행은 승인
# "full-auto"      → 모든 작업 자동 실행 (주의 필요)

# 샌드박스 정책
sandbox_policy = "permissive"
# "docker"         → Docker 컨테이너 격리
# "permissive"     → 제한 없음
```

**승인 정책 상세**

승인 정책은 Codex CLI의 안전 메커니즘이다. Claude Code의 `--dangerously-skip-permissions`와 유사하지만, 더 세분화된 단계를 제공한다.

| 정책 | 파일 읽기 | 파일 쓰기 | 명령 실행 | 용도 |
|------|-----------|-----------|-----------|------|
| `suggest` | 자동 | 승인 필요 | 승인 필요 | 최대 안전 (기본값) |
| `auto-edit` | 자동 | 자동 | 승인 필요 | 일반 개발 |
| `full-auto` | 자동 | 자동 | 자동 | CI/CD, 자동화 |

**CLI 플래그로 오버라이드**

설정 파일의 값은 CLI 플래그로 임시 변경할 수 있다.

```bash
# 모델 변경
codex --model gpt-4.1 "이 버그를 수정해줘"

# 승인 정책 변경
codex --approval-mode full-auto "테스트를 실행하고 실패하는 것을 수정해줘"

# 추론 레벨 변경
codex --reasoning-effort high "이 아키텍처를 분석해줘"
```

**언제 쓰는가**: 프로젝트마다 다른 모델이나 안전 수준을 적용하고 싶을 때. 예를 들어 오픈소스 프로젝트에서는 `suggest` 모드로, 개인 실험 프로젝트에서는 `full-auto` 모드로 동작하게 할 수 있다.

---

## 3. 마켓플레이스: 존재하지 않는다

### Claude Code와의 가장 큰 차이

Claude Code는 `/plugin marketplace add`로 마켓플레이스를 등록하고, `/plugin install`로 플러그인을 설치하는 **앱스토어 모델**을 갖추고 있다. Anthropic이 관리하는 공식 마켓플레이스도 존재한다.

Codex CLI에는 이에 해당하는 것이 **전혀 없다**.

### 그러면 어떻게 공유하는가

| 확장 포인트 | 배포 방식 |
|-------------|-----------|
| MCP 서버 | npm 패키지로 개별 배포 (`npx -y @패키지명`) |
| Skills | Git 저장소를 클론해서 `~/.codex/skills/`에 복사 |
| 슬래시 명령어 | Markdown 파일을 직접 공유 |
| AGENTS.md | 프로젝트 저장소에 커밋 |
| config.toml | dotfiles 저장소로 관리 |

MCP 서버의 경우, npm 생태계가 사실상 마켓플레이스 역할을 한다. `@modelcontextprotocol/server-github`, `@modelcontextprotocol/server-postgres` 같은 공식 패키지가 npm에 게시되어 있으며, `npx -y` 명령으로 설치 없이 바로 실행할 수 있다.

---

## 4. 제약사항

### 훅 시스템의 한계

Claude Code는 4가지 이벤트 훅을 제공한다.

| 훅 | Claude Code | Codex CLI |
|----|-------------|-----------|
| PreToolUse | 지원 (도구 실행 전 차단 가능) | **미지원** |
| PostToolUse | 지원 (도구 실행 후 반응) | **미지원** |
| Notification | 지원 | notify만 제한적 지원 |
| Stop | 지원 (작업 완료 후 정리) | **미지원** |

Codex CLI에서는 `notify` 훅만 사용할 수 있다. Claude Code처럼 "파일을 수정한 직후 자동으로 Prettier를 실행"하는 PostToolUse 훅이나, "위험한 명령 실행을 차단"하는 PreToolUse 훅은 구현할 수 없다. 이 기능은 GitHub Issue #2109에서 요청되고 있으며, 향후 추가될 가능성이 있다.

### 기타 제약사항

| 제약 | 상세 |
|------|------|
| LSP 미지원 | Claude Code는 LSP 서버를 연결하여 IDE 수준의 코드 이해가 가능하지만, Codex CLI는 미지원 |
| Skills 실험적 | CLI에서 Skills가 아직 실험 단계이며 안정성이 보장되지 않음 |
| 에이전트(Subagent) 미지원 | Claude Code의 독립 컨텍스트 서브에이전트에 대응하는 기능 없음 |
| 통합 패키지 배포 없음 | 여러 확장 포인트를 하나로 묶어 배포하는 메커니즘 부재 |
| 프로젝트별 슬래시 명령어 없음 | `prompts/`는 글로벌만 지원하며, 프로젝트별 명령어 정의 불가 |

---

## 5. Claude Code Plugin과의 종합 비교

| 구분 | Claude Code | Codex CLI |
|------|-------------|-----------|
| **통합 패키지** | Plugin (`plugin.json`) | 없음 (개별 확장 포인트) |
| **마켓플레이스** | 공식 + 커스텀 마켓 지원 | 없음 |
| **MCP 서버** | `.mcp.json`으로 설정 | `config.toml`로 설정 (**주력**) |
| **슬래시 명령어** | `.claude/commands/` | `~/.codex/prompts/` |
| **Skills** | `.claude/skills/` (안정) | `.codex/skills/` (**실험적**) |
| **에이전트** | `agents/` (서브에이전트) | 미지원 |
| **프로젝트 지시** | `CLAUDE.md` | `AGENTS.md` |
| **글로벌 지시** | `~/.claude/CLAUDE.md` | `~/.codex/instructions.md` |
| **훅 이벤트** | 4종 (Pre/Post/Notify/Stop) | notify만 |
| **LSP** | 지원 | 미지원 |
| **승인 정책** | 단일 (`--dangerously-skip-permissions`) | 3단계 (`suggest` / `auto-edit` / `full-auto`) |
| **MCP 서버로 동작** | 미지원 | `codex --mcp`로 가능 |

이 표에서 주목할 점은 두 가지이다.

첫째, Codex CLI가 우위인 영역이 있다. 승인 정책이 더 세분화되어 있고, Codex 자체를 MCP 서버로 노출하는 기능은 Claude Code에 없다.

둘째, 확장 체계의 성숙도에서 Claude Code가 앞서 있다. 통합 패키지, 마켓플레이스, 훅 시스템, LSP, 서브에이전트 등에서 Codex CLI가 아직 지원하지 않는 부분이 많다.

---

## 6. 실용 예시: "플러그인처럼" 구성하기

Codex CLI에 통합 플러그인 시스템이 없다고 해서 유사한 경험을 만들 수 없는 것은 아니다. 여러 확장 포인트를 수동으로 조합하면 된다.

### 시나리오: "프론트엔드 개발 킷" 만들기

프론트엔드 개발에 필요한 도구를 하나의 Git 저장소로 관리하는 예시이다.

**저장소 구조**

```
codex-frontend-kit/
├── README.md
├── install.sh               ← 설치 스크립트
├── config/
│   └── config.toml           ← MCP 서버 설정 (병합용)
├── skills/
│   ├── react-expert/
│   │   └── SKILL.md          ← React 코딩 규칙
│   ├── accessibility/
│   │   └── SKILL.md          ← 접근성 검사 규칙
│   └── performance/
│       └── SKILL.md          ← 성능 최적화 규칙
├── prompts/
│   ├── component.md          ← /component 명령어
│   ├── test.md               ← /test 명령어
│   └── storybook.md          ← /storybook 명령어
└── agents-template.md        ← AGENTS.md 템플릿
```

**설치 스크립트**

```bash
#!/bin/bash
# install.sh - Codex Frontend Kit 설치

CODEX_DIR="$HOME/.codex"
KIT_DIR="$(cd "$(dirname "$0")" && pwd)"

echo "=== Codex Frontend Kit 설치 ==="

# 1. Skills 복사
echo "[1/3] Skills 설치 중..."
mkdir -p "$CODEX_DIR/skills"
cp -r "$KIT_DIR/skills/"* "$CODEX_DIR/skills/"

# 2. 슬래시 명령어 복사
echo "[2/3] 슬래시 명령어 설치 중..."
mkdir -p "$CODEX_DIR/prompts"
cp "$KIT_DIR/prompts/"*.md "$CODEX_DIR/prompts/"

# 3. config.toml에 MCP 서버 추가 안내
echo "[3/3] MCP 서버 설정..."
echo ""
echo "다음 내용을 ~/.codex/config.toml에 추가하세요:"
echo ""
cat "$KIT_DIR/config/config.toml"

echo ""
echo "=== 설치 완료 ==="
echo "새 Codex CLI 세션을 시작하면 적용됩니다."
```

**Skills 예시: React 전문가**

```markdown
<!-- skills/react-expert/SKILL.md -->
# React 개발 전문가

## 컴포넌트 작성 규칙
- 함수형 컴포넌트만 사용한다
- Props는 반드시 TypeScript interface로 정의한다
- `React.FC` 대신 명시적 반환 타입을 사용한다
- 커스텀 훅은 `use` 접두사를 붙인다

## 상태 관리
- 컴포넌트 로컬 상태: useState
- 서버 상태: TanStack Query
- 전역 상태: Zustand (Redux 사용 금지)

## 성능
- 리스트 렌더링 시 반드시 key prop 사용
- 불필요한 리렌더링 방지: useMemo, useCallback 적절히 사용
- 이미지는 next/image 컴포넌트 사용
```

**슬래시 명령어 예시: 컴포넌트 생성기**

```markdown
<!-- prompts/component.md -->
---
description: "React 컴포넌트 생성"
argument_hint: "[컴포넌트명]"
---

$1 이름의 React 컴포넌트를 생성해주세요.

생성할 파일:
1. `src/components/$1/$1.tsx` - 컴포넌트 본체
2. `src/components/$1/$1.test.tsx` - 테스트
3. `src/components/$1/$1.stories.tsx` - Storybook 스토리
4. `src/components/$1/index.ts` - barrel export

TypeScript strict, 함수형 컴포넌트, Props interface 필수.
```

### 버전 관리 전략

이 "킷"을 Git 저장소로 관리하면, 팀원들은 다음처럼 설치한다.

```bash
# 설치
git clone https://github.com/myteam/codex-frontend-kit.git
cd codex-frontend-kit
bash install.sh

# 업데이트
cd codex-frontend-kit
git pull
bash install.sh
```

Claude Code의 `/plugin install`에 비하면 번거롭지만, 동일한 목적을 달성할 수 있다.

---

## 7. 전체 확장 포인트 한눈에 보기

```
Codex CLI 확장 체계 (개별 확장 포인트의 조합)
│
├── MCP 서버 (주력)                → 외부 서비스 연동
│   ├── config.toml에 등록
│   ├── STDIO / Streaming HTTP 지원
│   └── codex --mcp로 자체 MCP 서버화
│
├── 커스텀 슬래시 명령어            → 사용자가 명시적으로 호출
│   ├── ~/.codex/prompts/*.md
│   └── 위치 인자 + 명명 변수 지원
│
├── Skills (실험적)                → 맥락 기반 전문 지식
│   ├── ~/.codex/skills/ (글로벌)
│   ├── .codex/skills/ (프로젝트)
│   └── SKILL.md + 보조 스크립트
│
├── AGENTS.md                     → 프로젝트 컨텍스트/규칙
│   ├── 프로젝트 루트 + 서브디렉토리
│   └── 계층적 로드 (하위 우선)
│
└── config.toml 프로필              → 모델/승인/샌드박스 설정
    ├── ~/.codex/config.toml (글로벌)
    └── ./codex.toml (프로젝트)
```

---

## 8. 향후 전망

### 커스텀 프롬프트에서 Skills로의 전환

Codex CLI 개발 로드맵에서 `~/.codex/prompts/` 기반 커스텀 슬래시 명령어는 레거시로 분류되고 있다. Skills 시스템이 안정화되면 커스텀 프롬프트를 대체할 것으로 예상된다. 새로운 프로젝트에서는 처음부터 Skills를 사용하는 것이 권장된다.

### 훅 시스템 확장

GitHub Issue #2109에서 PreToolUse/PostToolUse 훅에 대한 커뮤니티 요구가 활발하다. 이 기능이 추가되면 Claude Code의 훅 시스템에 근접한 자동화가 가능해진다.

### 통합 패키지 시스템의 가능성

현재로서는 공식적인 로드맵에 통합 플러그인 시스템이 포함되어 있지 않다. 그러나 OpenAI가 Codex CLI를 오픈소스로 공개한 만큼, 커뮤니티 기여를 통해 이런 기능이 추가될 가능성은 열려 있다.

---

## 9. 자주 하는 실수

**실수 1**: config.toml에서 MCP 서버 환경변수를 직접 하드코딩한다
```toml
# 위험: 토큰이 파일에 직접 노출
[mcp_servers.github.env]
GITHUB_TOKEN = "ghp_abc123실제토큰"

# 권장: 환경변수를 참조하거나 별도 관리
# 시스템 환경변수에 GITHUB_TOKEN을 설정한 뒤 config.toml에서 참조
```

**실수 2**: Skills와 슬래시 명령어의 용도를 혼동한다. 슬래시 명령어는 "사용자가 명시적으로 실행"하는 것이고, Skills는 "Codex가 맥락에 따라 참조"하는 것이다. 반복 워크플로우는 슬래시 명령어로, 코딩 규칙은 Skills로 만든다.

**실수 3**: `full-auto` 승인 정책을 무분별하게 사용한다. `full-auto`는 모든 파일 수정과 명령 실행을 자동 승인한다. 프로덕션 코드베이스에서 사용하면 의도치 않은 변경이 발생할 수 있다. 신뢰할 수 있는 환경(CI/CD, 샌드박스)에서만 사용한다.

**실수 4**: AGENTS.md를 `.gitignore`에 추가한다. AGENTS.md는 프로젝트의 규칙을 정의하는 파일이므로 팀원 전체가 공유해야 한다. 반드시 버전 관리에 포함한다.

---

## 참고 자료

- Codex CLI GitHub 저장소: https://github.com/openai/codex
- MCP 공식 서버 목록: https://github.com/modelcontextprotocol/servers
- Codex CLI 설정 문서: https://github.com/openai/codex/blob/main/docs/config.md
- 훅 시스템 요청 이슈: https://github.com/openai/codex/issues/2109
