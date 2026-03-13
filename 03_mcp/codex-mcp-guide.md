# Codex CLI MCP 서버 연동 가이드

## 학습 목표

MCP(Model Context Protocol)가 Codex CLI에서 어떤 역할을 하는지 이해하고, TOML 기반 설정 방식으로 외부 서비스를 연결하여 코드 작업 중 실제 데이터를 조회하고 작업을 수행할 수 있다.

## 사전 준비

- Codex CLI 설치 완료 (`npm install -g @openai/codex` 또는 최신 설치 방법 확인)
- OpenAI API 키 설정 완료
- Node.js 18 이상 (`node --version`으로 확인)
- GitHub Personal Access Token (GitHub MCP 실습 시 필요)

> **⚠️ 주의:** Codex CLI의 MCP 지원은 2025년 기준 **실험적 기능(experimental)**이다. 설정 방식과 지원 범위가 향후 변경될 수 있다. 프로덕션 환경에서는 Claude Code 또는 Cursor의 MCP 지원을 권장한다.

---

## 전체 흐름 한눈에 보기

Codex CLI는 코드 생성 및 편집에 특화된 AI 도구이다. MCP를 통해 GitHub 코드 검색, Jira 이슈 조회 등 코딩 워크플로우와 연관된 외부 서비스를 연결할 수 있다.

1. **MCP 개념 이해** — Codex CLI에서 MCP의 역할과 제한사항 파악
2. **MCP 서버 설정** — `~/.codex/config.toml` TOML 형식 설정
3. **주요 서버 활용** — GitHub, Filesystem 등 실제 서비스 연동
4. **고급 활용** — 다중 서버 설정, 환경별 분리, 트러블슈팅

---

## Phase 1: Codex CLI에서 MCP의 역할

### 목표

MCP가 Codex CLI에서 어떤 방식으로 동작하고, 다른 도구와 어떤 차이가 있는지 설명할 수 있다.

### 단계별 구현

#### Step 1.1 — MCP의 필요성 파악하기

> **💡 개념 설명: MCP (Model Context Protocol)**
>
> Codex CLI에서 "이 저장소의 관련 이슈 찾아줘"라고 하면 어떻게 될까? 기본 상태에서는 로컬 파일만 볼 수 있다 — GitHub API 접근 방법도 없고 인증 정보도 없다.
>
> MCP는 AI 에이전트가 외부 시스템과 **표준화된 방식**으로 통신할 수 있게 해주는 프로토콜이다. Codex CLI에서는 코딩 작업을 보조하는 컨텍스트 소스로 활용된다.
>
> **핵심 한 줄:** Codex CLI + MCP = 외부 정보를 참조하는 코드 작성 에이전트

```
Codex CLI ←→ MCP 프로토콜 ←→ MCP 서버 ←→ 외부 서비스
                                  GitHub MCP    GitHub API
                                  Jira MCP      Jira API
                                  DB MCP        PostgreSQL
```

#### Step 1.2 — Codex CLI MCP의 특성

Codex CLI의 MCP 지원은 다른 도구와 비교했을 때 다음과 같은 특성을 가진다:

| 특성 | 설명 |
|------|------|
| 설정 형식 | TOML (JSON이 아닌 TOML 형식 사용) |
| 설정 파일 위치 | `~/.codex/config.toml` (전역만 지원) |
| 전송 방식 | stdio 방식 주로 지원 |
| 지원 상태 | 실험적 기능 |
| CLI 명령 | `codex mcp list`, `codex mcp add` (제한적 지원) |

#### Step 1.3 — TOML vs JSON 설정 형식 비교

Codex CLI는 JSON이 아닌 TOML 형식을 사용한다. 두 형식의 차이를 이해해두면 설정 작성 시 오류를 줄일 수 있다:

```toml
# TOML 방식 (Codex CLI)
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_TOKEN = "ghp_..."
```

```json
// JSON 방식 (Claude Code, Cursor)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": { "GITHUB_TOKEN": "ghp_..." }
    }
  }
}
```

> **💡 TOML 기본 규칙**
>
> - `[section]`으로 섹션 구분
> - 중첩 섹션은 `[parent.child]`로 표현
> - 문자열은 큰따옴표 사용
> - 배열은 `["항목1", "항목2"]` 형식

### 체크포인트

Codex CLI MCP의 실험적 특성과 TOML 설정 형식을 이해하고, JSON 형식과의 차이를 설명할 수 있는가?

---

## Phase 2: MCP 서버 설정하기

### 목표

`~/.codex/config.toml`에 MCP 서버를 등록하고, `codex mcp list` 명령으로 인식되는지 확인한다.

### 단계별 구현

#### Step 2.1 — config.toml 파일 위치 확인

```bash
# 설정 디렉토리 확인
ls ~/.codex/

# config.toml이 없으면 생성
mkdir -p ~/.codex
touch ~/.codex/config.toml
```

#### Step 2.2 — GitHub MCP 서버 등록

> **💡 개념 설명: config.toml의 MCP 설정 구조**
>
> `[mcp_servers.서버이름]` 섹션에 서버 실행 명령을 정의한다. `[mcp_servers.서버이름.env]` 하위 섹션에 환경 변수를 추가한다. 섹션 이름의 `.`은 계층 구조를 나타낸다.
>
> **핵심 한 줄:** `[mcp_servers.github]` = "github라는 이름의 MCP 서버 설정"

```toml
# ~/.codex/config.toml

# 기본 Codex 설정
model = "o4-mini"
approval_policy = "auto"

# GitHub MCP 서버
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_TOKEN = "ghp_여기에_토큰_붙여넣기"
```

> **⚠️ 주의:** 토큰을 파일에 직접 작성하면 보안 위험이 있다. 가능하면 환경 변수를 참조하는 방식을 사용한다. 단, Codex CLI TOML에서는 `${VAR}` 치환이 지원되지 않을 수 있으므로, 대신 `codex --env GITHUB_TOKEN=...` 플래그로 전달하거나 별도 `.env` 파일을 활용한다.

#### Step 2.3 — Filesystem MCP 서버 추가

```toml
# ~/.codex/config.toml (이어서 추가)

# Filesystem MCP 서버
[mcp_servers.filesystem]
command = "npx"
args = [
  "-y",
  "@modelcontextprotocol/server-filesystem",
  "/Users/사용자명/Projects",
  "/Users/사용자명/Documents"
]
```

#### Step 2.4 — MCP 서버 연결 확인

```bash
# 등록된 MCP 서버 목록 확인
codex mcp list
```

정상 설정 시 출력:

```
MCP Servers:
  github    (stdio) npx -y @modelcontextprotocol/server-github
  filesystem (stdio) npx -y @modelcontextprotocol/server-filesystem ...
```

MCP 서버 추가 명령 (CLI 방식):

```bash
# CLI로 서버 추가 (실험적 지원)
codex mcp add github

# 도움말 확인
codex mcp --help
```

### 체크포인트

`codex mcp list` 실행 시 등록한 서버 목록이 표시되면 성공이다.

---

## Phase 3: 주요 MCP 서버 활용

### 목표

GitHub와 Filesystem MCP 서버를 실제로 활용하고, Codex CLI 대화에서 외부 데이터를 참조하여 코드를 작성한다.

### 단계별 구현

#### Step 3.1 — GitHub MCP 활용

**GitHub Personal Access Token 발급:**

1. [github.com/settings/tokens](https://github.com/settings/tokens) 접속
2. **Generate new token (classic)** 클릭
3. 다음 권한 선택:
   - `repo` (저장소 읽기/쓰기)
   - `read:org` (조직 정보 읽기)
4. 생성된 토큰 복사

**설정 완료 후 활용 예시:**

```bash
# Codex CLI 실행
codex

# 대화에서 GitHub 정보 참조
> 저장소의 열려있는 이슈 목록을 보고 가장 중요한 버그 수정 코드 작성해줘
> PR #15의 리뷰 코멘트 내용을 반영해서 코드 수정해줘
> 이 코드와 관련된 GitHub 이슈가 있는지 확인해줘
```

#### Step 3.2 — Filesystem MCP 활용

Filesystem MCP는 Codex CLI의 기본 파일 접근 권한 외에 추가 경로를 명시적으로 허용한다:

```toml
# ~/.codex/config.toml

[mcp_servers.filesystem]
command = "npx"
args = [
  "-y",
  "@modelcontextprotocol/server-filesystem",
  "/Users/사용자명/shared-docs",
  "/tmp/workspace"
]
```

**활용 예시:**

```bash
> /Users/사용자명/shared-docs/api-spec.md 파일을 읽고 이에 맞는 API 클라이언트 코드 작성해줘
> /tmp/workspace의 로그 파일을 분석해서 오류 패턴 파악해줘
```

#### Step 3.3 — PostgreSQL MCP 활용

데이터베이스 스키마를 참조하여 정확한 쿼리 코드를 작성할 수 있다:

```toml
# ~/.codex/config.toml

[mcp_servers.postgres]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-postgres"]

[mcp_servers.postgres.env]
POSTGRES_CONNECTION_STRING = "postgresql://user:pass@localhost:5432/mydb"
```

**활용 예시:**

```bash
> 데이터베이스 스키마를 보고 사용자 통계를 조회하는 SQL 쿼리와 TypeScript 함수 작성해줘
> users 테이블 구조에 맞는 TypeORM 엔티티 클래스 만들어줘
```

#### Step 3.4 — 주요 MCP 서버 목록

| 서버 이름 | 패키지 | 주요 기능 | Codex CLI 적합성 |
|-----------|--------|-----------|-----------------|
| GitHub | `@modelcontextprotocol/server-github` | 저장소, PR, 이슈 | 높음 |
| Filesystem | `@modelcontextprotocol/server-filesystem` | 로컬 파일 | 높음 |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | DB 스키마·쿼리 | 높음 |
| Brave Search | `@modelcontextprotocol/server-brave-search` | 웹 검색 | 중간 |
| Slack | `@modelcontextprotocol/server-slack` | 팀 커뮤니케이션 | 낮음 |

### 체크포인트

GitHub MCP를 통해 실제 저장소 정보를 참조하여 코드 작성이 이루어지면 성공이다.

---

## Phase 4: 고급 활용

### 목표

환경별 설정 분리, 프로젝트 특화 설정, 트러블슈팅 방법을 익힌다.

### 단계별 구현

#### Step 4.1 — 전체 config.toml 구조 이해

Codex CLI의 `config.toml`은 MCP 설정 외에도 여러 설정을 담는다:

```toml
# ~/.codex/config.toml 전체 구조 예시

# ── 기본 모델 설정 ──────────────────────────────────
model = "o4-mini"                    # 사용할 OpenAI 모델
approval_policy = "auto"             # 도구 실행 승인 정책
# "suggest" | "auto-edit" | "auto"

# ── 기본 프롬프트 설정 ──────────────────────────────
# notify = true

# ── GitHub MCP 서버 ─────────────────────────────────
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_TOKEN = "ghp_..."

# ── Filesystem MCP 서버 ─────────────────────────────
[mcp_servers.filesystem]
command = "npx"
args = [
  "-y",
  "@modelcontextprotocol/server-filesystem",
  "/Users/사용자명/Projects"
]

# ── PostgreSQL MCP 서버 ─────────────────────────────
[mcp_servers.postgres]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-postgres"]

[mcp_servers.postgres.env]
POSTGRES_CONNECTION_STRING = "postgresql://localhost:5432/mydb"
```

#### Step 4.2 — 환경 변수로 토큰 분리하기

보안을 위해 토큰을 설정 파일에 직접 쓰지 않는 방법:

**방법 1: 셸 프로파일 활용**

```bash
# ~/.zshrc 또는 ~/.bashrc에 추가
export GITHUB_TOKEN="ghp_..."
export POSTGRES_CONNECTION_STRING="postgresql://..."
```

그런 다음 Codex CLI를 실행할 때 환경 변수가 자동으로 전달되도록 래퍼 스크립트를 작성한다:

```bash
#!/bin/bash
# ~/bin/codex-with-env.sh

export GITHUB_TOKEN="$(cat ~/.secrets/github_token)"
exec codex "$@"
```

**방법 2: 실행 시 직접 전달**

```bash
GITHUB_TOKEN=ghp_... codex
```

#### Step 4.3 — MCP 연결 디버깅

MCP 서버가 연결되지 않을 때 단계별 디버깅:

```bash
# 1. npx 명령이 직접 실행되는지 확인
npx -y @modelcontextprotocol/server-github

# 2. 토큰이 올바른지 확인
GITHUB_TOKEN=ghp_... npx -y @modelcontextprotocol/server-github

# 3. config.toml 문법 검사
# TOML 유효성 검사 도구 활용
npx toml-cli validate ~/.codex/config.toml

# 4. Codex CLI 디버그 모드로 실행
codex --debug
```

#### Step 4.4 — 프로젝트별 MCP 설정 전략

> **💡 개념 설명: 전역 설정의 한계**
>
> Codex CLI는 2025년 기준 전역 `~/.codex/config.toml`만 지원하며, 프로젝트별 설정 파일은 공식 지원하지 않는다. 프로젝트마다 다른 설정이 필요할 경우 별도 설정 파일을 관리하는 방법을 사용한다.
>
> **핵심 한 줄:** 프로젝트별 설정이 필요하다면 Claude Code나 Cursor를 함께 사용하는 것이 현실적이다.

임시 방편으로 프로젝트별 설정을 전환하는 방법:

```bash
# 프로젝트 A용 설정으로 전환
cp ~/.codex/config.project-a.toml ~/.codex/config.toml
codex

# 프로젝트 B용 설정으로 전환
cp ~/.codex/config.project-b.toml ~/.codex/config.toml
codex
```

또는 `CODEX_CONFIG` 환경 변수로 설정 파일 경로를 지정한다 (버전에 따라 지원 여부 확인 필요):

```bash
CODEX_CONFIG=./project-config.toml codex
```

### 체크포인트

여러 MCP 서버가 동시에 `codex mcp list`에 등록되어 있고, 각각의 서버를 활용한 작업이 가능하면 성공이다.

---

## 도구 간 비교표

| 항목 | Codex CLI | Claude Code | Gemini CLI | Cursor |
|------|-----------|-------------|------------|--------|
| 설정 형식 | TOML | JSON | JSON | JSON |
| 설정 파일 | `~/.codex/config.toml` | `.mcp.json` | `~/.gemini/settings.json` | `.cursor/mcp.json` |
| 프로젝트별 설정 | 미지원 (실험적) | `.mcp.json` 지원 | `.gemini/settings.json` 지원 | `.cursor/mcp.json` 지원 |
| CLI 추가 명령 | `codex mcp add` (실험적) | `claude mcp add` | 없음 | 없음 |
| HTTP/OAuth 지원 | 미지원 | 지원 | 제한적 | SSE 지원 |
| 지원 상태 | 실험적 | 안정 | 안정 | 안정 |
| 상태 확인 | `codex mcp list` | `/mcp` | `/mcp list` | Settings UI |

---

## 마무리

### 이 가이드에서 배운 것

- Codex CLI MCP의 실험적 특성과 TOML 기반 설정 방식
- `~/.codex/config.toml`에 `[mcp_servers.이름]` 섹션으로 서버 등록하는 방법
- GitHub, Filesystem, PostgreSQL MCP 서버 활용 예시
- 환경 변수로 토큰을 안전하게 관리하는 방법
- 프로젝트별 설정이 제한적인 Codex CLI의 한계와 대안

### 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| `codex mcp list`에서 서버가 안 보임 | config.toml 문법 오류 | TOML 유효성 검사기로 확인 |
| MCP 도구가 실행 안 됨 | 실험적 기능 제한 | Codex CLI 최신 버전으로 업데이트 |
| npx 실행 실패 | Node.js 미설치 또는 구버전 | `node --version` 확인 후 18+ 버전 설치 |
| 토큰 인증 실패 | 토큰 권한 부족 | GitHub 토큰 권한 범위 재확인 |
| 프로젝트별 설정 안 됨 | 전역 설정만 지원 | Claude Code나 Cursor 병행 사용 검토 |

### 다음 단계

- MCP 지원이 안정화된 [Claude Code MCP 가이드](./claude-code-mcp-guide.md)와 비교하여 사용 목적에 맞는 도구 선택
- [MCP 서버 공식 목록](https://github.com/modelcontextprotocol/servers)에서 코딩 워크플로우에 유용한 서버 탐색
- Codex CLI 공식 릴리즈 노트에서 MCP 지원 업데이트 추적
