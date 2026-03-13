# Cursor MCP 서버 연동 가이드

## 학습 목표

MCP(Model Context Protocol)가 Cursor 에디터에서 어떤 역할을 하는지 이해하고, GitHub, Slack 등 외부 서비스를 연결하여 에디터 내 AI Agent 모드에서 실제 데이터를 조회하고 작업을 수행할 수 있다.

## 사전 준비

- Cursor 에디터 설치 완료 (최신 버전 권장)
- Node.js 18 이상 (`node --version`으로 확인)
- GitHub Personal Access Token (GitHub MCP 실습 시 필요)
- Cursor Pro 이상 플랜 (Agent 모드에서 MCP 활용 시 권장)

---

## 전체 흐름 한눈에 보기

Cursor는 기본적으로 로컬 코드베이스를 대상으로 AI 작업을 수행한다. MCP 서버를 추가하면 GitHub 이슈 조회, Slack 메시지 전송, 데이터베이스 쿼리 등 외부 시스템과 직접 상호작용할 수 있게 된다. Cursor는 1,800개 이상의 커뮤니티 MCP 서버를 지원한다.

1. **MCP 개념 이해** — Cursor Agent 모드에서 MCP의 역할 파악
2. **MCP 서버 설정** — `.cursor/mcp.json`(프로젝트) 또는 Settings UI(전역) 설정
3. **주요 서버 활용** — GitHub, Slack, Notion 등 실제 서비스 연동
4. **고급 활용** — SSE 서버, 도구 한도 관리, 커뮤니티 서버 탐색

---

## Phase 1: Cursor에서 MCP의 역할

### 목표

MCP가 Cursor의 Agent 모드와 어떻게 연동되고, 어떤 방식으로 도구를 호출하는지 설명할 수 있다.

### 단계별 구현

#### Step 1.1 — MCP의 필요성 파악하기

> **💡 개념 설명: MCP (Model Context Protocol)**
>
> Cursor에서 "현재 저장소의 GitHub 이슈를 보고 코드 수정해줘"라고 하면 어떻게 될까? 기본 상태에서는 로컬 파일만 볼 수 있다 — GitHub API 접근 방법도, 인증 정보도 없다.
>
> MCP는 AI 에이전트가 외부 시스템과 **표준화된 방식**으로 통신할 수 있게 해주는 프로토콜이다. Cursor의 Agent 모드는 MCP 서버에 등록된 도구를 자동으로 선택·실행한다.
>
> **핵심 한 줄:** Cursor Agent + MCP = 외부 세계와 연결된 코딩 에이전트

```
Cursor Editor
    └── Agent 모드
            ↕ MCP 프로토콜
    MCP 서버들
        ├── GitHub MCP → GitHub API
        ├── Slack MCP  → Slack API
        └── DB MCP     → PostgreSQL
```

#### Step 1.2 — Cursor MCP의 특성

| 특성 | 내용 |
|------|------|
| 지원 플랜 | 모든 플랜 (Hobby, Pro, Business) |
| 최대 도구 수 | 40개 (초과 시 자동 비활성화) |
| 전송 방식 | stdio, SSE (Server-Sent Events) |
| 커뮤니티 서버 | 1,800개+ |
| 설정 UI | Settings → MCP 탭에서 관리 |

> **⚠️ 도구 한도 주의:** Cursor는 MCP 도구를 최대 40개까지만 활성화한다. 여러 서버를 등록할 경우 도구 수 합계가 40개를 넘지 않도록 관리한다.

#### Step 1.3 — 설정 스코프 계층 이해하기

| 설정 방법 | 적용 범위 | 특징 |
|-----------|-----------|------|
| Cursor Settings UI | 전역 (모든 프로젝트) | GUI 편집, 토큰 직접 입력 |
| `.cursor/mcp.json` (프로젝트 루트) | 해당 프로젝트 | Git 커밋 가능, 팀 공유 |

프로젝트 설정이 전역 설정에 **추가**된다 (오버라이드가 아닌 병합).

### 체크포인트

Cursor MCP의 40개 도구 한도와 두 가지 설정 스코프를 설명할 수 있는가?

---

## Phase 2: MCP 서버 설정하기

### 목표

`.cursor/mcp.json` 파일 생성과 Cursor Settings UI 두 가지 방법으로 MCP 서버를 설정하고, Agent 모드에서 인식되는지 확인한다.

### 단계별 구현

#### Step 2.1 — 프로젝트 설정 파일 생성

> **💡 개념 설명: .cursor/mcp.json 구조**
>
> `.cursor/mcp.json`은 프로젝트 루트의 `.cursor/` 디렉토리에 위치하는 팀 공유 MCP 설정 파일이다. `mcpServers` 키 하위에 각 서버를 이름과 함께 등록한다. 이 파일을 Git에 커밋하면 팀원 모두가 동일한 MCP 환경을 사용할 수 있다.
>
> **핵심 한 줄:** `.cursor/mcp.json` = 이 프로젝트에서 쓸 외부 도구 목록

```bash
# 프로젝트 루트에서 디렉토리 생성
mkdir -p .cursor
```

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_여기에_토큰_붙여넣기"
      }
    }
  }
}
```

> **⚠️ 주의:** 토큰을 `.cursor/mcp.json`에 직접 작성하면 Git에 실수로 커밋될 위험이 있다. 팀 환경에서는 환경 변수 참조 방식을 사용한다:
>
> ```json
> "GITHUB_TOKEN": "${GITHUB_TOKEN}"
> ```
>
> 그리고 각 팀원은 자신의 셸 프로파일에 `export GITHUB_TOKEN=ghp_...`을 추가한다.

#### Step 2.2 — Cursor Settings UI로 전역 설정

GUI를 통한 전역 MCP 서버 추가:

1. Cursor 열기 → **Settings** (⌘, 또는 Ctrl+,)
2. **MCP** 탭 클릭
3. **Add MCP Server** 버튼 클릭
4. 서버 정보 입력:
   - **Name:** `github`
   - **Type:** `command`
   - **Command:** `npx`
   - **Args:** `-y @modelcontextprotocol/server-github`
   - **Env:** `GITHUB_TOKEN=ghp_...`

Settings UI에서 저장하면 Cursor 전역 설정 파일에 자동으로 기록된다.

#### Step 2.3 — MCP 서버 연결 확인

Cursor Settings → MCP 탭에서 상태 확인:

```
● github       ✓ Connected   14 tools
● filesystem   ✓ Connected    6 tools
```

또는 Agent 채팅에서 확인:

```
어떤 MCP 도구를 사용할 수 있어?
```

Agent가 사용 가능한 MCP 도구 목록을 자동으로 나열한다.

### 체크포인트

Cursor Settings → MCP 탭에서 `Connected` 상태와 도구 수가 표시되면 성공이다.

---

## Phase 3: 주요 MCP 서버 활용

### 목표

GitHub, Slack, Notion MCP 서버를 설정하고, Cursor Agent 모드에서 자연어로 외부 서비스를 제어한다.

### 단계별 구현

#### Step 3.1 — GitHub MCP 활용

**GitHub Personal Access Token 발급:**

1. [github.com/settings/tokens](https://github.com/settings/tokens) 접속
2. **Generate new token (classic)** 클릭
3. 권한 선택: `repo`, `read:org`, `read:user`
4. 생성된 토큰 복사 (`ghp_...` 형태)

**설정:**

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

**Agent 모드에서 활용:**

Cursor 채팅에서 `⌘L` (Mac) 또는 `Ctrl+L` (Windows)로 채팅을 열고 **Agent** 모드 선택:

```
> 내 GitHub에서 열려있는 PR 목록 보여줘
> 저장소 my-user/my-project의 최근 이슈 5개 나열해줘
> 이슈 #42에 "검토 완료" 코멘트 달아줘
> 현재 파일의 변경사항으로 PR 만들어줘
```

#### Step 3.2 — Slack MCP 활용

**Slack Bot Token 발급:**

1. [api.slack.com/apps](https://api.slack.com/apps) → **Create New App**
2. **OAuth & Permissions** → Bot Token Scopes 추가:
   - `channels:read`, `channels:history`
   - `chat:write`, `users:read`
3. **Install to Workspace** → Bot User OAuth Token 복사 (`xoxb-...`)

**설정:**

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "github": { ... },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${SLACK_TEAM_ID}"
      }
    }
  }
}
```

**활용 예시:**

```
> #dev 채널에 배포 완료 메시지 보내줘
> 어제 #general 채널에서 오류 관련 메시지 검색해줘
> 이 버그 수정 사항을 #bugs 채널에 공유해줘
```

#### Step 3.3 — Notion MCP 활용

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer ${NOTION_TOKEN}\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

**활용 예시:**

```
> Notion의 API 스펙 문서를 읽고 타입스크립트 인터페이스 정의해줘
> Notion 프로젝트 계획의 요구사항에 맞게 이 함수 구현해줘
```

#### Step 3.4 — Brave Search MCP 활용

최신 라이브러리 문서나 스택 오버플로우 해결책을 검색하여 코드를 작성할 수 있다:

**Brave Search API Key 발급:**
- [brave.com/search/api](https://brave.com/search/api) 접속 → API Key 발급

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      }
    }
  }
}
```

**활용 예시:**

```
> React 19의 새로운 hook API를 검색하고 이 컴포넌트에 적용해줘
> 이 오류 메시지로 검색해서 해결 방법 찾아줘
```

#### Step 3.5 — 주요 MCP 서버 목록 (도구 수 포함)

| 서버 이름 | 패키지 | 도구 수 (대략) | 주요 기능 |
|-----------|--------|----------------|-----------|
| GitHub | `@modelcontextprotocol/server-github` | ~14 | 저장소, PR, 이슈 |
| Filesystem | `@modelcontextprotocol/server-filesystem` | ~6 | 로컬 파일 |
| Slack | `@modelcontextprotocol/server-slack` | ~5 | 채널, 메시지 |
| Notion | `@notionhq/notion-mcp-server` | ~8 | 페이지, 데이터베이스 |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | ~3 | DB 쿼리 |
| Brave Search | `@modelcontextprotocol/server-brave-search` | ~2 | 웹 검색 |

> **💡 도구 수 관리:** 위 서버를 모두 등록하면 약 38개로 40개 한도에 근접한다. 실제로 사용하는 서버만 등록하는 것을 권장한다.

### 체크포인트

Agent 모드에서 자연어 요청으로 실제 GitHub 이슈 목록이 반환되면 성공이다.

---

## Phase 4: 고급 활용

### 목표

SSE 방식의 원격 MCP 서버 연결, 커뮤니티 서버 탐색, Agent 모드 최적화 방법을 익힌다.

### 단계별 구현

#### Step 4.1 — SSE 방식 원격 MCP 서버 연결

> **💡 개념 설명: SSE (Server-Sent Events)**
>
> stdio 방식은 로컬 프로세스를 실행하지만, SSE 방식은 이미 실행 중인 원격 서버에 HTTP로 연결한다. 사내 MCP 서버를 공유 인프라로 운영하거나, 클라우드 기반 MCP 서비스를 사용할 때 SSE 방식을 사용한다.
>
> **핵심 한 줄:** SSE = 원격에 있는 MCP 서버에 HTTP로 연결

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "company-mcp": {
      "url": "https://mcp.company.internal/sse",
      "env": {
        "API_KEY": "${COMPANY_MCP_API_KEY}"
      }
    }
  }
}
```

로컬에서 SSE 서버를 실행하는 경우:

```json
{
  "mcpServers": {
    "local-sse": {
      "url": "http://localhost:3001/sse"
    }
  }
}
```

#### Step 4.2 — 커뮤니티 MCP 서버 탐색

Cursor는 1,800개 이상의 커뮤니티 MCP 서버를 공식 디렉토리에서 제공한다:

- **Cursor 내에서:** Settings → MCP → **Browse MCP Servers**
- **웹에서:** [cursor.com/mcp](https://cursor.com/mcp)
- **공식 목록:** [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)

커뮤니티 서버 추가 예시 (Linear 이슈 트래커):

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@linear/mcp-server"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    }
  }
}
```

#### Step 4.3 — Agent 모드에서 MCP 최적화

**도구 선택 힌트 제공:**

Agent가 더 정확한 MCP 도구를 선택하도록 요청에 컨텍스트를 추가한다:

```
// 모호한 요청 (비권장)
> 이슈 목록 보여줘

// 명확한 요청 (권장)
> GitHub MCP를 사용해서 my-org/my-repo의 open 이슈 목록 보여줘
> Slack #dev 채널의 오늘 메시지를 검색해서 오류 관련 스레드 찾아줘
```

**Cursor Rules와 MCP 조합:**

`.cursor/rules` 파일에 MCP 사용 지침을 추가하면 Agent가 자동으로 적절한 MCP를 활용한다:

```
// .cursor/rules

## MCP 사용 지침
- GitHub 관련 작업(이슈, PR 조회)은 항상 GitHub MCP 도구를 사용한다
- 외부 라이브러리 문서가 필요하면 Brave Search MCP를 활용한다
- 데이터베이스 스키마 확인 시 PostgreSQL MCP를 먼저 조회한다
```

#### Step 4.4 — 전체 .cursor/mcp.json 예시

실제 팀 프로젝트에서 사용하는 전체 설정 예시:

```json
// .cursor/mcp.json

{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${SLACK_TEAM_ID}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${DATABASE_URL}"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${BRAVE_API_KEY}"
      }
    }
  }
}
```

### 체크포인트

`.cursor/mcp.json`에 여러 서버를 등록하고, 도구 수 합계가 40개 미만임을 Cursor Settings에서 확인하면 성공이다.

---

## 도구 간 비교표

| 항목 | Cursor | Claude Code | VS Code Copilot | Antigravity |
|------|--------|-------------|-----------------|-------------|
| 설정 파일 | `.cursor/mcp.json` | `.mcp.json` | `.vscode/settings.json` | `~/.gemini/settings.json` |
| 전역 설정 UI | Settings → MCP | 없음 (파일만) | Settings Editor | 없음 (파일만) |
| 최대 도구 수 | 40개 | 80개 권장 | 제한 없음 | 제한 없음 |
| 전송 방식 | stdio, SSE | stdio, HTTP | stdio, HTTP | stdio |
| 커뮤니티 서버 | 1,800개+ 디렉토리 | 수동 탐색 | GitHub MCP Registry | 수동 탐색 |
| Agent 자동 활용 | Agent 모드 | 대화형 | Copilot Chat | Agent Manager |
| 상태 확인 | Settings UI + 채팅 | `/mcp` | Command Palette | `/mcp list` |
| 프로젝트 설정 | `.cursor/mcp.json` | `.mcp.json` | workspace settings | `.agent/settings.json` |

---

## 마무리

### 이 가이드에서 배운 것

- Cursor MCP의 40개 도구 한도와 stdio/SSE 두 가지 전송 방식
- `.cursor/mcp.json`(프로젝트)와 Settings UI(전역) 두 가지 설정 방법
- GitHub, Slack, Notion, Brave Search MCP 서버 연동 방법
- SSE 방식으로 원격 MCP 서버 연결하는 방법
- Agent 모드에서 MCP 도구를 정확하게 활용하는 팁

### 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| Settings → MCP에서 서버가 안 보임 | `.cursor/mcp.json` 문법 오류 | JSON 유효성 검사기로 확인 |
| `Disconnected` 상태 | 토큰 오류 또는 npx 실행 실패 | 터미널에서 직접 npx 명령 실행해 오류 확인 |
| 도구가 40개 초과 | 서버 너무 많음 | 자주 사용하지 않는 서버 제거 |
| Agent가 MCP 도구를 안 씀 | 일반 채팅 모드 사용 중 | `⌘L` → **Agent** 모드로 전환 |
| `${GITHUB_TOKEN}` 미치환 | 환경 변수 미설정 | 셸 프로파일에 `export GITHUB_TOKEN=...` 추가 |
| SSE 서버 연결 실패 | URL 또는 인증 오류 | 브라우저에서 URL 직접 접속 테스트 |

### 다음 단계

- [Cursor MCP 서버 디렉토리](https://cursor.com/mcp)에서 1,800개+ 커뮤니티 서버 탐색
- [MCP 공식 서버 목록](https://github.com/modelcontextprotocol/servers)에서 추가 서버 발견
- `.cursor/rules`와 MCP를 조합하여 프로젝트 특화 AI 에이전트 구성
