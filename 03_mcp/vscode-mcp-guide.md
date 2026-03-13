# VS Code (GitHub Copilot) MCP 서버 연동 가이드

## 학습 목표

MCP(Model Context Protocol)가 VS Code의 GitHub Copilot에서 어떤 역할을 하는지 이해하고, GitHub, Slack 등 외부 서비스를 연결하여 Copilot Chat의 Agent 모드에서 실제 데이터를 조회하고 작업을 수행할 수 있다.

## 사전 준비

- VS Code 최신 버전 설치 (MCP 지원은 VS Code 1.99+ 권장, 원격 MCP는 1.101+)
- GitHub Copilot 구독 및 GitHub Copilot 확장 설치
- Node.js 18 이상 (`node --version`으로 확인)
- GitHub Personal Access Token (GitHub MCP 실습 시 필요)

---

## 전체 흐름 한눈에 보기

VS Code의 GitHub Copilot은 기본적으로 로컬 코드베이스를 대상으로 AI 작업을 수행한다. MCP 서버를 추가하면 GitHub 이슈 조회, Slack 메시지 전송, 데이터베이스 쿼리 등 외부 시스템과 직접 상호작용할 수 있게 된다. Agent 플러그인을 활용하면 MCP, Skills, Hooks를 하나의 패키지로 구성할 수도 있다.

1. **MCP 개념 이해** — VS Code Copilot에서 MCP의 역할 파악
2. **MCP 서버 설정** — `.vscode/settings.json` 설정 방법
3. **주요 서버 활용** — GitHub, Slack, Notion 등 실제 서비스 연동
4. **고급 활용** — Agent 플러그인, 원격 MCP 서버, GitHub MCP Registry

---

## Phase 1: VS Code Copilot에서 MCP의 역할

### 목표

MCP가 VS Code Copilot의 Agent 모드와 어떻게 연동되고, 어떤 방식으로 도구를 호출하는지 설명할 수 있다.

### 단계별 구현

#### Step 1.1 — MCP의 필요성 파악하기

> **💡 개념 설명: MCP (Model Context Protocol)**
>
> VS Code Copilot에서 "현재 저장소의 GitHub 이슈를 보고 코드 수정해줘"라고 하면 어떻게 될까? Copilot은 로컬 파일은 볼 수 있지만 GitHub API 접근 방법이 없다.
>
> MCP는 AI 에이전트가 외부 시스템과 **표준화된 방식**으로 통신할 수 있게 해주는 프로토콜이다. Copilot Chat의 Agent 모드는 MCP 서버에 등록된 도구를 자동으로 선택·실행한다.
>
> **핵심 한 줄:** VS Code Copilot Agent + MCP = 외부 세계와 연결된 에디터 AI

```
VS Code Editor
    └── GitHub Copilot Extension
            └── Copilot Chat (Agent 모드)
                    ↕ MCP 프로토콜
            MCP 서버들
                ├── GitHub MCP → GitHub API
                ├── Slack MCP  → Slack API
                └── DB MCP     → PostgreSQL
```

#### Step 1.2 — VS Code MCP의 특성

| 특성 | 내용 |
|------|------|
| 필요 확장 | GitHub Copilot (구독 필요) |
| 설정 파일 | `.vscode/settings.json` |
| 전송 방식 | stdio, HTTP (VS Code 1.101+) |
| 원격 MCP | VS Code 1.101+에서 지원 |
| Agent 플러그인 | MCP + Skills + Hooks 통합 패키지 |
| MCP Registry | GitHub MCP Registry에서 큐레이션 서버 탐색 |

#### Step 1.3 — Copilot Chat 모드 이해하기

VS Code Copilot Chat은 세 가지 모드를 제공하며, MCP는 **Agent 모드**에서만 동작한다:

| 모드 | 설명 | MCP 사용 가능 여부 |
|------|------|-------------------|
| Ask | 코드 관련 질문·설명 | 불가 |
| Edit | 파일 편집 제안 | 불가 |
| Agent | 자율적 작업 수행, 도구 활용 | **가능** |

Copilot Chat에서 Agent 모드 선택: 채팅창 상단 드롭다운 → **Agent**

#### Step 1.4 — 설정 스코프 계층 이해하기

| 설정 파일 위치 | 적용 범위 | 특징 |
|----------------|-----------|------|
| `~/.config/Code/User/settings.json` (사용자) | 전역 | VS Code User Settings |
| `.vscode/settings.json` (프로젝트) | 해당 프로젝트 | Git 커밋 가능, 팀 공유 |

### 체크포인트

VS Code MCP는 Agent 모드에서만 동작하며, `.vscode/settings.json`에 설정함을 설명할 수 있는가?

---

## Phase 2: MCP 서버 설정하기

### 목표

`.vscode/settings.json`에 MCP 서버를 설정하고, Copilot Chat Agent 모드에서 인식되는지 확인한다.

### 단계별 구현

#### Step 2.1 — .vscode/settings.json에 MCP 서버 등록

> **💡 개념 설명: settings.json의 MCP 설정 키**
>
> VS Code에서 MCP 설정은 `github.copilot.chat.mcp.servers` 키를 사용한다. 다른 도구의 `mcpServers`와 구조는 동일하지만 키 이름이 다르다. 환경 변수 참조는 `${env:변수명}` 형식을 사용한다.
>
> **핵심 한 줄:** `github.copilot.chat.mcp.servers` = VS Code Copilot이 사용할 MCP 서버 목록

```json
// .vscode/settings.json

{
  "github.copilot.chat.mcp.servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    }
  }
}
```

> **⚠️ 주의:** VS Code의 환경 변수 참조 형식은 `${env:변수명}`이다. Claude Code나 Cursor의 `${변수명}` 형식과 다르다. 잘못된 형식을 사용하면 환경 변수가 치환되지 않는다.

#### Step 2.2 — 여러 MCP 서버 등록

```json
// .vscode/settings.json

{
  "github.copilot.chat.mcp.servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/사용자명/Projects"
      ]
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${env:SLACK_BOT_TOKEN}",
        "SLACK_TEAM_ID": "${env:SLACK_TEAM_ID}"
      }
    }
  }
}
```

#### Step 2.3 — VS Code Command Palette로 MCP 상태 확인

```
Ctrl+Shift+P (또는 ⌘+Shift+P) → "MCP" 검색
```

사용 가능한 명령:
- **MCP: List Servers** — 등록된 MCP 서버 목록
- **MCP: Start Server** — 특정 서버 수동 시작
- **MCP: Stop Server** — 특정 서버 중지
- **MCP: Show Server Output** — 서버 출력 로그 확인

Copilot Chat에서 확인:

```
> #github 어떤 도구들을 쓸 수 있어?
```

Agent가 사용 가능한 MCP 도구 목록을 자동으로 나열한다.

### 체크포인트

Command Palette → MCP: List Servers에서 등록한 서버가 보이고, Copilot Chat Agent 모드에서 MCP 도구가 인식되면 성공이다.

---

## Phase 3: 주요 MCP 서버 활용

### 목표

GitHub, Slack, Notion MCP 서버를 설정하고, VS Code Copilot Chat Agent 모드에서 자연어로 외부 서비스를 제어한다.

### 단계별 구현

#### Step 3.1 — GitHub MCP 활용

**GitHub Personal Access Token 발급:**

1. [github.com/settings/tokens](https://github.com/settings/tokens) 접속
2. **Generate new token (classic)** 클릭
3. 권한 선택: `repo`, `read:org`, `read:user`
4. 생성된 토큰 복사

**셸 프로파일에 환경 변수 추가:**

```bash
# ~/.zshrc 또는 ~/.bashrc
export GITHUB_TOKEN="ghp_..."
```

**설정:**

```json
// .vscode/settings.json

{
  "github.copilot.chat.mcp.servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    }
  }
}
```

**Copilot Chat Agent 모드에서 활용:**

```
> 이 저장소의 열려있는 이슈 목록 보여줘
> PR #42의 리뷰 코멘트를 반영해서 현재 파일 수정해줘
> 이슈 #15에 "작업 시작" 코멘트 달아줘
> 현재 변경사항으로 GitHub PR 만들어줘
```

#### Step 3.2 — Notion MCP 활용

**Notion Integration Token 발급:**

> **⚠️ 주의:** Notion 설정에 비슷한 메뉴가 두 개 있다. **Build → Internal integrations**를 사용해야 한다.

1. [notion.so/my-integrations](https://www.notion.so/my-integrations) 접속
2. **+ New integration** → 이름 입력 → Submit
3. **Capabilities**에서 Read content 체크 → **Save changes**
4. **Secrets** → **Copy**로 토큰 복사
5. 접근할 Notion 페이지 우상단 **⋯ → Connections → 생성한 integration 추가**

```json
// .vscode/settings.json

{
  "github.copilot.chat.mcp.servers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer ${env:NOTION_TOKEN}\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

**활용 예시:**

```
> Notion의 API 스펙 문서를 읽고 타입스크립트 인터페이스 작성해줘
> Notion 프로젝트 계획의 요구사항 목록 보여줘
> 오늘 작업 완료한 내용을 Notion 회의록에 추가해줘
```

#### Step 3.3 — PostgreSQL MCP 활용

```json
// .vscode/settings.json

{
  "github.copilot.chat.mcp.servers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${env:DATABASE_URL}"
      }
    }
  }
}
```

**활용 예시:**

```
> 데이터베이스 스키마를 보고 users 테이블의 TypeORM 엔티티 클래스 만들어줘
> orders 테이블과 관련된 API 엔드포인트 코드 작성해줘
```

#### Step 3.4 — `#` 컨텍스트 참조 방식

VS Code Copilot Chat에서는 `#서버명`으로 특정 MCP 서버를 명시적으로 참조할 수 있다:

```
> #github 이 저장소의 최근 커밋 목록 보여줘
> #notion API 스펙 문서 읽고 구현해줘
> #postgres users 테이블 스키마 확인해줘
```

#### Step 3.5 — 주요 MCP 서버 목록

| 서버 이름 | 패키지 | 주요 기능 |
|-----------|--------|-----------|
| GitHub | `@github/github-mcp-server` | 저장소, PR, 이슈 관리 |
| Filesystem | `@modelcontextprotocol/server-filesystem` | 로컬 파일 읽기/쓰기 |
| Notion | `@notionhq/notion-mcp-server` | 페이지, 데이터베이스 |
| Slack | `@modelcontextprotocol/server-slack` | 채널 메시지, DM |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | DB 쿼리 실행 |
| Brave Search | `@modelcontextprotocol/server-brave-search` | 웹 검색 |

### 체크포인트

Copilot Chat Agent 모드에서 `#github` 참조로 실제 이슈 목록이 반환되면 성공이다.

---

## Phase 4: 고급 활용

### 목표

원격 MCP 서버 연결(VS Code 1.101+), Agent 플러그인을 통한 MCP·Skills·Hooks 통합, GitHub MCP Registry 활용 방법을 익힌다.

### 단계별 구현

#### Step 4.1 — 원격 MCP 서버 연결 (VS Code 1.101+)

> **💡 개념 설명: 원격 MCP 서버**
>
> VS Code 1.101부터 HTTP 방식의 원격 MCP 서버를 지원한다. 원격 서버는 OAuth 2.0 인증을 통해 토큰 없이 안전하게 연결할 수 있다. 사내 공유 MCP 서버나 클라우드 서비스를 연결할 때 유용하다.
>
> **핵심 한 줄:** 원격 MCP = URL만 있으면 연결, OAuth 자동 처리

```json
// .vscode/settings.json

{
  "github.copilot.chat.mcp.servers": {
    "notion-remote": {
      "type": "http",
      "url": "https://mcp.notion.com/mcp"
    },
    "company-mcp": {
      "type": "http",
      "url": "https://mcp.company.internal/sse",
      "headers": {
        "Authorization": "Bearer ${env:COMPANY_MCP_TOKEN}"
      }
    }
  }
}
```

처음 연결 시 브라우저가 자동으로 열려 OAuth 인증을 진행한다.

#### Step 4.2 — Agent 플러그인 개발

> **💡 개념 설명: VS Code Agent 플러그인**
>
> VS Code의 Agent 플러그인은 MCP 서버, Skills(슬래시 명령), Hooks(이벤트 핸들러)를 하나의 VS Code 확장으로 패키징한다. 팀 전체가 설치할 수 있는 사내 AI 도구를 만들 때 사용한다.
>
> **핵심 한 줄:** Agent 플러그인 = MCP + Skills + Hooks를 하나의 VS Code 확장으로

기본 Agent 플러그인 구조:

```typescript
// extension.ts (VS Code 확장)

import * as vscode from "vscode";

export function activate(context: vscode.ExtensionContext) {
  // MCP 서버 등록
  const mcpServer = vscode.lm.registerMcpServerDefinitionProvider(
    "company-mcp",
    {
      provideMcpServerDefinitions: async () => [
        {
          label: "Company Internal API",
          command: "node",
          args: [context.asAbsolutePath("dist/mcp-server.js")],
          env: { INTERNAL_KEY: process.env.INTERNAL_KEY ?? "" }
        }
      ]
    }
  );

  // 슬래시 명령 (Skills) 등록
  const skill = vscode.chat.createChatParticipant(
    "company.dev-assistant",
    async (request, context, response, token) => {
      if (request.command === "review") {
        response.markdown("코드 리뷰를 시작합니다...");
        // MCP를 통해 GitHub PR 정보 가져와서 리뷰
      }
    }
  );

  context.subscriptions.push(mcpServer, skill);
}
```

`package.json`에 플러그인 등록:

```json
{
  "contributes": {
    "chatParticipants": [
      {
        "id": "company.dev-assistant",
        "fullName": "Company Dev Assistant",
        "name": "dev",
        "commands": [
          { "name": "review", "description": "PR 코드 리뷰 시작" },
          { "name": "deploy", "description": "배포 상태 확인" }
        ]
      }
    ]
  }
}
```

#### Step 4.3 — GitHub MCP Registry 활용

GitHub는 공식 큐레이션된 MCP 서버 목록인 MCP Registry를 운영한다:

- **웹 탐색:** [github.com/github/github-mcp-server](https://github.com/github/github-mcp-server)
- **VS Code 내:** Command Palette → **GitHub Copilot: Browse MCP Servers**

Registry에서 검증된 서버를 추가하면 한 줄로 설정이 완료된다:

```json
// VS Code Settings에서 Registry 서버 추가 예시

{
  "github.copilot.chat.mcp.servers": {
    "github-official": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    }
  }
}
```

#### Step 4.4 — 전체 .vscode/settings.json 예시

팀 프로젝트에서 사용하는 완전한 설정:

```json
// .vscode/settings.json

{
  // ── GitHub Copilot MCP 설정 ─────────────────────────
  "github.copilot.chat.mcp.servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer ${env:NOTION_TOKEN}\", \"Notion-Version\": \"2022-06-28\"}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "${env:DATABASE_URL}"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "${env:BRAVE_API_KEY}"
      }
    }
  },

  // ── Copilot 일반 설정 ───────────────────────────────
  "github.copilot.enable": {
    "*": true
  },
  "github.copilot.chat.agent.enabled": true
}
```

### 체크포인트

원격 MCP 서버(`type: "http"`)가 설정 파일에 추가되고, VS Code 1.101+에서 정상 연결되면 성공이다.

---

## 도구 간 비교표

| 항목 | VS Code Copilot | Claude Code | Cursor | Antigravity |
|------|-----------------|-------------|--------|-------------|
| 설정 키 | `github.copilot.chat.mcp.servers` | `mcpServers` | `mcpServers` | `mcpServers` |
| 설정 파일 | `.vscode/settings.json` | `.mcp.json` | `.cursor/mcp.json` | `~/.gemini/settings.json` |
| 환경 변수 참조 | `${env:VAR_NAME}` | `${VAR_NAME}` | `${VAR_NAME}` | `${VAR_NAME}` |
| MCP 사용 모드 | Agent 모드만 | 모든 대화 | Agent 모드 권장 | Agent Manager |
| 원격 MCP | 1.101+ (HTTP) | 기본 지원 (HTTP) | SSE 지원 | 제한적 |
| Agent 플러그인 | VS Code 확장으로 패키징 | Skills (별도) | Rules + MCP | 설정 파일 |
| 서버 탐색 | GitHub MCP Registry | 수동 탐색 | 1,800개+ 디렉토리 | 수동 탐색 |
| 상태 확인 | Command Palette → MCP | `/mcp` | Settings UI | `/mcp list` |

---

## 마무리

### 이 가이드에서 배운 것

- VS Code Copilot MCP는 Agent 모드에서만 동작하며, `github.copilot.chat.mcp.servers` 키로 설정
- VS Code 환경 변수 참조 형식 `${env:변수명}`의 특수성
- GitHub, Notion, PostgreSQL 등 실제 서비스 연동 방법
- VS Code 1.101+의 원격 MCP 서버(HTTP/OAuth) 연결 방법
- Agent 플러그인으로 MCP + Skills + Hooks를 통합 패키지화하는 방법

### 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| MCP 도구가 응답 안 함 | Ask/Edit 모드 사용 중 | **Agent** 모드로 전환 |
| `${env:GITHUB_TOKEN}` 미치환 | 환경 변수 미설정 | 셸 프로파일에 `export GITHUB_TOKEN=...` 추가 후 VS Code 재시작 |
| Command Palette에서 서버가 안 보임 | settings.json 문법 오류 | JSON 유효성 검사기로 확인 |
| MCP 서버 시작 실패 | npx 실행 오류 | Terminal에서 직접 `npx -y @github/github-mcp-server` 실행해 오류 확인 |
| 원격 MCP 연결 실패 | VS Code 버전 부족 | VS Code 1.101+ 버전으로 업데이트 |
| Agent 모드에서 MCP 도구 미사용 | Agent 모드 비활성화 | `"github.copilot.chat.agent.enabled": true` 설정 확인 |

### 다음 단계

- [GitHub MCP Registry](https://github.com/github/github-mcp-server) 탐색
- [VS Code MCP 공식 문서](https://code.visualstudio.com/docs/copilot/chat/mcp-servers)로 최신 기능 파악
- `04_skill/` 가이드에서 VS Code Agent 플러그인의 Skills 작성 방법 학습
- Agent 플러그인 개발로 팀 전체가 사용할 수 있는 사내 AI 도구 구성
