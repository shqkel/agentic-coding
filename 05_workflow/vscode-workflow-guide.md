# VS Code Copilot 워크플로우 가이드

## 학습 목표

VS Code GitHub Copilot의 **Agent Hooks**, **Prompt Files**, **Copilot Coding Agent**, **VS Code Tasks 통합**을 이해하고, 에디터 이벤트 기반 자동화부터 GitHub 이슈 자율 구현까지 단계적으로 구성할 수 있다.

> **다른 도구와의 대응**: Claude Code의 **Hooks**, Cursor의 **Automations + Agent Hooks**, Antigravity의 **Workflows**에 해당하는 VS Code 기능이다. VS Code Copilot은 에디터 네이티브 환경에서 GitHub과 깊이 통합된 자동화를 제공한다.

## 사전 준비

- VS Code 최신 버전 설치 (1.93 이상)
- GitHub Copilot 구독 (개인 또는 팀)
- GitHub 계정 연결 완료
- Copilot Coding Agent 사용 시: Copilot Pro+ 또는 Team 구독

---

## 전체 흐름 한눈에 보기

VS Code Copilot 자동화는 네 계층으로 구성된다. **Prompt Files**는 재사용 가능한 프롬프트 템플릿이다. **Agent Hooks**는 에디터 이벤트에 셸 명령을 자동 연결한다. **VS Code Tasks**는 빌드/테스트 파이프라인과 통합한다. **Copilot Coding Agent**는 GitHub 이슈를 받아 원격에서 자율 구현한다.

1. **개념 이해** — 네 계층의 역할과 GitHub 통합 방식
2. **Prompt Files 작성** — `.github/prompts/` 기반 템플릿 관리
3. **Agent Hooks + Tasks** — 에디터 이벤트 자동화
4. **Copilot Coding Agent** — 완전 자율 구현 파이프라인

---

## Phase 1: VS Code Copilot 자동화 개념

### 목표

VS Code Copilot의 네 가지 자동화 계층을 이해하고, GitHub과의 통합 방식을 파악한다.

### 단계별 구현

#### Step 1.1 — 네 가지 자동화 계층

> **💡 개념 설명: VS Code Copilot이 특별한 이유**
>
> VS Code Copilot은 GitHub과 동일한 회사(Microsoft)의 제품이다. Copilot Coding Agent는 GitHub 이슈에 Copilot을 직접 할당하면 원격 VM에서 코드를 구현하고 PR을 생성한다. 로컬 환경이 전혀 필요 없다.
>
> **핵심 한 줄:** GitHub 이슈 → Copilot 할당 → 자율 구현 → PR = 가장 간단한 자율 개발 파이프라인

| 계층 | 역할 | 설정 위치 |
|------|------|-----------|
| **Prompt Files** | 재사용 프롬프트 템플릿 | `.github/prompts/*.prompt.md` |
| **Agent Hooks** | 에디터 이벤트 자동화 | `.vscode/settings.json` |
| **VS Code Tasks** | 빌드/테스트 파이프라인 | `.vscode/tasks.json` |
| **Copilot Coding Agent** | 원격 자율 구현 | GitHub 이슈 할당 |

#### Step 1.2 — Copilot Coding Agent vs Copilot Chat 비교

| 항목 | Copilot Chat | Copilot Coding Agent |
|------|-------------|----------------------|
| 실행 환경 | 로컬 VS Code | 원격 GitHub VM |
| 대화 필요 | 필요 | 불필요 |
| 파일 수정 | 사용자 확인 후 | 자율 처리 |
| PR 생성 | 수동 | 자동 |
| CI/CD 연동 | 수동 | 자동 (GitHub Actions 감지) |
| 접근 방법 | 채팅창 | GitHub 이슈 담당자 할당 |

#### Step 1.3 — `.github/copilot-instructions.md`

모든 Copilot 상호작용에 자동으로 적용되는 전역 지침 파일이다:

```markdown
# .github/copilot-instructions.md

## 프로젝트 개요
이 프로젝트는 Node.js + TypeScript 기반 REST API 서버이다.

## 코드 스타일
- 모든 함수에 JSDoc 주석 필수
- async/await 사용 (callback 금지)
- any 타입 사용 금지
- 에러 처리는 custom Error 클래스 사용

## 테스트
- Jest 사용
- 파일명: `*.test.ts`
- 각 함수마다 최소 3가지 테스트 케이스

## 금지 사항
- console.log 프로덕션 코드에 사용 금지 (logger 모듈 사용)
- 하드코딩된 API 키 또는 비밀번호 금지
```

### 체크포인트

네 가지 자동화 계층과 각각의 설정 파일 위치를 나열할 수 있고, `copilot-instructions.md`의 역할을 설명할 수 있는가?

---

## Phase 2: Prompt Files 작성

### 목표

`.github/prompts/` 디렉토리에 팀이 공유하는 Prompt Files를 작성하고 Chat에서 호출한다.

### 단계별 구현

#### Step 2.1 — Prompt Files 파일 구조

```
.github/
├── copilot-instructions.md    # 전역 지침 (항상 적용)
└── prompts/
    ├── code-review.prompt.md      # 코드 리뷰
    ├── generate-tests.prompt.md   # 테스트 생성
    ├── write-docs.prompt.md       # 문서화
    └── security-audit.prompt.md   # 보안 감사
```

#### Step 2.2 — 코드 리뷰 Prompt File

```markdown
---
mode: agent
tools: [codebase, terminal]
description: 선택된 파일에 대한 포괄적인 코드 리뷰 수행
---
다음 변경사항을 검토해줘: $ARGUMENTS

## 검토 기준

### 코드 품질
- SOLID 원칙 준수 여부 확인
- 함수 길이 (20줄 이하 권장)
- 매직 넘버/문자열 상수화 여부

### 보안 검사
- 입력값 검증 및 살균(sanitization) 누락 여부
- 인증/인가 로직 정확성
- 민감 정보 노출 여부

### 타입 안전성 (TypeScript)
- `any` 타입 사용 여부
- null/undefined 처리 누락
- 제네릭 타입 올바른 사용

## 출력 형식
심각도를 CRITICAL/HIGH/MEDIUM/LOW로 분류하여
각 이슈마다 파일명:라인번호와 수정 코드 예시를 제공해줘.
```

Chat에서 호출:
```
@workspace /review src/controllers/auth.controller.ts
```

또는 설정에서 활성화된 경우:
```
/code-review
```

#### Step 2.3 — PR 리뷰 자동화 Prompt File

```markdown
---
mode: agent
tools: [codebase, terminal, github]
description: 현재 PR의 전체 변경사항에 대한 자동 리뷰 수행
---
현재 브랜치와 main 브랜치의 차이를 검토해줘.

## 검토 과정

1. `git diff main...HEAD` 실행하여 변경사항 파악
2. 변경된 각 파일에 대해 다음 검토 수행:
   - 기능 정확성: 요구사항을 올바르게 구현했는가?
   - 하위 호환성: 기존 API를 변경하지 않았는가?
   - 테스트 커버리지: 새 코드에 대한 테스트가 있는가?

3. 전체 테스트 스위트 실행: `npm test`

## 출력
검토 결과를 PR 댓글 형식(Markdown)으로 작성해줘.
승인(Approve) / 수정 요청(Request Changes) 권고도 포함할 것.
```

#### Step 2.4 — 문서화 Prompt File

```markdown
---
mode: agent
tools: [codebase]
description: 지정된 파일에 JSDoc/docstring 자동 추가
---
다음 파일에 문서화를 추가해줘: $ARGUMENTS

## 문서화 기준

### JSDoc (TypeScript/JavaScript)
- 모든 exported 함수와 클래스에 `@description` 추가
- 파라미터에 `@param {타입} 이름 - 설명` 형식
- 반환값에 `@returns {타입} 설명`
- 사용 예시 `@example` 포함

### 한국어 작성
- 모든 설명은 한국어로 작성
- 기술 용어는 영문 그대로 사용 가능

파일을 직접 수정하여 문서를 추가해줘.
수정 후 TypeScript 컴파일 오류가 없는지 확인할 것.
```

### 체크포인트

`.github/prompts/code-review.prompt.md` 작성 후 Chat에서 호출 시 리뷰가 실행되면 성공이다.

---

## Phase 3: Agent Hooks + VS Code Tasks 통합

### 목표

에디터 이벤트에 자동화를 연결하고, VS Code Tasks와 Copilot을 통합하여 개발 파이프라인을 구성한다.

### 단계별 구현

#### Step 3.1 — Agent Hooks 설정

```json
// .vscode/settings.json
{
  "github.copilot.chat.agent.hooks": {
    "preCommand": "npm run lint 2>/dev/null || true",
    "postEdit": "npx prettier --write ${file} 2>/dev/null || true"
  }
}
```

#### Step 3.2 — 언어별 세밀한 포매팅

```json
// .vscode/settings.json
{
  "github.copilot.chat.agent.hooks": {
    "postEdit": "bash -c 'EXT=\"${file##*.}\"; case \"$EXT\" in ts|tsx|js|jsx) npx prettier --write \"${file}\" 2>/dev/null;; py) black \"${file}\" && isort \"${file}\" 2>/dev/null;; go) gofmt -w \"${file}\" 2>/dev/null;; esac' || true"
  },
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "explicit"
  }
}
```

#### Step 3.3 — VS Code Tasks 정의

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Copilot: 린트 + 포매팅",
      "type": "shell",
      "command": "npm run lint:fix && npm run format",
      "group": "build",
      "presentation": {
        "reveal": "silent",
        "panel": "shared"
      },
      "problemMatcher": "$eslint-stylish"
    },
    {
      "label": "Copilot: 테스트 실행",
      "type": "shell",
      "command": "npm test -- --coverage",
      "group": {
        "kind": "test",
        "isDefault": true
      },
      "presentation": {
        "reveal": "always",
        "panel": "dedicated"
      }
    },
    {
      "label": "Copilot: PR 준비",
      "dependsOrder": "sequence",
      "dependsOn": [
        "Copilot: 린트 + 포매팅",
        "Copilot: 테스트 실행"
      ],
      "group": "build"
    }
  ]
}
```

Tasks를 Chat과 연동:
```
@workspace PR 제출 전에 'Copilot: PR 준비' 태스크를 실행하고 오류가 있으면 수정해줘
```

#### Step 3.4 — 키보드 단축키로 Copilot Tasks 실행

```json
// .vscode/keybindings.json
[
  {
    "key": "ctrl+shift+t",
    "command": "workbench.action.tasks.runTask",
    "args": "Copilot: 테스트 실행"
  },
  {
    "key": "ctrl+shift+p ctrl+shift+r",
    "command": "workbench.action.tasks.runTask",
    "args": "Copilot: PR 준비"
  }
]
```

### 체크포인트

파일 수정 후 자동 포매팅이 실행되고, `Ctrl+Shift+T`로 테스트가 실행되면 성공이다.

---

## Phase 4: Copilot Coding Agent

### 목표

GitHub 이슈에 Copilot을 담당자로 할당하여 원격에서 자율적으로 코드를 구현하고 PR을 생성하는 파이프라인을 구성한다.

### 단계별 구현

#### Step 4.1 — Copilot Coding Agent 활성화

1. GitHub 저장소 Settings → Copilot → Agent 활성화
2. 필요한 권한 부여:
   - `Contents: Read & Write` (코드 수정)
   - `Pull requests: Read & Write` (PR 생성)
   - `Issues: Read` (이슈 읽기)

#### Step 4.2 — 이슈 → 자율 구현 흐름

```
1. GitHub 이슈 작성
   제목: feat: 이메일 알림 구독/해제 API 구현
   본문:
     ## 요구사항
     - POST /notifications/subscribe: 이메일 구독 등록
     - DELETE /notifications/subscribe: 구독 해제
     - 인증 필요 (JWT Bearer 토큰)
     - 이미 구독 중이면 409 Conflict 반환

2. 담당자(Assignee)에 "Copilot" 할당

3. Copilot Coding Agent 자동 시작
   - 코드베이스 분석
   - 기존 인증 미들웨어 패턴 파악
   - 라우터, 컨트롤러, 서비스 레이어 구현
   - 단위 테스트 + 통합 테스트 작성
   - GitHub Actions CI 통과 확인

4. PR 자동 생성
   제목: feat: 이메일 알림 구독/해제 API (#42)
   본문: 구현 내용 요약, 테스트 결과, 변경 파일 목록
```

#### Step 4.3 — 이슈 작성 팁 (Copilot이 잘 이해하는 형식)

```markdown
## 구현할 기능
[한 문장으로 명확히]

## 요구사항
- [구체적인 기능 명세 1]
- [구체적인 기능 명세 2]

## 기술 스택 (해당 시)
- 언어/프레임워크
- 의존해야 할 기존 모듈

## 완료 조건 (Definition of Done)
- [ ] 기능 구현 완료
- [ ] 단위 테스트 작성 (커버리지 80% 이상)
- [ ] API 문서 업데이트
- [ ] CI 통과

## 참고 파일
- `src/controllers/user.controller.ts` (패턴 참고)
- `src/middlewares/auth.middleware.ts` (인증 참고)
```

#### Step 4.4 — GitHub Actions와 자동 연동

Copilot Coding Agent는 저장소의 GitHub Actions를 자동으로 감지하고 CI 결과를 확인한다:

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage

  # Copilot이 이 결과를 확인하고 실패 시 자동으로 수정 시도
```

> **💡 개념 설명: Copilot과 CI 연동**
>
> Copilot Coding Agent가 PR을 생성하면 GitHub Actions가 자동 실행된다. CI가 실패하면 Copilot이 실패 로그를 분석하고 코드를 수정하여 다시 커밋한다. 사람이 개입하지 않아도 CI가 통과할 때까지 자동으로 반복한다.
>
> **핵심 한 줄:** Copilot + GitHub Actions = CI 실패 시 자동 수정 루프

#### Step 4.5 — 완성된 프로젝트 구성 예시

```
프로젝트 루트/
├── .github/
│   ├── copilot-instructions.md        # 전역 Copilot 지침
│   ├── prompts/
│   │   ├── code-review.prompt.md      # /code-review
│   │   ├── generate-tests.prompt.md   # /generate-tests
│   │   ├── write-docs.prompt.md       # /write-docs
│   │   └── security-audit.prompt.md   # /security-audit
│   └── workflows/
│       └── ci.yml                     # Copilot Agent와 연동
├── .vscode/
│   ├── settings.json                  # Agent Hooks 설정
│   ├── tasks.json                     # Tasks 정의
│   └── extensions.json                # 권장 확장 목록
```

#### Step 4.6 — 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| Copilot Agent가 이슈를 처리 안 함 | Agent 미활성화 | 저장소 Settings → Copilot → Agent 확인 |
| PR이 CI에서 계속 실패 | instructions.md에 기술 스택 미명시 | `copilot-instructions.md`에 상세 스택 추가 |
| Prompt File이 Chat에서 안 보임 | 파일 확장자 오류 | `.prompt.md` 확장자 확인 |
| Agent Hooks 미동작 | 설정 키 오류 | `github.copilot.chat.agent.hooks` 키 정확히 입력 |
| 원치 않는 파일 수정 | 범위 지정 미흡 | 이슈 본문에 수정 금지 파일 명시 |

### 체크포인트

GitHub 이슈에 Copilot을 담당자로 할당했을 때 PR이 자동 생성되고, CI가 통과하면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- `copilot-instructions.md` = 모든 Copilot 상호작용에 자동 적용되는 전역 지침
- Prompt Files (`.github/prompts/*.prompt.md`) = 팀이 공유하는 재사용 프롬프트
- Agent Hooks = 에디터 이벤트에 연결된 자동 포매팅/린트
- VS Code Tasks = Copilot과 통합된 빌드/테스트 파이프라인
- Copilot Coding Agent = GitHub 이슈 → 자율 구현 → PR 생성

### 실용적인 자동화 아이디어

| 시나리오 | 기능 | 설명 |
|----------|------|------|
| 저장 시 자동 정리 | Agent Hooks | 포매팅, 린트 자동 실행 |
| 표준 리뷰 체크리스트 | Prompt Files | 팀 공유 리뷰 기준 통일 |
| 이슈 자동 구현 | Copilot Coding Agent | 이슈 할당 → PR 자동 생성 |
| PR 통과 자동화 | Copilot + GitHub Actions | CI 실패 시 자동 수정 |
| 일일 보안 감사 | Prompt Files + Task | 정기 보안 취약점 스캔 |
| 신입 온보딩 지원 | copilot-instructions.md | 프로젝트 규칙 자동 적용 |

---

## 다른 도구와의 비교표

| 기능 | VS Code Copilot | Claude Code Hooks | Cursor Automations | Codex CLI | Antigravity Workflows |
|------|-----------------|-------------------|--------------------|-----------|----------------------|
| 에디터 이벤트 훅 | Agent Hooks | PostToolUse | Agent Hooks | 미지원 | Rules (간접) |
| 재사용 프롬프트 | Prompt Files | Skills | Prompt Files | 프로파일 | Workflows |
| 원격 자율 에이전트 | Copilot Coding Agent | 미지원 | Background Agent | CI 환경 | 미지원 |
| GitHub 이슈 트리거 | 이슈 할당 방식 | GitHub Actions | Automations | GitHub Actions | 미지원 |
| CI/CD 자동 연동 | GitHub Actions (자동 감지) | GitHub Actions | GitHub Actions | GitHub Actions | 제한적 |
| 설정 파일 위치 | `.github/` + `.vscode/` | `.claude/` | `.cursor/` | `~/.codex/` | `.agent/` |
| 팀 공유 방식 | 저장소 커밋 | 저장소 커밋 | 저장소 커밋 | 개인 설정 | 저장소 커밋 |
| 로컬 실행 필요 | Agent는 불필요 | 필수 | Agent는 불필요 | 필수 | 필수 |
| HTTP 훅 지원 | 미지원 | 지원 | Webhook 수신 | 셸 명령으로 | 미지원 |
