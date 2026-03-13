# Antigravity Workflows 가이드

## 학습 목표

Antigravity의 **Workflows 시스템**을 이해하고, YAML 프론트매터 형식의 워크플로우 파일을 작성하여 반복적인 멀티스텝 작업을 채팅 명령어 한 줄로 실행할 수 있다. 또한 Rules와 Workflows를 조합하여 Hooks에 준하는 자동화를 구현한다.

> **다른 도구와의 대응**: Claude Code의 **Hooks**, Codex CLI의 **`codex exec` 파이프라인**, Cursor의 **Automations**에 해당하는 Antigravity 기능이다. Antigravity는 이벤트 훅 대신 **저장된 프롬프트 파이프라인**으로 반복 작업을 자동화한다.

## 사전 준비

- Antigravity 기본 사용법 숙지 (`01_agentic_coding_tool/02_editor_native/antigravity-guide.md` 참고)
- YAML 프론트매터 기본 이해
- Gemini CLI 설치 완료

---

## 전체 흐름 한눈에 보기

매번 "1단계 이렇게 하고, 2단계 저렇게 하고..."를 채팅창에 타이핑하는 대신, 워크플로우 파일 하나에 전체 절차를 저장한다. 이후 `/워크플로우명`으로 호출하면 에이전트가 파일에 정의된 순서대로 자율 실행한다.

1. **개념 이해** — Workflows의 역할과 Rules와의 차이
2. **기본 워크플로우 작성** — YAML 프론트매터 형식과 단계 정의
3. **고급 파이프라인** — Skills 연결, 조건 분기, 전역 vs 프로젝트 스코프
4. **실전 예시** — 개발 사이클 전체 자동화

---

## Phase 1: Workflows 개념 이해

### 목표

Antigravity Workflows가 해결하는 문제를 이해하고, Rules와의 역할 차이를 파악한다.

### 단계별 구현

#### Step 1.1 — Workflows가 해결하는 문제

> **💡 개념 설명: Workflow vs 일반 채팅**
>
> "기능 개발 → 테스트 → 문서화 → 코드 리뷰" 사이클을 매번 수동으로 지시하면 단계를 빠뜨리거나 순서가 바뀌기 쉽다. Workflow는 이 절차를 파일로 고정하고, 호출 시마다 동일한 순서로 실행되도록 보장한다.
>
> **핵심 한 줄:** Workflow = 반복되는 멀티스텝 작업의 표준 절차서

#### Step 1.2 — 파일 위치와 스코프

```
전역 워크플로우:    ~/.gemini/antigravity/workflows/
프로젝트 워크플로우: <project-root>/.agent/workflows/
```

| 스코프 | 위치 | 적합한 용도 |
|--------|------|-------------|
| 전역 | `~/.gemini/antigravity/workflows/` | 모든 프로젝트에서 공통 사용 |
| 프로젝트 | `.agent/workflows/` | 특정 프로젝트의 고유 절차 |

우선순위: 프로젝트 워크플로우가 전역 워크플로우보다 우선 적용된다.

#### Step 1.3 — Workflows vs Rules vs Skills 차이

| 구성 요소 | 역할 | 실행 방식 |
|-----------|------|-----------|
| **Rules** | 에이전트가 항상 따르는 행동 규칙 | 자동 (별도 호출 불필요) |
| **Workflows** | 멀티스텝 작업의 표준 절차 | 명시적 (`/워크플로우명`) |
| **Skills** | 재사용 가능한 단일 작업 단위 | 명시적 또는 Workflow에서 호출 |

> **💡 개념 설명: Rules로 Hooks 흉내 내기**
>
> Antigravity에는 Claude Code의 `PostToolUse` 같은 이벤트 훅이 없다. 대신 Rules에 "파일을 수정하면 반드시 린트를 실행한다"처럼 행동 지침을 작성하면, 에이전트가 스스로 이 규칙을 따른다. 완전 자동화보다는 느슨하지만, 에이전트 판단으로 적용된다.
>
> **핵심 한 줄:** Rules = 이벤트 훅을 대체하는 AI 행동 지침

#### Step 1.4 — 워크플로우 기본 파일 형식

```yaml
---
name: 워크플로우-이름
description: 이 워크플로우가 하는 일 설명
---
1. 첫 번째 단계 설명
2. 두 번째 단계 설명
3. 세 번째 단계 설명
```

파일 확장자: `.md` (Markdown)
호출 방법: 채팅창에서 `/워크플로우-이름` 입력

### 체크포인트

Workflows, Rules, Skills의 역할 차이를 설명할 수 있고, 워크플로우 파일의 기본 위치와 호출 방법을 알고 있는가?

---

## Phase 2: 기본 워크플로우 작성

### 목표

실제 동작하는 첫 번째 워크플로우 파일을 작성하고 `/명령어`로 실행되는지 확인한다.

### 단계별 구현

#### Step 2.1 — 첫 번째 워크플로우: 커밋 메시지 생성

```markdown
---
name: smart-commit
description: 스테이징된 변경사항을 분석하여 Conventional Commits 형식의 커밋 메시지를 생성하고 커밋한다
---
1. `git diff --staged` 실행하여 스테이징된 변경사항 확인
2. 변경사항의 의도와 범위를 분석
3. Conventional Commits 형식으로 커밋 메시지 작성
   - 형식: `type(scope): 한국어 설명`
   - type: feat / fix / docs / refactor / test / chore
   - 제목 50자 이내
4. 사용자에게 메시지 확인 요청
5. 승인 시 `git commit -m "메시지"` 실행
```

저장 위치: `.agent/workflows/smart-commit.md`

실행:
```
/smart-commit
```

#### Step 2.2 — 두 번째 워크플로우: 단위 테스트 생성

```markdown
---
name: generate-unit-tests
description: 지정된 파일의 모든 함수에 대한 pytest 단위 테스트를 생성한다
---
1. 대상 파일 경로 확인 (인자로 전달되지 않으면 사용자에게 요청)
2. 파일의 모든 함수와 클래스 분석
   - 함수 시그니처, 파라미터 타입, 반환값
   - 존재하는 docstring 확인
3. 각 함수에 대한 테스트 케이스 설계
   - 정상 케이스 (happy path)
   - 경계값 케이스 (edge case)
   - 예외 케이스 (exception case)
4. `tests/` 디렉토리에 `test_[원본파일명].py` 생성
5. 생성된 테스트 실행하여 동작 확인
6. 실패한 테스트가 있으면 원인 분석 후 수정
```

저장 위치: `.agent/workflows/generate-unit-tests.md`

실행:
```
/generate-unit-tests src/utils/auth.py
```

#### Step 2.3 — 인자 전달 방법

워크플로우는 호출 시 인자를 받을 수 있다. 파일 내에서 인자를 명시적으로 처리하도록 단계에 서술한다:

```markdown
---
name: refactor-file
description: 지정된 파일을 SOLID 원칙에 따라 리팩토링한다
---
1. 인자로 전달된 파일을 읽고 현재 구조 파악
   (인자 없이 호출 시: "어떤 파일을 리팩토링할까요?" 질문)
2. 코드 품질 문제점 파악
   - 단일 책임 원칙 위반 여부
   - 함수/클래스 길이 확인 (함수 20줄, 클래스 200줄 초과 검사)
   - 중복 코드 탐지
3. 리팩토링 계획 수립 및 사용자에게 계획 공유
4. 승인 후 단계별 리팩토링 실행
5. 기존 테스트가 있으면 실행하여 동작 검증
6. 리팩토링 전후 변경사항 요약 보고
```

### 체크포인트

`/generate-unit-tests src/main.py` 실행 시 에이전트가 `tests/test_main.py` 파일을 생성하면 성공이다.

---

## Phase 3: 고급 파이프라인 구성

### 목표

Skills 연결, Rules와의 조합, 다중 단계 조건 분기를 활용하여 복잡한 워크플로우를 구성한다.

### 단계별 구현

#### Step 3.1 — Rules와 Workflows 조합: 자동 품질 관리

Rules 파일에 파일 수정 후 자동 실행 지침을 추가한다:

```markdown
# .agent/rules.md

## 코드 수정 후 반드시 수행

파일을 수정하거나 새로 생성한 후에는 다음 단계를 반드시 수행한다:

1. **린트 검사**: `npm run lint [파일명]` 또는 `flake8 [파일명]` 실행
2. **타입 검사**: TypeScript 파일이면 `tsc --noEmit` 실행
3. **관련 테스트 실행**: 수정된 파일과 연관된 테스트 파일 자동 탐색 후 실행
4. **오류가 있으면**: 자동으로 수정하고 다시 검사 (무한 루프 방지: 최대 3회)

## 새 함수/클래스 생성 시

새로운 public 함수 또는 클래스를 생성할 때는:
- 반드시 docstring을 포함한다
- 해당 함수의 단위 테스트를 함께 작성한다
- 타입 어노테이션을 추가한다
```

#### Step 3.2 — Skills 연결 파이프라인

여러 Skills를 순서대로 연결하는 풀 사이클 워크플로우:

```markdown
---
name: full-cycle
description: 기능 요청을 받아 코드 구현, 테스트 작성, 문서화, 리뷰까지 전체 사이클을 자동 실행한다
---
1. 구현할 기능 설명 확인 (인자 또는 대화로 입력)
2. **기존 코드 품질 점검**: 관련 파일들의 현재 상태 분석
   - @skill:code-review 실행하여 기존 코드 이슈 파악
   - 발견된 이슈가 구현에 영향을 주면 먼저 수정
3. **기능 구현**:
   - 필요한 파일 및 함수 설계
   - TDD 방식: 테스트 먼저 작성 후 구현
   - Rules에 따라 린트 및 타입 검사 자동 실행
4. **테스트 자동 생성**: @skill:generate-tests 실행
   - 정상/경계/예외 케이스 모두 포함
   - 전체 테스트 스위트 실행하여 기존 테스트 통과 여부 확인
5. **문서화**: @skill:write-docs 실행
   - 새로 추가된 함수에 docstring 추가
   - CHANGELOG.md 업데이트
6. **최종 리뷰**: @skill:code-review 재실행
   - 구현된 코드의 품질 최종 확인
   - 이슈가 있으면 수정
7. 완료 보고: 구현 내용, 테스트 결과, 변경 파일 목록 요약
```

#### Step 3.3 — 조건 분기 워크플로우

파일 유형이나 상태에 따라 다른 경로를 실행하는 워크플로우:

```markdown
---
name: smart-review
description: 파일 유형을 감지하여 적절한 리뷰 전략을 자동 선택한다
---
1. 대상 파일 또는 PR 번호 확인
2. **파일 유형 감지**:
   - Python 파일: PEP 8, 타입 힌트, 도큐멘테이션 품질 검사
   - TypeScript/JavaScript: ESLint 규칙, 타입 안전성, React 패턴 (해당 시)
   - SQL: 인덱스 사용, N+1 쿼리, 인젝션 취약점 검사
   - 설정 파일 (YAML/JSON): 스키마 검증, 시크릿 노출 여부 확인
3. **보안 검사** (모든 파일 유형):
   - 하드코딩된 API 키, 비밀번호, 토큰 탐색
   - 취약한 암호화 알고리즘 사용 여부
4. 유형별 리뷰 결과를 통합하여 심각도별 분류
   - CRITICAL: 즉시 수정 필요
   - HIGH: PR 병합 전 수정 권장
   - MEDIUM/LOW: 향후 개선 사항
5. 리뷰 보고서를 `reviews/` 디렉토리에 저장
```

#### Step 3.4 — 전역 워크플로우: 모든 프로젝트 공통

```markdown
---
name: standup
description: 오늘의 Git 활동을 기반으로 스탠드업 미팅 보고서를 자동 생성한다
---
1. 어제부터 오늘까지의 Git 커밋 이력 조회
   - `git log --since="yesterday" --author="현재 사용자" --oneline`
2. 변경된 파일과 커밋 메시지 분석
3. 다음 형식으로 스탠드업 보고서 작성:

   **어제 한 일:**
   - (커밋 분석 기반 자동 작성)

   **오늘 할 일:**
   - (현재 열린 이슈나 브랜치 기반 추론)

   **블로커:**
   - (실패한 테스트, 미해결 머지 충돌 등)

4. 보고서를 클립보드에 복사 또는 파일로 저장
```

저장 위치: `~/.gemini/antigravity/workflows/standup.md` (전역)

### 체크포인트

`/full-cycle "사용자 로그인 기능 구현"` 실행 시 에이전트가 코드 구현부터 테스트, 문서화까지 자동으로 수행하면 성공이다.

---

## Phase 4: 실전 예시

### 목표

실제 팀에서 운영할 수 있는 완성된 워크플로우 세트를 구성한다.

### 단계별 구현

#### Step 4.1 — PR 준비 워크플로우

```markdown
---
name: pr-ready
description: 현재 브랜치를 PR 제출 가능한 상태로 만든다
---
1. 현재 브랜치와 base 브랜치(main/develop) 차이 분석
2. **자동 정리 작업**:
   - 디버그용 console.log / print 문 제거
   - TODO 주석 목록화
   - 미사용 import 정리
3. **품질 검사**:
   - 전체 테스트 스위트 실행
   - 린트 오류 수정
   - 타입 오류 수정
4. **문서 업데이트**:
   - 변경된 함수의 docstring 검토 및 보완
   - CHANGELOG.md에 변경사항 추가
5. **PR 초안 작성**:
   - 제목: 주요 변경사항 한 줄 요약
   - 본문: 변경 이유, 주요 변경사항, 테스트 방법
6. `git push origin [현재 브랜치]` 실행
7. PR 초안 제공 (GitHub CLI로 생성 여부 확인)
```

#### Step 4.2 — 디렉토리 구조 추천

```
.agent/
├── rules.md                    # 에이전트 행동 규칙 (항상 적용)
├── workflows/
│   ├── smart-commit.md         # /smart-commit
│   ├── generate-unit-tests.md  # /generate-unit-tests
│   ├── refactor-file.md        # /refactor-file
│   ├── full-cycle.md           # /full-cycle
│   ├── smart-review.md         # /smart-review
│   └── pr-ready.md             # /pr-ready
└── skills/
    ├── code-review.md          # @skill:code-review
    ├── generate-tests.md       # @skill:generate-tests
    └── write-docs.md           # @skill:write-docs

~/.gemini/antigravity/workflows/
├── standup.md                  # /standup (전역)
└── weekly-review.md            # /weekly-review (전역)
```

#### Step 4.3 — 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| 워크플로우가 목록에 안 보임 | 파일 경로 오류 | `.agent/workflows/` 또는 `~/.gemini/antigravity/workflows/` 확인 |
| 단계가 건너뜀 | 설명이 모호함 | 각 단계에 구체적인 명령어 또는 검증 기준 명시 |
| Skills 호출 실패 | Skills 파일 없음 | `.agent/skills/` 디렉토리에 스킬 파일 생성 |
| Rules가 적용 안 됨 | rules.md 위치 오류 | `.agent/rules.md` (프로젝트 루트 기준) 확인 |
| 무한 루프 발생 | 자동 수정 반복 | Rules에 최대 반복 횟수 명시 ("최대 3회") |

### 체크포인트

`/pr-ready` 실행 시 테스트 통과, 린트 정리, PR 초안 생성이 모두 완료되면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- Workflows = 반복 절차를 YAML 프론트매터로 저장한 표준 프롬프트 파이프라인
- Rules = 이벤트 훅 대신 에이전트 행동을 자동 제어하는 지침
- 전역(`~/.gemini/antigravity/workflows/`) vs 프로젝트(`.agent/workflows/`) 스코프 분리
- Skills를 Workflow에서 `@skill:이름`으로 연결하여 모듈식 파이프라인 구성

### 실용적인 워크플로우 아이디어

| 워크플로우 | 설명 |
|-----------|------|
| `/smart-commit` | 스테이징 변경사항 분석 후 Conventional Commits 형식 커밋 |
| `/generate-unit-tests [파일]` | 파일의 모든 함수에 대한 테스트 자동 생성 |
| `/pr-ready` | PR 제출 전 품질 검사 및 자동 정리 |
| `/full-cycle [기능]` | 기능 구현부터 테스트, 문서화까지 전체 사이클 |
| `/standup` | 오늘의 커밋 기반 스탠드업 보고서 자동 생성 |
| `/smart-review [파일]` | 파일 유형 감지 후 적절한 리뷰 자동 실행 |

---

## 다른 도구와의 비교표

| 기능 | Antigravity Workflows | Claude Code Hooks | Codex CLI 프로파일 | Cursor Automations |
|------|----------------------|--------------------|---------------------|---------------------|
| 트리거 방식 | 명시적 채팅 명령어 | 이벤트 자동 트리거 | 스크립트/CI 호출 | 이벤트 자동 트리거 |
| 설정 형식 | Markdown + YAML 프론트매터 | JSON | TOML | GUI 설정 |
| 코드 차단 기능 | Rules로 간접 구현 | PreToolUse 종료 코드 | 승인 정책 | 제한적 |
| 모듈화 | Skills 연결 | 훅 배열 조합 | 프로파일 분기 | Prompt Files |
| 외부 서비스 연동 | 에이전트 도구 활용 | HTTP 훅 직접 지원 | 셸 명령으로 연동 | Webhook, Slack 기본 지원 |
| 비대화형 실행 | 미지원 | GitHub Actions 지원 | 핵심 기능 | 백그라운드 에이전트 |
| 학습 난이도 | 낮음 (Markdown 작성) | 중간 (JSON + 셸) | 낮음 (TOML + CLI) | 낮음 (GUI) |
