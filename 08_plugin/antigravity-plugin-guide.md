# Antigravity Plugin/Extension 학습 가이드

## 개요

Antigravity의 확장 시스템은 Claude Code와 근본적으로 다른 구조를 가진다. Claude Code가 모든 확장 기능을 **하나의 Plugin 패키지**로 통합하는 반면, Antigravity는 **세 가지 독립된 확장 시스템**이 공존한다. 이 구조적 차이를 이해하는 것이 Antigravity 확장 학습의 출발점이다.

---

## 1. Antigravity의 확장 체계

### 세 가지 독립된 확장 시스템

Antigravity에는 다음 세 가지 확장 시스템이 존재하며, 각각 독립적으로 동작한다.

| 확장 시스템 | 확장 대상 | 배포 채널 | 비유 |
|-------------|-----------|-----------|------|
| **VS Code 호환 확장** | 에디터 기능 | OpenVSX Registry | 스마트폰의 기본 앱 스토어 |
| **MCP Store** | 에이전트 도구 | MCP Store (내장) | AI 전용 도구 상점 |
| **Skills** | 에이전트 지식 | 디렉토리 기반 (로컬) | 직원의 업무 매뉴얼 |

이 세 시스템은 서로를 대체하지 않는다. 각각 고유한 역할이 있으며, 필요에 따라 조합해서 사용한다.

### Claude Code와의 근본적 차이

Claude Code의 Plugin은 **하나의 `plugin.json`** 안에 커스텀 명령어, 스킬, 에이전트, MCP 서버, 훅, LSP 서버를 모두 묶는다. 앱스토어에서 앱 하나를 설치하면 모든 기능이 한 번에 들어오는 것과 같다.

Antigravity는 이와 달리 **각 기능을 독립된 채널에서 관리**한다. 에디터 테마가 필요하면 OpenVSX에서, PostgreSQL 연결이 필요하면 MCP Store에서, 코드 리뷰 규칙이 필요하면 Skills 디렉토리에서 각각 가져온다. 통합 패키지라는 개념 자체가 없다.

```
Claude Code:
  Plugin (plugin.json)
  ├── Commands
  ├── Skills
  ├── Agents
  ├── MCP Servers
  ├── Hooks
  └── LSP Servers

Antigravity:
  VS Code 호환 확장 (OpenVSX)   ← 에디터 기능
  MCP Store                      ← 에이전트 도구
  Skills (.agent/skills/)        ← 에이전트 지식
  (세 시스템 간에 패키지 의존 관계 없음)
```

이 구조는 장단점이 있다. 통합 패키지가 없으므로 "하나 설치하면 끝"이라는 편리함은 없지만, 필요한 것만 골라서 가볍게 사용할 수 있다는 유연성이 있다.

---

## 2. VS Code 호환 확장 (OpenVSX Registry)

### 개요

Antigravity는 VS Code 오픈소스(Code-OSS) 기반의 에디터이다. 따라서 VS Code 확장 생태계를 그대로 활용할 수 있다. 다만 **Microsoft의 VS Code Marketplace 대신 OpenVSX Registry를 사용**한다는 점이 핵심 차이이다.

OpenVSX는 Eclipse Foundation이 관리하는 오픈소스 확장 레지스트리이다. VS Code Marketplace에 등록된 확장 중 상당수가 OpenVSX에도 등록되어 있지만, 전부는 아니다. 이것이 실질적으로 가장 큰 제약이다.

### VS Code 호환 확장이 담당하는 영역

이 확장들은 **에디터 자체의 기능**을 확장한다. AI 에이전트와는 직접 관련이 없다.

- **테마**: 색상 테마, 아이콘 테마
- **언어 지원**: 구문 강조, 코드 스니펫, 린터, 포매터
- **편집 도구**: Git 통합, 디버거, 터미널 확장
- **UI 확장**: 파일 탐색기, 사이드바 위젯

### 설치 방법

**방법 1: Extensions 패널에서 검색**

가장 기본적인 방법이다. 사이드바의 Extensions 아이콘을 클릭하거나 `Ctrl+Shift+X`를 누르면 OpenVSX Registry에서 확장을 검색하고 설치할 수 있다.

**방법 2: VSIX 파일에서 직접 설치**

OpenVSX에 없는 확장은 VSIX 파일을 직접 설치할 수 있다.

1. 확장의 VSIX 파일을 다운로드한다 (GitHub Release 페이지 등에서)
2. Command Palette(`Ctrl+Shift+P`) → `Extensions: Install from VSIX...` 선택
3. 다운로드한 `.vsix` 파일을 지정한다

**방법 3: VS Code Marketplace로 우회 (비공식)**

OpenVSX에 없는 확장이 반드시 필요한 경우, 마켓플레이스 URL을 변경하는 우회 방법이 있다. 설정 파일(`product.json` 또는 환경 변수)에서 확장 갤러리 URL을 VS Code Marketplace로 변경하는 것이다. 다만 이 방법은 Microsoft의 이용약관에 위배될 수 있으므로 주의가 필요하다.

### 제약사항

**Microsoft 전용 확장 사용 불가**

다음과 같은 확장은 Microsoft가 VS Code에서만 사용하도록 라이선스를 제한하고 있어 Antigravity에서 사용할 수 없다.

- C# Dev Kit
- Visual Studio IntelliCode
- Remote Development Pack (Remote-SSH, Remote-WSL 등)
- Pylance (Python 언어 서버)
- Live Share

이들은 마켓플레이스 URL을 우회하더라도 설치 후 정상 동작하지 않을 수 있다.

**OpenVSX에 없는 인기 확장**

일부 인기 확장은 제작자가 OpenVSX에 별도로 등록하지 않아 검색되지 않는다. 이 경우 VSIX 수동 설치나 대체 확장을 찾아야 한다.

**다른 에디터와 확장 공유 불가**

같은 머신에 VS Code와 Antigravity가 모두 설치되어 있어도 확장은 공유되지 않는다. 각 에디터는 독립된 확장 디렉토리를 사용하므로, 원하는 확장을 Antigravity에서 별도로 설치해야 한다.

---

## 3. MCP Store (에이전트 확장)

### 개요

MCP Store는 Antigravity의 **AI 에이전트가 외부 도구와 서비스에 접근하는 통로**이다. VS Code 호환 확장이 에디터 기능을 확장하는 것이라면, MCP Store는 에이전트의 능력을 확장한다.

MCP Store에는 **1,500개 이상의 사전 구축된 MCP 서버**가 등록되어 있다. 데이터베이스, 클라우드 서비스, SaaS API, 개발 도구 등 다양한 영역을 커버한다. Claude Code에서 `.mcp.json`에 수동으로 MCP 서버를 설정하는 것과 비교하면, 몇 번의 클릭으로 동일한 연동을 완료할 수 있다.

### MCP Store가 담당하는 영역

MCP Store의 서버는 에이전트에게 **도구(tool)**를 제공한다. 에이전트는 이 도구를 사용해 외부 시스템과 상호작용한다.

- **데이터베이스**: AlloyDB for PostgreSQL, BigQuery, Cloud SQL, Firestore
- **클라우드 서비스**: Firebase, Google Cloud Storage, AWS S3
- **개발 도구**: GitHub, GitLab, Jira, Linear
- **커뮤니케이션**: Slack, Discord, Microsoft Teams
- **자동화**: n8n, Composio, Zapier
- **기타**: Stripe, Twilio, SendGrid 등

### 설치 방법

MCP 서버 설치는 에디터 내에서 직접 이루어진다.

```
1. 메뉴 → MCP Servers (또는 Command Palette에서 "MCP" 검색)
2. 원하는 서비스를 검색하거나 카테고리에서 선택
3. Install 버튼 클릭
4. 필요한 인증 정보 입력 (API 키, 토큰 등)
```

설치가 완료되면 프로젝트의 `mcp_config.json` 파일이 자동으로 업데이트된다. 이후 에이전트에게 "GitHub 이슈 목록을 보여줘" 같은 요청을 하면, 에이전트가 설치된 MCP 서버의 도구를 사용해 직접 처리한다.

### mcp_config.json 구조

MCP Store에서 설치하면 자동 생성되지만, 수동으로 편집할 수도 있다.

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    },
    "firebase": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-firebase"],
      "env": {
        "FIREBASE_PROJECT_ID": "my-project"
      }
    }
  }
}
```

### 주요 MCP 서버 예시

| 서버 이름 | 용도 | 제공 도구 예시 |
|-----------|------|---------------|
| GitHub | 코드 저장소 연동 | 이슈 관리, PR 생성, 코드 검색 |
| BigQuery | 데이터 분석 | SQL 쿼리 실행, 테이블 조회 |
| Firebase | 앱 백엔드 | Firestore CRUD, Auth 관리 |
| Slack | 팀 커뮤니케이션 | 메시지 전송, 채널 검색 |
| n8n | 워크플로우 자동화 | 자동화 트리거, 실행 상태 조회 |
| Composio | 통합 API | 200+ 서비스의 통합 액션 |
| Jira | 프로젝트 관리 | 티켓 생성, 스프린트 조회 |

### 서버당 도구 제한

하나의 MCP 서버가 노출할 수 있는 도구 수는 **최대 500개**로 제한된다. 이는 에이전트의 컨텍스트 창이 도구 설명으로 과도하게 채워지는 것을 방지하기 위한 것이다. 실제로 대부분의 MCP 서버는 10~50개 정도의 도구를 제공하므로 이 제한에 걸리는 경우는 드물다.

---

## 4. Skills 시스템 (에이전트 지식 확장)

### 개요

Skills는 VS Code 확장도 MCP 서버도 아닌, **Antigravity만의 고유한 에이전트 지식 확장 시스템**이다. 에이전트에게 "이런 상황에서는 이렇게 해라"는 전문 지식을 주입하는 메커니즘이다.

Claude Code의 Skills가 Plugin 패키지 안에 번들되는 것과 달리, Antigravity의 Skills는 **파일 시스템 디렉토리에 직접 배치**된다. 별도의 마켓플레이스나 설치 절차 없이, 디렉토리에 파일을 만들면 바로 작동한다.

### 디렉토리 구조

Skills는 두 가지 범위로 배치할 수 있다.

```
# 프로젝트 스코프 (현재 프로젝트에서만 활성화)
.agent/skills/<skill-name>/SKILL.md

# 글로벌 스코프 (모든 프로젝트에서 활성화)
~/.gemini/antigravity/skills/<skill-name>/SKILL.md
```

각 스킬은 반드시 `SKILL.md` 파일을 포함하는 디렉토리여야 한다. 이 파일이 스킬의 본문이자 에이전트에게 전달되는 지식이다.

### SKILL.md 작성 예시

```markdown
<!-- .agent/skills/code-reviewer/SKILL.md -->

# 코드 리뷰 전문가

이 스킬은 코드 리뷰 요청 시 활성화된다.

## 리뷰 기준

1. **버그 가능성**: null 참조, 경계값 오류, 경쟁 상태
2. **성능**: 불필요한 루프, N+1 쿼리, 메모리 누수
3. **보안**: 입력값 검증, SQL 인젝션, XSS
4. **가독성**: 네이밍 컨벤션, 함수 길이, 주석 적절성

## 출력 형식

발견된 이슈를 심각도(Critical/Warning/Info) 순으로 정리한다.
각 이슈에는 파일 경로, 라인 번호, 수정 제안을 포함한다.
```

### Progressive Disclosure (점진적 공개)

Skills의 핵심 동작 원리이다. 모든 스킬이 항상 로드되는 것이 아니라, **에이전트가 현재 맥락을 분석하여 관련 있는 스킬만 선택적으로 로드**한다.

예를 들어 사용자가 "이 코드 리뷰해줘"라고 요청하면, 에이전트는 `code-reviewer` 스킬의 `SKILL.md`를 읽고 그 기준에 따라 리뷰를 수행한다. "배포해줘"라고 하면 `deploy` 스킬이 로드된다. 요청과 무관한 스킬은 로드되지 않으므로 컨텍스트 창을 낭비하지 않는다.

이는 Claude Code의 Skills가 YAML frontmatter의 `description` 필드를 기반으로 매칭하는 것과 유사한 메커니즘이다.

### Skills를 사용해야 하는 경우

- 프로젝트 고유의 코딩 컨벤션을 에이전트에게 학습시키고 싶을 때
- 배포 절차, 테스트 규칙 같은 반복적인 워크플로우를 정의할 때
- 팀 전체가 동일한 에이전트 행동 규칙을 공유하고 싶을 때 (Git에 커밋)
- CLAUDE.md/GEMINI.md에 적기엔 너무 길고 상세한 전문 지식을 분리할 때

---

## 5. 세 가지 시스템의 역할 분담

세 확장 시스템은 각각 다른 계층을 담당한다. 이 역할 분담을 이해하면 "어떤 시스템을 써야 하는가"에 대한 판단이 쉬워진다.

```
┌──────────────────────────────────────┐
│  에이전트 지식 (Skills)              │  ← "어떻게 해야 하는지 안다"
│  .agent/skills/*/SKILL.md            │     코드 리뷰 기준, 배포 절차
├──────────────────────────────────────┤
│  에이전트 도구 (MCP Store)           │  ← "외부 시스템에 접근할 수 있다"
│  mcp_config.json                     │     GitHub, DB, Slack 연동
├──────────────────────────────────────┤
│  에디터 기능 (VS Code 확장)          │  ← "편집 환경이 풍부하다"
│  OpenVSX Registry                    │     테마, 린터, 디버거
└──────────────────────────────────────┘
```

### 상세 비교

| 구분 | VS Code 확장 | MCP Store | Skills |
|------|-------------|-----------|--------|
| **확장 대상** | 에디터 UI/기능 | 에이전트의 외부 도구 | 에이전트의 내부 지식 |
| **배포 방법** | OpenVSX Registry | MCP Store (내장) | 파일 시스템 (Git 커밋) |
| **설치 방법** | Extensions 패널 | MCP Servers 메뉴 | 디렉토리에 파일 배치 |
| **설정 파일** | (에디터 내부 관리) | mcp_config.json | SKILL.md |
| **사용 주체** | 개발자 (사람) | 에이전트 (AI) | 에이전트 (AI) |
| **예시** | ESLint, Prettier, 테마 | GitHub 연동, DB 쿼리 | 코드 리뷰 규칙, 배포 절차 |
| **오프라인 사용** | 설치 후 가능 | 서버 실행 필요 | 항상 가능 |

### 판단 플로우

```
에디터 UI를 바꾸고 싶다 (테마, 키바인딩 등)
  → VS Code 확장

에이전트가 외부 서비스에 접근해야 한다 (API, DB 등)
  → MCP Store

에이전트에게 특정 규칙이나 절차를 가르치고 싶다
  → Skills
```

---

## 6. Claude Code Plugin과의 비교표

| 구분 | Claude Code | Antigravity |
|------|-------------|-------------|
| **통합 패키지** | Plugin (`plugin.json`) | 없음 (3개 시스템 분리) |
| **에디터 확장** | 없음 (CLI이므로) | VS Code 호환 확장 (OpenVSX) |
| **MCP 서버** | Plugin에 번들 (`.mcp.json`) | MCP Store에서 독립 설치 |
| **Skills** | Plugin에 번들 (`skills/`) | 디렉토리 기반으로 독립 관리 |
| **Hooks** | Plugin에 번들 (`hooks.json`) | Rules/Denylist로 대체 |
| **LSP** | Plugin에 번들 (`.lsp.json`) | VS Code 확장으로 제공 |
| **Agents** | Plugin에 번들 (`agents/`) | 직접 대응 없음 |
| **마켓플레이스** | Plugin Marketplace | OpenVSX + MCP Store (이원화) |
| **배포 단위** | Git 저장소 하나 = 플러그인 하나 | 시스템별 개별 설치 |
| **팀 공유** | 플러그인 설치 URL 공유 | 설정 파일 + Skills 디렉토리 Git 커밋 |

### 아키텍처 철학의 차이

**Claude Code**: "하나의 플러그인이 모든 것을 포함한다." 플러그인 제작자가 명령어, 스킬, MCP 서버, 훅을 하나의 패키지로 묶어 배포한다. 사용자는 설치 한 번으로 모든 기능을 얻는다.

**Antigravity**: "각 확장은 독립적으로 존재한다." 에디터 기능, 에이전트 도구, 에이전트 지식이 각각 별도의 채널에서 관리된다. 통합의 편리함보다 독립성과 유연성을 선택한 구조이다.

어느 쪽이 우월하다고 단정할 수는 없다. Claude Code 방식은 "잘 만들어진 플러그인 하나로 복잡한 워크플로우를 한 번에 구성"할 수 있다는 장점이 있고, Antigravity 방식은 "필요한 조각만 가져다 쓸 수 있다"는 유연성이 있다.

---

## 7. 빠른 시작

### 7-1. MCP Store에서 GitHub 서버 설치

```
1. Command Palette (Ctrl+Shift+P) → "MCP Servers" 검색
2. MCP Store 패널에서 "GitHub" 검색
3. Install 클릭
4. GitHub Personal Access Token 입력
5. 에이전트에게 테스트: "내 GitHub 저장소 목록 보여줘"
```

설치 후 `mcp_config.json`에 GitHub 서버 설정이 자동으로 추가된다.

### 7-2. VS Code 확장 설치 (ESLint 예시)

```
1. Extensions 패널 (Ctrl+Shift+X) 열기
2. "ESLint" 검색
3. dbaeumer.vscode-eslint 확장의 Install 클릭
4. 프로젝트에 .eslintrc 설정이 있으면 바로 동작 시작
```

### 7-3. Skills 생성 (코드 리뷰 스킬 예시)

```bash
# 프로젝트 루트에서 실행
mkdir -p .agent/skills/code-reviewer
```

`.agent/skills/code-reviewer/SKILL.md` 파일을 생성하고 다음 내용을 작성한다.

```markdown
# 코드 리뷰 전문가

코드 리뷰 요청 시 다음 기준으로 검토한다.

## 검토 항목
1. 버그 가능성: null 체크, 경계값, 타입 안전성
2. 성능: 불필요한 연산, 메모리 사용
3. 보안: 입력값 검증, 인증/인가
4. 코드 스타일: 프로젝트 컨벤션 준수 여부

## 출력 형식
- Critical/Warning/Info 등급으로 분류
- 각 이슈에 파일 경로와 수정 제안 포함
```

이후 에이전트에게 "이 PR 코드 리뷰해줘"라고 요청하면 이 스킬이 자동으로 활성화된다.

### 7-4. 세 시스템을 조합한 실전 시나리오

"코드 리뷰 후 Slack으로 결과를 전송하는" 워크플로우를 구성한다고 가정하자.

1. **VS Code 확장**: ESLint, Prettier 설치 (코드 품질 기본 환경)
2. **MCP Store**: GitHub 서버 + Slack 서버 설치 (PR 접근 + 메시지 전송)
3. **Skills**: `code-reviewer` 스킬 생성 (리뷰 기준 정의)

이 세 가지를 조합하면 에이전트에게 "이 PR 리뷰하고 결과를 #code-review 채널에 보내줘"라고 한 문장으로 요청할 수 있다.

---

## 8. 보안 고려사항

### MCP Store 서버의 권한

MCP 서버는 에이전트에게 외부 시스템 접근 권한을 부여한다. 이는 강력한 기능이지만 보안 위험도 수반한다.

- **API 키 관리**: MCP 서버에 입력하는 API 키와 토큰은 `mcp_config.json`에 평문으로 저장된다. 이 파일을 Git에 커밋하지 않도록 `.gitignore`에 반드시 추가한다.
- **최소 권한 원칙**: GitHub 토큰을 생성할 때 `repo:read` 만 필요하다면 `repo` 전체 권한을 부여하지 않는다. MCP 서버가 요구하는 최소 권한만 부여한다.
- **서버 신뢰성**: MCP Store에 등록된 서버라고 모두 안전한 것은 아니다. 서버의 소스 코드와 제작자를 확인하고, 알려진 제작자의 서버를 우선 사용한다.

### VS Code 확장의 보안

- VS Code 확장은 로컬 파일 시스템에 접근할 수 있다. 출처가 불분명한 확장은 설치하지 않는다.
- OpenVSX에 등록된 확장도 악의적인 코드를 포함할 수 있다. 다운로드 수와 리뷰를 확인한다.
- VSIX 파일을 수동 설치할 때는 특히 주의한다. 공식 채널 외의 경로에서 다운로드한 파일은 검증이 어렵다.

### Skills의 보안

- Skills는 에이전트의 행동을 지시하는 텍스트이므로, **악의적인 지시를 포함한 스킬**이 프로젝트에 포함될 수 있다. 외부에서 클론한 저장소의 `.agent/skills/` 디렉토리를 반드시 검토한다.
- 글로벌 스킬(`~/.gemini/antigravity/skills/`)은 모든 프로젝트에 영향을 미치므로 더욱 신중하게 관리한다.

### 자율 실행 모드의 주의

Antigravity의 에이전트에게 작업을 위임할 때, MCP 서버를 통한 외부 시스템 변경(파일 삭제, 데이터베이스 수정, 메시지 전송 등)은 되돌리기 어려울 수 있다. 중요한 작업은 에이전트에게 "실행 전에 계획을 먼저 보여줘"라고 요청하는 습관을 들인다.

---

## 9. 자주 하는 실수

### 실수 1: VS Code 확장과 MCP 서버를 혼동한다

```
"GitHub 확장을 설치했는데 에이전트가 이슈를 못 읽어요"
→ VS Code의 GitHub 확장은 에디터 UI에 GitHub 기능을 추가하는 것이다.
  에이전트가 GitHub에 접근하려면 MCP Store에서 GitHub MCP 서버를 설치해야 한다.
```

### 실수 2: OpenVSX에 없는 확장을 찾지 못해 당황한다

```
"VS Code에서 쓰던 확장이 검색 안 돼요"
→ OpenVSX에 등록되지 않은 확장일 수 있다.
  VSIX 수동 설치를 시도하거나, OpenVSX에서 동일 기능의 대체 확장을 찾는다.
```

### 실수 3: mcp_config.json을 Git에 커밋한다

```
"팀원이 내 API 키를 볼 수 있게 됐어요"
→ mcp_config.json에는 API 키와 토큰이 평문으로 저장된다.
  .gitignore에 mcp_config.json을 추가한다.
  이미 커밋했다면 git filter-branch로 히스토리에서 제거하고 키를 즉시 교체한다.
```

### 실수 4: Skills 디렉토리 경로를 잘못 만든다

```
"스킬을 만들었는데 에이전트가 인식을 못해요"
→ 올바른 경로: .agent/skills/my-skill/SKILL.md
  흔한 실수:
  ❌ .agent/my-skill/SKILL.md          (skills/ 디렉토리 누락)
  ❌ .agent/skills/my-skill/skill.md   (파일명 대소문자 오류)
  ❌ .agent/skills/SKILL.md            (스킬 이름 디렉토리 누락)
```

### 실수 5: Claude Code Plugin처럼 하나의 패키지로 묶으려 한다

```
"MCP 서버와 스킬을 하나의 확장으로 배포하고 싶어요"
→ Antigravity에는 통합 패키지 개념이 없다.
  MCP 서버는 MCP Store에서, 스킬은 Git 저장소에서 각각 관리해야 한다.
  팀 공유가 목적이라면 스킬은 프로젝트 저장소에 커밋하고,
  MCP 설정은 README에 설치 가이드를 작성하는 방식으로 해결한다.
```

### 실수 6: Microsoft 전용 확장을 설치하려 한다

```
"C# Dev Kit이 안 깔려요"
→ Microsoft 전용 라이선스 확장은 Antigravity에서 사용할 수 없다.
  C#의 경우 오픈소스 대안인 OmniSharp 확장을 사용한다.
  Python의 Pylance 대신 Pyright 기반 확장을 찾는다.
```

---

## 참고 자료

- Antigravity 공식 문서: [Google Antigravity 공식 사이트]
- OpenVSX Registry: https://open-vsx.org/
- MCP 공식 사양: https://modelcontextprotocol.io/
- Gemini Skills 문서: [Gemini Agent Skills 가이드]
