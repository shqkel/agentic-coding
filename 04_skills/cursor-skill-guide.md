# Cursor "Skills" 작성 가이드 — Notepads와 Agent Requested Rules

## 학습 목표

Cursor에는 명시적인 "Skills" 시스템이 없지만, Notepads와 Agent Requested Rules 두 가지 기능을 조합하여 다른 도구의 Skills와 동일한 효과를 낼 수 있다. 각 기능의 역할과 적합한 사용 시점을 이해하고, 실제로 작성하여 활용할 수 있다.

## 사전 준비

- Cursor 에디터 설치 및 기본 사용법 숙지
- Cursor Pro 또는 Team 플랜 (Notepads 기능)
- 작업할 프로젝트 디렉토리

---

## 전체 흐름 한눈에 보기

Cursor의 Rule은 `.cursor/rules/` 디렉토리의 `.mdc` 파일로 관리된다. "Skills"에 해당하는 기능은 두 가지 방식으로 구현한다:

- **Notepads**: 복잡한 프롬프트를 영구 저장하고 `@notepad:이름`으로 참조하는 방식
- **Agent Requested Rules**: 에이전트가 요청 내용에 따라 스스로 로드 여부를 판단하는 조건부 Rule

1. **개념 이해** — Cursor에서 Skills 개념이 어떻게 구현되는지
2. **Notepads 활용** — 재사용 가능한 프롬프트 저장 및 공유
3. **Agent Requested Rules** — 조건부 자동 로드 규칙 작성
4. **실전 예시** — 실무에서 바로 쓸 수 있는 설정 작성

---

## Phase 1: Cursor에서 Skills 개념 이해

### 목표

Cursor의 Rule 시스템에서 Skills에 해당하는 부분을 정확히 파악하고, 각 기능의 특성을 이해한다.

### 단계별 구현

#### Step 1.1 — Cursor Rule의 4가지 유형

> **💡 개념 설명: Cursor Rule 유형과 Skills의 관계**
>
> Cursor의 `.cursor/rules/*.mdc` 파일은 `alwaysApply`와 `globs` 설정에 따라 4가지 유형으로 동작한다. 이 중 **Agent Requested** 유형이 다른 도구의 Skills와 가장 유사하다. 에이전트가 description을 읽고 현재 요청에 필요한지 스스로 판단해서 로드한다.
>
> **핵심 한 줄:** Agent Requested Rule = "에이전트가 필요하다고 판단할 때만 자동 로드되는 전문 지침"

| 유형 | alwaysApply | globs | 동작 | Skills 유사도 |
|------|-------------|-------|------|--------------|
| Always | true | 없음 | 항상 로드 | ❌ (Rule과 동일) |
| Auto Attached | false | 패턴 있음 | 해당 파일 작업 시 자동 로드 | 부분적 |
| Agent Requested | false | 없음 | 에이전트가 description 보고 판단 | ✅ (Skills과 가장 유사) |
| Manual | false | 없음, description 없음 | `@rule이름`으로 수동 참조 | ✅ (Notepads와 유사) |

#### Step 1.2 — Notepads vs Agent Requested Rules 비교

| 구분 | Notepads | Agent Requested Rules |
|------|----------|-----------------------|
| 저장 위치 | Cursor UI 내부 | `.cursor/rules/*.mdc` |
| 버전 관리 | 불가 (UI 전용) | 가능 (파일로 저장) |
| 팀 공유 | Cursor Team 기능 | git으로 공유 |
| 호출 방식 | `@notepad:이름` (수동) | 에이전트 자동 판단 |
| 적합한 용도 | 개인 작업, 복잡한 one-off 프롬프트 | 팀 공유, 반복적 작업 패턴 |
| 인자 전달 | 없음 | 없음 |

### 체크포인트

Notepads와 Agent Requested Rules 중 팀 공유가 필요한 작업에는 무엇을 써야 하는가?

---

## Phase 2: Notepads — 재사용 가능한 프롬프트 저장소

### 목표

Cursor Notepads에 복잡한 프롬프트를 저장하고 `@notepad:이름`으로 참조하는 방법을 익힌다.

### 단계별 구현

#### Step 2.1 — Notepad 생성

1. Cursor 에디터 좌측 사이드바에서 **Notepads** 탭을 열거나
2. `Cmd+Shift+P` → "Notepads: New Notepad" 실행

또는 Chat 창에서 직접:
```
@notepad:새이름 만들어줘
```

#### Step 2.2 — Notepad 내용 작성

> **💡 개념 설명: Notepad 활용 패턴**
>
> Notepad는 긴 프롬프트를 매번 타이핑하지 않아도 되도록 저장해두는 공간이다. "이 컨텍스트를 항상 기억하면서 작업해줘"라는 배경 정보나, "이 형식으로 출력해줘"라는 상세한 형식 지정 등에 유용하다.
>
> **핵심 한 줄:** Notepad = "자주 쓰는 복잡한 프롬프트를 이름 붙여 저장한 것"

```markdown
<!-- Notepad 이름: api-review-context -->

# API 리뷰 컨텍스트

이 프로젝트의 API 설계 기준:

## 기술 스택
- Runtime: Node.js + Express
- DB: PostgreSQL (Prisma ORM)
- Auth: JWT (access token 15분, refresh token 7일)
- 응답 형식: { data, meta, error } 표준 구조

## 우리 팀 API 규칙
- URL은 /api/v1/ 접두사 필수
- 페이지네이션은 cursor-based (offset 금지)
- 날짜는 ISO 8601 형식 (UTC)
- 에러 코드는 SCREAMING_SNAKE_CASE

## 리뷰 시 중점 확인 사항
1. 위 규칙 준수 여부
2. 트랜잭션 처리 여부 (복수 DB 작업 시)
3. 응답 시간 최적화 (N+1 쿼리 등)
4. 입력 검증 (Joi/Zod schema 사용 여부)
```

#### Step 2.3 — Notepad 참조 방법

Chat 창에서 `@` 기호로 Notepad를 참조한다:

```
@notepad:api-review-context 이 파일의 API 엔드포인트를 리뷰해줘
```

여러 Notepad를 동시에 참조할 수 있다:

```
@notepad:api-review-context @notepad:security-checklist 이 코드 검토해줘
```

파일과 함께 참조:

```
@notepad:api-review-context @src/routes/users.ts 리뷰해줘
```

#### Step 2.4 — 팀 공유 (Cursor Team)

Cursor Team 플랜에서는 Notepads를 팀과 공유할 수 있다:

1. Notepad 편집 화면에서 **Share** 버튼 클릭
2. 팀 멤버에게 공유 링크 전달

> 팀 공유 없이 버전 관리가 필요하다면 Notepad 대신 Agent Requested Rules를 사용하는 것을 권장한다.

### 체크포인트

Notepad를 생성하고 `@notepad:이름` 형식으로 Chat에서 참조했을 때 해당 컨텍스트가 적용되면 성공이다.

---

## Phase 3: Agent Requested Rules — 조건부 자동 로드 지침

### 목표

`.cursor/rules/` 디렉토리에 Agent Requested 유형의 `.mdc` 파일을 작성하여, 에이전트가 요청 내용에 따라 자동으로 로드하도록 구성한다.

### 단계별 구현

#### Step 3.1 — `.mdc` 파일 기본 구조

> **💡 개념 설명: .mdc 파일 frontmatter**
>
> `.mdc` 파일은 YAML frontmatter와 Markdown 본문으로 구성된다. Agent Requested 유형으로 만들려면 `alwaysApply: false`로 설정하고 `globs`는 비워두며, `description`에 트리거 조건을 명확히 작성한다. 에이전트는 이 description을 읽고 현재 요청에 이 Rule이 필요한지 판단한다.
>
> **핵심 한 줄:** description = 에이전트가 로드 여부를 판단하는 기준 설명

```markdown
<!-- .cursor/rules/django-deploy.mdc -->

---
description: Django 배포 체크리스트 및 AWS EB 배포 절차. '배포', 'deploy', 'AWS 배포', '프로덕션 올리기' 요청 시 에이전트가 로드
globs: ""
alwaysApply: false
---

# Django 배포 규칙

배포 작업 시 반드시 아래 절차를 따른다.

## 배포 전 체크리스트

1. 모든 테스트 통과 확인: `python manage.py test`
2. 배포 설정 검사: `python manage.py check --deploy`
3. 미적용 마이그레이션 확인: `python manage.py showmigrations`
4. requirements.txt 최신화: `pip freeze > requirements.txt`

## 배포 명령어

```bash
# Elastic Beanstalk 배포
eb deploy

# 상태 확인
eb status && eb health
```

## 롤백

배포 실패 시: `eb deploy --version previous`
```

#### Step 3.2 — Auto Attached Rule (파일 패턴 기반)

특정 파일 작업 시 자동 로드되는 Rule은 Skill의 일부 특성을 가진다:

```markdown
<!-- .cursor/rules/react-component.mdc -->

---
description: React 컴포넌트 작성 가이드
globs: "src/components/**/*.tsx,src/components/**/*.jsx"
alwaysApply: false
---

# React 컴포넌트 규칙

`src/components/` 하위 파일 작업 시 자동으로 적용된다.

## 컴포넌트 구조

```tsx
// 권장 구조
interface Props {
  // 타입 정의 필수
}

const ComponentName: React.FC<Props> = ({ prop1, prop2 }) => {
  // 1. Hooks (useState, useEffect 등)
  // 2. 파생 상태 계산
  // 3. 이벤트 핸들러
  // 4. JSX 반환
  return <div>{/* ... */}</div>;
};

export default ComponentName;
```

## 규칙
- Props 타입 정의 필수 (interface 사용)
- 컴포넌트당 1개 파일
- 스타일은 CSS Module 또는 Tailwind만 사용
- 불필요한 useEffect 지양 (파생 상태는 useMemo 활용)
```

#### Step 3.3 — 여러 Rule 조합 전략

```bash
# 권장 .cursor/rules/ 구조
.cursor/rules/
├── always/
│   └── coding-style.mdc      # alwaysApply: true — 항상 적용
├── auto/
│   ├── react-component.mdc   # globs: "src/components/**" — 파일 감지
│   ├── api-routes.mdc        # globs: "src/api/**"
│   └── test-files.mdc        # globs: "**/*.test.*"
└── agent/
    ├── deploy.mdc            # description 기반 — 배포 요청 시
    ├── code-review.mdc       # description 기반 — 리뷰 요청 시
    └── db-migration.mdc      # description 기반 — 마이그레이션 요청 시
```

> **주의**: Cursor는 파일명이나 디렉토리 구조로 유형을 구분하지 않는다. 디렉토리 구조는 사람이 관리하기 위한 구성이고, 실제 동작은 frontmatter의 `alwaysApply`와 `globs` 값으로 결정된다.

#### Step 3.4 — Cursor 설정에서 Rules 확인

`Cmd+Shift+P` → "Cursor: View Rules" 또는 Cursor Settings → Rules에서:

- 현재 활성화된 Rules 목록 확인
- 각 Rule의 적용 여부 수동 토글

### 체크포인트

Agent Requested Rule을 작성하고 "배포해줘"라고 요청했을 때 에이전트가 해당 Rule을 로드하여 배포 절차를 안내해주면 성공이다.

---

## Phase 4: 실전 예시

### 목표

실무에서 즉시 활용할 수 있는 Notepad 1개와 Agent Requested Rule 2개를 작성한다.

### 예시 1: 코드 리뷰 Agent Requested Rule

```markdown
<!-- .cursor/rules/code-review.mdc -->

---
description: 코드 품질 분석 및 PR 리뷰. '코드 리뷰', 'PR 리뷰', '검토', 'review' 요청 시 에이전트가 자동으로 로드
globs: ""
alwaysApply: false
---

# 코드 리뷰 가이드

## 리뷰 순서

1. **가독성 분석** — 이름, 함수 길이, 중첩 깊이
2. **보안 검토** — 입력 검증, 인증/인가, 민감 정보 노출
3. **성능 분석** — 알고리즘, DB 쿼리, 캐싱 가능성
4. **테스트 커버리지** — 테스트 없는 로직이 있는가?

## 피드백 형식

반드시 이 구조를 사용한다:

```
### 전체 평가: [1-5점]

#### ✅ 잘된 점
- (구체적 항목, 최소 1개)

#### 🔧 개선 제안
- **[파일:라인]** 문제 설명
  ```
  // Before
  // After
  ```
  이유: ...

#### 🚨 즉시 수정
- (있는 경우만)
```

## 우리 팀 기준

- 함수 최대 50줄
- 중첩 최대 3단계
- any 타입 사용 금지 (TypeScript)
- 모든 async 함수에 에러 처리 필수
```

### 예시 2: 성능 최적화 Agent Requested Rule

```markdown
<!-- .cursor/rules/performance.mdc -->

---
description: 코드 성능 분석 및 최적화. '성능', '느려', '최적화', 'performance', '속도 개선' 요청 시 에이전트가 로드
globs: ""
alwaysApply: false
---

# 성능 최적화 가이드

## 분석 우선순위

### 1. 데이터베이스 (영향 높음)
- N+1 쿼리 패턴 탐지
- 인덱스 누락 확인
- 불필요한 JOIN이나 서브쿼리
- ORM이 생성하는 실제 SQL 확인 (`explain analyze`)

### 2. 알고리즘 복잡도 (영향 중간)
- O(n²) 이상의 중첩 루프
- 불필요한 정렬 연산
- 선형 탐색 → 해시맵으로 교체 가능 여부

### 3. 프론트엔드 (영향 다양)
- 불필요한 리렌더링 (React)
- 큰 번들 사이즈
- 이미지 최적화 미적용

## 측정 먼저, 최적화 나중

최적화 전 반드시 벤치마크를 먼저 측정하고,
최적화 후 동일 벤치마크로 개선 효과를 수치로 제시한다.

## 결과 보고 형식

| 항목 | 현재 | 개선 후(예상) | 난이도 | 우선순위 |
|------|------|--------------|--------|---------|
| DB 쿼리 수 | 23개 | 3개 | 낮음 | 높음 |
| 응답시간 | 850ms | 120ms | 중간 | 높음 |
```

### 예시 3: 아키텍처 컨텍스트 Notepad

```markdown
<!-- Notepad 이름: project-architecture -->

# 프로젝트 아키텍처 컨텍스트

## 기술 스택
- Frontend: Next.js 14 (App Router) + TypeScript + Tailwind CSS
- Backend: FastAPI + Python 3.11
- Database: PostgreSQL 15 + Redis (캐싱/세션)
- Infrastructure: AWS ECS + RDS + ElastiCache

## 디렉토리 구조
```
src/
├── app/           # Next.js App Router
├── components/    # 공유 UI 컴포넌트
├── hooks/         # 커스텀 훅
├── lib/           # 유틸리티 및 설정
└── types/         # 공유 TypeScript 타입

api/
├── routers/       # FastAPI 라우터
├── models/        # SQLAlchemy 모델
├── schemas/       # Pydantic 스키마
└── services/      # 비즈니스 로직
```

## 주요 설계 결정
- 상태 관리: Zustand (전역), TanStack Query (서버 상태)
- 인증: NextAuth.js + JWT
- API 통신: axios + custom hook
- 에러 처리: 중앙집중식 ErrorBoundary

이 컨텍스트를 바탕으로 코드 작성/리뷰 시 아키텍처 결정을 고려해줘.
```

호출:

```
@notepad:project-architecture 새로운 알림 기능을 어디에 어떻게 추가해야 할까?
```

### 체크포인트

예시 파일들을 실제로 작성하고, 트리거 키워드로 요청했을 때 에이전트가 적절한 Rule을 자동으로 로드하는지 확인했는가?

---

## 다른 도구와의 비교

| 기능 | Cursor | Claude Code | Antigravity | VS Code Copilot |
|------|--------|-------------|-------------|-----------------|
| Skills 방식 | Notepads + Agent Requested Rules | `.claude/commands/*.md` | `.agent/skills/SKILL.md` | Prompt Files + Instructions |
| 파일 위치 | `.cursor/rules/*.mdc` | `.claude/commands/` | `.agent/skills/<name>/` | `.github/prompts/` |
| 호출 방식 | 자동(에이전트 판단) 또는 `@notepad:이름` | `/명령어` (명시적) | 자동 (description 매칭) | 자동 또는 `/명령어` |
| 인자 전달 | 없음 | `$ARGUMENTS` | 없음 | `${input:변수}` |
| 버전 관리 | `.mdc` 파일은 가능, Notepads는 불가 | 가능 | 가능 | 가능 |
| 팀 공유 | `.mdc` git 공유, Notepads는 Team 플랜 | git 공유 | git 공유 | git 공유 |
| 파일 번들링 | 없음 (단일 파일) | 없음 (단일 파일) | 디렉토리 번들 | 없음 |
| 전역 설정 | 없음 | `~/.claude/commands/` | `~/.gemini/antigravity/skills/` | User Instructions |

---

## 마무리

### 이 가이드에서 배운 것

- Cursor에는 명시적 Skills 시스템이 없지만 Notepads + Agent Requested Rules로 동일한 효과
- Notepads = 개인 사용, 복잡한 컨텍스트 저장, `@notepad:이름` 수동 참조
- Agent Requested Rules = 팀 공유, 버전 관리, 에이전트 자동 로드
- `alwaysApply: false` + `globs: ""` + 명확한 `description` = Agent Requested 유형

### 효과적인 description 작성 팁

```markdown
# ❌ 나쁜 예 (너무 짧고 모호함)
description: 배포 관련 규칙

# ✅ 좋은 예 (구체적 트리거 표현 열거)
description: Django 배포 체크리스트 및 AWS EB 배포 절차. '배포', 'deploy', 'AWS 배포', '프로덕션 올리기', 'release' 요청 시 에이전트가 로드
```

### 다음 단계

- `05_workflow` — Agent Requested Rules를 조합한 멀티스텝 워크플로우
- `02_rule/cursor-rule-guide.md` — 4가지 Rule 유형 전략적 활용 방법
