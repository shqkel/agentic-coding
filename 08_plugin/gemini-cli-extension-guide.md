# Gemini CLI Extension 학습 가이드

## 개요

Gemini CLI의 Extension 시스템은 **프롬프트, MCP 서버, 커스텀 명령어, 테마, 훅, 서브에이전트, 정책을 하나의 패키지로 묶어 배포하고 설치할 수 있는 확장 메커니즘**이다. Claude Code의 Plugin에 대응하는 개념으로, 개인 설정 파일에 흩어진 구성을 버전 관리되는 배포 단위로 만들 수 있으며, 팀 전체가 동일한 워크플로우를 일관되게 사용하게 한다.

---

## 1. Extension이란 무엇인가

### 한 줄 정의

> Extension = MCP 서버 + 커스텀 명령어 + 에이전트 스킬 + 훅 + 서브에이전트 + 정책 + 테마 + 컨텍스트 파일이 **하나의 패키지로 묶인 확장 단위**이다.

이것은 단순한 파일 묶음이 아니다. 각 구성 요소는 서로 다른 역할을 담당하며, 하나의 Extension 안에서 유기적으로 협력한다. 스마트폰의 앱에 비유하면, 앱 하나가 UI(명령어), 자동화(훅), 외부 서비스 연결(MCP), AI 서브기능(에이전트), 보안 규칙(정책), 외관(테마) 등을 함께 제공하는 것과 같다.

### Extension이 해결하는 문제

개발자가 Gemini CLI를 사용하다 보면 자연스럽게 커스텀 명령어, GEMINI.md 규칙, MCP 서버 설정을 만들게 된다. 처음에는 괜찮지만 시간이 지나면 문제가 생긴다.

- 팀원에게 공유할 방법이 없다
- 버전 관리를 어떻게 할지 모르겠다
- 명령어 이름이 충돌한다
- 프로젝트마다 같은 설정을 반복 복사한다
- MCP 서버 설정과 관련 정책을 따로 관리해야 한다

Extension은 이 혼란을 해결한다. 흩어진 설정을 **버전 관리되는 배포 가능한 패키지**로 만들어주며, 500개 이상의 공식 Extension Gallery를 통해 검증된 확장을 즉시 설치할 수 있다.

---

## 2. Extension의 구조

Extension은 정해진 디렉토리 구조를 따른다. 이 구조를 이해하는 것이 Extension 학습의 첫 번째 단계이다.

```
my-extension/
├── gemini-extension.json    ← 유일한 필수 파일 (매니페스트)
├── commands/                ← 커스텀 명령어 (TOML 파일들)
├── skills/                  ← 에이전트 스킬 (각각 SKILL.md 포함)
├── hooks/
│   └── hooks.json           ← 훅 설정
├── agents/                  ← 서브에이전트 정의
├── policies/                ← 정책 규칙 (TOML 파일들)
├── themes/                  ← 테마 정의
└── GEMINI.md                ← 컨텍스트 파일 (Extension 전용 규칙)
```

### gemini-extension.json (매니페스트)

Extension의 신원증명서이다. 이 파일만이 유일한 필수 파일이며, Extension의 이름, 버전, 포함된 구성 요소를 선언한다.

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "description": "무엇을 하는 Extension인지 설명",
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["./dist/server.js"],
      "env": {
        "API_KEY": "${settings.apiKey}"
      }
    }
  },
  "contextFileName": "GEMINI.md",
  "excludeTools": ["dangerousTool"],
  "settings": {
    "apiKey": {
      "type": "string",
      "description": "API 키를 입력한다",
      "sensitive": true
    }
  },
  "themes": {
    "dark-pro": "./themes/dark-pro.json"
  },
  "plan": "pro",
  "migratedTo": null
}
```

**매니페스트의 주요 필드**

| 필드 | 설명 | 필수 여부 |
|------|------|-----------|
| `name` | Extension 고유 이름 | 필수 |
| `version` | 시맨틱 버전 | 권장 |
| `description` | Extension 설명 | 권장 |
| `mcpServers` | MCP 서버 설정 | 선택 |
| `contextFileName` | 컨텍스트 파일 경로 (기본: GEMINI.md) | 선택 |
| `excludeTools` | 차단할 도구 목록 | 선택 |
| `settings` | 사용자 설정 스키마 (API 키 등) | 선택 |
| `themes` | 테마 파일 경로 | 선택 |
| `plan` | 필요한 Gemini 플랜 (free, pro 등) | 선택 |
| `migratedTo` | 다른 Extension으로 이전된 경우의 대상 | 선택 |

---

## 3. Extension의 8가지 구성 요소 상세 설명

### 3-1. 커스텀 명령어 (Commands)

**한 줄 설명**: 사용자가 직접 입력해서 실행하는 `/명령어` 형태의 워크플로우이다.

**어떻게 동작하는가**

명령어는 TOML 파일로 정의된다. Claude Code의 Markdown 기반 명령어와 달리, Gemini CLI는 TOML 형식을 사용하여 명령어의 메타데이터와 실행 로직을 구조적으로 분리한다.

```toml
# commands/deploy.toml
[command]
name = "deploy"
description = "스테이징 환경에 배포한다"

[command.prompt]
text = """
현재 브랜치의 변경사항을 스테이징 환경에 배포한다.

1. `git status`로 커밋되지 않은 변경사항이 없는지 확인한다
2. 테스트 스위트를 실행한다: `npm test`
3. 빌드를 생성한다: `npm run build`
4. 배포 스크립트를 실행한다: `./scripts/deploy-staging.sh`
"""
```

```toml
# commands/review.toml
[command]
name = "review"
description = "현재 변경사항에 대한 코드 리뷰를 수행한다"

[command.args]
file = { type = "string", description = "리뷰할 파일 경로", required = false }

[command.prompt]
text = """
다음 기준으로 코드를 리뷰한다:
- 버그 가능성
- 성능 이슈
- 보안 취약점
{{#if args.file}}
대상 파일: {{args.file}}
{{/if}}
"""
```

**네임스페이싱**: Extension 이름으로 자동 격리된다. `my-extension`의 `deploy` 명령어는 `/my-extension:deploy`로 호출되므로 다른 Extension과 이름이 충돌하지 않는다.

**언제 쓰는가**: 특정 워크플로우를 명시적으로 실행하고 싶을 때. "내가 원할 때 실행한다"는 의도가 있다.

---

### 3-2. 에이전트 스킬 (Skills)

**한 줄 설명**: 사용자가 명시적으로 호출하지 않아도 **맥락에 따라 자동으로 활성화되는 전문 지식 블록**이다.

**어떻게 동작하는가**

스킬은 `SKILL.md` 파일을 가진 디렉토리이다. `SKILL.md`의 YAML frontmatter에 트리거 조건을 적는다. Gemini는 대화 맥락이 이 조건과 일치하면 해당 스킬을 자동으로 읽어서 적용한다.

```markdown
<!-- skills/code-reviewer/SKILL.md -->
---
name: code-reviewer
description: |
  PR 코드 리뷰를 수행한다.
  "코드 리뷰해줘", "이 코드 어때", "PR 검토" 등의
  표현이 나오면 자동 활성화한다.
---

# 코드 리뷰 전문가

이 스킬이 활성화되면 다음 기준으로 코드를 검토한다:

- 버그 가능성
- 성능 이슈
- 보안 취약점
- 가독성 및 유지보수성

리뷰 결과는 우선순위별로 분류하여 제시한다.
```

**Commands와의 차이**

| 구분 | Commands | Skills |
|------|----------|--------|
| 실행 방식 | 사용자가 `/명령어` 직접 입력 | Gemini가 맥락 파악 후 자동 활성화 |
| 비유 | 앱의 메뉴 버튼 | 직원의 전문 지식 (물어보지 않아도 앎) |
| 용도 | 명시적 워크플로우 실행 | "이런 상황에선 이렇게 해라" 규칙 내재화 |

**언제 쓰는가**: "이런 상황에서는 항상 이렇게 해달라"는 규칙을 내재화하고 싶을 때.

---

### 3-3. MCP 서버 (Model Context Protocol Server)

**한 줄 설명**: Gemini가 **외부 시스템(API, DB, 서비스)과 실시간으로 연결되는 통로**이다.

**왜 필요한가**

Gemini는 기본적으로 파일 시스템과 터미널만 접근할 수 있다. GitHub, Jira, Slack, PostgreSQL 같은 외부 서비스에 접근하려면 MCP 서버가 필요하다.

**어떻게 동작하는가**

MCP 서버는 매니페스트(`gemini-extension.json`)의 `mcpServers` 필드에서 설정한다. `settings`를 통해 사용자별 API 키 등을 안전하게 주입할 수 있다.

```json
{
  "name": "github-integration",
  "version": "1.0.0",
  "description": "GitHub 연동 Extension",
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${settings.githubToken}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${settings.dbUrl}"]
    }
  },
  "settings": {
    "githubToken": {
      "type": "string",
      "description": "GitHub Personal Access Token",
      "sensitive": true
    },
    "dbUrl": {
      "type": "string",
      "description": "PostgreSQL 연결 URL",
      "sensitive": true
    }
  }
}
```

`settings`의 `sensitive: true` 플래그를 사용하면 API 키와 같은 민감한 값이 로그에 노출되지 않는다. 사용자는 Extension 설치 후 설정 UI를 통해 자신의 키를 안전하게 입력한다.

**언제 쓰는가**: 외부 서비스와의 연동이 필요할 때. 여러 외부 시스템을 하나의 Extension으로 묶어 배포할 때 강력하다.

---

### 3-4. 훅 (Hooks)

**한 줄 설명**: Gemini의 **특정 행동(파일 쓰기, 명령 실행 등)에 자동으로 반응하는 이벤트 핸들러**이다.

**왜 필요한가**

Gemini가 코드를 수정할 때마다 자동으로 포매터를 실행하거나, 민감한 파일을 수정하려 할 때 경고를 띄우거나, 모든 bash 명령을 로그로 남기고 싶을 때 훅을 사용한다.

**어떻게 동작하는가**

훅은 `hooks/hooks.json`에 정의한다. Claude Code의 훅과 유사한 구조를 따른다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "WriteFile|EditFile",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $GEMINI_TOOL_RESULT_FILE",
            "timeout": 30
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "ExecuteCommand",
        "hooks": [
          {
            "type": "command",
            "command": "${GEMINI_EXTENSION_ROOT}/hooks/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

**실용 예시**: `prettier-on-save` Extension은 Gemini가 파일을 수정할 때마다 자동으로 Prettier를 실행하여 코드 스타일을 일관되게 유지한다. 이것이 PostToolUse 훅 하나로 구현된다.

**언제 쓰는가**: "특정 행동이 일어날 때마다 자동으로 무언가를 실행하고 싶다"는 요구가 있을 때.

---

### 3-5. 서브에이전트 (Agents / Subagents)

**한 줄 설명**: **독립된 컨텍스트 창을 가지는 전문화된 AI 인스턴스**이다. 메인 Gemini가 서브태스크를 위임하는 대상이다.

**왜 필요한가**

복잡한 작업을 하나의 Gemini 인스턴스가 모두 처리하면 컨텍스트가 오염되고 토큰이 낭비된다. 에이전트는 서브태스크를 **격리된 환경**에서 처리한 뒤 결과만 메인으로 돌려준다.

```markdown
<!-- agents/security-auditor.md -->
---
name: security-auditor
description: 코드의 보안 취약점을 전문적으로 분석하는 에이전트이다.
  "보안 검토", "취약점 분석", "security audit" 시 호출한다.
tools:
  - ReadFile
  - Search
  - ExecuteCommand
---

# 보안 감사 에이전트

나는 보안 전문가 역할을 수행하는 서브에이전트이다.

주어진 코드에서 다음을 검사한다:
- SQL 인젝션
- XSS 취약점
- 하드코딩된 시크릿
- 안전하지 않은 의존성

발견된 취약점은 CVSS 점수와 함께 심각도 순으로 정리한다.
```

**Skills와의 차이**

| 구분 | Skills | Agents |
|------|--------|--------|
| 컨텍스트 | 메인 대화에 로드 | 독립된 컨텍스트 창 |
| 실행 | 자동 활성화 | 메인 Gemini가 명시적 위임 |
| 토큰 사용 | 메인 컨텍스트에 영향 | 격리되어 메인에 영향 없음 |
| 용도 | 규칙 내재화 | 병렬 처리, 복잡한 서브태스크 |

**언제 쓰는가**: 메인 작업과 별개로 전문화된 분석이나 생성 작업이 필요할 때.

---

### 3-6. 정책 규칙 (Policies)

**한 줄 설명**: Gemini의 **행동 경계를 선언적으로 정의하는 규칙 시스템**이다. Claude Code에는 없는 Gemini CLI 고유 기능이다.

**왜 필요한가**

훅이 "이벤트 발생 시 스크립트를 실행"하는 절차적 접근이라면, 정책은 "이런 행동은 허용/금지한다"고 선언적으로 명시하는 접근이다. 조직의 보안 규칙이나 코딩 표준을 Gemini에게 강제할 수 있다.

```toml
# policies/security.toml
[policy]
name = "no-production-access"
description = "프로덕션 데이터베이스에 대한 직접 접근을 금지한다"

[[policy.rules]]
action = "deny"
tool = "ExecuteCommand"
condition = "command contains 'prod' and command contains 'psql'"
message = "프로덕션 DB 직접 접근은 금지되어 있습니다. 스테이징 환경을 사용하세요."

[[policy.rules]]
action = "deny"
tool = "WriteFile"
condition = "path matches '*.env.production'"
message = "프로덕션 환경 변수 파일은 수동으로 수정할 수 없습니다."
```

```toml
# policies/code-standards.toml
[policy]
name = "code-standards"
description = "팀 코딩 표준을 강제한다"

[[policy.rules]]
action = "warn"
tool = "WriteFile"
condition = "path matches '*.ts' and content contains 'any'"
message = "TypeScript에서 'any' 타입 사용은 지양합니다. 구체적인 타입을 명시하세요."
```

**훅과의 차이**

| 구분 | Hooks | Policies |
|------|-------|----------|
| 접근 방식 | 절차적 (스크립트 실행) | 선언적 (규칙 명시) |
| 실행 | 외부 프로세스 호출 | Gemini 내부에서 평가 |
| 용도 | 자동화 작업 | 행동 제한/경고 |
| 성능 | 프로세스 생성 비용 | 거의 비용 없음 |

**언제 쓰는가**: 조직의 보안 규칙이나 코딩 표준을 Gemini에게 강제하고 싶을 때. 훅보다 가볍고 선언적이다.

---

### 3-7. 테마 (Themes)

**한 줄 설명**: Gemini CLI의 **시각적 외관을 커스터마이징하는 설정**이다. Claude Code에는 없는 Gemini CLI 고유 기능이다.

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "themes": {
    "ocean-dark": "./themes/ocean-dark.json",
    "forest-light": "./themes/forest-light.json"
  }
}
```

Extension에 테마를 포함시키면, 기능적 확장과 시각적 커스터마이징을 하나의 패키지로 배포할 수 있다. 예를 들어 팀 전용 Extension에 팀 브랜드 색상의 테마를 포함시키는 것이다.

**언제 쓰는가**: 팀 브랜딩이나 개인 취향에 맞는 시각적 환경을 패키지에 포함시키고 싶을 때.

---

### 3-8. 컨텍스트 파일 (GEMINI.md)

**한 줄 설명**: Extension이 활성화될 때 Gemini에게 **자동으로 주입되는 지시 파일**이다.

```markdown
<!-- GEMINI.md -->
# My Extension 컨텍스트

이 Extension이 활성화되면 다음 규칙을 따른다:

- 모든 API 호출은 에러 핸들링을 포함해야 한다
- 로그 메시지는 구조화된 JSON 형식으로 작성한다
- 테스트 파일은 반드시 `*.test.ts` 네이밍 컨벤션을 따른다
```

매니페스트의 `contextFileName` 필드로 파일명을 변경할 수 있다. 지정하지 않으면 기본값인 `GEMINI.md`를 사용한다.

**언제 쓰는가**: Extension이 활성화될 때 Gemini가 항상 인지해야 하는 규칙이나 맥락이 있을 때.

---

## 4. Extension Gallery (마켓플레이스)

### Gallery란 무엇인가

Extension Gallery는 **Extension 카탈로그**이다. 앱스토어에 비유하면 정확하다. 브라우저에서 Gallery를 탐색하고, CLI에서 설치 명령을 실행하면 된다.

**공식 Gallery 주소**: https://geminicli.com/extensions/

### Gallery 규모와 생태계

Gallery에는 **500개 이상의 Extension**이 등록되어 있다. Google이 직접 관리하는 공식 Extension뿐 아니라 파트너사와 커뮤니티의 기여도 포함된다.

| 구분 | 제공사 예시 |
|------|-------------|
| Google 공식 | Google Cloud, Firebase, Android 관련 Extension |
| 파트너사 | Shopify, Figma, Postman, Snyk, Elastic, MongoDB 등 |
| 커뮤니티 | 개인 개발자가 GitHub에 공개한 Extension |

### Gallery에서 설치하기

```bash
# Gallery에서 Extension 설치 (GitHub URL 기반)
gemini extensions install https://github.com/user/my-extension

# Gallery 브라우저에서 원하는 Extension을 찾고,
# 표시된 설치 명령어를 복사하여 실행하면 된다
```

---

## 5. 설치 및 관리

### 기본 설치

```bash
# GitHub URL로 설치
gemini extensions install https://github.com/user/my-extension

# 로컬 경로로 설치 (개발 중인 Extension 테스트)
gemini extensions install /path/to/my-extension
```

### Extension 생성

```bash
# 새 Extension 프로젝트 생성 (템플릿 기반)
gemini extensions new my-extension mcp-server
gemini extensions new my-extension custom-commands

# 사용 가능한 템플릿 타입:
# - mcp-server: MCP 서버 기반 Extension
# - custom-commands: 커스텀 명령어 기반 Extension
```

### 목록 확인 및 상태 관리

```bash
# 설치된 Extension 목록 확인
gemini extensions list

# 세션 내에서 활성 Extension 확인
/extensions list

# Extension 활성화/비활성화
gemini extensions enable my-extension
gemini extensions disable my-extension

# 특정 스코프에서만 비활성화
gemini extensions disable my-extension --scope=workspace
```

### 개발 및 빌드

```bash
# 로컬 개발용 링크 (심볼릭 링크 방식)
gemini extensions link .

# TypeScript로 작성한 Extension 빌드
gemini extensions build
```

`link` 명령어는 개발 중인 Extension 디렉토리를 Gemini가 인식하는 위치에 심볼릭 링크로 연결한다. 소스 코드를 수정하면 즉시 반영되므로 개발-테스트 사이클이 빨라진다.

---

## 6. 스코프 계층

Extension은 세 가지 범위(스코프)로 존재하며, 우선순위가 있다.

```
Workspace (프로젝트 레벨)          ← 최우선
    ↓
User (글로벌 레벨)                 ← 차순위
    ↓
Gallery (마켓플레이스 기본값)       ← 최하위
```

| 스코프 | 위치 | 의미 |
|--------|------|------|
| Workspace | `<project>/.gemini/extensions/` | 현재 프로젝트에서만 적용 |
| User (글로벌) | `~/.gemini/extensions/` | 모든 프로젝트에서 적용 |
| Gallery | 공식 마켓플레이스 | 설치된 Extension의 기본 설정 |

**우선순위 규칙**: 같은 이름의 Extension이 여러 스코프에 존재하면, Workspace > User > Gallery 순으로 적용된다. 이를 통해 프로젝트별로 Extension의 동작을 오버라이드할 수 있다.

**예시**: 팀 전체가 `code-formatter` Extension을 User 스코프로 설치했더라도, 특정 프로젝트에서는 Workspace 스코프에 다른 설정의 `code-formatter`를 두어 프로젝트 고유의 포매팅 규칙을 적용할 수 있다.

---

## 7. Extension 만들기 (빠른 시작)

### 단계 1: 프로젝트 생성

```bash
# MCP 서버 기반 Extension 생성
gemini extensions new weather-info mcp-server
cd weather-info
```

이 명령은 다음 구조를 생성한다:

```
weather-info/
├── gemini-extension.json
├── src/
│   └── server.ts
├── package.json
└── tsconfig.json
```

### 단계 2: 매니페스트 설정

```json
{
  "name": "weather-info",
  "version": "0.1.0",
  "description": "현재 날씨 정보를 제공하는 Extension",
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["./dist/server.js"],
      "env": {
        "WEATHER_API_KEY": "${settings.weatherApiKey}"
      }
    }
  },
  "settings": {
    "weatherApiKey": {
      "type": "string",
      "description": "OpenWeatherMap API 키",
      "sensitive": true
    }
  }
}
```

### 단계 3: MCP 서버 구현

```typescript
// src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "weather",
  version: "0.1.0"
});

server.tool(
  "get-weather",
  "지정한 도시의 현재 날씨를 조회한다",
  { city: z.string().describe("도시 이름 (영문)") },
  async ({ city }) => {
    const apiKey = process.env.WEATHER_API_KEY;
    const res = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${apiKey}&units=metric&lang=kr`
    );
    const data = await res.json();
    return {
      content: [{
        type: "text",
        text: `${data.name}: ${data.weather[0].description}, ${data.main.temp}°C`
      }]
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### 단계 4: 빌드 및 로컬 테스트

```bash
# TypeScript 빌드
gemini extensions build

# 로컬에서 테스트
gemini extensions link .

# Gemini CLI를 시작하고 Extension 동작 확인
gemini
> 서울의 현재 날씨를 알려줘
```

### 단계 5: 배포

```bash
# GitHub에 Push
git init
git add .
git commit -m "Initial release of weather-info extension"
git remote add origin https://github.com/user/weather-info-extension.git
git push -u origin main

# 다른 사용자가 설치할 때
gemini extensions install https://github.com/user/weather-info-extension
```

---

## 8. Claude Code Plugin과의 비교

### 구조 비교

| 구분 | Claude Code (Plugin) | Gemini CLI (Extension) |
|------|---------------------|----------------------|
| 매니페스트 | `.claude-plugin/plugin.json` | `gemini-extension.json` |
| 매니페스트 위치 | 서브디렉토리 안 | 프로젝트 루트 |
| 명령어 형식 | Markdown (.md) | TOML (.toml) |
| 스킬 형식 | SKILL.md | SKILL.md (동일) |
| 에이전트 형식 | Markdown (.md) | Markdown (.md) (동일) |
| MCP 설정 위치 | `.mcp.json` (별도 파일) | `gemini-extension.json` 내부 |
| 훅 위치 | `hooks/hooks.json` | `hooks/hooks.json` (동일) |

### 배포 비교

| 구분 | Claude Code | Gemini CLI |
|------|-------------|-----------|
| 마켓플레이스 | Marketplace (Git 기반) | Extension Gallery (500+ 등록) |
| 설치 명령 | `/plugin install name@market` | `gemini extensions install <URL>` |
| 마켓플레이스 추가 | `/plugin marketplace add` | Gallery는 단일 (추가 불필요) |

### 구성 요소 비교

| 구성 요소 | Claude Code | Gemini CLI |
|-----------|-------------|-----------|
| 커스텀 명령어 | O | O |
| 에이전트 스킬 | O | O |
| 서브에이전트 | O | O |
| MCP 서버 | O | O |
| 훅 | O | O |
| LSP 서버 | O | X |
| 정책 (Policies) | X | O |
| 테마 (Themes) | X | O |
| Settings UI | X | O |

**핵심 차이점 요약**:
- Claude Code는 **LSP 서버**를 지원하여 IDE 수준의 코드 인텔리전스를 제공한다
- Gemini CLI는 **정책(Policies)**, **테마(Themes)**, **Settings UI**를 지원하여 선언적 행동 제어와 시각적 커스터마이징이 가능하다
- Gemini CLI의 `settings`는 민감 정보를 안전하게 관리하는 UI를 제공하며, 이는 Claude Code에 없는 고유 기능이다

---

## 9. 보안 고려사항

### Google은 서드파티 Extension을 검증하지 않는다

Extension Gallery에 등록된 Extension 중 Google 공식이 아닌 것은 Google의 보안 검증을 거치지 않는다. 따라서 설치 전에 소스 코드를 직접 확인하는 것이 권장된다.

```bash
# 설치 전에 GitHub 저장소를 먼저 확인한다
# 1. gemini-extension.json의 mcpServers에서 실행되는 프로세스 확인
# 2. hooks/hooks.json에서 실행되는 명령어 확인
# 3. excludeTools가 불필요하게 광범위하지 않은지 확인
```

### settings의 sensitive 플래그

API 키, 토큰 같은 민감한 값은 반드시 `sensitive: true`로 표시해야 한다.

```json
{
  "settings": {
    "apiKey": {
      "type": "string",
      "description": "서비스 API 키",
      "sensitive": true
    },
    "region": {
      "type": "string",
      "description": "서비스 리전",
      "sensitive": false
    }
  }
}
```

`sensitive: true`인 설정값은 로그 출력, 디버그 정보에서 마스킹 처리되어 실수로 노출되는 것을 방지한다.

### excludeTools로 도구 차단

Extension이 특정 도구를 사용하지 못하도록 제한할 수 있다. 이는 Extension의 권한 범위를 최소화하는 보안 원칙이다.

```json
{
  "name": "read-only-analyzer",
  "excludeTools": ["WriteFile", "EditFile", "ExecuteCommand"],
  "description": "코드를 분석만 하고 수정하지 않는 Extension"
}
```

### 보안 체크리스트

Extension을 설치하기 전에 다음을 확인한다:

1. **소스 코드 확인**: GitHub 저장소의 코드를 직접 읽는다
2. **매니페스트 점검**: `mcpServers`에서 실행되는 프로세스가 신뢰할 수 있는지 확인한다
3. **훅 점검**: `hooks.json`에서 실행되는 외부 명령이 안전한지 확인한다
4. **정책 점검**: `policies/`의 규칙이 합리적인지 확인한다
5. **Settings 점검**: `sensitive: true`가 적절히 사용되었는지 확인한다
6. **커뮤니티 평판**: GitHub 스타, 이슈, 최근 업데이트 빈도를 확인한다

---

## 10. 자주 하는 실수

**실수 1**: 매니페스트 파일명을 잘못 쓴다

```
# Gemini CLI
gemini-extension.json     (O)
.gemini-extension.json    (X) ← 점(.) 붙이지 않는다
extension.json            (X) ← 정확한 이름을 사용한다

# Claude Code와 비교
# Claude: .claude-plugin/plugin.json (서브디렉토리 안)
# Gemini: gemini-extension.json (프로젝트 루트에 직접)
```

**실수 2**: 커스텀 명령어를 Markdown으로 작성한다

```
# Claude Code 방식 (Markdown)
commands/deploy.md        (Claude Code에서는 O)

# Gemini CLI 방식 (TOML)
commands/deploy.toml      (Gemini CLI에서는 O)
commands/deploy.md        (Gemini CLI에서는 X)
```

Claude Code에서 Gemini CLI로 이전할 때 가장 많이 하는 실수이다. Gemini CLI의 명령어는 TOML 형식이다.

**실수 3**: MCP 서버 설정을 별도 파일에 넣는다

```json
// Claude Code 방식: .mcp.json (별도 파일)
// Gemini CLI 방식: gemini-extension.json 안의 mcpServers 필드

// 잘못된 방식
// my-extension/.mcp.json ← Gemini에서는 이 파일을 읽지 않는다

// 올바른 방식
// my-extension/gemini-extension.json 내의 "mcpServers" 필드에 설정
```

**실수 4**: `gemini extensions link` 없이 로컬 Extension을 테스트한다

```bash
# 개발 중인 Extension을 테스트하려면 반드시 link를 먼저 실행한다
cd my-extension/
gemini extensions link .

# link 없이 Gemini를 시작하면 개발 중인 Extension이 인식되지 않는다
```

**실수 5**: 스코프를 고려하지 않고 Extension을 비활성화한다

```bash
# 전역으로 비활성화 (모든 프로젝트에서 비활성화됨)
gemini extensions disable my-extension

# 특정 프로젝트에서만 비활성화 (다른 프로젝트에서는 계속 활성화)
gemini extensions disable my-extension --scope=workspace
```

전역 비활성화는 모든 프로젝트에 영향을 미친다. 특정 프로젝트에서만 비활성화하려면 반드시 `--scope=workspace`를 사용한다.

**실수 6**: TypeScript Extension을 빌드하지 않고 배포한다

```bash
# TypeScript로 작성한 경우 반드시 빌드 후 배포한다
gemini extensions build
git add dist/
git commit -m "Build extension"
git push
```

`gemini extensions build`를 실행하지 않으면 TypeScript 소스가 JavaScript로 변환되지 않아 설치한 사용자 측에서 Extension이 동작하지 않는다.

---

## 11. 전체 구성 요소 한눈에 보기

```
Extension (확장 단위)
│
├── Commands (커스텀 명령어)       → 사용자가 명시적으로 호출 (TOML)
│   └── 예: /deploy, /review, /format
│
├── Skills (에이전트 스킬)          → Gemini가 맥락 파악 후 자동 활성화
│   └── 예: 코드 리뷰 요청 시 자동으로 리뷰 기준 적용
│
├── Agents (서브에이전트)           → 독립 컨텍스트에서 서브태스크 처리
│   └── 예: 보안 감사, 테스트 생성, 문서화 전담
│
├── MCP Servers (외부 연결)        → 외부 서비스 API 연동
│   └── 예: GitHub, Jira, PostgreSQL, Slack 연결
│
├── Hooks (이벤트 핸들러)           → Gemini 행동에 자동 반응
│   └── 예: 파일 수정 후 자동 포매팅, 명령 실행 전 보안 검사
│
├── Policies (정책 규칙)            → 선언적 행동 제한/경고 [Gemini 고유]
│   └── 예: 프로덕션 DB 접근 금지, any 타입 사용 경고
│
├── Themes (테마)                  → 시각적 외관 커스터마이징 [Gemini 고유]
│   └── 예: 팀 브랜드 색상, 다크/라이트 테마
│
└── GEMINI.md (컨텍스트 파일)       → Extension 활성화 시 자동 주입 규칙
    └── 예: 팀 코딩 규칙, API 사용 가이드라인
```

---

## 참고 자료

- Extension Gallery: https://geminicli.com/extensions/
- Gemini CLI GitHub: https://github.com/google-gemini/gemini-cli
- MCP 프로토콜 명세: https://modelcontextprotocol.io/
