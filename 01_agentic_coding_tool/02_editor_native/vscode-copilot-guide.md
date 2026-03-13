# VS Code GitHub Copilot 실습 가이드

> **작성일**: 2026-03-12
> **대상 도구**: GitHub Copilot for Visual Studio Code (Agent Mode 중심)
> **요구 환경**: VS Code 1.99+, GitHub 계정

---

## 개요

### GitHub Copilot이란?

GitHub Copilot은 GitHub과 OpenAI가 공동 개발한 AI 기반 코딩 어시스턴트로, VS Code에 가장 깊이 통합된 에이전틱 코딩 도구이다. 단순한 코드 자동완성을 넘어, **Agent Mode**를 통해 자율적으로 코드를 분석하고, 파일을 생성/수정하며, 터미널 명령을 실행하고, 오류를 자동 수정하는 에이전트 기반 개발 워크플로우를 제공한다.

### Copilot만의 핵심 강점

| 강점 | 설명 |
|------|------|
| **IDE 완전 통합** | 에디터, 터미널, 디버거, Git 등 VS Code 전체와 유기적으로 연동 |
| **실시간 인라인 제안** | 타이핑하는 즉시 코드 완성 제안 (Tab 한 번으로 적용) |
| **멀티 모델 지원** | GPT-5, Claude Sonnet 4, Gemini 등 다양한 모델 선택 가능 |
| **Auto 모델 선택** | 작업 복잡도에 따라 최적 모델을 자동 라우팅 |
| **@workspace 컨텍스트** | 프로젝트 전체 코드베이스를 이해하는 대화 |
| **확장 생태계** | 수천 개의 VS Code 확장과 함께 사용 가능 |
| **무료 티어 제공** | Copilot Free로 월 2,000회 코드 완성 + 50회 프리미엄 요청 |
| **PR/커밋/문서** | 코드 작성 외에도 PR 리뷰, 커밋 메시지 생성, 문서화 지원 |

### Agent Mode란?

Agent Mode는 Copilot이 **자율적 프로그래밍 파트너**로 작동하는 모드이다. 사용자의 요청을 받으면 다음을 자동으로 수행한다:

1. 코드베이스를 분석하여 관련 파일을 파악
2. 구현 계획을 수립
3. 여러 파일에 걸쳐 코드를 생성/수정
4. 터미널 명령을 실행 (빌드, 테스트 등)
5. 컴파일 오류, 린트 오류를 감지하고 자동 수정
6. 테스트 결과를 확인하고 실패 시 재수정

이 루프를 작업이 완료될 때까지 반복하며, CLI 도구(Claude Code, Gemini CLI)의 에이전트 모드와 유사하지만 **VS Code UI 내에서 시각적으로** 이루어진다는 점이 차별점이다.

### CLI 도구 대비 Copilot의 장점

```
CLI 도구 (Claude Code, Gemini CLI)
  - 터미널 중심 인터페이스
  - 별도 도구 전환 필요
  - 텍스트 기반 diff 확인

VS Code Copilot
  - 에디터 내에서 직접 동작 (컨텍스트 전환 없음)
  - 인라인 diff 뷰, 시각적 코드 리뷰
  - 디버거/터미널/Git과 원클릭 연동
  - 타이핑 중 실시간 코드 제안
```

---

## Phase 1: 설치 및 설정

### 1-1. VS Code 확장 설치

VS Code에서 확장(Extensions) 패널을 열고 "GitHub Copilot"을 검색하여 설치한다.

```
설치할 확장:
  1. GitHub Copilot          - 인라인 코드 완성 + Agent Mode
  2. GitHub Copilot Chat     - 채팅 인터페이스 (최신 버전에서는 Copilot에 통합)
```

> 💡 **참고**: VS Code 1.99 이상에서는 GitHub Copilot 확장 하나로 인라인 제안, 채팅, Agent Mode가 모두 통합되어 있다.

**커맨드 팔레트로 설치하는 방법:**

```
Ctrl+Shift+P (Windows/Linux) 또는 Cmd+Shift+P (macOS)
→ "Extensions: Install Extensions" 입력
→ "GitHub Copilot" 검색 후 Install
```

### 1-2. GitHub 계정 및 구독

Copilot을 사용하려면 GitHub 계정에 Copilot 구독이 활성화되어 있어야 한다.

| 플랜 | 가격 | 주요 기능 |
|------|------|----------|
| **Copilot Free** | $0/월 | 월 2,000 코드 완성, 50 프리미엄 요청, GPT-4o + Claude 3.5 Sonnet |
| **Copilot Pro** | $10/월 | 무제한 코드 완성, 프리미엄 모델 접근, Coding Agent |
| **Copilot Pro+** | $39/월 | Pro 전체 + 더 많은 프리미엄 요청, 모든 모델 접근 |
| **Copilot Business** | $19/사용자/월 | 조직용, 관리 정책, IP 보호 |
| **Copilot Enterprise** | $39/사용자/월 | 기업용, 맞춤 모델 튜닝, 감사 로그 |

> 💡 **Copilot Free 시작하기**: GitHub 계정만 있으면 무료로 시작할 수 있다. 검증된 학생, 교사, 인기 오픈소스 유지관리자는 Copilot Pro를 무료로 사용할 수 있다.

### 1-3. 인증

VS Code에서 확장 설치 후 최초 실행 시:

1. VS Code 우측 하단에 "Sign in to GitHub" 알림이 표시된다
2. 클릭하면 브라우저가 열리며 GitHub OAuth 인증 진행
3. 권한 승인 후 VS Code로 자동 복귀
4. 상태 표시줄에 Copilot 아이콘이 활성화되면 완료

```
인증 상태 확인:
  VS Code 하단 상태 표시줄 → Copilot 아이콘 (활성: 체크마크, 비활성: 경고)
```

### 1-4. 핵심 단축키

| 동작 | Windows/Linux | macOS |
|------|---------------|-------|
| 인라인 제안 수락 | `Tab` | `Tab` |
| 인라인 제안 거부 | `Esc` | `Esc` |
| 다음/이전 제안 순회 | `Alt+]` / `Alt+[` | `Option+]` / `Option+[` |
| Copilot Chat 열기 | `Ctrl+Shift+I` | `Cmd+Shift+I` |
| 인라인 Chat | `Ctrl+I` | `Cmd+I` |
| 빠른 Chat | `Ctrl+Shift+Alt+L` | `Cmd+Shift+Option+L` |

> 💡 **체크포인트**: Copilot 설치 후 아무 코드 파일을 열고 함수 시그니처를 입력해 보자. 회색 텍스트로 코드 제안이 나타나면 설정이 올바른 것이다.

---

## Phase 2: 기본 사용법

### 2-1. 인라인 코드 제안 (Tab Completion)

Copilot의 가장 기본적인 기능이다. 코드를 타이핑하면 실시간으로 다음 코드를 제안한다.

```python
# 함수 시그니처만 작성하면 Copilot이 본문을 제안한다
def calculate_fibonacci(n: int) -> int:
    # ← 여기서 Tab을 누르면 전체 구현이 완성됨
```

```typescript
// 주석으로 의도를 설명하면 더 정확한 제안을 받을 수 있다
// JWT 토큰을 검증하고 사용자 정보를 반환하는 미들웨어
export const authMiddleware = async (req, res, next) => {
    // ← Copilot이 토큰 검증 로직을 제안
};
```

**Next Edit Suggestions (NES)**: 최신 Copilot은 현재 커서 위치뿐 아니라, 다음에 편집해야 할 위치까지 예측하여 제안한다. 구문 하이라이팅이 적용된 고스트 텍스트로 표시된다.

### 2-2. Copilot Chat 패널

`Ctrl+Shift+I`로 사이드 패널에서 Copilot Chat을 열 수 있다. 자연어로 질문하고 코드를 생성할 수 있다.

```
사용 예시:
  "이 프로젝트의 인증 로직을 설명해줘"
  "이 함수에 에러 핸들링을 추가해줘"
  "REST API를 GraphQL로 마이그레이션하는 방법을 알려줘"
```

### 2-3. Chat Participants (채팅 참여자)

Chat에서 `@` 접두사로 특정 컨텍스트 제공자를 호출할 수 있다.

| 참여자 | 역할 | 사용 예시 |
|--------|------|----------|
| `@workspace` | 전체 코드베이스 컨텍스트 | `@workspace 이 프로젝트의 아키텍처를 설명해줘` |
| `@terminal` | 터미널 출력/명령 컨텍스트 | `@terminal 방금 발생한 에러를 분석해줘` |
| `@vscode` | VS Code 설정/명령 | `@vscode 탭 크기를 4로 변경하는 방법` |
| `@github` | GitHub 이슈/PR/저장소 | `@github 이 저장소의 오픈 이슈를 보여줘` |

```
실전 활용:
  @workspace 프로젝트에서 사용 중인 인증 방식은 무엇인가?
  @terminal 방금 실행한 테스트에서 실패한 원인을 분석해줘
  @vscode Python 가상환경을 인터프리터로 설정하는 방법
```

> 💡 **@workspace의 강력함**: `@workspace`는 프로젝트 전체 파일을 인덱싱하여 컨텍스트로 활용한다. "이 프로젝트에서 데이터베이스 연결은 어떻게 하고 있어?"와 같은 질문에 관련 파일을 찾아 정확히 답변할 수 있다.

### 2-4. Slash Commands (슬래시 명령어)

Chat에서 `/` 접두사로 특정 작업을 빠르게 실행할 수 있다.

| 명령어 | 기능 | 사용 예시 |
|--------|------|----------|
| `/explain` | 선택한 코드 설명 | 코드 선택 후 `/explain` |
| `/fix` | 버그 수정 제안 | 에러가 있는 코드 선택 후 `/fix` |
| `/tests` | 테스트 코드 생성 | 함수 선택 후 `/tests` |
| `/doc` | 문서/주석 생성 | 함수 선택 후 `/doc` |
| `/new` | 새 프로젝트/파일 생성 | `/new Express API 서버 생성` |
| `/clear` | 대화 초기화 | `/clear` |

```
실전 활용 시나리오:

1. 코드 이해: 레거시 코드 선택 → /explain
2. 버그 수정: 에러 발생 코드 선택 → /fix
3. 테스트 작성: 비즈니스 로직 선택 → /tests
4. 문서화: 공개 API 함수 선택 → /doc
```

### 2-5. Agent Mode 기본 사용

Agent Mode는 Copilot Chat에서 활성화할 수 있다.

**활성화 방법:**

1. Copilot Chat 패널 열기 (`Ctrl+Shift+I`)
2. 채팅 입력란 상단의 모드 선택 드롭다운에서 **Agent** 선택
3. 자연어로 작업 요청

```
Agent Mode 요청 예시:
  "Express로 REST API 서버를 만들어줘. 사용자 CRUD와 JWT 인증을 포함해."
  "이 프로젝트의 테스트 커버리지를 80% 이상으로 올려줘."
  "TypeScript 마이그레이션을 진행해줘. src/ 디렉토리의 .js 파일을 .ts로 변환해."
```

Agent Mode에서 Copilot은 다음 단계를 자율적으로 수행한다:
1. 코드베이스 분석 (관련 파일 탐색)
2. 파일 생성/수정
3. 터미널 명령 실행 (npm install, 테스트 등)
4. 오류 발생 시 자동 수정 루프

> 💡 **체크포인트**: Chat 패널에서 Agent 모드를 선택하고, "현재 프로젝트의 구조를 분석하고 개선점을 제안해줘"라고 입력해 보자. Copilot이 파일을 탐색하고 분석 결과를 제공하는 것을 확인할 수 있다.

---

## Phase 3: Copilot Edits & Agent Mode 심화

### 3-1. Copilot Edits (멀티 파일 편집)

Copilot Edits는 여러 파일에 걸친 변경사항을 한 번에 요청하고 리뷰할 수 있는 기능이다.

**Working Set 개념**: 편집 대상 파일을 명시적으로 지정하는 UI이다.

```
Working Set에 파일 추가하는 방법:
  1. 파일 탐색기에서 드래그 앤 드롭
  2. 에디터 탭을 드래그 앤 드롭
  3. Chat에서 # 기호로 파일 참조 (예: #src/auth/login.ts)
```

**편집 흐름:**

```
1. Copilot Edits 패널 열기
2. Working Set에 대상 파일 추가
3. 변경 요청 입력 (예: "모든 API 엔드포인트에 에러 핸들링 추가")
4. Copilot이 제안한 변경사항을 인라인 diff로 확인
5. 각 변경사항을 Accept 또는 Discard
```

### 3-2. Agent Mode 자율 워크플로우

Agent Mode에서는 Copilot이 전체 워크플로우를 자율적으로 실행한다.

```
요청: "React 대시보드 앱을 만들어줘. 사용자 통계, 차트, 다크모드를 포함해."

Copilot Agent의 자율 실행 과정:
  1. [분석] 프로젝트 구조 파악, 기존 의존성 확인
  2. [계획] 필요한 패키지와 파일 구조 결정
  3. [실행] npm install react-router-dom recharts ...
  4. [생성] src/components/Dashboard.tsx 생성
  5. [생성] src/components/Chart.tsx 생성
  6. [생성] src/hooks/useDarkMode.ts 생성
  7. [수정] src/App.tsx에 라우팅 추가
  8. [검증] npm run build → 빌드 오류 발생
  9. [수정] 타입 오류 자동 수정
  10. [검증] npm run build → 성공
  11. [완료] 변경사항 요약 보고
```

### 3-3. 터미널 통합

Agent Mode에서 터미널 명령을 직접 실행할 수 있다. 명령 실행 전 사용자에게 승인을 요청한다.

```
Copilot이 제안하는 터미널 명령 예시:
  > npm install express cors helmet
  > npx prisma migrate dev --name init
  > npm run test -- --coverage

각 명령은 실행 전 [Run] [Skip] 버튼이 표시되어 사용자가 제어한다.
```

**자동 승인 설정 (YOLO 모드):**

Chat 입력란에서 `/autoApprove` 또는 `/yolo`를 입력하면 터미널 명령 자동 승인을 토글할 수 있다.

### 3-4. Hooks (에이전트 라이프사이클 자동화)

2026년 2월 릴리스에서 도입된 Hooks는 에이전트의 핵심 라이프사이클 이벤트에 커스텀 스크립트를 연결하는 기능이다.

```json
// .vscode/settings.json
{
  "github.copilot.chat.agent.hooks": {
    "preCommand": "npm run lint",
    "postEdit": "npx prettier --write ${file}"
  }
}
```

이를 통해 에이전트가 명령을 실행하기 전에 린트를 실행하거나, 파일 편집 후 자동으로 포매팅을 적용할 수 있다.

### 3-5. Explore 서브에이전트

내장된 Explore 서브에이전트는 경량 모델을 사용하여 코드베이스를 병렬로 탐색한다. 메인 에이전트가 Plan을 수립할 때 Explore가 관련 파일과 코드 경로를 빠르게 수집하여 전달한다.

### 3-6. Vision (이미지 입력)

Copilot Chat에 이미지를 첨부하여 시각적 컨텍스트를 제공할 수 있다.

```
활용 시나리오:
  - UI 목업 이미지 → HTML/CSS 코드 생성
  - 에러 스크린샷 → 원인 분석 및 수정
  - 화이트보드 다이어그램 → 아키텍처 코드 구현
  - 디자인 시안 → 컴포넌트 코드 생성
```

### 3-7. Claude Code와의 비교

| 기능 | VS Code Copilot | Claude Code (CLI) |
|------|-----------------|-------------------|
| 인터페이스 | GUI (에디터 통합) | CLI (터미널) |
| 인라인 코드 제안 | 실시간 Tab 완성 | 없음 |
| 파일 변경 확인 | 인라인 diff 뷰 | 텍스트 diff |
| 터미널 실행 | UI 버튼으로 승인 | 프롬프트로 승인 |
| 멀티 모델 | GPT-5, Claude, Gemini 등 선택 | Claude 모델 전용 |
| 자율 모드 | Agent Mode | `--dangerously-skip-permissions` |
| 프로젝트 규칙 | `copilot-instructions.md` | `CLAUDE.md` |
| MCP 지원 | 지원 | 지원 |
| 가격 | Free 티어 있음 ($0~$39/월) | Pro 플랜 필요 |

> 💡 **체크포인트**: Agent Mode에서 간단한 앱을 처음부터 만들어 보자. 예를 들어 "Express + SQLite로 TODO API를 만들어줘"를 요청하고, Copilot이 파일 생성부터 테스트까지 자율적으로 수행하는 과정을 관찰해 보자.

---

## Phase 4: 설정 파일 위치

### 4-1. VS Code settings.json (Copilot 설정)

Copilot 관련 설정은 VS Code의 `settings.json`에서 관리한다.

**사용자 설정** (모든 프로젝트에 적용):

```
위치:
  Windows: %APPDATA%\Code\User\settings.json
  macOS:   ~/Library/Application Support/Code/User/settings.json
  Linux:   ~/.config/Code/User/settings.json
```

**워크스페이스 설정** (현재 프로젝트에만 적용):

```
위치: <project-root>/.vscode/settings.json
```

**주요 Copilot 설정 예시:**

```json
{
  // Copilot 인라인 제안 활성화/비활성화
  "github.copilot.enable": {
    "*": true,
    "markdown": false,
    "plaintext": false
  },

  // Chat에서 사용할 기본 모델
  "github.copilot.chat.model": "auto",

  // Agent Mode에서 터미널 명령 자동 승인
  "github.copilot.chat.agent.autoApprove": false,

  // 인라인 제안 지연 시간 (ms)
  "github.copilot.inlineSuggest.debounce": 75,

  // Next Edit Suggestions 활성화
  "github.copilot.nextEditSuggestions.enabled": true
}
```

### 4-2. copilot-instructions.md (프로젝트 규칙)

프로젝트 전체에 적용되는 Copilot 지침 파일이다. Claude Code의 `CLAUDE.md`, Gemini CLI의 `GEMINI.md`에 해당한다.

```
위치: <project-root>/.github/copilot-instructions.md
```

**copilot-instructions.md 예시:**

```markdown
# 프로젝트 규칙

## 기술 스택
- Backend: Node.js + Express + TypeScript
- Database: PostgreSQL + Prisma ORM
- Frontend: React + Next.js
- Testing: Jest + React Testing Library

## 코딩 규칙
- TypeScript strict 모드 사용
- 모든 함수에 반환 타입 명시
- `any` 타입 사용 금지
- 에러 처리는 커스텀 AppError 클래스 사용

## 네이밍 규칙
- 컴포넌트: PascalCase (예: UserProfile.tsx)
- 유틸리티: camelCase (예: formatDate.ts)
- 상수: UPPER_SNAKE_CASE
- API 엔드포인트: kebab-case

## 커밋 규칙
- Conventional Commits 형식
- 한국어로 작성
```

**경로별 개별 지침 파일** (2025년 7월 이후):

YAML Frontmatter의 `applyTo` 속성으로 특정 경로에만 적용되는 지침을 만들 수 있다.

```markdown
---
applyTo: "src/api/**"
---
# API 모듈 지침
- 모든 엔드포인트에 OpenAPI 문서 작성
- Rate limiting 미들웨어 필수 적용
- 응답은 항상 { data, error, meta } 형식
```

```
파일별 지침 위치:
  <project-root>/.github/instructions/*.instructions.md
```

### 4-3. 콘텐츠 제외 설정

Copilot이 특정 파일에 접근하지 못하도록 제외할 수 있다.

**방법 1: VS Code 설정으로 제외**

```json
// .vscode/settings.json
{
  "github.copilot.chat.codeGeneration.excludeFiles": [
    "**/secrets/**",
    "**/.env*",
    "**/credentials/**"
  ]
}
```

**방법 2: GitHub 조직 설정 (Business/Enterprise)**

GitHub 조직 관리 페이지에서 저장소 단위로 콘텐츠 제외 정책을 설정할 수 있다. 제외된 파일은 인라인 제안, Chat 응답, 코드 리뷰에서 모두 무시된다.

**방법 3: .copilotignore 확장 (커뮤니티)**

`.gitignore`와 유사한 문법의 `.copilotignore` 파일을 지원하는 서드파티 확장도 있다.

```
# .copilotignore 예시 (서드파티 확장 필요)
node_modules/
dist/
.env
*.secret
credentials/
```

### 4-4. 설정 계층 구조

```
우선순위 (높은 것이 낮은 것을 오버라이드):
  1. 워크스페이스 설정     (.vscode/settings.json)
  2. 사용자 설정           (User settings.json)
  3. 기본값               (VS Code 내장 기본값)

지침 파일 로딩:
  1. .github/copilot-instructions.md         (프로젝트 전체)
  2. .github/instructions/*.instructions.md  (경로별 개별 지침)
```

### Claude Code 설정 대응표

| Claude Code | Copilot | 설명 |
|-------------|---------|------|
| `CLAUDE.md` | `.github/copilot-instructions.md` | 프로젝트 규칙 |
| `~/.claude/CLAUDE.md` | User settings.json | 전역 설정 |
| 서브디렉토리 `CLAUDE.md` | `*.instructions.md` (applyTo) | 경로별 지침 |
| `.claudeignore` | 콘텐츠 제외 설정 | 파일 제외 |
| `.claude/settings.json` | `.vscode/settings.json` | 도구 동작 설정 |

> 💡 **체크포인트**: 프로젝트 루트에 `.github/copilot-instructions.md` 파일을 만들고, 코딩 규칙을 몇 가지 작성해 보자. 이후 Copilot Chat에서 코드를 요청하면 지침이 반영되는 것을 확인할 수 있다.

---

## Phase 5: 핵심 기능 미리보기

이 섹션에서는 Copilot의 핵심 기능들이 이 프로젝트의 다른 가이드와 어떻게 연결되는지 안내한다. 각 주제는 별도의 상세 가이드에서 다룬다.

### 5-1. copilot-instructions.md (규칙) → `02_rule/`

프로젝트 규칙 파일은 Copilot이 코드를 생성할 때 따라야 할 지침을 정의한다.

```
대응 관계:
  Gemini CLI  → GEMINI.md / .gemini/GEMINI.md
  Claude Code → CLAUDE.md
  Copilot     → .github/copilot-instructions.md

Copilot 고유 기능:
  - applyTo: 경로별 선택적 적용
  - YAML Frontmatter 지원
  - 조직 수준 정책 상속 (Business/Enterprise)
```

상세 내용은 `02_rule/` 디렉토리의 가이드를 참고한다.

### 5-2. MCP 서버 지원 → `03_mcp/`

Copilot은 MCP(Model Context Protocol) 서버를 통해 외부 도구와 연동할 수 있다.

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
    "figma": {
      "command": "npx",
      "args": ["-y", "@figma/mcp-server"]
    }
  }
}
```

```
Copilot MCP 특징:
  - VS Code 설정으로 MCP 서버 관리
  - Agent 플러그인: MCP 서버 + Skills + Hooks를 하나의 패키지로 설치
  - 원격 MCP 서버 지원 (VS Code 1.101+)
  - GitHub MCP Registry에서 큐레이션된 서버 목록 제공
```

상세 내용은 `03_mcp/` 디렉토리의 가이드를 참고한다.

### 5-3. Custom Instructions & Chat Participants → `04_skill/`

Copilot의 Chat Participant와 Custom Agent는 특정 도메인에 특화된 AI 기능을 제공한다.

```
Copilot의 Skill 개념:
  - Chat Participants (@workspace, @terminal, @vscode, @github)
  - Custom Chat Participants (확장으로 추가 가능)
  - Agent Skills (slash command로 트리거되는 전문 기능)
  - Agent 플러그인 (Skills + Tools + Hooks + MCP를 번들링)

대응 관계:
  Antigravity Skills  → Copilot Agent 플러그인
  Gemini CLI Skills   → Copilot Chat Participants + Agent Skills
  Claude Code Task()  → Copilot Explore 서브에이전트
```

상세 내용은 `04_skill/` 디렉토리의 가이드를 참고한다.

### 5-4. Tasks & Agent Workflows → `05_workflow/`

Copilot Agent Mode에서 복잡한 작업을 체계적으로 수행하는 워크플로우를 구성할 수 있다.

```
Copilot Workflow 구성 요소:
  - Prompt Files (.prompt.md): 재사용 가능한 프롬프트 템플릿
  - Agent Hooks: 라이프사이클 이벤트에 자동화 연결
  - Custom Slash Commands: /명령어로 Agent Skills 트리거
  - Copilot Coding Agent: GitHub 이슈를 자동으로 구현 (원격 에이전트)

대응 관계:
  Antigravity Workflows → Copilot Prompt Files + Hooks
  Gemini CLI Commands   → Copilot Custom Slash Commands
  Claude Code hooks     → Copilot Agent Hooks
```

상세 내용은 `05_workflow/` 디렉토리의 가이드를 참고한다.

### 5-5. Multi-Agent 패턴 → `06_subagent/`

Copilot은 여러 에이전트 패턴을 지원한다.

```
Copilot의 Multi-Agent 구조:
  - Explore 서브에이전트: 코드베이스 탐색을 병렬로 위임
  - Copilot Coding Agent: GitHub 이슈 기반 원격 자율 에이전트
  - Claude Agent SDK 통합: Anthropic Claude 모델로 에이전트 태스크 위임
  - Agent 플러그인 확장: 서드파티 에이전트 기능 추가

Copilot Coding Agent:
  - GitHub 이슈에 Copilot을 할당하면 자율적으로 브랜치 생성, 코드 구현, PR 생성
  - 원격에서 실행되므로 로컬 환경 불필요
  - CI/CD 파이프라인과 자동 연동
```

상세 내용은 `06_subagent/` 디렉토리의 가이드를 참고한다.

> 💡 **체크포인트**: 위 5가지 핵심 기능 중 가장 관심 있는 영역을 하나 선택하고, 해당 상세 가이드를 읽어보자.

---

## 모델 선택 가이드

Copilot은 다양한 AI 모델을 지원하며, 작업에 따라 최적의 모델을 선택할 수 있다.

### 사용 가능한 모델 (2026년 3월 기준)

| 모델 | 특징 | 적합한 작업 |
|------|------|------------|
| **Auto** (기본값) | 작업에 따라 최적 모델 자동 선택 | 대부분의 일반 작업 |
| **GPT-5** | OpenAI 최신 모델, 강력한 추론 | 복잡한 알고리즘, 설계 |
| **GPT-5 mini** | 빠른 응답, 가벼운 작업 | 간단한 코드 생성, 질문 |
| **Claude Sonnet 4** | Anthropic 모델, 코드 품질 우수 | 리팩터링, 코드 리뷰 |
| **Gemini** | Google 모델, 대용량 컨텍스트 | 대규모 코드베이스 분석 |

### 모델 변경 방법

```
1. Chat 입력란 하단의 모델 선택 드롭다운 클릭
2. 원하는 모델 선택
3. 또는 커맨드 팔레트에서 "Copilot: Select Model" 실행
```

> 💡 **Auto 모드 팁**: Auto 모드는 실시간 가용성과 작업 복잡도를 기반으로 모델을 선택한다. 응답 상단에 마우스를 올리면 실제 사용된 모델과 프리미엄 요청 배수를 확인할 수 있다.

---

## 마무리

### Copilot의 위치

GitHub Copilot은 에이전틱 코딩 도구 생태계에서 **IDE 통합 최적화**라는 고유한 위치를 차지한다.

```
도구 스펙트럼:

  CLI 중심                          IDE 중심
  ←──────────────────────────────────────→
  Gemini CLI    Claude Code    Antigravity    Copilot
  (터미널)      (터미널)       (전용 에디터)   (VS Code 통합)
```

- **CLI 도구**가 강력한 자동화와 스크립팅에 적합하다면
- **Copilot**은 일상적인 코딩 워크플로우에서의 원활한 AI 통합에 강점을 가진다

### 권장 활용 전략

1. **일상 코딩**: Copilot 인라인 제안 + Tab 완성으로 속도 향상
2. **코드 이해**: `@workspace`와 `/explain`으로 코드베이스 파악
3. **기능 구현**: Agent Mode로 새 기능을 자율적으로 생성
4. **코드 품질**: `/fix`, `/tests`로 버그 수정 및 테스트 보강
5. **팀 협업**: `copilot-instructions.md`로 팀 코딩 규칙 공유
6. **외부 연동**: MCP 서버로 GitHub, Figma 등 외부 도구 통합

### 다음 단계

이 가이드에서 개요를 파악했다면, 각 상세 가이드로 진행하자:

| 순서 | 가이드 | 내용 |
|------|--------|------|
| 1 | `02_rule/` | copilot-instructions.md 작성법 |
| 2 | `03_mcp/` | MCP 서버 설정 및 활용 |
| 3 | `04_skill/` | Chat Participants, Agent 플러그인 |
| 4 | `05_workflow/` | Prompt Files, Hooks, 자동화 |
| 5 | `06_subagent/` | Coding Agent, Multi-Agent 패턴 |

---

## 참고 링크

- [GitHub Copilot 공식 문서](https://code.visualstudio.com/docs/copilot/overview)
- [GitHub Copilot 기능 치트시트](https://code.visualstudio.com/docs/copilot/reference/copilot-vscode-features)
- [Copilot 구독 플랜](https://github.com/features/copilot/plans)
- [copilot-instructions.md 작성 가이드](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [MCP 서버 설정](https://code.visualstudio.com/docs/copilot/customization/mcp-servers)
- [AI 모델 선택](https://code.visualstudio.com/docs/copilot/customization/language-models)
- [Agent Mode 소개 블로그](https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode)
- [awesome-copilot (커뮤니티 리소스)](https://github.com/github/awesome-copilot)
