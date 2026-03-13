# Gemini CLI Agent Skills 작성 가이드

## 학습 목표

Agent Skill이 GEMINI.md Rule과 어떻게 다른지 이해하고, SKILL.md를 처음부터 작성하여 Gemini CLI가 적절한 상황에서 자동으로 활성화하도록 구성할 수 있다.

## 사전 준비

- Gemini CLI 기본 사용법 숙지 (`01_agentic_coding_tool/01_cli_driven/gemini-cli-guide.md` 참고)
- 텍스트 에디터
- (선택) Node.js — 스크립트 번들링 실습 시 필요

---

## 전체 흐름 한눈에 보기

GEMINI.md Rule은 항상 컨텍스트에 로드되어 모든 대화에 적용된다. 반면 Skill은 사용자 요청이 특정 설명과 매칭될 때만 로드된다. 덕분에 컨텍스트 낭비 없이 전문화된 지식을 필요할 때만 주입할 수 있다.

1. **개념 이해** — Rule vs Skill 차이와 적합한 사용 시점
2. **첫 Skill 만들기** — SKILL.md 구조와 frontmatter 작성
3. **리소스 추가** — 스크립트, 참고 자료 번들링
4. **Skill 관리** — 목록 확인, 활성화, 우선순위

---

## Phase 1: Rule vs Skill — 언제 무엇을 써야 하나

### 목표

GEMINI.md Rule과 Agent Skill의 차이를 이해하고 각 상황에 맞는 선택을 내릴 수 있다.

### 단계별 구현

#### Step 1.1 — Rule과 Skill 비교

> **💡 개념 설명: 왜 Skill이 필요한가?**
>
> GEMINI.md에 모든 전문 지식을 넣으면 어떨까? Python 코딩 가이드, Django 배포 절차, AWS 설정 방법, 데이터베이스 마이그레이션 체크리스트... 금세 수천 줄이 된다. 매번 모든 내용이 컨텍스트에 로드되면 비용이 증가하고 응답 품질이 저하된다.
>
> Skill은 "이 전문 지식이 필요한 순간"을 AI 스스로 판단하게 한다. "배포해줘"라고 하면 Django 배포 Skill만 활성화되고, "코드 리뷰해줘"라고 하면 코드 리뷰 Skill만 활성화된다.
>
> **핵심 한 줄:** Rule = 항상 적용되는 기본 원칙 / Skill = 필요할 때만 꺼내는 전문 매뉴얼

| 구분 | GEMINI.md Rule | Agent Skill |
|------|----------------|-------------|
| 로드 시점 | 항상 (세션 시작 시) | 요청 매칭 시에만 |
| 적합한 내용 | 코딩 스타일, 커밋 규칙 등 보편적 원칙 | 배포 절차, 특정 도구 사용법 등 전문 지식 |
| 컨텍스트 영향 | 항상 소비 | 필요 시에만 소비 |
| 파일 형식 | Markdown | SKILL.md + 추가 파일들 |

#### Step 1.2 — Skill이 적합한 상황 식별

다음 기준 중 하나라도 해당하면 Skill로 만드는 것이 좋다:

- 특정 작업(배포, 테스트, 리뷰 등)에만 필요한 지식
- 단계별 절차가 있는 워크플로우
- 특정 명령어나 스크립트가 포함된 작업
- "항상 켜두기엔 너무 긴" 내용

### 체크포인트

팀의 코딩 컨벤션은 Rule, AWS 배포 절차는 Skill로 구분할 수 있는가?

---

## Phase 2: 첫 번째 Skill 만들기

### 목표

코드 리뷰 Skill을 처음부터 작성하고 Gemini CLI에서 인식되는지 확인한다.

### 단계별 구현

#### Step 2.1 — Skill 디렉토리 생성

사용자 글로벌 Skill을 만들어본다:

```bash
# ~/.gemini/skills/code-reviewer/

mkdir -p ~/.gemini/skills/code-reviewer
```

프로젝트 전용 Skill은 아래에 생성한다:

```bash
# <project-root>/.gemini/skills/code-reviewer/

mkdir -p .gemini/skills/code-reviewer
```

#### Step 2.2 — SKILL.md 작성

> **💡 개념 설명: SKILL.md frontmatter**
>
> SKILL.md 파일 최상단의 `---`로 감싸진 블록이 frontmatter이다. Gemini는 이 메타데이터의 `description`을 읽어서 사용자 요청과 매칭한다. description을 잘 쓸수록 자동 활성화 정확도가 높아진다.
>
> **핵심 한 줄:** description = "이런 요청이 오면 나를 써" 라는 신호

```markdown
# ~/.gemini/skills/code-reviewer/SKILL.md

---
name: code-reviewer
description: >
  코드 품질 분석 및 개선 제안 전문가.
  사용자가 "코드 리뷰", "코드 검토", "리팩토링 제안",
  "PR 리뷰" 등을 요청할 때 활성화된다.
---

# Code Reviewer Skill

당신은 시니어 소프트웨어 엔지니어로서 코드 품질 분석 전문가이다.

## 리뷰 관점

코드를 리뷰할 때 다음 순서로 분석한다:

1. **가독성**: 변수/함수명이 의도를 명확히 전달하는가?
2. **단일 책임**: 각 함수/클래스가 하나의 역할만 하는가?
3. **에러 처리**: 예외 상황이 적절히 처리되는가?
4. **성능**: 불필요한 연산이나 N+1 쿼리가 있는가?
5. **보안**: 입력 검증, SQL 인젝션 등 취약점이 있는가?

## 피드백 형식

리뷰 결과를 다음 형식으로 작성한다:

### ✅ 잘된 점
- (구체적인 칭찬)

### 🔧 개선 제안
- **[파일명:라인번호]** 문제 설명
  - 현재 코드: `기존 코드`
  - 제안: `개선된 코드`
  - 이유: 개선이 필요한 근거

### ⚡ 우선순위 높은 수정사항
- (즉시 수정이 필요한 항목)
```

#### Step 2.3 — Skill 인식 확인

Gemini CLI를 재시작하고 Skill이 목록에 나타나는지 확인한다:

```bash
/skills list
```

예상 출력:

```
Available Skills:
  ● code-reviewer — 코드 품질 분석 및 개선 제안 전문가
```

### 체크포인트

`/skills list`에 `code-reviewer`가 표시되고, "이 함수 코드 리뷰해줘"라고 했을 때 Skill이 자동 활성화되면 성공이다.

---

## Phase 3: 스크립트와 리소스 번들링

### 목표

SKILL.md만이 아니라 실행 가능한 스크립트와 참고 자료를 Skill에 포함시켜 더 강력한 자동화를 구현한다.

### 단계별 구현

#### Step 3.1 — Skill 디렉토리 구조 이해

```
~/.gemini/skills/code-reviewer/
├── SKILL.md           ← 필수: Skill 정의 및 지침
├── scripts/           ← 선택: 실행 스크립트
│   └── run-lint.sh
├── resources/         ← 선택: 참고 자료
│   └── checklist.md
└── assets/            ← 선택: 예시 파일
    └── bad-good-examples.md
```

#### Step 3.2 — 보조 스크립트 추가

```bash
# ~/.gemini/skills/code-reviewer/scripts/run-lint.sh

#!/bin/bash
# 코드 리뷰 전 린트 실행 스크립트

echo "=== ESLint 실행 중 ==="
npx eslint . --format=compact 2>&1

echo "=== Prettier 검사 중 ==="
npx prettier --check . 2>&1

echo "=== 완료 ==="
```

```bash
# 실행 권한 부여
chmod +x ~/.gemini/skills/code-reviewer/scripts/run-lint.sh
```

#### Step 3.3 — SKILL.md에서 스크립트 참조

스크립트를 SKILL.md 지침에서 참조한다:

```markdown
# ~/.gemini/skills/code-reviewer/SKILL.md (일부 발췌)

## 리뷰 절차

1. `scripts/run-lint.sh`를 실행하여 자동 감지 가능한 오류를 먼저 수집한다
2. 린트 결과를 바탕으로 코드 품질 분석을 진행한다
3. 위 리뷰 형식에 따라 피드백을 작성한다

## 참고 리소스

- `resources/checklist.md`: 리뷰 체크리스트 항목
```

#### Step 3.4 — 참고 자료 파일 추가

```markdown
# ~/.gemini/skills/code-reviewer/resources/checklist.md

# 코드 리뷰 체크리스트

## 필수 확인 항목
- [ ] 함수 길이가 50줄 이내인가?
- [ ] 중첩 depth가 3 이하인가?
- [ ] 매직 넘버(하드코딩된 숫자)가 없는가?
- [ ] 민감한 정보가 코드에 직접 포함되지 않았는가?

## Python 특화
- [ ] type hints가 있는가?
- [ ] docstring이 작성되었는가?

## JavaScript/TypeScript 특화
- [ ] `any` 타입 사용이 없는가?
- [ ] Promise 에러 처리가 있는가?
```

### 체크포인트

"PR 리뷰해줘"라고 요청 시 Skill이 활성화되고, 스크립트가 실행된 결과를 바탕으로 리뷰가 제공되면 성공이다.

---

## Phase 4: Skill 관리와 우선순위

### 목표

여러 Skill을 체계적으로 관리하고 같은 이름의 Skill이 충돌할 때 우선순위를 이해한다.

### 단계별 구현

#### Step 4.1 — Skill 관리 명령어

| 명령어 | 설명 |
|--------|------|
| `/skills list` | 사용 가능한 Skill 전체 목록 |
| `/skills show code-reviewer` | 특정 Skill 상세 정보 |
| `/skills enable code-reviewer` | Skill 활성화 |
| `/skills disable code-reviewer` | Skill 비활성화 (세션 내) |
| `/skills reload` | Skill 파일 변경 후 다시 로드 |

#### Step 4.2 — Skill 우선순위

같은 이름의 Skill이 여러 위치에 있을 경우 아래 순서로 우선한다:

```
워크스페이스 Skill  >  사용자 Skill  >  확장 Skill
<project>/.gemini/skills/  >  ~/.gemini/skills/  >  확장 프로그램
```

**활용 팁**: 글로벌 Skill을 프로젝트에서 재정의할 수 있다. 예를 들어 전역 `code-reviewer`가 있어도 특정 프로젝트에서는 `.gemini/skills/code-reviewer/`를 만들어 Python 특화 버전으로 덮어쓸 수 있다.

#### Step 4.3 — 공식 Skills 설치

검증된 공식 Skill을 설치하는 방법이다:

```bash
# Firebase 관련 Skills 설치
npx skills.sh install firebase-basics
npx skills.sh install firebase-auth-basics
npx skills.sh install firebase-firestore-basics

# 설치 확인
/skills list
```

### 체크포인트

프로젝트별 Skill이 글로벌 Skill보다 우선 적용되는지 확인했는가?

---

## 마무리

### 이 가이드에서 배운 것

- Rule vs Skill: 항상 적용 vs 요청 시 활성화
- SKILL.md frontmatter의 `description`이 자동 매칭의 핵심
- 스크립트(`scripts/`)와 참고자료(`resources/`)를 번들링하는 방법
- 워크스페이스 Skill이 글로벌 Skill보다 우선한다

### 효과적인 description 작성 팁

```markdown
# ❌ 나쁜 예 (너무 모호함)
description: 코드와 관련된 도움을 제공한다.

# ✅ 좋은 예 (구체적인 트리거 키워드 포함)
description: >
  코드 품질 분석 전문가.
  "코드 리뷰", "PR 리뷰", "리팩토링 제안", "코드 검토" 요청 시 활성화.
```

### 다음 단계

- `05_workflow` — 반복 작업을 재사용 가능한 슬래시 명령어로 만들기
- 여러 Skill을 연결하는 복합 워크플로우 구성
