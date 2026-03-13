# VS Code GitHub Copilot Plugin 학습 가이드

## 개요

VS Code GitHub Copilot의 확장 시스템은 **세 가지 레이어로 구성된 다층 확장 체계**이다. Agent Plugin(Preview), Copilot Extensions(일반 확장), VS Code Extension API(프로그래밍 확장)가 각각 다른 수준에서 Copilot의 기능을 확장한다. 이 가이드에서는 각 레이어의 역할과 사용법을 체계적으로 다룬다.

---

## 1. VS Code Copilot의 확장 체계

### 세 가지 확장 레이어

Copilot을 확장하는 방법은 세 가지이다. 각각의 성격이 다르므로 혼동하지 않는 것이 중요하다.

| 레이어 | 성격 | 비유 | 상태 |
|--------|------|------|------|
| **Agent Plugin** | 통합 패키지 (번들) | 스마트폰의 앱 패키지 | Preview (VS Code 1.110+) |
| **Copilot Extensions** | 개별 확장 (VS Code 확장) | 앱스토어의 개별 앱 | GA (정식) |
| **VS Code Extension API** | 프로그래밍 확장 | SDK로 직접 개발 | GA (정식) |

**핵심 포인트**: Claude Code의 Plugin에 가장 가까운 개념은 **Agent Plugin**이다. 커스텀 명령어, 스킬, 에이전트, 훅, MCP 서버를 하나의 패키지로 묶어 배포한다는 점이 동일하다. Copilot Extensions는 이와 별개로, VS Code Marketplace를 통해 설치하는 개별 확장이다.

```
VS Code Copilot 확장 체계
│
├── Agent Plugin (Preview)           → 통합 패키지, Git 저장소 기반 배포
│   ├── Slash Commands
│   ├── Agent Skills
│   ├── Custom Agents
│   ├── Hooks
│   └── MCP Servers
│
├── Copilot Extensions (GA)          → VS Code Marketplace 기반 개별 확장
│   ├── Language Model Tool          → Agent Mode용
│   ├── Chat Participant API         → Ask Mode용
│   ├── Language Model API           → 직접 AI 통합
│   └── MCP Tool                     → 크로스 환경
│
└── VS Code Extension API            → 프로그래밍으로 직접 확장
```

---

## 2. Agent Plugin (Preview, 2026년 2월~)

### Agent Plugin이란 무엇인가

> Agent Plugin = Slash Commands + Agent Skills + Custom Agents + Hooks + MCP Servers가 **하나의 패키지로 묶인 확장 단위**이다.

Claude Code의 Plugin과 가장 유사한 개념이다. 여러 구성 요소를 하나의 Git 저장소에 묶어 배포하고, 사용자는 설치 한 번으로 모든 기능을 사용할 수 있다. 2026년 2월부터 Preview로 제공되기 시작했으며, VS Code 1.110 이상이 필요하다.

### Agent Plugin이 해결하는 문제

Copilot을 사용하다 보면 MCP 서버 설정, 커스텀 명령어, 특정 워크플로우를 팀원에게 공유하고 싶어진다. 이전에는 이를 위해 각각을 개별적으로 설정해야 했다.

- MCP 서버는 `.vscode/mcp.json`에 직접 추가
- 커스텀 명령어는 각자 만들어서 사용
- 워크플로우 규칙은 문서로 전달

Agent Plugin은 이 모든 것을 **하나의 패키지로 묶어 Git 저장소 기반 마켓플레이스에서 배포**한다. 팀원은 설치 한 번으로 동일한 환경을 구성할 수 있다.

### 5가지 구성 요소

#### 2-1. Slash Commands

**한 줄 설명**: 사용자가 `/command-name` 형태로 직접 호출하는 추가 명령어이다.

Claude Code의 Commands와 동일한 개념이다. 채팅 창에서 `/`를 입력하면 설치된 플러그인의 명령어가 자동 완성 목록에 나타난다. 명령어를 선택하면 미리 정의된 프롬프트와 워크플로우가 실행된다.

**사용 예시**:

```
/deploy-staging     → 스테이징 환경에 배포하는 워크플로우 실행
/review-security    → 보안 관점에서 현재 파일을 리뷰
/generate-tests     → 현재 파일에 대한 테스트 코드 자동 생성
```

**특징**: Agent Plugin의 Slash Commands는 플러그인 이름으로 네임스페이싱되므로, 서로 다른 플러그인 간 명령어 이름이 충돌하지 않는다.

**언제 쓰는가**: 특정 워크플로우를 명시적으로, 반복적으로 실행하고 싶을 때.

---

#### 2-2. Agent Skills

**한 줄 설명**: 에이전트 워크플로우 내에서 **맥락에 따라 자동으로 활성화되는 전문 기능 블록**이다.

Claude Code의 Skills와 대응된다. 사용자가 명시적으로 호출하지 않아도 대화의 맥락이 특정 조건과 일치하면 자동으로 활성화된다.

| 구분 | Slash Commands | Agent Skills |
|------|---------------|--------------|
| 실행 방식 | 사용자가 `/명령어` 직접 입력 | Copilot이 맥락 파악 후 자동 활성화 |
| 비유 | 앱의 메뉴 버튼 | 전문가의 내재된 지식 |
| 용도 | 명시적 워크플로우 실행 | "이런 상황에서는 이렇게 해라" 규칙 |

**언제 쓰는가**: "코드 리뷰 요청 시 항상 특정 기준으로 검토해라", "배포 관련 질문이 나오면 우리 팀의 배포 규칙을 적용해라"와 같은 자동 규칙이 필요할 때.

---

#### 2-3. Custom Agents

**한 줄 설명**: 특정 도메인에 전문화된 **AI 페르소나**이다.

Claude Code의 Subagents와 대응된다. 보안 전문가, 아키텍처 리뷰어, 데이터베이스 전문가 등 특화된 역할을 수행하는 별도의 AI 인스턴스를 정의한다. 메인 Copilot이 서브태스크를 Custom Agent에게 위임하여 처리한다.

**왜 필요한가**: 하나의 AI가 모든 도메인을 동일한 수준으로 다루기 어렵다. Custom Agent를 사용하면 특정 영역에 대한 전문 시스템 프롬프트, 도구 접근 권한, 행동 규칙을 분리할 수 있다.

**언제 쓰는가**: 보안 감사, 성능 분석, 문서화 등 전문적이고 반복적인 서브태스크가 있을 때.

---

#### 2-4. Hooks

**한 줄 설명**: Copilot의 **특정 이벤트에 자동으로 반응하는 핸들러**이다.

Claude Code의 Hooks와 동일한 개념이다. 차이점은 이벤트 유형의 이름이 다르다는 것이다.

| Copilot Hook 유형 | 실행 시점 | Claude Code 대응 |
|-------------------|-----------|-------------------|
| `PreCommand` | 명령이 실행되기 직전 | `PreToolUse` |
| `PostEdit` | 파일이 수정된 직후 | `PostToolUse` (Write/Edit 매칭) |
| `PostToolUse` | 도구 사용 직후 | `PostToolUse` |

**실용 예시**:

- **PostEdit**: 파일이 수정될 때마다 자동으로 Prettier/ESLint를 실행한다
- **PreCommand**: 위험한 명령(예: `rm -rf`, `git push --force`)을 실행하기 전에 경고를 띄운다
- **PostToolUse**: 터미널 명령 실행 후 로그를 남긴다

**언제 쓰는가**: "코드가 수정되면 자동으로 포매팅", "위험한 명령 전에 확인" 같은 자동 반응이 필요할 때.

---

#### 2-5. MCP Servers

**한 줄 설명**: Copilot이 **외부 시스템과 연결되는 표준화된 통로**이다.

Claude Code의 MCP 서버와 동일한 역할이다. GitHub, Jira, 데이터베이스, 내부 API 등 외부 서비스의 기능을 Copilot이 호출할 수 있는 도구(tool)로 노출한다. Agent Plugin에 MCP 서버를 포함시키면, 설치 한 번으로 팀 전체가 동일한 외부 연동 환경을 구성한다.

**언제 쓰는가**: 외부 서비스와의 연동이 필요할 때. 예를 들어 "Jira 티켓을 읽고, 코드를 수정하고, PR을 만드는" 워크플로우를 하나의 플러그인으로 배포할 때.

---

### Agent Plugin 관리 방법

#### 설치된 플러그인 확인

Extensions 사이드바에서 `@agentPlugins`로 검색하면 설치된 Agent Plugin 목록을 볼 수 있다. 또는 **Agent Plugins - Installed** 뷰에서 직접 관리한다.

#### 마켓플레이스 등록

Agent Plugin은 Git 저장소 기반 마켓플레이스에서 배포된다. 커스텀 마켓플레이스를 등록하여 팀 전용 플러그인을 배포할 수 있다.

```
1. VS Code 설정(Settings)에서 "Agent Plugin Marketplace" 검색
2. 커스텀 마켓플레이스의 Git 저장소 URL 추가
3. Extensions 사이드바에서 @agentPlugins로 검색하여 설치
```

#### 로컬 플러그인 수동 등록

개발 중인 플러그인을 테스트할 때 로컬 경로로 직접 등록할 수 있다. Git에 올리기 전 로컬에서 동작을 확인하는 용도로 사용한다.

---

## 3. Copilot Extensions (일반 확장)

Agent Plugin이 "통합 패키지"라면, Copilot Extensions는 **VS Code Marketplace를 통해 설치하는 개별 확장**이다. 이미 정식(GA) 상태이며, 네 가지 통합 방법을 제공한다.

### 확장 통합 방법 4가지

#### A. Language Model Tool (Agent Mode용)

**한 줄 설명**: Agent Mode에서 Copilot이 **자율적으로 호출하는 도메인 도구**이다.

**어떻게 동작하는가**

VS Code 확장 호스트에서 실행되므로 VS Code API 전체에 접근할 수 있다. 확장이 제공하는 도구를 Agent Mode의 Copilot이 자율적으로 판단하여 호출한다. 예를 들어 데이터베이스 쿼리 도구를 등록하면, Copilot이 "이 데이터를 조회해야 한다"고 판단할 때 자동으로 호출한다.

**핵심 특징**:
- VS Code 확장 호스트에서 실행 → VS Code API 전체 접근 가능
- Agent Mode 내에서 자율적으로 동작
- 사용자가 명시적으로 호출하지 않아도 Copilot이 판단하여 사용

**언제 쓰는가**: Agent Mode에서 도메인 특화 기능(DB 쿼리, API 호출, 파일 변환 등)을 추가하고 싶을 때.

---

#### B. Chat Participant API (Ask Mode용)

**한 줄 설명**: `@participant-name`으로 호출하는 **전문 어시스턴트**이다.

**어떻게 동작하는가**

Chat Participant는 채팅 창에서 `@이름`으로 호출하는 전문화된 대화 상대이다. 일반적인 Copilot 대화와 달리, 대화 흐름 전체를 참여자(Participant)가 제어한다.

**빌트인 Chat Participants**:

| 참여자 | 역할 | 사용 예시 |
|--------|------|-----------|
| `@workspace` | 워크스페이스 전체 맥락으로 질문 | `@workspace 이 프로젝트의 인증 방식은?` |
| `@terminal` | 터미널 관련 질문 | `@terminal 마지막 에러를 어떻게 해결해?` |
| `@vscode` | VS Code 설정/기능 질문 | `@vscode 자동 저장을 활성화하려면?` |
| `@github` | GitHub 관련 질문 | `@github 최근 이슈 목록을 보여줘` |

**커스텀 Chat Participant 만들기**: VS Code Extension API를 사용하여 새로운 `@참여자`를 만들 수 있다. 데이터베이스 전문가(`@database`), CI/CD 관리자(`@cicd`) 같은 도메인 전문 어시스턴트를 구현할 수 있다.

**중요**: Chat Participant는 **Ask Mode에서만 동작**한다. Agent Mode에서는 사용할 수 없으므로, Agent Mode에서 확장 기능이 필요하면 Language Model Tool을 사용해야 한다.

**언제 쓰는가**: 특정 도메인에 대한 전문화된 대화 인터페이스가 필요할 때.

---

#### C. Language Model API (직접 AI 통합)

**한 줄 설명**: 프로그래밍으로 **AI 모델에 직접 접근하는 API**이다.

**어떻게 동작하는가**

채팅 인터페이스를 거치지 않고, VS Code 확장 코드에서 직접 Copilot의 AI 모델을 호출한다. Code Actions, Hover Providers, 커스텀 뷰 등 VS Code의 어떤 기능에서든 AI를 활용할 수 있다.

**활용 예시**:
- 코드에 마우스를 올렸을 때(Hover Provider) AI가 설명을 생성
- 코드 액션(전구 아이콘)에서 "AI로 리팩터링" 옵션 추가
- 커스텀 사이드바에서 코드 분석 결과 표시

**핵심 특징**:
- 채팅 인터페이스 불필요 → UI 어디서든 AI를 사용
- `vscode.lm.selectChatModels()` API로 모델 선택
- 토큰 스트리밍 지원

**언제 쓰는가**: 채팅이 아닌 VS Code의 다른 UI 요소(에디터, 사이드바, 상태바 등)에 AI 기능을 직접 통합하고 싶을 때.

---

#### D. MCP Tool (크로스 환경)

**한 줄 설명**: VS Code 외부에서도 동작하는 **범용 도구 연결**이다.

**어떻게 동작하는가**

MCP(Model Context Protocol) 서버를 통해 외부 도구와 서비스를 연결한다. VS Code에만 종속되지 않으므로 Claude Code, JetBrains, 터미널 등 MCP를 지원하는 모든 환경에서 재사용할 수 있다.

**VS Code에서 MCP 서버 연결 방법**:

```json
// .vscode/mcp.json
{
  "servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/data"]
    }
  }
}
```

**Language Model Tool과의 차이**:

| 구분 | Language Model Tool | MCP Tool |
|------|-------------------|----------|
| 실행 환경 | VS Code 확장 호스트 | 별도 프로세스 |
| API 접근 | VS Code API 전체 | 없음 (MCP 프로토콜만) |
| 이식성 | VS Code 전용 | 크로스 환경 |
| 배포 | VS Code Marketplace | npm/pip 등 |

**언제 쓰는가**: 여러 AI 도구(VS Code, Claude Code, JetBrains 등)에서 공통으로 사용할 도구를 만들고 싶을 때.

---

## 4. Agent Tools 3가지

Copilot의 Agent Mode에서 사용할 수 있는 도구는 세 가지 출처에서 온다. 이 구분을 이해하면 "어떤 도구를 어떻게 추가하는가"에 대한 답이 명확해진다.

### 4-1. Built-in Tools (기본 내장 도구)

Copilot에 기본으로 포함된 도구이다. 별도 설치 없이 사용할 수 있다.

| 도구 | 기능 |
|------|------|
| 파일 읽기/쓰기 | 워크스페이스의 파일을 읽고 수정 |
| 터미널 | 명령어 실행 |
| 검색 | 워크스페이스 내 코드 검색 |
| 디버깅 | 디버그 세션 제어 |
| VS Code API | 에디터 설정, 확장 관리 등 |

### 4-2. Extension Tools (확장 도구)

Copilot Extensions의 Language Model Tool로 등록된 도구이다. VS Code Marketplace에서 확장을 설치하면 자동으로 Agent Mode에서 사용 가능해진다.

**특징**: VS Code 확장 호스트에서 실행되므로 VS Code API에 직접 접근할 수 있다. 에디터 상태를 읽거나, 알림을 띄우거나, 다른 확장과 상호작용하는 것이 가능하다.

### 4-3. MCP Tools (MCP 도구)

MCP 서버를 통해 연결된 외부 도구이다. `.vscode/mcp.json`이나 사용자 설정에서 MCP 서버를 등록하면 Agent Mode에서 해당 도구를 사용할 수 있다.

**특징**: 별도 프로세스로 실행되며, VS Code에 종속되지 않는다. 다른 AI 도구에서도 동일한 MCP 서버를 재사용할 수 있다.

```
Agent Mode에서 사용 가능한 도구
│
├── Built-in Tools        → 파일 읽기/쓰기, 터미널, 검색, 디버깅
│   └── 설치 불필요, 기본 제공
│
├── Extension Tools       → Language Model Tool로 등록된 확장
│   └── VS Code Marketplace에서 설치
│
└── MCP Tools             → MCP 서버를 통한 외부 도구
    └── .vscode/mcp.json에서 설정
```

---

## 5. Claude Code Plugin과의 비교표

두 도구 모두 "플러그인"이라는 이름으로 확장 시스템을 제공하지만, 세부 구조와 상태가 다르다. 아래 표를 통해 개념을 대응시킬 수 있다.

| 구분 | Claude Code | VS Code Copilot |
|------|-------------|-----------------|
| **통합 패키지** | Plugin (`plugin.json`) | Agent Plugin (Preview) |
| **마켓플레이스** | Plugin Marketplace (Git 기반) | Git 저장소 기반 마켓플레이스 |
| **Commands** | Slash Commands (`.md` 파일) | Slash Commands |
| **Skills** | Skills (`SKILL.md`) | Agent Skills |
| **Agents** | Subagents (`.md` 파일) | Custom Agents |
| **MCP** | `.mcp.json` | `.vscode/mcp.json` |
| **Hooks** | 4종 (PreToolUse, PostToolUse, Notification, Stop) | PreCommand, PostEdit, PostToolUse 등 |
| **LSP** | `.lsp.json`으로 지원 | VS Code 네이티브 (별도 설정 불필요) |
| **고유 기능** | LSP 서버 번들링 | Chat Participants, Language Model API |
| **상태** | GA (정식) | Preview |
| **패키지 형식** | Git 저장소 + `plugin.json` | Git 저장소 기반 |
| **설치 방법** | `/plugin install` CLI 명령 | Extensions 사이드바에서 설치 |

**주목할 차이점**:

1. **LSP 처리 방식**: Claude Code는 LSP 서버를 플러그인에 포함시켜야 하지만, VS Code는 이미 LSP를 네이티브로 지원하므로 별도 설정이 필요 없다.

2. **고유 기능**: VS Code Copilot은 Chat Participant API와 Language Model API라는 독자적인 확장 방식을 제공한다. 이들은 Claude Code에 직접 대응되는 개념이 없다.

3. **성숙도**: Claude Code의 Plugin은 GA(정식) 상태인 반면, VS Code의 Agent Plugin은 아직 Preview 상태이다. 기능이나 API가 변경될 수 있다.

---

## 6. Agent Plugin vs Copilot Extensions 비교

VS Code 생태계 내에서 Agent Plugin과 Copilot Extensions는 같은 목적(Copilot 확장)을 위한 서로 다른 접근 방식이다. 이 둘을 혼동하면 "어디서 설치하고 관리하는가"에서 혼란이 생긴다.

| 구분 | Agent Plugin | Copilot Extensions |
|------|-------------|-------------------|
| **배포 방식** | Git 저장소 기반 마켓플레이스 | VS Code Marketplace |
| **형식** | 번들 패키지 (여러 구성 요소 포함) | 개별 확장 (단일 기능) |
| **관리 위치** | Agent Plugins 뷰 (`@agentPlugins`) | Extensions 뷰 |
| **설치 방법** | Git URL 또는 마켓플레이스에서 설치 | Marketplace에서 검색 후 설치 |
| **개발 언어** | 설정 파일 + 스크립트 (저코드) | TypeScript/JavaScript (VS Code API) |
| **API 접근** | 제한적 (정의된 인터페이스만) | VS Code API 전체 |
| **상태** | Preview | GA (정식) |
| **적합한 상황** | 팀 워크플로우 표준화, 빠른 배포 | 고급 커스터마이제이션, 복잡한 UI |

**선택 기준**:

- **팀 워크플로우를 빠르게 공유**하고 싶다면 → Agent Plugin
- **VS Code API를 사용한 고급 기능**이 필요하다면 → Copilot Extensions
- **여러 AI 도구에서 공통 사용**할 도구라면 → MCP Server (둘 다에서 사용 가능)

---

## 7. 빌트인 Chat Participants 상세

Chat Participants는 VS Code Copilot의 독자적인 기능이다. Claude Code에는 직접 대응되는 개념이 없다. 채팅 창에서 `@이름`으로 호출하여 특정 영역의 전문가에게 질문한다.

### @workspace

**역할**: 워크스페이스 전체를 맥락으로 질문에 답한다.

```
@workspace 이 프로젝트에서 인증을 처리하는 파일은 어디에 있어?
@workspace 이 프로젝트의 아키텍처를 설명해줘
@workspace package.json에 정의된 빌드 스크립트를 설명해줘
```

프로젝트 전체 구조를 이해한 상태에서 답변하므로, 개별 파일이 아닌 프로젝트 수준의 질문에 적합하다.

### @terminal

**역할**: 터미널 관련 질문에 답한다.

```
@terminal 마지막 에러 메시지를 분석해줘
@terminal 이 프로젝트를 빌드하려면 어떤 명령어를 실행해야 해?
@terminal npm 의존성 충돌을 해결하는 방법은?
```

터미널의 현재 상태(출력, 에러 등)를 읽고 이를 기반으로 답변한다.

### @vscode

**역할**: VS Code 자체의 설정과 기능에 대해 답한다.

```
@vscode 자동 저장을 활성화하려면 어떻게 해?
@vscode 탭 크기를 4칸으로 바꾸려면?
@vscode 키바인딩을 수정하는 방법은?
```

VS Code 사용법에 대한 질문에 특화된 참여자이다.

### @github

**역할**: GitHub 관련 정보에 접근하고 질문에 답한다.

```
@github 이 저장소의 최근 이슈를 보여줘
@github PR #42의 변경사항을 요약해줘
@github 이 파일의 최근 커밋 히스토리를 보여줘
```

GitHub API와 연동되어 이슈, PR, 커밋 등의 정보를 직접 조회할 수 있다.

---

## 8. 빠른 시작

### Agent Plugin 설치 예시

```
1. VS Code를 1.110 이상으로 업데이트
2. Extensions 사이드바 열기 (Ctrl+Shift+X)
3. 검색창에 @agentPlugins 입력
4. "Agent Plugins - Installed" 뷰에서 마켓플레이스 설정
5. 원하는 플러그인 검색 후 설치
```

로컬 개발 중인 플러그인을 테스트하려면:

```
1. Settings (Ctrl+,) 열기
2. "Agent Plugin" 검색
3. "Agent Plugin: Local Plugins" 항목에 로컬 경로 추가
4. VS Code 새로고침 (Ctrl+Shift+P → "Developer: Reload Window")
```

### MCP 서버 연결 예시

프로젝트 루트에 `.vscode/mcp.json` 파일을 생성한다.

```json
{
  "servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    }
  }
}
```

사용자 레벨에서 MCP 서버를 등록하려면 VS Code 설정(`settings.json`)에 추가한다.

```json
{
  "github.copilot.chat.mcp.servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/data"]
    }
  }
}
```

MCP 서버가 연결되면 Agent Mode 채팅에서 해당 도구를 자동으로 사용할 수 있다.

### Chat Participant 사용 예시

채팅 창에서 `@`를 입력하면 사용 가능한 참여자 목록이 나타난다.

```
@workspace 이 프로젝트의 전체 구조를 설명해줘
@terminal 마지막 빌드 에러를 분석해줘
@vscode 에디터 폰트 크기를 변경하는 방법은?
@github 현재 브랜치의 PR 상태를 알려줘
```

**주의**: Chat Participant는 Ask Mode에서만 동작한다. Agent Mode에서는 사용할 수 없다.

---

## 9. 보안 고려사항

### 로컬 코드 실행 위험

Agent Plugin과 Copilot Extensions 모두 **로컬 머신에서 코드를 실행할 수 있다**. 특히 다음 구성 요소는 주의가 필요하다.

| 구성 요소 | 위험 수준 | 이유 |
|-----------|-----------|------|
| Hooks | 높음 | 자동으로 스크립트 실행 |
| MCP Servers | 높음 | 별도 프로세스로 시스템 접근 |
| Agent Skills/Commands | 중간 | Copilot을 통해 간접 실행 |
| Chat Participants | 낮음 | 대화 응답에 한정 |

### 설치 전 검토 사항

1. **출처 확인**: 신뢰할 수 있는 저장소/마켓플레이스에서만 설치한다
2. **Hooks 검토**: 설치 전 어떤 훅이 포함되어 있는지 확인한다. 특히 PreCommand 훅은 모든 명령 실행 전에 동작하므로 민감하다
3. **MCP 서버 검토**: 어떤 외부 서비스에 접근하는지, 어떤 환경 변수(토큰 등)를 요구하는지 확인한다
4. **권한 최소화**: MCP 서버에 전달하는 토큰은 필요한 최소한의 권한만 부여한다

### 팀 환경에서의 보안

```
팀 공유 플러그인 보안 체크리스트:

□ 플러그인 저장소의 코드를 리뷰했는가
□ Hooks가 실행하는 스크립트 내용을 확인했는가
□ MCP 서버가 접근하는 외부 서비스 목록을 파악했는가
□ 환경 변수(토큰 등)가 최소 권한으로 설정되었는가
□ 플러그인 업데이트 시 변경사항을 리뷰하는 프로세스가 있는가
```

---

## 10. 자주 하는 실수

**실수 1**: Agent Plugin과 Copilot Extensions를 혼동한다

```
❌ "Copilot Extensions에서 Agent Plugin을 설치한다"
✅ Agent Plugin은 @agentPlugins 뷰에서, Copilot Extensions는 Extensions 뷰에서 관리한다
```

이 둘은 배포 방식, 관리 위치, 형식이 모두 다르다. Agent Plugin은 Git 저장소 기반의 번들 패키지이고, Copilot Extensions는 VS Code Marketplace의 개별 확장이다.

**실수 2**: Agent Mode에서 Chat Participant를 사용하려 한다

```
❌ Agent Mode에서 "@workspace 이 코드를 리팩터링해줘"
✅ Ask Mode로 전환 후 "@workspace 이 코드 구조를 설명해줘"
   Agent Mode에서 리팩터링은 Chat Participant 없이 직접 요청한다
```

Chat Participant(`@workspace`, `@terminal` 등)는 **Ask Mode에서만 동작**한다. Agent Mode에서는 Copilot이 Built-in Tools, Extension Tools, MCP Tools를 자율적으로 사용하므로 Chat Participant가 필요하지 않다.

**실수 3**: MCP 서버 설정 파일 위치를 잘못 지정한다

```
❌ 프로젝트 루트에 .mcp.json (Claude Code 방식)
✅ .vscode/mcp.json (VS Code Copilot 방식)
```

Claude Code는 프로젝트 루트의 `.mcp.json`을 사용하지만, VS Code Copilot은 `.vscode/mcp.json`을 사용한다. 두 도구를 함께 사용하는 경우 각각의 위치에 설정 파일을 만들어야 한다.

**실수 4**: Agent Plugin이 정식 기능이라고 생각한다

```
❌ "Agent Plugin은 안정적이니까 프로덕션 워크플로우에 전면 도입하자"
✅ Agent Plugin은 Preview 상태이다. API와 기능이 변경될 수 있으므로
   핵심 워크플로우에는 점진적으로 도입한다
```

Agent Plugin은 2026년 2월에 Preview로 시작되었다. 아직 API나 구성 방식이 변경될 수 있으므로, 중요한 팀 워크플로우에 전면 도입하기보다는 점진적으로 테스트하면서 도입하는 것을 권장한다.

**실수 5**: Language Model Tool과 MCP Tool의 차이를 무시한다

```
❌ "MCP Tool이 있으니까 Language Model Tool은 필요 없다"
✅ VS Code API 접근이 필요하면 Language Model Tool,
   크로스 환경 재사용이 필요하면 MCP Tool을 사용한다
```

Language Model Tool은 VS Code 확장 호스트에서 실행되므로 VS Code API(에디터 상태, 알림, 다른 확장 연동 등)에 직접 접근할 수 있다. MCP Tool은 별도 프로세스이므로 VS Code API에 접근할 수 없지만, 다른 AI 도구에서도 재사용할 수 있다. 용도에 따라 선택해야 한다.

---

## 전체 구성 요소 한눈에 보기

```
VS Code GitHub Copilot 확장 시스템
│
├── Agent Plugin (Preview) ─────────── 통합 패키지 (Git 저장소 배포)
│   ├── Slash Commands                → 사용자가 /명령어로 직접 호출
│   ├── Agent Skills                  → 맥락에 따라 자동 활성화
│   ├── Custom Agents                 → 전문화된 AI 페르소나
│   ├── Hooks                         → 이벤트 핸들러 (PreCommand, PostEdit 등)
│   └── MCP Servers                   → 외부 서비스 연결
│
├── Copilot Extensions (GA) ────────── 개별 확장 (VS Code Marketplace)
│   ├── Language Model Tool           → Agent Mode용 도메인 도구
│   ├── Chat Participant API          → Ask Mode용 전문 어시스턴트
│   ├── Language Model API            → 프로그래밍으로 AI 직접 접근
│   └── MCP Tool                      → 크로스 환경 도구 연결
│
└── Agent Tools ────────────────────── Copilot이 사용하는 도구 3종
    ├── Built-in Tools                → 파일, 터미널, 검색, 디버깅
    ├── Extension Tools               → Language Model Tool로 등록된 확장
    └── MCP Tools                     → MCP 서버로 연결된 외부 도구
```

---

## 참고 자료

- VS Code Copilot Extensions 공식 문서: https://code.visualstudio.com/docs/copilot/copilot-extensibility-overview
- Chat Participant API 가이드: https://code.visualstudio.com/api/extension-guides/chat
- Language Model API 가이드: https://code.visualstudio.com/api/extension-guides/language-model
- MCP 서버 설정 가이드: https://code.visualstudio.com/docs/copilot/chat/mcp-servers
- Agent Plugin 공식 문서: https://code.visualstudio.com/docs/copilot/chat/agent-plugins
