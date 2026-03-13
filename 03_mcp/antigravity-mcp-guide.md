# Google Antigravity (Gemini CLI) MCP 서버 연동 가이드

## 학습 목표

MCP(Model Context Protocol)가 Google Antigravity(에디터 네이티브 Gemini 에이전트)에서 어떤 역할을 하는지 이해하고, Gemini CLI와 공유되는 설정 방식을 통해 외부 서비스를 연결하여 에디터 내에서 AI 에이전트가 외부 데이터를 활용할 수 있도록 설정한다.

## 사전 준비

- Google Antigravity 설치 완료 (`01_agentic_coding_tool/02_editor_native/antigravity-guide.md` 참고)
- Gemini CLI 설치 및 인증 완료 (`01_agentic_coding_tool/01_cli_driven/gemini-cli-guide.md` 참고)
- Node.js 18 이상 (`node --version`으로 확인)
- GitHub Personal Access Token (GitHub MCP 실습 시 필요)

---

## 전체 흐름 한눈에 보기

Google Antigravity는 에디터(IDE) 내에서 동작하는 Gemini 기반 에이전트 도구이다. Gemini CLI와 동일한 MCP 설정 파일을 공유하므로, 한 번 설정하면 CLI와 에디터 양쪽에서 동일한 MCP 서버를 사용할 수 있다.

1. **MCP 개념 이해** — Antigravity에서 MCP의 역할과 Gemini CLI와의 관계
2. **MCP 서버 설정** — `~/.gemini/settings.json` 및 프로젝트 설정
3. **주요 서버 활용** — GitHub, Notion 등 에이전트가 자동 활용
4. **고급 활용** — Agent Manager와 MCP 연동, 커스텀 서버

---

## Phase 1: Antigravity에서 MCP의 역할

### 목표

MCP가 Antigravity 에디터 환경에서 어떻게 동작하고, Gemini CLI와의 관계를 설명할 수 있다.

### 단계별 구현

#### Step 1.1 — Antigravity와 MCP의 관계 파악하기

> **💡 개념 설명: Antigravity와 Gemini CLI의 공유 설정**
>
> Google Antigravity는 에디터 내에서 동작하는 Gemini 에이전트이다. 내부적으로 Gemini CLI의 설정 인프라를 공유한다. 즉, `~/.gemini/settings.json`에 등록한 MCP 서버는 Gemini CLI와 Antigravity 양쪽에서 동시에 사용된다.
>
> **핵심 한 줄:** 설정 한 번 = Gemini CLI와 Antigravity 모두에서 MCP 사용 가능

```
에디터 (VS Code/JetBrains/etc.)
    └── Antigravity Plugin
            ↕ 공유 설정
        Gemini CLI
            ↕
    ~/.gemini/settings.json
            ↕
    MCP 서버들 (GitHub, Notion, Slack ...)
```

#### Step 1.2 — 에디터 네이티브 MCP 활용의 장점

| 장점 | 설명 |
|------|------|
| 인라인 컨텍스트 | 에디터에서 파일 편집 중 MCP 데이터를 즉시 참조 |
| Agent Manager 통합 | 에이전트가 MCP 도구를 자동으로 선택·실행 |
| 공유 설정 | Gemini CLI와 동일한 설정 재사용 |
| 멀티 에이전트 | 여러 에이전트가 동일 MCP 서버를 병렬 활용 |

#### Step 1.3 — 설정 스코프 계층 이해하기

| 설정 파일 위치 | 적용 범위 | 우선순위 |
|----------------|-----------|----------|
| `~/.gemini/settings.json` | 전역 (모든 프로젝트) | 낮음 |
| `.agent/settings.json` (프로젝트 루트) | 해당 프로젝트 | 높음 |
| `.gemini/settings.json` (프로젝트 루트) | 해당 프로젝트 (Gemini CLI 호환) | 높음 |

프로젝트 설정이 전역 설정을 오버라이드(덮어쓰기)한다.

### 체크포인트

Antigravity가 Gemini CLI와 설정을 공유하는 이유와 설정 스코프 계층을 설명할 수 있는가?

---

## Phase 2: MCP 서버 설정하기

### 목표

`~/.gemini/settings.json`과 프로젝트 `.agent/settings.json`에 MCP 서버를 설정하고, Antigravity에서 인식되는지 확인한다.

### 단계별 구현

#### Step 2.1 — GitHub Personal Access Token 발급

GitHub MCP 서버는 GitHub API에 접근하기 위해 인증 토큰이 필요하다.

1. [github.com/settings/tokens](https://github.com/settings/tokens) 접속
2. **Generate new token (classic)** 클릭
3. 다음 권한 선택:
   - `repo` (저장소 읽기/쓰기)
   - `read:org` (조직 정보 읽기)
   - `read:user` (사용자 정보 읽기)
4. 생성된 토큰 복사 (`ghp_...` 형태)

#### Step 2.2 — 전역 설정 파일에 MCP 서버 등록

> **💡 개념 설명: settings.json 구조**
>
> `~/.gemini/settings.json`은 Gemini CLI와 Antigravity가 공유하는 전역 설정 파일이다. `mcpServers` 키 하위에 각 MCP 서버를 이름과 함께 등록한다. 이 설정은 모든 프로젝트에 자동 적용된다.
>
> **핵심 한 줄:** settings.json의 mcpServers = Antigravity 에이전트가 사용할 수 있는 외부 도구 목록

```json
// ~/.gemini/settings.json

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

> **⚠️ 주의:** 토큰을 파일에 직접 작성하면 보안 위험이 있다. 반드시 환경 변수 참조 방식(`${GITHUB_TOKEN}`)을 사용하고, 셸 프로파일(`.bashrc`, `.zshrc`)에 `export GITHUB_TOKEN=ghp_...`을 추가한다.

#### Step 2.3 — 프로젝트별 MCP 설정

팀 프로젝트에서는 프로젝트 전용 MCP 설정을 사용한다:

```json
// <project-root>/.agent/settings.json

{
  "mcpServers": {
    "project-db": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${PROJECT_DB_URL}"
      }
    },
    "project-docs": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "./docs",
        "./specs"
      ]
    }
  }
}
```

이 파일을 Git에 커밋하면 팀원 모두가 동일한 MCP 환경을 공유한다 (토큰은 개인 환경 변수로 관리).

#### Step 2.4 — MCP 서버 연결 확인

Gemini CLI에서 확인하면 Antigravity에서도 동일하게 인식된다:

```bash
# Gemini CLI 실행
gemini

# MCP 서버 목록 확인
/mcp list
```

정상 설정 시 출력:

```
● github (connected)
  - create_issue
  - get_pull_request
  - list_repos
  - search_code
  ... (여러 도구 목록)

● filesystem (connected)
  - read_file
  - write_file
  - list_directory
```

에디터 내 Antigravity에서도 같은 MCP 서버들이 에이전트에게 노출된다.

### 체크포인트

`/mcp list` 실행 시 등록한 서버들이 `connected` 상태로 표시되면 성공이다.

---

## Phase 3: 주요 MCP 서버 활용

### 목표

Antigravity의 Agent Manager가 MCP 도구를 자동으로 활용하는 방식을 이해하고, 에디터 내에서 GitHub, Notion 등 서비스와 연동하여 작업한다.

### 단계별 구현

#### Step 3.1 — Agent Manager와 MCP 자동 활용

> **💡 개념 설명: Agent Manager와 MCP 도구**
>
> Antigravity의 Agent Manager는 사용자 요청을 분석하여 적절한 MCP 도구를 자동으로 선택·실행한다. "GitHub PR 목록을 보고 이 코드 작성해줘"라고 하면 에이전트가 스스로 GitHub MCP를 호출하여 PR 목록을 가져온 후 코드를 작성한다.
>
> **핵심 한 줄:** MCP를 설정하면 에이전트가 알아서 맥락에 맞는 도구를 고른다

에디터 내 Antigravity 채팅에서 자연어로 요청:

```
현재 저장소의 열려있는 이슈 목록을 보고, 가장 긴급한 버그를 수정하는 코드 작성해줘

이 파일의 함수와 관련된 GitHub 이슈가 있는지 찾아줘

PR #42의 리뷰 코멘트를 반영해서 현재 파일을 수정해줘
```

#### Step 3.2 — GitHub MCP 활용

```json
// ~/.gemini/settings.json

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

**에디터에서의 활용 흐름:**

```
1. 에디터에서 파일 열기
2. Antigravity 채팅창에서 요청
3. 에이전트가 GitHub MCP 자동 호출
4. 이슈/PR 정보를 컨텍스트로 코드 생성
5. 에디터에 인라인으로 변경사항 적용
```

#### Step 3.3 — Notion MCP 활용

기획 문서와 API 스펙을 Notion에서 직접 참조하여 코드를 작성할 수 있다:

**Notion Integration Token 발급:**

> **⚠️ 주의:** Notion 설정에 비슷한 메뉴가 두 개 있다. **Build → Internal integrations**를 사용해야 한다.

1. [notion.so/my-integrations](https://www.notion.so/my-integrations) 접속
2. **+ New integration** 클릭 → 이름·워크스페이스 입력 → Submit
3. **Capabilities**에서 Read content 체크 → **Save changes**
4. **Secrets** 섹션 → **Copy**로 토큰 복사 (`ntn_...` 형태)
5. 접근할 Notion 페이지 우상단 **⋯ → Connections → 생성한 integration 추가**

```json
// ~/.gemini/settings.json

{
  "mcpServers": {
    "github": { ... },
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

**에디터에서의 활용 예시:**

```
Notion의 API 스펙 문서를 읽고 해당 API를 호출하는 클라이언트 코드 작성해줘

Notion 프로젝트 계획 페이지의 요구사항에 맞게 이 모듈 리팩터링해줘
```

#### Step 3.4 — /mcp 관리 명령어

Gemini CLI와 Antigravity 모두에서 사용 가능한 MCP 명령어:

| 명령어 | 설명 |
|--------|------|
| `/mcp list` | 연결된 서버와 도구 목록 |
| `/mcp show` | 도구 설명 표시 (에이전트 선택에 도움) |
| `/mcp hide` | 도구 설명 숨기기 (컨텍스트 절약) |
| `/mcp schema` | 도구의 JSON 스키마 상세 확인 |

#### Step 3.5 — 주요 MCP 서버 목록

| 서버 이름 | 패키지 | 주요 기능 | Antigravity 활용 시나리오 |
|-----------|--------|-----------|--------------------------|
| GitHub | `@github/github-mcp-server` | PR, 이슈, 코드 | 이슈 기반 코드 생성 |
| Notion | `@notionhq/notion-mcp-server` | 문서, 데이터베이스 | 스펙 문서 기반 개발 |
| Filesystem | `@modelcontextprotocol/server-filesystem` | 로컬 파일 | 프로젝트 외 파일 참조 |
| Slack | `@modelcontextprotocol/server-slack` | 팀 채널, DM | 슬랙 스레드 기반 수정 |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | DB 쿼리 | 스키마 기반 코드 생성 |

### 체크포인트

에디터 내 Antigravity에서 자연어 요청 시 MCP 도구가 자동으로 호출되어 외부 데이터가 참조되면 성공이다.

---

## Phase 4: 고급 활용

### 목표

커스텀 MCP 서버를 Antigravity 에이전트와 연동하고, 멀티 에이전트 환경에서 MCP를 효율적으로 활용한다.

### 단계별 구현

#### Step 4.1 — 커스텀 MCP 서버와 Antigravity 연동

사내 시스템을 위한 커스텀 MCP 서버를 작성하여 에이전트가 사내 API에 접근하도록 한다:

```typescript
// custom-mcp-server.ts

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "company-api", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "search_internal_docs",
      description: "사내 문서 시스템에서 기술 문서를 검색한다",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string", description: "검색 키워드" },
          category: { type: "string", enum: ["api", "design", "runbook"] }
        },
        required: ["query"]
      }
    }
  ]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "search_internal_docs") {
    const { query, category } = request.params.arguments as {
      query: string;
      category?: string;
    };
    // 실제 사내 API 호출
    const results = await fetchInternalDocs(query, category);
    return {
      content: [{ type: "text", text: JSON.stringify(results) }]
    };
  }
  throw new Error(`Unknown tool: ${request.params.name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);

async function fetchInternalDocs(query: string, category?: string) {
  // 사내 API 호출 로직
  return [{ title: "예시 문서", url: "https://internal.example.com/docs/1" }];
}
```

빌드 후 설정에 등록:

```json
// ~/.gemini/settings.json

{
  "mcpServers": {
    "company-api": {
      "command": "node",
      "args": ["/path/to/custom-mcp-server.js"],
      "env": {
        "INTERNAL_API_KEY": "${INTERNAL_API_KEY}"
      }
    }
  }
}
```

#### Step 4.2 — 멀티 에이전트 환경에서 MCP 활용

> **💡 개념 설명: Antigravity 멀티 에이전트**
>
> Antigravity는 복잡한 작업을 여러 하위 에이전트로 분할하여 병렬 처리할 수 있다. 각 에이전트는 독립적으로 MCP 도구를 호출하므로, 예를 들어 한 에이전트는 GitHub에서 이슈를 조회하고, 다른 에이전트는 Notion에서 스펙을 읽는 동시 작업이 가능하다.
>
> **핵심 한 줄:** MCP 도구 수가 많을수록 에이전트의 병렬 처리 능력이 향상된다

멀티 에이전트 활용 예시:

```
이 기능을 구현해줘:
1. GitHub 이슈 #123의 요구사항 분석
2. Notion에서 관련 API 스펙 문서 조회
3. 기존 코드 패턴 파악
4. 위 세 가지를 통합하여 구현 코드 작성
```

에이전트가 자동으로 각 단계에서 적절한 MCP 도구를 선택·실행한다.

#### Step 4.3 — MCP 설정 최적화 팁

```json
// 프로젝트별 최적화 예시
// <project-root>/.agent/settings.json

{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  },
  "agentSettings": {
    "mcpToolDescriptions": "show",
    "maxConcurrentMcpCalls": 3
  }
}
```

### 체크포인트

커스텀 MCP 서버가 `/mcp list`에 등록되고, Antigravity 에이전트가 해당 도구를 사용하여 작업을 수행하면 성공이다.

---

## 도구 간 비교표

| 항목 | Antigravity | Gemini CLI | Claude Code | Cursor |
|------|-------------|------------|-------------|--------|
| 설정 파일 | `~/.gemini/settings.json` (공유) | `~/.gemini/settings.json` | `.mcp.json` | `.cursor/mcp.json` |
| 동작 환경 | 에디터 내 (인라인) | 터미널 | 터미널 | 에디터 내 |
| 설정 공유 | Gemini CLI와 공유 | Antigravity와 공유 | 독립 | 독립 |
| Agent 자동 활용 | Agent Manager가 자동 선택 | 자연어 요청 시 선택 | 자연어 요청 시 선택 | Agent 모드에서 선택 |
| 프로젝트 설정 | `.agent/settings.json` | `.gemini/settings.json` | `.mcp.json` | `.cursor/mcp.json` |
| 상태 확인 | `/mcp list` (공유) | `/mcp list` | `/mcp` | Settings UI |
| HTTP/OAuth | 제한적 | 제한적 | 지원 | SSE 지원 |

---

## 마무리

### 이 가이드에서 배운 것

- Antigravity가 Gemini CLI와 `~/.gemini/settings.json`을 공유한다는 핵심 개념
- 전역(`~/.gemini/settings.json`)과 프로젝트(`.agent/settings.json`) 설정 분리 방법
- GitHub, Notion 등 서비스를 에디터 내 에이전트와 연동하는 방법
- Agent Manager가 MCP 도구를 자동으로 선택·실행하는 방식
- 커스텀 MCP 서버 작성과 Antigravity 연동

### 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| `/mcp list`에서 서버가 안 보임 | settings.json 문법 오류 | JSON 유효성 검사기로 확인 |
| `(disconnected)` 상태 | 토큰 오류 또는 npx 실행 실패 | 터미널에서 직접 npx 명령 실행해 오류 확인 |
| 에디터에서만 MCP 미인식 | Antigravity 플러그인 재시작 필요 | 에디터 재시작 또는 플러그인 리로드 |
| 프로젝트 설정이 전역 설정과 충돌 | 동일 서버명으로 오버라이드됨 | 프로젝트 설정에서 서버명 차별화 |
| `${GITHUB_TOKEN}` 미치환 | 환경 변수 미설정 | 셸 프로파일에 `export GITHUB_TOKEN=...` 추가 |

### 다음 단계

- [Gemini CLI MCP 가이드](./mcp-guide.md)에서 CLI 기반 MCP 활용 방법 심화 학습
- [MCP 서버 공식 목록](https://github.com/modelcontextprotocol/servers) 탐색
- `05_workflow/` 가이드에서 Antigravity 에이전트와 MCP를 활용한 자동화 파이프라인 구성 방법 학습
