# Claude Code MCP 서버 연동 가이드

## 학습 목표

MCP(Model Context Protocol)가 Claude Code에서 어떤 역할을 하는지 이해하고, GitHub, Notion 등 외부 서비스를 연결하여 대화 중 실제 데이터를 조회하고 작업을 수행할 수 있다.

## 사전 준비

- Claude Code 설치 및 인증 완료 (`01_agentic_coding_tool/01_cli_driven/claude-code-guide.md` 참고)
- Node.js 18 이상 (`node --version`으로 확인)
- GitHub Personal Access Token (GitHub MCP 실습 시 필요)

---

## 전체 흐름 한눈에 보기

Claude Code는 기본적으로 파일 조작, 셸 실행, 코드 편집 등 로컬 작업만 할 수 있다. MCP 서버를 추가하면 GitHub 이슈 조회, Notion 페이지 편집, Slack 메시지 전송 등 외부 시스템과 직접 상호작용할 수 있게 된다.

1. **MCP 개념 이해** — Claude Code에서 MCP의 역할과 프로토콜 구조 파악
2. **MCP 서버 설정** — `.mcp.json`(프로젝트) 또는 `settings.json`(전역) 설정
3. **주요 서버 활용** — GitHub, Notion 등 실제 서비스 연동
4. **고급 활용** — HTTP MCP, OAuth 인증, 커스텀 서버 작성

---

## Phase 1: Claude Code에서 MCP의 역할

### 목표

MCP가 해결하는 문제를 이해하고, Claude Code에서 어떤 방식으로 동작하는지 설명할 수 있다.

### 단계별 구현

#### Step 1.1 — MCP의 필요성 파악하기

> **💡 개념 설명: MCP (Model Context Protocol)**
>
> Claude Code에서 "내 GitHub PR 목록 보여줘"라고 하면 어떻게 될까? 기본 상태에서는 대답할 수 없다 — GitHub API 인증 정보도 없고 호출 방법도 모른다.
>
> MCP는 AI 에이전트가 외부 시스템과 **표준화된 방식**으로 통신할 수 있게 해주는 프로토콜이다. Anthropic이 설계하고 여러 회사가 채택한 개방형 표준으로, 각 외부 서비스는 MCP 서버 형태로 자신의 기능을 노출한다.
>
> **핵심 한 줄:** MCP = Claude Code가 외부 세계와 통신하는 표준 언어

```
Claude Code ←→ MCP 프로토콜 ←→ MCP 서버 ←→ 외부 서비스
                                  GitHub MCP    GitHub API
                                  Notion MCP    Notion API
                                  Slack MCP     Slack API
```

#### Step 1.2 — MCP 전송 방식 파악하기

Claude Code는 다음 세 가지 전송 방식을 지원한다:

| 전송 방식 | 설명 | 사용 예시 |
|-----------|------|-----------|
| `stdio` | 로컬 프로세스를 표준 입출력으로 연결 | npx로 실행하는 패키지 |
| `http` | 원격 HTTP 서버 연결 (OAuth 지원) | Notion, 클라우드 서비스 |
| `sse` | Server-Sent Events 방식의 스트리밍 | 실시간 이벤트 처리 |

이 가이드에서는 **stdio (npx)** 방식과 **HTTP (OAuth)** 방식을 모두 다룬다.

#### Step 1.3 — 설정 범위(스코프) 이해하기

| 설정 파일 위치 | 적용 범위 | 용도 |
|----------------|-----------|------|
| `~/.claude/settings.json` | 전역 (모든 프로젝트) | 개인 도구, 항상 쓰는 서버 |
| `.mcp.json` (프로젝트 루트) | 해당 프로젝트 | 팀 공유 도구, Git 커밋 가능 |
| `.claude/settings.local.json` | 로컬 오버라이드 | 개인 토큰, Git 제외 필수 |

### 체크포인트

MCP 전송 방식 3가지와 설정 스코프 2단계를 설명할 수 있는가?

---

## Phase 2: MCP 서버 설정하기

### 목표

`claude mcp add` CLI 명령과 설정 파일 직접 편집 두 가지 방법으로 MCP 서버를 등록하고, `/mcp` 명령으로 연결 상태를 확인한다.

### 단계별 구현

#### Step 2.1 — CLI로 MCP 서버 추가하기

Claude Code는 설정 파일을 직접 편집하지 않고 CLI 명령으로 MCP 서버를 추가할 수 있다:

```bash
# stdio 방식 (npx 패키지)
claude mcp add --transport stdio github -- npx -y @github/github-mcp-server

# HTTP 방식 (OAuth 지원)
claude mcp add --transport http notion https://mcp.notion.com/mcp

# 환경 변수와 함께 추가
claude mcp add --transport stdio github \
  -e GITHUB_TOKEN=ghp_... \
  -- npx -y @github/github-mcp-server
```

> **💡 개념 설명: `--` (더블 대시)의 역할**
>
> `--` 뒤의 내용은 `claude mcp add` 명령 자체의 인수가 아니라, MCP 서버를 시작하는 명령어로 그대로 전달된다. 즉 `npx -y @github/github-mcp-server`가 MCP 서버 실행 명령이 된다.

#### Step 2.2 — 설정 파일 직접 편집하기

> **💡 개념 설명: .mcp.json 구조**
>
> `.mcp.json`은 프로젝트 루트에 위치하는 팀 공유 MCP 설정 파일이다. `mcpServers` 키 하위에 각 서버를 이름과 함께 등록한다. 이 파일을 Git에 커밋하면 팀원 모두가 동일한 MCP 환경을 사용할 수 있다.
>
> **핵심 한 줄:** `.mcp.json` = "이 프로젝트에서 사용하는 외부 도구 목록"

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
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/사용자명/Projects"
      ]
    }
  }
}
```

전역 설정은 `~/.claude/settings.json`에 동일한 형식으로 작성한다:

```json
// ~/.claude/settings.json

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

> **⚠️ 주의:** 토큰을 파일에 직접 작성하면 Git에 실수로 커밋될 위험이 있다. 반드시 환경 변수 참조 방식(`${GITHUB_TOKEN}`)을 사용하고, 셸 프로파일(`.bashrc`, `.zshrc`)에 `export GITHUB_TOKEN=ghp_...`을 추가한다.

#### Step 2.3 — MCP 서버 연결 확인

Claude Code 대화형 세션에서 `/mcp` 명령으로 상태를 확인한다:

```bash
claude
```

```
> /mcp
```

정상 설정 시 출력:

```
● github (connected)
  Tools: create_issue, get_pull_request, list_repos, search_code ...
● filesystem (connected)
  Tools: read_file, write_file, list_directory ...
```

MCP 서버 목록 관리 명령어:

```bash
# 등록된 서버 목록 확인
claude mcp list

# 특정 서버 상세 정보
claude mcp get github

# 서버 제거
claude mcp remove github
```

### 체크포인트

`/mcp` 실행 시 `github (connected)` 상태와 도구 목록이 표시되면 성공이다.

---

## Phase 3: 주요 MCP 서버 활용

### 목표

GitHub, Notion, Slack MCP 서버를 실제로 설정하고, Claude Code 대화에서 자연어로 외부 서비스를 제어한다.

### 단계별 구현

#### Step 3.1 — GitHub MCP 활용

**GitHub Personal Access Token 발급:**

1. [github.com/settings/tokens](https://github.com/settings/tokens) 접속
2. **Generate new token (classic)** 클릭
3. 다음 권한 선택:
   - `repo` (저장소 읽기/쓰기)
   - `read:org` (조직 정보 읽기)
   - `read:user` (사용자 정보 읽기)
4. 생성된 토큰 복사 (`ghp_...` 형태)

**설정:**

```json
// .mcp.json

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

**활용 예시:**

```
> 내 GitHub에서 열려있는 PR 목록 보여줘
> 저장소 my-user/my-project의 최근 이슈 5개 나열해줘
> 이슈 #42에 "검토 완료" 코멘트 달아줘
> 이 코드 변경사항으로 PR 만들어줘
```

#### Step 3.2 — Notion MCP 활용 (HTTP 방식)

Notion MCP는 HTTP 전송 방식으로 OAuth 인증을 지원한다. 별도 토큰 발급 없이 인증 흐름이 자동으로 처리된다:

```bash
# HTTP 방식으로 추가 (OAuth 자동 처리)
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

또는 Notion Integration Token을 사용하는 stdio 방식:

```json
// .mcp.json

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
> Notion에서 프로젝트 계획 페이지 읽어줘
> Notion에 오늘 회의록 페이지 만들어줘
> Notion 데이터베이스에서 진행 중인 작업 목록 보여줘
```

#### Step 3.3 — Slack MCP 활용

```json
// .mcp.json

{
  "mcpServers": {
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
> #general 채널에 배포 완료 메시지 보내줘
> 어제 #dev 채널에서 오류 관련 메시지 검색해줘
```

#### Step 3.4 — 주요 MCP 서버 목록

| 서버 이름 | 패키지 | 주요 기능 |
|-----------|--------|-----------|
| GitHub | `@github/github-mcp-server` | 저장소, PR, 이슈 관리 |
| Notion (HTTP) | `https://mcp.notion.com/mcp` | 페이지·데이터베이스 (OAuth) |
| Notion (stdio) | `@notionhq/notion-mcp-server` | 페이지·데이터베이스 (토큰) |
| Filesystem | `@modelcontextprotocol/server-filesystem` | 로컬 파일 읽기/쓰기 |
| Slack | `@modelcontextprotocol/server-slack` | 채널 메시지, DM |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | DB 쿼리 실행 |
| Brave Search | `@modelcontextprotocol/server-brave-search` | 웹 검색 |

### 체크포인트

자연어 요청으로 GitHub 저장소 목록 또는 이슈 목록이 실제로 반환되면 성공이다.

---

## Phase 4: 고급 활용

### 목표

커스텀 MCP 서버를 작성하고, 인증 보안을 강화하며, 컨텍스트 윈도우 영향을 최소화하는 방법을 익힌다.

### 단계별 구현

#### Step 4.1 — 컨텍스트 윈도우 영향 관리

> **💡 개념 설명: MCP와 컨텍스트 윈도우**
>
> MCP 서버가 많을수록 사용 가능한 도구 목록이 Claude의 컨텍스트 윈도우를 차지한다. 도구가 너무 많으면 실제 대화에 쓸 수 있는 공간이 줄어들고 응답 품질이 저하될 수 있다.
>
> **권장 한도:** MCP 서버 10개 이하, 총 도구 수 80개 이하

현재 도구 수를 확인하는 방법:

```
> /mcp
```

출력 예시에서 각 서버의 도구 수를 합산하여 80개 미만인지 확인한다.

불필요한 서버는 비활성화한다:

```bash
# 특정 서버 임시 제거
claude mcp remove filesystem

# 필요할 때만 추가
claude mcp add --transport stdio filesystem -- npx -y @modelcontextprotocol/server-filesystem /tmp
```

#### Step 4.2 — 커스텀 MCP 서버 작성하기

> **💡 개념 설명: 커스텀 MCP 서버**
>
> 사내 API, 레거시 시스템, 자체 데이터베이스 등 공개 MCP 서버가 없는 서비스를 연결하려면 직접 MCP 서버를 작성한다. MCP SDK는 TypeScript와 Python을 공식 지원한다.

간단한 커스텀 MCP 서버 (TypeScript):

```typescript
// my-mcp-server.ts

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-custom-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// 도구 목록 등록
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_company_info",
      description: "회사 내부 정보를 조회한다",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string", description: "조회할 정보" }
        },
        required: ["query"]
      }
    }
  ]
}));

// 도구 실행 핸들러
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_company_info") {
    const query = request.params.arguments?.query as string;
    // 실제 로직: 사내 API 호출, DB 쿼리 등
    return {
      content: [{ type: "text", text: `조회 결과: ${query}에 대한 정보` }]
    };
  }
  throw new Error(`Unknown tool: ${request.params.name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

설정 파일에 등록:

```json
{
  "mcpServers": {
    "company": {
      "command": "node",
      "args": ["/path/to/my-mcp-server.js"]
    }
  }
}
```

#### Step 4.3 — HTTP MCP와 OAuth 인증

HTTP 전송 방식은 토큰 없이 OAuth 2.0 인증 흐름을 자동으로 처리한다:

```bash
# OAuth 지원 서비스 연결
claude mcp add --transport http notion https://mcp.notion.com/mcp
claude mcp add --transport http linear https://mcp.linear.app/mcp
```

처음 연결 시 브라우저가 자동으로 열려 OAuth 인증을 진행한다. 이후 토큰은 자동으로 관리된다.

#### Step 4.4 — 프로젝트별 MCP 설정과 보안

팀 환경에서의 권장 설정 구조:

```
project/
├── .mcp.json              # 팀 공유: 서버 목록 (토큰 없음), Git 커밋
└── .claude/
    └── settings.local.json  # 개인 전용: 실제 토큰 값, .gitignore에 추가
```

`.mcp.json` (Git 커밋 가능):

```json
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

`.claude/settings.local.json` (`.gitignore`에 추가 필수):

```json
{
  "env": {
    "GITHUB_TOKEN": "ghp_실제토큰값"
  }
}
```

### 체크포인트

커스텀 MCP 서버를 등록하고 `/mcp`에서 해당 서버와 도구가 보이면 성공이다.

---

## 도구 간 비교표

| 항목 | Claude Code | Gemini CLI | Cursor | VS Code Copilot |
|------|-------------|------------|--------|-----------------|
| 설정 파일 | `.mcp.json` / `settings.json` | `~/.gemini/settings.json` | `.cursor/mcp.json` | `.vscode/settings.json` |
| CLI 추가 명령 | `claude mcp add` | 없음 (파일 직접 편집) | 없음 (파일 직접 편집) | 없음 (파일 직접 편집) |
| HTTP/OAuth 지원 | 있음 (기본 지원) | 제한적 | SSE 지원 | VS Code 1.101+ |
| 전역/프로젝트 분리 | 있음 (2단계) | 있음 (2단계) | 있음 (2단계) | workspace 설정으로 분리 |
| 도구 수 권장 한도 | 80개 이하 | 제한 없음 | 40개 | 제한 없음 |
| 상태 확인 명령 | `/mcp` | `/mcp list` | Settings UI | Command Palette |

---

## 마무리

### 이 가이드에서 배운 것

- MCP 프로토콜의 역할: Claude Code와 외부 서비스를 연결하는 표준 인터페이스
- `claude mcp add` CLI와 `.mcp.json` 파일 두 가지 설정 방법
- stdio(npx)와 HTTP(OAuth) 두 가지 전송 방식의 차이
- GitHub, Notion, Slack 등 실제 서비스 연동 방법
- 커스텀 MCP 서버 작성과 컨텍스트 윈도우 관리

### 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| `/mcp`에서 서버가 안 보임 | `.mcp.json` 문법 오류 | JSON 유효성 검사기로 확인 |
| `(disconnected)` 상태 | 토큰 오류 또는 npx 실행 실패 | 터미널에서 직접 npx 명령을 실행해 오류 확인 |
| 도구 호출 실패 | 권한 부족 | GitHub 토큰 권한 범위 재확인 |
| 응답이 느림 | 도구 수 과다 | MCP 서버를 10개 이하로 줄이기 |
| `${GITHUB_TOKEN}` 미치환 | 환경 변수 미설정 | 셸 프로파일에 `export GITHUB_TOKEN=...` 추가 |

### 다음 단계

- [MCP 서버 공식 목록](https://github.com/modelcontextprotocol/servers) 탐색
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)로 커스텀 서버 작성
- `04_skill/` 가이드에서 Claude Code Skills와 MCP의 조합 방법 학습
