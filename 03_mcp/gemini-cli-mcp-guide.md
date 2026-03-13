# Gemini CLI MCP 서버 연동 가이드

## 학습 목표

MCP(Model Context Protocol)가 무엇인지 이해하고, GitHub 등 외부 서비스를 Gemini CLI에 연결하여 대화 중 실제 데이터를 조회하고 작업을 수행할 수 있다.

## 사전 준비

- Gemini CLI 설치 및 인증 완료 (`01_agentic_coding_tool/01_cli_driven/gemini-cli-guide.md` 참고)
- Node.js 18 이상 (`node --version`으로 확인)
- GitHub Personal Access Token (GitHub MCP 실습 시 필요)

---

## 전체 흐름 한눈에 보기

Gemini CLI는 기본적으로 파일 조작, 셸 실행 등 로컬 작업만 할 수 있다. MCP 서버를 추가하면 GitHub 이슈 조회, 슬랙 메시지 전송, 데이터베이스 쿼리 등 외부 시스템과 직접 상호작용할 수 있게 된다.

1. **MCP 개념 이해** — 프로토콜 구조와 동작 방식 파악
2. **GitHub MCP 설정** — 첫 번째 MCP 서버 연결
3. **대화에서 활용** — @ 표기법으로 MCP 도구 호출
4. **추가 서버 탐색** — Filesystem, Slack 등 확장

---

## Phase 1: MCP란 무엇인가

### 목표

MCP가 해결하는 문제를 이해하고, Gemini CLI에서 어떤 역할을 하는지 설명할 수 있다.

### 단계별 구현

#### Step 1.1 — MCP의 필요성 파악하기

> **💡 개념 설명: MCP (Model Context Protocol)**
>
> Gemini CLI에서 "오늘 내 GitHub PR 목록 보여줘"라고 하면 어떻게 될까? 기본 상태에서는 대답할 수 없다 — GitHub API 인증 정보도 없고 호출 방법도 모른다.
>
> MCP는 AI 에이전트가 외부 시스템과 **표준화된 방식**으로 통신할 수 있게 해주는 프로토콜이다. Anthropic이 설계하고 여러 회사가 채택한 개방형 표준으로, 각 외부 서비스는 MCP 서버 형태로 자신의 기능을 노출한다.
>
> **핵심 한 줄:** MCP = AI 에이전트가 외부 세계와 통신하는 표준 언어

```
Gemini CLI ←→ MCP 프로토콜 ←→ MCP 서버 ←→ 외부 서비스
                                  GitHub MCP    GitHub API
                                  Slack MCP     Slack API
                                  DB MCP        PostgreSQL
```

#### Step 1.2 — MCP 서버 유형 파악하기

MCP 서버는 크게 세 가지 방식으로 실행된다:

| 유형 | 실행 방식 | 예시 |
|------|-----------|------|
| npx | Node.js 패키지를 즉시 실행 | GitHub, Slack MCP |
| 로컬 실행파일 | 직접 설치한 바이너리 | 커스텀 MCP |
| HTTP/SSE | 원격 서버 연결 | 클라우드 서비스 |

이 가이드에서는 가장 일반적인 **npx 방식**을 사용한다.

### 체크포인트

MCP가 없을 때와 있을 때 Gemini CLI가 할 수 있는 일의 차이를 설명할 수 있는가?

---

## Phase 2: GitHub MCP 서버 설정하기

### 목표

`~/.gemini/settings.json`에 GitHub MCP 서버를 설정하고 Gemini CLI에서 인식되는지 확인한다.

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

#### Step 2.2 — settings.json에 MCP 서버 등록

> **💡 개념 설명: settings.json 구조**
>
> `~/.gemini/settings.json`은 Gemini CLI의 전역 설정 파일이다. `mcpServers` 키 하위에 각 MCP 서버를 이름과 함께 등록한다. `command`와 `args`는 해당 서버를 시작하는 셸 명령과 동일하다.
>
> **핵심 한 줄:** settings.json의 mcpServers = "이 도구들도 쓸 수 있어"라고 Gemini에게 알리는 목록

```json
# ~/.gemini/settings.json

{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "ghp_여기에_토큰_붙여넣기"
      }
    }
  }
}
```

> **⚠️ 주의:** 토큰을 파일에 직접 작성하면 보안 위험이 있다. 팀 환경에서는 환경 변수 참조 방식을 사용한다:
>
> ```json
> "GITHUB_TOKEN": "${GITHUB_TOKEN}"
> ```
>
> 그리고 셸 프로파일(`.bashrc`, `.zshrc`)에 `export GITHUB_TOKEN=ghp_...` 추가

#### Step 2.3 — MCP 서버 연결 확인

Gemini CLI를 재시작하고 대화형 세션에서 확인한다:

```bash
gemini
```

```bash
/mcp list
```

정상 설정 시 출력:

```
● github (connected)
  - create_issue
  - get_pull_request
  - list_repos
  ... (여러 도구 목록)
```

### 체크포인트

`/mcp list` 실행 시 `github (connected)` 상태와 도구 목록이 표시되면 성공이다.

---

## Phase 3: MCP 서버 활용하기

### 목표

Gemini CLI 대화에서 GitHub MCP 도구를 실제로 사용하여 데이터를 조회하고 작업을 수행한다.

### 단계별 구현

#### Step 3.1 — 자연어로 MCP 도구 호출

MCP 서버가 연결되면 특별한 명령어 없이 자연어로 요청한다. Gemini가 적절한 MCP 도구를 자동으로 선택한다:

```bash
> 내 GitHub에서 열려있는 PR 목록 보여줘
> 저장소 my-user/my-project의 최근 이슈 5개 나열해줘
> 이슈 #42에 "검토 완료" 라고 코멘트 달아줘
```

#### Step 3.2 — @ 표기법으로 명시적 지정

특정 MCP 서버의 도구를 명시적으로 지정하려면 `@서버명` 표기법을 사용한다:

```bash
> @github 오늘 머지된 PR 요약해줘
> @github 이슈 #15 닫아줘
```

#### Step 3.3 — /mcp 관리 명령어

| 명령어 | 설명 |
|--------|------|
| `/mcp list` | 연결된 서버와 도구 목록 |
| `/mcp show` | 도구 설명 표시 |
| `/mcp hide` | 도구 설명 숨기기 (컨텍스트 절약) |
| `/mcp schema` | 도구의 JSON 스키마 상세 확인 |

### 체크포인트

"내 GitHub 저장소 목록 보여줘" 요청에 실제 저장소 목록이 반환되면 성공이다.

---

## Phase 4: 추가 MCP 서버 확장

### 목표

여러 MCP 서버를 동시에 설정하고, 용도에 따라 적합한 서버를 선택할 수 있다.

### 단계별 구현

#### Step 4.1 — Filesystem MCP 추가

로컬 파일시스템을 안전하게 제어하는 MCP 서버이다. 허용할 디렉토리를 명시적으로 지정한다:

```json
# ~/.gemini/settings.json (일부 발췌)

{
  "mcpServers": {
    "github": { ... },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/사용자명/Documents",
        "/Users/사용자명/Projects"
      ]
    }
  }
}
```


#### Step 4.2 — Notion MCP 추가

Notion 페이지·데이터베이스를 AI에서 직접 읽고 편집할 수 있다.

**Notion Integration Token 발급:**

> **⚠️ 주의:** Notion 설정에 비슷한 메뉴가 두 개 있다. **Build → Internal integrations**를 사용해야 한다. Listings → Integrations(마켓플레이스)는 MCP 연결 시 사용하지 않는다.

1. [notion.so/my-integrations](https://www.notion.so/my-integrations) 접속 (좌측 사이드바 **Build → Internal integrations**)
2. **+ New integration** 클릭 → 이름·워크스페이스 입력 → Submit
3. **Capabilities**에서 Read / Update / Insert content 체크 → **Save changes**
4. **Secrets** 섹션 → **Show** → **Copy**로 토큰 복사 (`ntn_...` 또는 `secret_...` 형태)
5. 접근할 Notion 페이지 우상단 **⋯ → Connections → 방금 만든 integration 추가**

> **💡 팁:** 최상위 페이지에 연결하면 모든 하위 페이지도 자동으로 접근 가능하다.

```json
# ~/.gemini/settings.json (일부 발췌)

{
  "mcpServers": {
    "github": { ... },
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer ntn_YOUR_TOKEN_HERE\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

> **⚠️ 주의:** 토큰을 파일에 직접 작성하면 보안 위험이 있다. 셸 프로파일에 `export NOTION_TOKEN=ntn_...`을 추가하고 설정 파일에서는 `${NOTION_TOKEN}`으로 참조한다.

설정 후 Gemini CLI에서 자연어로 요청한다:

```bash
> Notion에서 프로젝트 계획 페이지 읽어줘
> Notion에 오늘 회의록 페이지 만들어줘
> Notion 데이터베이스에서 진행 중인 작업 보여줘
```

#### Step 4.3 — 주요 MCP 서버 목록

| 서버 이름 | 패키지 | 주요 기능 |
|-----------|--------|-----------|
| GitHub | `@github/github-mcp-server` | 저장소, PR, 이슈 관리 |
| Filesystem | `@modelcontextprotocol/server-filesystem` | 로컬 파일 읽기/쓰기 |
| Notion | `@notionhq/notion-mcp-server` | 페이지·데이터베이스 읽기/쓰기 |
| Slack | `@slack/mcp-server` | 채널 메시지, DM |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | DB 쿼리 실행 |
| Brave Search | `@modelcontextprotocol/server-brave-search` | 웹 검색 |

> **💡 개념 설명: MCP 서버 탐색하기**
>
> 공개 MCP 서버는 [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) 에서 전체 목록을 확인할 수 있다. 커뮤니티가 만든 서버까지 포함하면 수백 개 이상이 존재한다.
>
> **핵심 한 줄:** 연결하고 싶은 서비스가 있다면 "{서비스명} MCP server"로 검색해보자.

#### Step 4.4 — 프로젝트별 MCP 설정

팀 프로젝트에서는 프로젝트 전용 MCP 설정을 사용한다:

```json
# <project-root>/.gemini/settings.json

{
  "mcpServers": {
    "project-db": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_URL": "postgresql://localhost:5432/mydb"
      }
    }
  }
}
```

이 파일을 Git에 커밋하면 팀원 모두가 동일한 MCP 환경을 공유한다 (토큰은 개인 환경 변수로 관리).

### 체크포인트

`/mcp list`에서 두 개 이상의 서버가 `connected` 상태로 표시되면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- MCP 프로토콜의 역할: AI와 외부 서비스를 연결하는 표준 인터페이스
- `~/.gemini/settings.json`의 `mcpServers`에 서버를 등록하는 방법
- 자연어와 `@서버명` 표기법으로 MCP 도구 호출하기
- 전역/프로젝트 범위의 MCP 설정 분리

### 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| `/mcp list`에서 서버가 안 보임 | settings.json 문법 오류 | JSON 유효성 검사기로 확인 |
| `(disconnected)` 상태 | 토큰 오류 또는 npx 실행 실패 | 터미널에서 직접 npx 명령을 실행해 오류 확인 |
| 도구 호출 실패 | 권한 부족 | GitHub 토큰 권한 범위 재확인 |

