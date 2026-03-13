# Cursor 자동화 워크플로우 가이드

## 학습 목표

Cursor의 **Automations**, **Agent Hooks**, **Prompt Files**, **Background Agent**를 이해하고, 이벤트 트리거 기반 자동화 파이프라인을 구성하여 GitHub 이슈 생성부터 PR 제출까지의 사이클을 자율 실행할 수 있다.

> **다른 도구와의 대응**: Claude Code의 **Hooks**, Codex CLI의 **`codex exec` + 프로파일**, Antigravity의 **Workflows**에 해당하는 Cursor 기능이다. Cursor는 GUI 기반 이벤트 트리거와 원격 자율 에이전트를 함께 지원한다.

## 사전 준비

- Cursor 에디터 설치 (버전 0.42 이상)
- GitHub 계정 연결 완료
- Cursor Pro 구독 (Automations 기능 사용 시)

---

## 전체 흐름 한눈에 보기

Cursor 자동화는 크게 세 계층으로 구성된다. **Prompt Files**는 재사용 가능한 프롬프트 템플릿이다. **Agent Hooks**는 에디터 이벤트에 셸 명령을 연결한다. **Automations**는 외부 이벤트(GitHub, Slack, 스케줄 등)를 트리거로 Background Agent를 자율 실행한다.

1. **개념 이해** — 세 계층의 역할과 관계
2. **Prompt Files 작성** — 재사용 가능한 프롬프트 관리
3. **Agent Hooks 설정** — 에디터 이벤트 기반 자동화
4. **Automations + Background Agent** — 완전 자율 실행 파이프라인

---

## Phase 1: Cursor 자동화 개념

### 목표

Cursor의 세 가지 자동화 계층을 이해하고, 각각이 어떤 문제를 해결하는지 파악한다.

### 단계별 구현

#### Step 1.1 — 세 가지 자동화 계층

> **💡 개념 설명: 왜 세 계층인가?**
>
> 에디터 내 작업(파일 저장 시 포매팅)과 외부 이벤트 기반 작업(GitHub 이슈 등록 시 자동 구현)은 전혀 다른 메커니즘이 필요하다. Cursor는 이를 각각 Agent Hooks(에디터 내)와 Automations(외부 이벤트)로 분리하여 처리한다.
>
> **핵심 한 줄:** Hooks = 에디터 내부 이벤트, Automations = 외부 이벤트로 분리

| 계층 | 트리거 | 실행 위치 | 적합한 용도 |
|------|--------|-----------|-------------|
| **Prompt Files** | 수동 호출 (`/파일명`) | 로컬 | 재사용 프롬프트 템플릿 |
| **Agent Hooks** | 에디터 이벤트 | 로컬 | 저장 시 포매팅, 린트 |
| **Automations** | 외부 이벤트 | 원격 VM | 이슈 자동 구현, 정기 작업 |

#### Step 1.2 — Automations 트리거 유형

| 트리거 | 설명 | 예시 |
|--------|------|------|
| **Schedule** | 시간 기반 정기 실행 | 매일 오전 9시 코드 품질 검사 |
| **GitHub PR** | PR 생성/업데이트 시 | 자동 코드 리뷰 댓글 작성 |
| **GitHub Issue** | 이슈 생성 시 | 이슈 분석 및 구현 시작 |
| **Slack** | Slack 메시지 수신 시 | `@cursor` 멘션으로 작업 지시 |
| **Linear** | Linear 이슈 상태 변경 시 | 작업 시작 시 자동 브랜치 생성 |
| **PagerDuty** | 인시던트 발생 시 | 장애 원인 자동 분석 |
| **Webhook** | HTTP POST 수신 시 | 사용자 정의 이벤트 연동 |

#### Step 1.3 — Background Agent

> **💡 개념 설명: Background Agent란?**
>
> Background Agent는 사용자의 로컬 환경이 아닌 Cursor의 원격 VM에서 실행되는 자율 에이전트다. GitHub 이슈를 받아 분석하고, 코드를 작성하고, 테스트를 실행하고, PR을 생성하는 일련의 작업을 사람 개입 없이 처리한다.
>
> **핵심 한 줄:** Background Agent = 클라우드에서 24시간 자율 실행되는 개발 에이전트

### 체크포인트

Prompt Files, Agent Hooks, Automations의 차이를 설명하고, Background Agent가 실행되는 위치(로컬 vs 원격)를 말할 수 있는가?

---

## Phase 2: Prompt Files 작성

### 목표

`.cursor/prompts/` 디렉토리에 재사용 가능한 프롬프트 파일을 작성하고 Chat에서 호출한다.

### 단계별 구현

#### Step 2.1 — Prompt Files 기본 구조

```
.cursor/
└── prompts/
    ├── code-review.md
    ├── generate-tests.md
    └── write-docs.md
```

파일 형식:
```markdown
---
description: 이 프롬프트가 하는 일
mode: agent              # agent | ask | edit
tools: [codebase, terminal, web]
---
여기에 프롬프트 내용 작성
```

#### Step 2.2 — 코드 리뷰 Prompt File

```markdown
---
description: 선택된 파일 또는 변경사항에 대한 포괄적 코드 리뷰 수행
mode: agent
tools: [codebase, terminal]
---
다음 코드를 검토해줘: $ARGUMENTS

## 검토 기준

### 코드 품질
- 함수와 클래스가 단일 책임 원칙을 따르는지 확인
- 변수명과 함수명이 의도를 명확히 표현하는지 확인
- 중복 코드가 있는지 확인

### 보안
- 입력값 검증 누락 여부
- SQL Injection, XSS, CSRF 취약점
- 하드코딩된 시크릿 또는 API 키

### 성능
- 불필요한 반복 연산
- N+1 쿼리 패턴
- 메모리 누수 가능성

## 출력 형식
각 이슈를 심각도(CRITICAL/HIGH/MEDIUM/LOW)로 분류하고
파일명:라인번호와 함께 구체적인 수정 방안을 제시해줘.
```

Chat에서 호출:
```
/code-review src/api/auth.ts
```

#### Step 2.3 — 테스트 생성 Prompt File

```markdown
---
description: 지정된 파일에 대한 포괄적인 단위 테스트 자동 생성
mode: agent
tools: [codebase, terminal]
---
다음 파일에 대한 단위 테스트를 생성해줘: $ARGUMENTS

## 테스트 작성 원칙

1. **AAA 패턴** 준수: Arrange → Act → Assert
2. **커버리지 목표**: 각 함수당 최소 3가지 케이스
   - 정상 케이스 (happy path)
   - 경계값 케이스 (edge case)
   - 예외/오류 케이스 (error case)
3. **명확한 테스트명**: `test_함수명_상황_기대결과` 형식
4. **테스트 격리**: 외부 의존성은 Mock 처리

## 실행
테스트 파일 생성 후 `npm test` 또는 `pytest`로 실행하여
모든 테스트가 통과하는지 확인해줘.
```

#### Step 2.4 — 노트패드에서 Prompt 관리

Cursor 노트패드(Notepad)에도 프롬프트를 저장할 수 있다:

- `Ctrl+Shift+P` → "New Notepad" 생성
- 노트패드에 프롬프트 작성 후 Chat에서 `@노트패드명`으로 참조

```
@code-style-guide 이 파일을 우리 팀 스타일 가이드에 맞게 수정해줘
```

### 체크포인트

`.cursor/prompts/code-review.md` 생성 후 Chat에서 `/code-review src/main.ts`를 실행하면 리뷰가 진행되면 성공이다.

---

## Phase 3: Agent Hooks 설정

### 목표

에디터 이벤트에 연결된 Agent Hooks를 설정하여 파일 저장 시 자동 포매팅과 린트를 구성한다.

### 단계별 구현

#### Step 3.1 — Agent Hooks 설정 위치

Cursor Settings(`Ctrl+,`) → Features → Agent → Hooks 섹션에서 설정하거나, `.vscode/settings.json`에 직접 작성한다:

```json
// .cursor/settings.json 또는 .vscode/settings.json
{
  "cursor.agent.hooks": {
    "preCommand": "npm run lint 2>/dev/null || true",
    "postEdit": "npx prettier --write ${file} 2>/dev/null || true"
  }
}
```

#### Step 3.2 — 훅 이벤트 유형

| 이벤트 | 발생 시점 | 예시 활용 |
|--------|-----------|-----------|
| `preCommand` | 명령 실행 전 | 린트 사전 실행 |
| `postEdit` | 파일 수정 후 | 자동 포매팅 |
| `onSave` | 파일 저장 시 | 타입 검사 실행 |

#### Step 3.3 — 언어별 포매팅 자동화

```json
{
  "cursor.agent.hooks": {
    "postEdit": "bash -c 'case \"${file}\" in *.ts|*.tsx|*.js|*.jsx) npx prettier --write \"${file}\" ;; *.py) black \"${file}\" && isort \"${file}\" ;; *.go) gofmt -w \"${file}\" ;; esac' 2>/dev/null || true"
  }
}
```

#### Step 3.4 — 프로젝트별 Rules Files (`.cursor/rules`)

에이전트의 행동을 제어하는 규칙 파일을 프로젝트에 포함한다:

```markdown
# .cursor/rules/coding-standards.mdc

## 코드 작성 규칙

- TypeScript에서 `any` 타입 사용 금지 — 대신 구체적인 타입 또는 `unknown` 사용
- 모든 async 함수에 try-catch 필수
- console.log는 프로덕션 코드에 사용 금지 (logger 모듈 사용)

## 파일 수정 후 자동 실행

파일을 수정하면 반드시:
1. `npm run lint -- ${file}` 실행
2. 린트 오류가 있으면 자동 수정
3. 수정된 파일을 포매터로 정리
```

### 체크포인트

파일을 수정할 때마다 자동으로 포매팅이 실행되고, Agent가 coding-standards.mdc 규칙을 준수하면 성공이다.

---

## Phase 4: Automations + Background Agent

### 목표

GitHub 이슈 트리거로 Background Agent가 자율적으로 이슈를 분석하고 코드를 구현하여 PR을 생성하는 완전 자율 파이프라인을 구성한다.

### 단계별 구현

#### Step 4.1 — Automations 설정

Cursor Dashboard(`cursor.com/dashboard`) → Automations → New Automation:

```
트리거 설정:
  - Type: GitHub Issue
  - Repository: your-org/your-repo
  - Filter: label = "cursor-agent"

Agent 설정:
  - Model: claude-4-5 (또는 최신 모델)
  - Max turns: 50
  - Allowed tools: terminal, file system, web search

지시사항:
  이슈를 분석하고 다음 단계로 구현해줘:
  1. 이슈 요구사항 파악 및 관련 코드 탐색
  2. 구현 계획 수립
  3. TDD 방식으로 코드 구현
  4. 전체 테스트 스위트 통과 확인
  5. PR 생성 (이슈 번호 참조)
```

#### Step 4.2 — GitHub 이슈 → 자동 구현 흐름

```
1. GitHub에서 이슈 생성
   예시 이슈 제목: "feat: 사용자 비밀번호 초기화 API 구현"
   라벨: cursor-agent

2. Automation 트리거 감지
   → Background Agent 시작 (원격 VM)

3. Background Agent 실행
   → 코드베이스 분석
   → 기존 인증 코드 탐색
   → 비밀번호 초기화 엔드포인트 구현
   → 단위 테스트 작성 및 통과 확인
   → PR 생성: "feat: 사용자 비밀번호 초기화 API (#123)"

4. 사용자 확인
   → PR 리뷰 후 머지
```

#### Step 4.3 — Slack 트리거 워크플로우

```
Slack 채널에서:
  @cursor feat/login-redesign 브랜치의 로그인 페이지 성능 최적화해줘

→ Background Agent:
  1. 해당 브랜치 체크아웃
  2. 로그인 페이지 성능 프로파일링
  3. 병목 지점 파악 및 최적화
  4. 성능 테스트 실행 (before/after 비교)
  5. PR 생성 + Slack에 결과 알림
```

#### Step 4.4 — 정기 자동화 (Schedule 트리거)

```
Schedule 설정:
  - 실행 시간: 매일 오전 6시 (팀 출근 전)
  - Cron: 0 6 * * 1-5

지시사항:
  1. main 브랜치의 어제 변경사항 분석
  2. 잠재적 버그와 기술 부채 파악
  3. 테스트 커버리지 감소 파일 탐지
  4. 분석 결과를 팀 Slack 채널(#dev-daily)에 보고
```

#### Step 4.5 — 완성된 자동화 설정 예시

```json
// .cursor/settings.json
{
  "cursor.agent.hooks": {
    "postEdit": "npx prettier --write ${file} 2>/dev/null || true",
    "preCommand": "npm run typecheck 2>/dev/null || true"
  },
  "cursor.rules": [
    ".cursor/rules/coding-standards.mdc",
    ".cursor/rules/security-guidelines.mdc"
  ]
}
```

```
.cursor/
├── settings.json           # Agent Hooks 설정
├── prompts/
│   ├── code-review.md      # /code-review
│   ├── generate-tests.md   # /generate-tests
│   └── write-docs.md       # /write-docs
└── rules/
    ├── coding-standards.mdc
    └── security-guidelines.mdc
```

#### Step 4.6 — 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| Automation이 트리거 안 됨 | GitHub 권한 미연결 | Cursor Dashboard에서 GitHub 앱 재설치 |
| Background Agent 무한 실행 | Max turns 미설정 | Automation 설정에서 Max turns 제한 |
| Hooks가 동작 안 함 | 설정 파일 경로 오류 | `.cursor/settings.json` 또는 `.vscode/settings.json` 위치 확인 |
| Prompt File이 Chat에서 안 보임 | 파일 경로 오류 | `.cursor/prompts/` 디렉토리 확인 |
| PR이 너무 많이 생성됨 | 트리거 필터 미설정 | 이슈 라벨 필터 추가 (`cursor-agent` 라벨) |

### 체크포인트

"cursor-agent" 라벨이 붙은 GitHub 이슈 생성 시 Background Agent가 자동으로 코드를 구현하고 PR을 생성하면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- Prompt Files = `.cursor/prompts/`에 저장된 재사용 가능한 프롬프트 템플릿
- Agent Hooks = 파일 저장/수정 시 자동 실행되는 셸 명령
- Automations = GitHub, Slack 등 외부 이벤트로 트리거되는 자율 실행
- Background Agent = 원격 VM에서 24시간 자율 실행되는 개발 에이전트

### 실용적인 자동화 아이디어

| 시나리오 | 계층 | 설명 |
|----------|------|------|
| 저장 시 자동 포매팅 | Agent Hooks | `postEdit`에 Prettier 연결 |
| 반복 코드 리뷰 | Prompt Files | `/code-review` 명령으로 표준화 |
| 이슈 자동 구현 | Automations | `cursor-agent` 라벨 이슈 자동 처리 |
| Slack 개발 요청 | Automations | `@cursor` 멘션으로 작업 지시 |
| 일일 품질 점검 | Automations/Schedule | 매일 오전 코드베이스 분석 보고 |
| PR 자동 리뷰 | Automations/GitHub PR | PR 생성 시 자동 리뷰 댓글 |

---

## 다른 도구와의 비교표

| 기능 | Cursor | Claude Code Hooks | Codex CLI | Antigravity Workflows | VS Code Copilot |
|------|--------|-------------------|-----------|-----------------------|-----------------|
| 에디터 내 이벤트 훅 | Agent Hooks | PostToolUse | 미지원 | Rules (간접) | Agent Hooks |
| 재사용 프롬프트 | Prompt Files | Skills | 프로파일 | Workflows | Prompt Files |
| 외부 이벤트 트리거 | Automations (다양) | GitHub Actions | GitHub Actions | 미지원 | Copilot Coding Agent |
| 원격 자율 에이전트 | Background Agent | 미지원 | CI 환경 실행 | 미지원 | Copilot Coding Agent |
| 설정 방식 | GUI + JSON | JSON | TOML | YAML + Markdown | JSON |
| Slack 연동 | 기본 지원 | HTTP 훅 | 셸 명령 | 미지원 | 미지원 |
| GitHub 이슈 트리거 | 기본 지원 | GitHub Actions | GitHub Actions | 미지원 | 기본 지원 |
| 로컬 실행 환경 필요 | 선택적 (VM 실행 가능) | 필수 | 필수 | 필수 | 선택적 |
