# Claude Code 실습 가이드

> Anthropic의 CLI 코딩 에이전트 Claude Code를 처음 쓰는 개발자를 위한 흐름 중심 가이드
> 최신 모델 **Claude Opus 4.6** 기준 · 작성일: 2026-03-12

---

## 개요

Claude Code는 Anthropic이 개발한 터미널 기반 AI 코딩 에이전트이다. 코드를 읽고, 파일을 편집하고, 셸 명령을 실행하며, Git 워크플로우까지 자율적으로 수행한다. 2025년 2월 리서치 프리뷰로 시작해 13개월 만에 GitHub 전체 공개 커밋의 약 4%를 차지할 만큼 성장했으며, Anthropic 자체 코드의 90%가 Claude Code로 작성되고 있다.

### 주요 특징

- **확장 사고(Extended Thinking)**: 모델의 추론 과정을 실시간으로 확인 가능. 복잡한 문제에 대해 단계적으로 사고하는 과정을 투명하게 보여준다
- **메모리 시스템**: 세션 간 학습 내용을 자동 저장하여 프로젝트별 컨텍스트를 누적한다
- **내장 비용 추적**: `/cost` 명령으로 세션별 토큰 사용량과 비용을 실시간 확인한다
- **Git 네이티브**: Git 컨텍스트를 이해하고, 커밋 생성/PR 작성을 직접 수행한다
- **멀티 모델 전환**: 세션 중 `/model` 명령으로 Opus(고난이도) ↔ Haiku(빠른 작업) 간 즉시 전환한다
- **Hooks 시스템**: PreToolUse, PostToolUse 등 이벤트 기반 자동화를 제공한다 (다른 CLI 도구에 없는 고유 기능)
- **IDE 통합**: VS Code 확장과 JetBrains 플러그인으로 터미널 밖에서도 사용 가능하다
- **MCP 지원**: 3,000개 이상의 외부 통합(GitHub, Slack, DB 등)을 표준 프로토콜로 연결한다

### 다른 도구와의 차별점

| 비교 항목 | Claude Code | Gemini CLI | Codex CLI | Cursor / VS Code Copilot |
|-----------|-------------|------------|-----------|--------------------------|
| 실행 환경 | 터미널 + IDE 확장 | 터미널 | 터미널 | 에디터 네이티브 |
| 확장 사고 | O (추론 과정 공개) | X | X | X |
| Hooks (이벤트 자동화) | O (PreToolUse, PostToolUse 등) | 제한적 | X | X |
| 내장 비용 추적 | O (`/cost`) | O (`/stats`) | X | X |
| 메모리 시스템 | O (CLAUDE.md + MEMORY.md) | O (GEMINI.md) | O (AGENTS.md) | 제한적 |
| 서브에이전트 | O (최대 7개 병렬) | X | 실험적 | X |
| Agent Teams | O (2026.02~) | X | X | X |
| 권한 모드 | 5단계 (plan~bypass) | 2단계 (기본/YOLO) | 3단계 | 에디터 의존 |

---

## Phase 1: 설치 및 인증

### 목표

Claude Code가 터미널에서 실행되고, Anthropic 계정 또는 API Key로 인증이 완료된 상태가 된다.

### 시스템 요구사항

| 항목 | 요구 사항 |
|------|-----------|
| OS | macOS / Linux / Windows (WSL 권장) |
| Node.js | v18 이상 (네이티브 설치 시 불필요) |
| 계정 | Anthropic API Key 또는 Claude Pro/Max/Team/Enterprise 구독 |
| Git | 설치 권장 (커밋/PR 자동화, undo 활용) |

### Step 1.1 --- Claude Code 설치

#### 방법 1: 네이티브 설치 (권장)

Node.js가 필요 없고, 자동 업데이트를 지원하는 권장 설치 방법이다.

```bash
# macOS / Linux
curl -fsSL https://claude.ai/install.sh | bash

# Windows (PowerShell)
irm https://claude.ai/install.ps1 | iex
```

#### 방법 2: npm 전역 설치 (레거시)

```bash
# Node.js 18+ 필요
npm install -g @anthropic-ai/claude-code

# 설치 확인
claude --version
```

> **💡 개념 설명: Claude Code란?**
>
> "이 함수에서 에러가 발생하니 원인을 찾아서 고쳐줘"라고 터미널에 자연어로 입력하면, Claude Code가 코드를 읽고, 파일을 수정하고, 테스트까지 실행하는 자율 코딩 에이전트이다. 단순 코드 제안(Copilot)과 달리, **파일 시스템과 셸을 직접 조작**하는 점이 핵심 차이이다.
>
> 핵심 요약: Claude Code = 터미널 안의 자율 코딩 에이전트

### Step 1.2 --- 인증

```bash
# 첫 실행 시 인증 프롬프트 표시
claude
```

인증 방법은 세 가지이다:

1. **OAuth 로그인** (Pro/Max/Team/Enterprise 구독자): 브라우저가 열리며 Anthropic 계정으로 OAuth 인증 진행. API Key 없이 구독 크레딧을 그대로 사용한다.
2. **API Key**: Anthropic Console에서 발급받은 키를 사용한다.
3. **Amazon Bedrock / Google Vertex AI**: 기업 환경에서 클라우드 공급자 인증을 사용한다.

```bash
# 환경 변수로 API Key 설정 (선택)
# ~/.bashrc 또는 ~/.zshrc

export ANTHROPIC_API_KEY="sk-ant-..."
```

### 체크포인트

아래 명령 후 대화형 프롬프트(`>`)가 보이면 성공이다.

```bash
claude
# ">" 프롬프트가 나타나면 인증 완료
```

---

## Phase 2: 기본 사용법

### 목표

대화형 모드와 원샷 모드를 사용하고, 주요 CLI 플래그와 슬래시 명령을 익힌다.

### Step 2.1 --- 대화형 모드 (Interactive Mode)

```bash
# 기본 실행 - 현재 디렉토리를 작업 컨텍스트로 사용
claude

# 특정 모델 지정
claude --model claude-opus-4-6

# 프로젝트 디렉토리에서 시작
cd ~/my-project && claude
```

대화형 모드에서는 자연어로 작업을 지시한다:

```
> src/auth/login.ts 파일에서 JWT 토큰 생성 로직을 분석해줘
> 이 프로젝트의 전체 구조를 설명해줘
> README.md를 한국어로 작성해줘
```

### Step 2.2 --- 원샷 모드 (Print Mode)

`-p` (또는 `--print`) 플래그로 비대화형 실행이 가능하다. 스크립팅과 CI/CD 파이프라인에 적합하다.

```bash
# 한 줄 명령으로 결과 받기
claude -p "이 프로젝트의 디렉토리 구조를 설명해줘"

# 파이프로 입력 전달
cat error.log | claude -p "이 에러 로그를 분석하고 해결 방법을 알려줘"

# 결과를 파일로 저장
claude -p "API 문서를 생성해줘" > api-docs.md

# JSON 형식 출력 (스크립팅용)
claude -p "프로젝트 의존성 분석해줘" --output-format json
```

### Step 2.3 --- 주요 CLI 플래그

| 플래그 | 설명 | 예시 |
|--------|------|------|
| `-p, --print` | 원샷(비대화형) 모드 | `claude -p "분석해줘"` |
| `--model` | 사용할 모델 선택 | `--model opus`, `--model sonnet` |
| `--permission-mode` | 권한 모드 설정 | `--permission-mode plan` |
| `--allowedTools` | 허용할 도구 지정 | `--allowedTools "Read" "Grep" "Bash(npm run test:*)"` |
| `--output-format` | 출력 형식 (text/json/stream-json) | `--output-format json` |
| `--max-turns` | 에이전트 루프 최대 횟수 | `--max-turns 10` |
| `--resume` | 이전 세션 이어하기 | `claude --resume` |
| `--dangerously-skip-permissions` | 모든 권한 확인 건너뜀 | (주의: 격리 환경에서만 사용) |

> **💡 개념 설명: --allowedTools란?**
>
> Claude Code가 사용할 수 있는 도구(Bash, Read, Write, Edit, Grep 등)를 세션 단위로 제한하는 플래그이다. 예를 들어 `--allowedTools "Read" "Grep"`으로 실행하면 파일 읽기와 검색만 가능하고 수정은 불가능하다. 특정 Bash 명령만 허용하려면 glob 패턴을 사용한다: `--allowedTools "Bash(npm run *)"`.

### Step 2.4 --- 슬래시 명령

대화형 세션에서 `/` 접두사로 시작하는 명령을 사용한다:

#### 세션 관리

| 명령 | 기능 |
|------|------|
| `/help` | 도움말 표시 |
| `/status` | 현재 모델, 권한 모드, 컨텍스트 상태 확인 |
| `/clear` | 대화 기록 초기화 (컨텍스트 200K 토큰 복구) |
| `/compact` | 대화 내용을 요약하여 컨텍스트 압축 |
| `/cost` | 현재 세션의 토큰 사용량과 비용 확인 |
| `/context` | 컨텍스트 윈도우 사용량 확인 |

#### 모델 및 동작

| 명령 | 기능 |
|------|------|
| `/model` | 모델 전환 (Opus 4.6 ↔ Sonnet 4.6 ↔ Haiku 4.5) |
| `/model sonnet` | 특정 모델로 즉시 전환 |
| `/memory` | 메모리(CLAUDE.md) 확인 및 편집 |
| `/plan` | 플랜 모드 토글 (분석만, 수정 안 함) |

#### 프로젝트 관련

| 명령 | 기능 |
|------|------|
| `/init` | 프로젝트 CLAUDE.md 자동 생성 |
| `/review` | 변경사항 코드 리뷰 |
| 사용자 정의 | `.claude/commands/`에 정의한 커스텀 명령 |

### Step 2.5 --- 세션 이어하기 (Resume)

Claude Code는 세션 기록을 로컬에 저장한다. 이전 작업을 이어서 하려면:

```bash
# 가장 최근 세션 재개
claude --resume

# 특정 세션 선택하여 재개
claude --resume <session-id>
```

### 체크포인트

TUI에서 `/model`을 입력했을 때 모델 목록(Opus 4.6, Sonnet 4.6, Haiku 4.5)이 나타나면 정상이다.

```bash
claude
# > /model
# 모델 목록에서 선택 가능 확인
```

---

## Phase 3: 권한 모드 (Permission Modes)

### 목표

5가지 권한 모드의 차이를 이해하고, 상황에 맞게 설정하여 안전하게 에이전트를 운영한다.

### Step 3.1 --- 권한 모드 이해하기

> **💡 개념 설명: 왜 권한 모드가 필요한가?**
>
> Claude Code는 파일을 편집하고 셸 명령을 실행할 수 있는 강력한 에이전트이다. 잘못된 명령이 실행되면 파일이 손상되거나 의도치 않은 부작용이 생길 수 있다. 권한 모드는 "Claude가 얼마나 자율적으로 행동할 수 있는가"를 결정하는 안전 장치이다.
>
> 핵심 요약: 모르는 코드베이스 = plan / 검증된 환경 = acceptEdits / 격리된 CI = bypassPermissions

| 모드 | 설명 | 사용 시기 |
|------|------|-----------|
| **default** | 모든 도구 사용 시 확인 요청 | 기본 동작, 가장 안전 |
| **acceptEdits** | 파일 편집 자동 승인, 셸 명령은 확인 | 일반 개발 작업 |
| **plan** | 분석만 가능, 파일 수정/명령 실행 불가 | 낯선 코드베이스 탐색 |
| **dontAsk** | 허용 목록의 도구는 자동 승인 | 신뢰할 수 있는 프로젝트 |
| **bypassPermissions** | 모든 권한 확인 건너뜀 (YOLO 모드) | Docker/VM 격리 환경만 |

### Step 3.2 --- 권한 모드 설정

```bash
# CLI 플래그로 설정
claude --permission-mode acceptEdits

# YOLO 모드 (격리 환경에서만!)
claude --dangerously-skip-permissions

# 세션 중 전환: Shift+Tab으로 순환
```

### Step 3.3 --- settings.json에서 허용 목록 설정

```json
// ~/.claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Grep",
      "Bash(npm run test:*)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Edit"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl *)"
    ]
  }
}
```

> **💡 개념 설명: 허용 목록(Allowlist)이란?**
>
> `Bash(npm run test:*)` 형식으로 특정 명령 패턴만 자동 승인할 수 있다. Gemini CLI의 `allowedCommands`와 유사하지만, Claude Code는 도구(Tool) 단위로 더 세밀하게 제어한다. 예를 들어 `Edit`은 모든 파일 편집을, `Bash(git *)`은 Git 관련 명령만 자동 승인한다.

### Step 3.4 --- 다른 도구와 비교

| 기능 | Claude Code | Gemini CLI | Codex CLI |
|------|-------------|------------|-----------|
| 기본 모드 | 모든 도구 확인 | 모든 도구 확인 | Auto (디렉토리 내 자동) |
| 자동 편집 | acceptEdits | - | Auto (기본) |
| 분석 전용 | plan | --plan | Read-only (untrusted) |
| 완전 자율 | bypassPermissions | --yolo | Full Access (never) |
| 세밀한 제어 | Tool별 allow/deny | allowedCommands | - |
| 안전 장치 | 계층별 deny 우선 | 샌드박스 자동 활성화 | 네트워크 차단 |

### Step 3.5 --- 안전하게 bypassPermissions 사용하기

```bash
# Docker 컨테이너에서 사용 (권장)
docker run -v $(pwd):/workspace -w /workspace \
  claude-code claude --dangerously-skip-permissions \
  -p "모든 lint 에러 수정하고 테스트 실행해줘"
```

```yaml
# GitHub Actions에서 사용 (CI/CD)
- name: Claude Code Review
  run: |
    claude -p "변경된 파일들의 보안 이슈 검토해줘" \
      --permission-mode bypassPermissions \
      --output-format json
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 체크포인트

세션 중 `Shift+Tab`을 눌러 권한 모드가 순환되면 정상이다.

---

## Phase 4: 프로젝트/사용자 스코프

### 목표

전역(Global), 프로젝트(Project) 스코프 계층 구조를 이해하고, 설정 파일의 위치와 역할을 파악한다.

### Step 4.1 --- 전역 설정 (User Scope)

사용자의 모든 프로젝트에 적용되는 설정이다.

```
~/.claude/
├── CLAUDE.md              # 전역 규칙 (모든 프로젝트에 적용)
├── settings.json          # 전역 설정 (MCP 서버, 권한, 테마 등)
├── commands/              # 사용자 정의 슬래시 명령어
│   ├── review.md          # /review 명령
│   └── git/
│       └── commit.md      # /git:commit 명령
└── memory/                # 자동 저장되는 학습 내용 (MEMORY.md)
```

**예시: 전역 CLAUDE.md**

```markdown
# ~/.claude/CLAUDE.md

나는 풀스택 개발자이다.
선호하는 프로그래밍 언어는 TypeScript와 Python이다.
코드 작성 시 항상 타입 힌트를 포함해줘.
주석과 커밋 메시지는 한국어로 작성한다.
```

**예시: 전역 settings.json**

```json
// ~/.claude/settings.json
{
  "permissions": {
    "allow": [
      "Read",
      "Grep",
      "Bash(git *)"
    ],
    "deny": [
      "Bash(rm -rf *)"
    ]
  },
  "env": {
    "ANTHROPIC_MODEL": "claude-opus-4-6"
  }
}
```

### Step 4.2 --- 프로젝트 설정 (Project Scope)

특정 프로젝트 디렉토리에만 적용되는 설정이다.

```
<project-root>/
├── CLAUDE.md                  # 프로젝트 규칙 (Git에 커밋 — 팀 공유)
├── .claude/
│   ├── settings.json          # 프로젝트 설정 (Git에 커밋 — 팀 공유)
│   ├── settings.local.json    # 로컬 전용 설정 (Git에 커밋 안 함)
│   ├── commands/              # 프로젝트 전용 슬래시 명령어
│   └── agents/                # 커스텀 에이전트 정의
├── .mcp.json                  # MCP 서버 설정 (프로젝트별)
└── .claudeignore              # 컨텍스트에서 제외할 파일 패턴
```

**예시: 프로젝트 CLAUDE.md**

```markdown
<!-- <project-root>/CLAUDE.md -->
# 프로젝트: E-commerce 백엔드 API

## 기술 스택
- Node.js + Express + TypeScript
- PostgreSQL + Prisma ORM
- Redis 캐시
- JWT 인증

## 코딩 규칙
- ESLint + Prettier 규칙 준수
- 모든 API 엔드포인트에 OpenAPI 문서 작성
- 에러 핸들링은 middleware/errorHandler.ts 사용
- 테스트는 Vitest, __tests__/ 디렉토리에 작성

## 빌드/테스트 명령
- `npm run build` — TypeScript 컴파일
- `npm run test` — 테스트 실행
- `npm run lint` — 린트 검사
```

**예시: .claudeignore**

```bash
# .claudeignore
node_modules/
dist/
*.log
.env
.env.*
coverage/
```

### Step 4.3 --- settings.json vs settings.local.json

| 파일 | Git 커밋 | 용도 |
|------|----------|------|
| `.claude/settings.json` | O (팀 공유) | 프로젝트 공통 허용 도구, MCP 서버 |
| `.claude/settings.local.json` | X (개인용) | 개인 API 키 경로, 실험적 설정 |

```json
// .claude/settings.json (팀 공유)
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(npx prisma *)"
    ]
  }
}
```

```json
// .claude/settings.local.json (개인용, .gitignore에 포함)
{
  "permissions": {
    "allow": [
      "Bash(docker *)"
    ]
  }
}
```

### Step 4.4 --- 스코프 우선순위

```
Managed Settings (조직 관리자)     ← 최우선, 재정의 불가
  └── CLI 플래그 (--model 등)      ← 세션 임시 재정의
      └── settings.local.json     ← 프로젝트 개인 설정
          └── .claude/settings.json ← 프로젝트 팀 설정
              └── ~/.claude/settings.json ← 전역 사용자 설정
```

> **💡 개념 설명: deny는 항상 우선한다**
>
> 어떤 레벨에서든 도구가 `deny`로 설정되면, 다른 레벨에서 `allow`하더라도 차단된다. 이 규칙은 보안을 위해 의도적으로 설계된 것이다. 예를 들어 전역 settings.json에서 `Bash(rm -rf *)`를 deny하면, 프로젝트 settings.json에서 allow하더라도 실행되지 않는다.

### Step 4.5 --- 컴포넌트 스코프 (서브디렉토리 CLAUDE.md)

프로젝트 내 특정 디렉토리에 CLAUDE.md를 두면, 해당 디렉토리 작업 시 추가 컨텍스트가 로드된다.

```
<project-root>/
├── CLAUDE.md                 # 프로젝트 전체 규칙
└── src/
    └── auth/
        └── CLAUDE.md         # auth 모듈 전용 규칙
```

```markdown
<!-- src/auth/CLAUDE.md -->
# 인증 모듈

이 디렉토리는 JWT 기반 인증을 처리한다.
- passport-jwt 라이브러리 사용
- 토큰 만료 시간: ACCESS_TOKEN_EXPIRY 환경변수 참조
- 리프레시 토큰: 7일
- 비밀번호 해싱: bcrypt (saltRounds=12)
```

### 체크포인트

프로젝트 루트에서 `claude`를 실행한 뒤 `/status`를 입력했을 때, CLAUDE.md가 로드된 것이 확인되면 정상이다.

```bash
# 프로젝트 루트에서
claude
# > /status
# "CLAUDE.md loaded" 또는 유사한 메시지 확인
```

---

## Phase 5: 핵심 기능 미리보기

### 목표

Claude Code의 확장 기능(Rule, MCP, Skill, Hooks, Workflow, Subagent)의 개념을 파악한다. 각 기능의 상세 내용은 후속 챕터에서 다룬다.

### 5.1 CLAUDE.md 규칙 → `02_rule`에서 상세

CLAUDE.md는 Claude Code가 따라야 할 프로젝트 규칙, 코딩 컨벤션, 아키텍처 지침을 정의하는 파일이다. Gemini CLI의 `GEMINI.md`, Codex CLI의 `AGENTS.md`에 대응한다.

```markdown
<!-- CLAUDE.md 규칙 예시 (미리보기) -->
## 코딩 규칙
- TypeScript strict 모드 사용
- any 타입 사용 금지
- 모든 함수에 JSDoc 주석 필수

## Git 커밋 규칙
- Conventional Commits 형식 (feat:, fix:, docs:)
- 커밋 메시지는 한국어로 작성
```

| Claude Code | Gemini CLI | Codex CLI |
|-------------|------------|-----------|
| CLAUDE.md | GEMINI.md | AGENTS.md |
| 프로젝트 루트 + 서브디렉토리 | `.gemini/` 디렉토리 내 | 프로젝트 루트 |

> 상세 작성법, 파일 임포트, 계층 병합 규칙 등은 **`02_rule/`** 챕터에서 다룬다.

### 5.2 MCP 서버 → `03_mcp`에서 상세

MCP(Model Context Protocol)는 Claude Code를 GitHub, Slack, 데이터베이스 등 외부 시스템과 연결하는 표준 프로토콜이다. 프로젝트 루트의 `.mcp.json` 또는 settings.json에서 설정한다.

```bash
# MCP 서버 추가 (CLI 명령)
claude mcp add --transport stdio github -- npx -y @github/github-mcp-server

# HTTP 기반 MCP 서버 (OAuth 지원)
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

```json
// .mcp.json (프로젝트 루트)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

> 서버 설정, OAuth 인증, 컨텍스트 관리, 커스텀 MCP 서버 개발 등은 **`03_mcp/`** 챕터에서 다룬다.

### 5.3 커스텀 슬래시 명령 (Skills) → `04_skill`에서 상세

`.claude/commands/` 디렉토리에 Markdown 파일을 만들면 커스텀 슬래시 명령이 된다. Gemini CLI의 `.toml` 기반 Slash Command와 유사하지만, Claude Code는 Markdown 형식을 사용한다.

```markdown
<!-- .claude/commands/review-pr.md -->
다음 PR의 변경사항을 검토해줘:

$ARGUMENTS

코드 품질, 보안 이슈, 성능 문제를 중점적으로 확인하고
개선 제안을 구체적으로 제시해줘.
```

```bash
# 사용법
> /review-pr 123

# 네임스페이스 명령
# .claude/commands/git/commit.md → /git:commit
```

| Claude Code | Gemini CLI | Codex CLI |
|-------------|------------|-----------|
| `.claude/commands/*.md` | `.gemini/commands/*.toml` | - |
| Markdown 형식 | TOML 형식 | - |
| `$ARGUMENTS` 치환 | `{{args}}` 치환 | - |

> 커스텀 명령 작성법, 셸 명령 주입, 파일 참조, 네임스페이스 등은 **`04_skill/`** 챕터에서 다룬다.

### 5.4 Hooks & Workflows → `05_workflow`에서 상세

Hooks는 Claude Code의 고유 기능으로, 도구 실행 전후에 스크립트를 자동 실행하는 이벤트 기반 자동화 시스템이다. Gemini CLI나 Codex CLI에는 없는 기능이다.

```json
// .claude/settings.json (Hooks 설정 미리보기)
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH"
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
            "command": "/path/to/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

**주요 Hook 이벤트:**

| 이벤트 | 시점 | 활용 예시 |
|--------|------|-----------|
| PreToolUse | 도구 실행 전 | 보안 검사, 권한 확인, 입력 검증 |
| PostToolUse | 도구 실행 후 | 자동 포매팅, 린트 실행, 로깅 |
| Notification | 알림 발생 시 | Slack 알림, 데스크톱 알림 |
| Stop | 에이전트 중단 시 | 정리 작업, 보고서 생성 |

> Hook 타입(command, http, prompt), 비동기 실행, 보안 설정 등은 **`05_workflow/`** 챕터에서 다룬다.

### 5.5 Agent / Task 서브에이전트 → `06_subagent`에서 상세

서브에이전트는 Claude Code가 스폰하는 자율적인 AI 작업자이다. 최대 7개가 병렬로 실행되며, 각각 독립된 컨텍스트에서 작업한 뒤 결과 요약만 메인 대화로 반환한다.

**내장 서브에이전트 유형:**

| 유형 | 역할 |
|------|------|
| Explore | 파일 탐색, 코드베이스 구조 파악 |
| Plan | 코드베이스 리서치, 구현 계획 수립 |
| General | 범용 멀티스텝 작업 수행 |

```
메인 세션: "auth 모듈 리팩토링하고 테스트도 작성해줘"
  ├── 서브에이전트 1: auth 모듈 구조 분석 (Explore)
  ├── 서브에이전트 2: 리팩토링 계획 수립 (Plan)
  └── 서브에이전트 3: 테스트 코드 작성 (General)
  → 결과 요약만 메인 컨텍스트로 반환
```

**Agent Teams (2026.02 ~):**

Agent Teams는 여러 독립 Claude 세션이 서로 메시지를 주고받으며 작업을 분담하는 기능이다. 서브에이전트가 "상사-부하" 관계라면, Agent Teams는 "동료 협업" 관계이다.

> 서브에이전트 설정, 모델 라우팅(탐색은 Haiku, 핵심 작업은 Opus), Agent Teams 구성 등은 **`06_subagent/`** 챕터에서 다룬다.

---

## Phase 6: IDE 통합

### 목표

VS Code와 JetBrains IDE에서 Claude Code를 사용하는 방법을 파악한다.

### 6.1 VS Code 확장

VS Code용 공식 확장을 설치하면 에디터 내에서 그래픽 채팅 패널, 체크포인트 기반 Undo, `@` 파일 참조, 병렬 대화를 사용할 수 있다.

```
# 설치 방법
1. Cmd+Shift+X (Mac) / Ctrl+Shift+X (Windows/Linux)
2. "Claude Code" 검색
3. Install 클릭
```

### 6.2 JetBrains 플러그인 (Beta)

JetBrains IDE(IntelliJ, WebStorm, PyCharm 등)용 공식 플러그인이 베타로 제공된다. IDE의 통합 터미널에서 Claude Code CLI를 실행하고, 변경 사항을 IDE의 diff 뷰어에서 검토할 수 있다.

```
# 설치 방법
1. Settings → Plugins → Marketplace
2. "Claude Code" 검색
3. Install 클릭
```

---

## 마무리

### 이 가이드에서 배운 것

- Claude Code를 네이티브 설치(또는 npm)로 설치하고 OAuth 또는 API Key로 인증한다
- 대화형 모드(`claude`)와 원샷 모드(`claude -p`)로 자연어 명령을 내린다
- 5가지 권한 모드(default, acceptEdits, plan, dontAsk, bypassPermissions)로 안전성과 자율성의 균형을 조절한다
- 전역(`~/.claude/`)과 프로젝트(`.claude/`) 스코프로 설정을 계층적으로 관리한다
- CLAUDE.md, MCP, 커스텀 명령, Hooks, 서브에이전트의 개념을 파악했다

### 자주 발생하는 실수

| 실수 | 해결 방법 |
|------|-----------|
| bypassPermissions를 로컬 머신에서 사용 | 반드시 Docker/VM 격리 환경에서만 사용. 로컬에서는 acceptEdits나 allowlist 활용 |
| 컨텍스트 윈도우 초과로 품질 저하 | `/compact`로 요약하거나 `/clear`로 초기화. MCP 서버는 10개 이하, 도구 80개 이하 유지 |
| CLAUDE.md에 민감한 정보 포함 | CLAUDE.md는 Git에 커밋되므로 API 키, 비밀번호 절대 포함 금지. 환경 변수 사용 |
| settings.local.json을 Git에 커밋 | `.gitignore`에 `settings.local.json` 추가 |
| 모든 MCP 서버를 한꺼번에 활성화 | 컨텍스트 윈도우가 200K에서 70K로 줄어들 수 있음. 필요한 서버만 활성화 |

### 다음 학습 단계

| 챕터 | 주제 | 내용 |
|------|------|------|
| `02_rule/` | CLAUDE.md 규칙 작성 | 계층 병합, 파일 임포트, 팀 공유 전략 |
| `03_mcp/` | MCP 서버 | 설정, OAuth, 커스텀 서버 개발 |
| `04_skill/` | 커스텀 슬래시 명령 | 명령 작성, 인자 치환, 네임스페이스 |
| `05_workflow/` | Hooks & Workflows | 이벤트 자동화, CI/CD 통합 |
| `06_subagent/` | 서브에이전트 & Agent Teams | 병렬 작업, 모델 라우팅, 팀 협업 |

### 유용한 리소스

- [Claude Code 공식 문서](https://code.claude.com/docs/en/)
- [Claude Code CLI 레퍼런스](https://code.claude.com/docs/en/cli-reference)
- [Claude Code 설정 레퍼런스](https://code.claude.com/docs/en/settings)
- [Hooks 레퍼런스](https://code.claude.com/docs/en/hooks)
- [Claude Code 변경 로그](https://code.claude.com/docs/en/changelog)
- [GitHub 저장소](https://github.com/anthropics/claude-code)

### 업데이트 방법

```bash
# 네이티브 설치 — 자동 업데이트됨 (수동 재설치 가능)
curl -fsSL https://claude.ai/install.sh | bash

# npm 설치
npm update -g @anthropic-ai/claude-code

# 버전 확인
claude --version
```

---

*작성 기준: Claude Code, Claude Opus 4.6 / Sonnet 4.6 / Haiku 4.5, 2026-03-12*
