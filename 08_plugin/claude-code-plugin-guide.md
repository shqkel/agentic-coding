# Claude Code Plugin 학습 가이드

## 개요

Claude Code의 Plugin 시스템은 **커스텀 기능을 하나의 패키지로 묶어 배포하고 설치할 수 있는 확장 메커니즘**이다. 개인 작업 디렉토리(`.claude/`)에 흩어진 설정을 버전 관리되는 배포 단위로 만들 수 있으며, 팀 전체가 동일한 워크플로우를 일관되게 사용하게 한다.

---

## 1. Plugin이란 무엇인가

### 한 줄 정의

> 플러그인 = 커스텀 명령어 + 스킬 + 에이전트 + MCP 서버 + 훅 + LSP 서버가 **하나의 패키지로 묶인 확장 단위**이다.

이것은 단순한 파일 묶음이 아니다. 각 구성 요소는 서로 다른 역할을 담당하며, 하나의 플러그인 안에서 유기적으로 협력한다. 스마트폰의 앱에 비유하면, 앱 하나가 UI(명령어), 자동화(훅), 외부 서비스 연결(MCP), AI 서브기능(에이전트) 등을 함께 제공하는 것과 같다.

### 플러그인이 해결하는 문제

개발자가 Claude Code를 사용하다 보면 자연스럽게 `.claude/commands/`에 커스텀 명령어를 만들게 된다. 처음에는 괜찮지만 시간이 지나면 문제가 생긴다.

- 팀원에게 공유할 방법이 없다
- 버전 관리를 어떻게 할지 모르겠다
- 명령어 이름이 충돌한다
- 프로젝트마다 같은 설정을 반복 복사한다

플러그인은 이 혼란을 해결한다. 흩어진 설정을 **버전 관리되는 배포 가능한 패키지**로 만들어준다.

---

## 2. Plugin의 구조

플러그인은 정해진 디렉토리 구조를 따른다. 이 구조를 이해하는 것이 플러그인 학습의 첫 번째 단계이다.

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        ← 유일한 필수 파일 (메타데이터 + 경로 선언)
├── commands/              ← 커스텀 명령어 (.md 파일들)
├── agents/                ← 서브에이전트 정의 (.md 파일들)
├── skills/                ← 스킬 디렉토리 (각각 SKILL.md 포함)
├── hooks/
│   └── hooks.json         ← 훅 설정
├── .mcp.json              ← MCP 서버 설정
└── .lsp.json              ← LSP 서버 설정
```

### plugin.json (메니페스트)

플러그인의 신원증명서이다. `name` 필드만 필수이며, 나머지는 선택이다.

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "무엇을 하는 플러그인인지 설명",
  "author": { "name": "작성자 이름" },
  "commands": "./commands/",
  "agents": "./agents/",
  "skills": "./skills/",
  "hooks": "./hooks/hooks.json",
  "mcpServers": "./.mcp.json",
  "lspServers": "./.lsp.json"
}
```

**중요**: 컴포넌트 디렉토리(commands, agents, skills 등)는 `.claude-plugin/` 안이 아닌 **플러그인 루트에** 위치해야 한다. 이 실수를 가장 자주 한다.

---

## 3. Plugin의 6가지 구성 요소 상세 설명

### 3-1. 커스텀 명령어 (Commands / Slash Commands)

**한 줄 설명**: 사용자가 직접 입력해서 실행하는 `/명령어` 형태의 워크플로우이다.

**어떻게 동작하는가**

명령어는 단순한 Markdown 파일이다. 파일 내용이 곧 Claude에게 전달되는 프롬프트가 된다. 사용자가 `/my-plugin:deploy`를 입력하면, Claude는 `commands/deploy.md`의 내용을 읽고 그 지시에 따라 작업을 수행한다.

```markdown
<!-- commands/deploy.md -->
---
description: 스테이징 환경에 배포한다
---

# 배포 명령어

현재 브랜치의 변경사항을 스테이징 환경에 배포한다.

1. `git status`로 커밋되지 않은 변경사항이 없는지 확인한다
2. 테스트 스위트를 실행한다: `npm test`
3. 빌드를 생성한다: `npm run build`
4. 배포 스크립트를 실행한다: `./scripts/deploy-staging.sh`
```

**네임스페이싱**: 플러그인 이름으로 자동 격리된다. `my-plugin`의 `deploy` 명령어는 `/my-plugin:deploy`로 호출되므로 다른 플러그인과 이름이 충돌하지 않는다.

**언제 쓰는가**: 특정 워크플로우를 명시적으로 실행하고 싶을 때. "내가 원할 때 실행한다"는 의도가 있다.

---

### 3-2. 스킬 (Skill)

**한 줄 설명**: 사용자가 명시적으로 호출하지 않아도 **맥락에 따라 자동으로 활성화되는 전문 지식 블록**이다.

**Commands와의 핵심 차이**

| 구분 | Commands | Skills |
|------|----------|--------|
| 실행 방식 | 사용자가 `/명령어` 직접 입력 | Claude가 맥락 파악 후 자동 활성화 |
| 비유 | 앱의 메뉴 버튼 | 직원의 전문 지식 (물어보지 않아도 앎) |
| 용도 | 명시적 워크플로우 실행 | "이런 상황에선 이렇게 해라" 규칙 내재화 |

**어떻게 동작하는가**

스킬은 `SKILL.md` 파일을 가진 디렉토리이다. `SKILL.md`의 YAML frontmatter에 트리거 조건을 적는다. Claude는 대화 맥락이 이 조건과 일치하면 해당 스킬을 자동으로 읽어서 적용한다.

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

**컨텍스트 격리**: Commands와 달리 스킬은 내용이 메인 대화 컨텍스트로 직접 로드된다. 이 때문에 스킬은 간결하게 작성해야 한다(권장: 150줄 이내).

**언제 쓰는가**: "이런 상황에서는 항상 이렇게 해달라"는 규칙을 내재화하고 싶을 때.

---

### 3-3. 에이전트 (Agent / Subagent)

**한 줄 설명**: **독립된 컨텍스트 창을 가지는 전문화된 AI 인스턴스**이다. 메인 Claude가 서브태스크를 위임하는 대상이다.

**왜 필요한가**

복잡한 작업을 하나의 Claude 인스턴스가 모두 처리하면 컨텍스트가 오염되고 토큰이 낭비된다. 에이전트는 서브태스크를 **격리된 환경**에서 처리한 뒤 결과만 메인으로 돌려준다.

```markdown
<!-- agents/security-auditor.md -->
---
name: security-auditor
description: 코드의 보안 취약점을 전문적으로 분석하는 에이전트이다.
  "보안 검토", "취약점 분석", "security audit" 시 호출한다.
tools:
  - Read
  - Grep
  - Bash
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
| 실행 | 자동 활성화 | 메인 Claude가 명시적 위임 |
| 토큰 사용 | 메인 컨텍스트에 영향 | 격리되어 메인에 영향 없음 |
| 용도 | 규칙 내재화 | 병렬 처리, 복잡한 서브태스크 |

**언제 쓰는가**: 메인 작업과 별개로 전문화된 분석이나 생성 작업이 필요할 때. 예를 들어 코드 작성 중에 보안 감사, 문서 생성, 테스트 작성을 병렬로 처리하는 경우이다.

---

### 3-4. MCP 서버 (Model Context Protocol Server)

**한 줄 설명**: Claude가 **외부 시스템(API, DB, 서비스)과 실시간으로 연결되는 통로**이다.

**왜 필요한가**

Claude는 기본적으로 파일 시스템과 터미널만 접근할 수 있다. GitHub, Jira, Slack, PostgreSQL 같은 외부 서비스에 접근하려면 MCP 서버가 필요하다.

**어떻게 동작하는가**

MCP 서버는 별도의 프로세스로 실행되는 Node.js/Python 프로그램이다. Claude와 표준화된 프로토콜(MCP)로 통신하며, 외부 서비스의 기능을 Claude가 호출할 수 있는 "도구(tool)"로 노출한다.

```json
// .mcp.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

이 설정이 적용되면 Claude는 GitHub 이슈를 직접 읽고 쓰며, PostgreSQL 쿼리를 실행할 수 있게 된다. 플러그인에 MCP 서버를 포함시키면, 설치 한 번으로 팀 전체가 이 연결을 사용할 수 있다.

**언제 쓰는가**: 외부 서비스와의 연동이 필요할 때. "Jira 티켓을 읽어서 구현하고 PR을 만드는" 워크플로우처럼, 여러 외부 시스템을 한 플러그인으로 묶어 배포할 때 강력하다.

---

### 3-5. 훅 (Hooks)

**한 줄 설명**: Claude의 **특정 행동(파일 쓰기, 명령 실행 등)에 자동으로 반응하는 이벤트 핸들러**이다.

**왜 필요한가**

Claude가 코드를 수정할 때마다 자동으로 포매터를 실행하거나, 민감한 파일을 수정하려 할 때 경고를 띄우거나, 모든 bash 명령을 로그로 남기고 싶을 때 훅을 사용한다.

**훅의 종류**

| 훅 유형 | 실행 시점 | 특징 |
|---------|-----------|------|
| `PreToolUse` | Claude가 도구를 쓰기 직전 | `block: true`로 실행 차단 가능 |
| `PostToolUse` | Claude가 도구를 쓴 직후 | 결과에 반응하는 추가 작업 |
| `Notification` | Claude가 알림을 보낼 때 | 커스텀 알림 처리 |
| `Stop` | Claude가 작업을 완료할 때 | 완료 후 정리 작업 |

```json
// hooks/hooks.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $CLAUDE_TOOL_RESULT_FILE",
            "timeout": 30
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
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/security-check.sh"
          }
        ]
      }
    ]
  }
}
```

**실용 예시**: `security-guidance` 플러그인은 Claude가 `.env`나 `config/secrets.yml` 같은 민감한 파일을 수정하려 할 때 자동으로 보안 모범 사례 경고를 띄운다. 이 모든 것이 PreToolUse 훅 하나로 구현된다.

**언제 쓰는가**: "특정 행동이 일어날 때마다 자동으로 무언가를 실행하고 싶다"는 요구가 있을 때.

---

### 3-6. LSP 서버 (Language Server Protocol Server)

**한 줄 설명**: Claude가 **IDE처럼 코드를 이해하게 해주는 실시간 언어 분석 서버**이다.

**왜 필요한가**

LSP 없이 Claude는 코드를 "텍스트"로만 읽는다. LSP가 연결되면 Claude는 "코드"로 읽는다. 타입 에러, 함수 정의로 이동, 모든 참조 찾기 같은 IDE 기능을 Claude도 사용할 수 있게 된다.

**어떻게 동작하는가**

LSP는 VS Code가 코드 인텔리전스를 제공하기 위해 사용하는 것과 동일한 프로토콜이다. TypeScript 코드를 작성할 때 VS Code가 타입 에러를 즉시 표시하는 것처럼, LSP 플러그인이 설치되면 Claude도 코드를 수정한 즉시 타입 에러를 파악할 수 있다.

```json
// .lsp.json
{
  "typescript": {
    "command": "typescript-language-server",
    "args": ["--stdio"],
    "extensionToLanguage": {
      ".ts": "typescript",
      ".tsx": "typescriptreact"
    }
  },
  "python": {
    "command": "pyright-langserver",
    "args": ["--stdio"]
  }
}
```

**공식 마켓플레이스의 LSP 플러그인 예시**

| 언어 | 플러그인 이름 | 필요한 바이너리 |
|------|-------------|----------------|
| TypeScript | `typescript-lsp` | `typescript-language-server` |
| Python | `pyright-lsp` | `pyright-langserver` |
| Rust | `rust-analyzer-lsp` | `rust-analyzer` |
| Go | `gopls-lsp` | `gopls` |

**언제 쓰는가**: 코드 정확도가 중요한 프로젝트에서. LSP를 사용하면 Claude가 grep 기반 검색보다 훨씬 정밀하게 코드를 탐색하고 수정할 수 있다.

---

## 4. Marketplace 개념

### Marketplace란 무엇인가

마켓플레이스는 **플러그인 카탈로그**이다. 앱스토어에 비유하면 정확하다. 앱스토어를 추가하면 그 스토어에 어떤 앱이 있는지 볼 수 있지만, 아직 아무것도 설치된 것은 아니다. 설치는 원하는 플러그인을 직접 골라서 한다.

```
마켓플레이스 추가 (앱스토어 등록)  →  플러그인 설치 (앱 다운로드)
/plugin marketplace add 주소         /plugin install 플러그인명@마켓명
```

### 공식 마켓플레이스

Anthropic이 직접 관리하는 `claude-plugins-official` 마켓플레이스는 Claude Code를 시작하면 자동으로 사용 가능하다. 별도 등록 없이 바로 설치할 수 있다.

공식 마켓플레이스는 내부 플러그인(`/plugins`)과 서드파티 파트너 플러그인(`/external_plugins`) 두 디렉토리로 구성된다.

### 마켓플레이스 추가 방법

```bash
# GitHub 저장소 형식
/plugin marketplace add anthropics/claude-code

# Git URL 형식 (GitLab, Bitbucket 등)
/plugin marketplace add https://gitlab.com/org/plugins.git

# 로컬 경로 형식 (개발 중인 마켓플레이스 테스트 시)
/plugin marketplace add /path/to/my-marketplace

# 단축 명령어
/plugin market add anthropics/claude-code
```

### 마켓플레이스 구조

마켓플레이스 자체도 Git 저장소이며, `.claude-plugin/marketplace.json` 파일에 포함된 플러그인 목록이 정의된다.

```json
// .claude-plugin/marketplace.json
{
  "name": "my-team-marketplace",
  "owner": { "name": "My Team" },
  "plugins": [
    {
      "name": "deploy-tools",
      "description": "배포 관련 도구 모음",
      "source": {
        "source": "github",
        "repo": "myteam/deploy-tools-plugin"
      }
    },
    {
      "name": "code-quality",
      "description": "코드 품질 자동화 도구",
      "source": {
        "source": "github",
        "repo": "myteam/code-quality-plugin"
      }
    }
  ]
}
```

### 플러그인 설치 및 관리

```bash
# 설치 가능한 플러그인 목록 확인
/plugin list

# 특정 마켓플레이스에서 설치
/plugin install typescript-lsp@claude-plugins-official

# 설치된 플러그인 확인
/plugin status

# 플러그인 업데이트
/plugin update my-plugin@my-market

# 플러그인 제거
/plugin uninstall my-plugin@my-market
# 또는 단축형
/plugin rm my-plugin@my-market
```

### 설치 범위 (Scope)

플러그인은 두 가지 범위로 설치할 수 있다.

| 범위 | 위치 | 의미 |
|------|------|------|
| 글로벌 | `~/.claude/` | 모든 프로젝트에서 사용 가능 |
| 프로젝트 | `./.claude/` | 현재 프로젝트에서만 사용 |

---

## 5. 전체 구성 요소 한눈에 보기

```
Plugin (확장 단위)
│
├── Commands (슬래시 명령어)     → 사용자가 명시적으로 호출
│   └── 예: /deploy, /review, /commit
│
├── Skills (스킬)                → Claude가 맥락 파악 후 자동 활성화
│   └── 예: 코드 리뷰 요청 시 자동으로 리뷰 기준 적용
│
├── Agents (서브에이전트)         → 독립 컨텍스트에서 서브태스크 처리
│   └── 예: 보안 감사, 테스트 생성, 문서화 전담
│
├── MCP Servers (외부 연결)      → 외부 서비스 API 연동
│   └── 예: GitHub, Jira, PostgreSQL, Slack 연결
│
├── Hooks (이벤트 핸들러)         → Claude 행동에 자동 반응
│   └── 예: 파일 수정 후 자동 포매팅, 명령 실행 전 보안 검사
│
└── LSP Servers (코드 인텔리전스) → IDE 수준의 코드 이해
    └── 예: 타입 에러 실시간 감지, 정의로 이동, 참조 찾기
```

---

## 6. 빠른 시작

```bash
# 1. 공식 마켓플레이스에서 TypeScript LSP 설치
/plugin install typescript-lsp@claude-plugins-official

# 2. GitHub 연동 플러그인 설치 (공식 마켓플레이스)
/plugin install github@claude-plugins-official

# 3. 데모 마켓플레이스 추가 (Anthropic 공식 예제)
/plugin marketplace add anthropics/claude-code

# 4. 설치된 플러그인 상태 확인
/plugin status

# 5. 에러 확인 (설치 문제 발생 시)
/plugin Errors
```

---

## 7. 자주 하는 실수

**실수 1**: 컴포넌트를 `.claude-plugin/` 안에 넣는다
```
❌ my-plugin/.claude-plugin/commands/deploy.md
✅ my-plugin/commands/deploy.md
```

**실수 2**: 마켓플레이스 추가와 플러그인 설치를 혼동한다. 마켓플레이스 추가는 카탈로그 등록이며, 이후 별도로 설치 명령을 실행해야 한다.

**실수 3**: LSP 플러그인을 설치했지만 바이너리가 없다. LSP 플러그인은 시스템에 언어 서버 바이너리(`typescript-language-server`, `pyright-langserver` 등)가 설치되어 있어야 동작한다.

---

## 참고 자료

- 공식 플러그인 레퍼런스: https://code.claude.com/docs/en/plugins-reference
- 공식 마켓플레이스 GitHub: https://github.com/anthropics/claude-plugins-official
- 플러그인 제출: https://platform.claude.com/plugins/submit
