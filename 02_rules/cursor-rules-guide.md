# Cursor Rules 작성 가이드

## 학습 목표

Cursor AI 에디터의 Rules 시스템을 이해하고, `.cursor/rules/` 디렉토리에 MDC 형식의 규칙 파일을 작성하여 프로젝트 맞춤형 AI 코딩 환경을 구축할 수 있다.

## 사전 준비

- Cursor AI 에디터 설치 (v2.0 이상)
- 프로젝트 디렉토리 구조에 대한 기본 이해
- 마크다운 문법 및 YAML 프론트매터 기초 지식
- Glob 패턴에 대한 기본 이해 (예: `**/*.py`, `src/**/*.ts`)

---

## 전체 흐름 한눈에 보기

Cursor Rules는 AI가 코드를 생성하거나 수정할 때 참고하는 "프로젝트별 행동 지침"이다. 다른 도구의 CLAUDE.md, GEMINI.md와 유사하지만, **4가지 규칙 유형**과 **Glob 기반 자동 첨부** 기능으로 가장 세분화된 규칙 시스템을 제공한다.

**주요 단계:**
1. Cursor Rules의 역할과 진화 과정 이해
2. 스코프 계층 (Global → Project → Legacy) 파악
3. MDC 형식의 규칙 파일 작성법 학습
4. Notepads(보조 컨텍스트) 활용법 이해
5. Django 블로그 프로젝트로 실전 적용

---

## Phase 1: Cursor Rules 이해하기

### 목표
Cursor Rules가 무엇인지, 어떻게 진화해왔는지, 다른 도구의 규칙 시스템과 어떻게 다른지 이해한다.

### 단계별 구현

#### Step 1.1 — Cursor Rules의 역할 파악하기

Cursor Rules는 AI 에디터에게 "이 프로젝트에서는 이렇게 코드를 작성해라"라고 알려주는 설정 파일이다. 채팅, Composer, 인라인 편집 등 모든 AI 상호작용에 자동으로 주입된다.

> **💡 개념 설명: 왜 Cursor Rules가 필요한가?**
>
> AI 모델은 범용적으로 학습되어 있어, 프로젝트 고유의 규칙을 모른다. Rules를 설정하면:
> - **일관성**: 팀 전체가 동일한 코딩 스타일로 AI 코드를 생성
> - **정확성**: 프로젝트의 기술 스택과 패턴에 맞는 코드 생성
> - **효율성**: 매번 "우리는 TypeScript를 써" 같은 지시를 반복할 필요 없음
> - **세분화**: 파일 유형별로 다른 규칙을 자동 적용 가능
>
> **한 줄 요약**: Cursor Rules는 AI의 프로젝트 맞춤형 행동 지침서다.

#### Step 1.2 — Rules 시스템의 진화

Cursor의 규칙 시스템은 크게 두 세대로 나뉜다.

```
[1세대] .cursorrules (레거시)
  └─ 프로젝트 루트에 단일 파일
  └─ 모든 대화에 항상 적용
  └─ 모놀리식 구조 (하나의 파일에 모든 규칙)

        ↓ 진화 (Cursor v0.45+)

[2세대] .cursor/rules/*.mdc (현재)
  └─ 디렉토리 기반 다중 파일
  └─ 4가지 적용 방식 (Always / Auto / Agent / Manual)
  └─ Glob 패턴으로 파일별 자동 적용
  └─ MDC 형식 (YAML 프론트매터 + 마크다운)
```

> **💡 개념 설명: 왜 MDC 형식으로 전환되었는가?**
>
> `.cursorrules`는 단일 파일이라 규칙이 많아지면 관리가 어렵고, 모든 규칙이 항상 적용되어 컨텍스트 낭비가 발생했다. MDC 형식은:
> - **모듈화**: 주제별로 파일을 분리하여 관리 용이
> - **선택적 적용**: 필요한 규칙만 컨텍스트에 포함
> - **Glob 매칭**: 파일 유형에 따라 자동으로 관련 규칙 첨부
>
> **한 줄 요약**: `.cursorrules`는 "전부 아니면 전무", `.cursor/rules/`는 "필요한 것만 골라서".

#### Step 1.3 — 다른 도구와의 비교

| 항목 | Cursor Rules | CLAUDE.md | GEMINI.md | AGENTS.md (Codex) |
|------|-------------|-----------|-----------|-------------------|
| **파일 위치** | `.cursor/rules/*.mdc` | 프로젝트 루트 `CLAUDE.md` | 프로젝트 루트 `GEMINI.md` | 프로젝트 루트 `AGENTS.md` |
| **형식** | MDC (YAML 프론트매터 + MD) | 순수 마크다운 | 순수 마크다운 | 순수 마크다운 |
| **규칙 유형** | 4가지 (Always/Auto/Agent/Manual) | 1가지 (항상 적용) | 1가지 (항상 적용) | 1가지 (항상 적용) |
| **파일별 규칙** | Glob 패턴으로 자동 첨부 | 미지원 | 미지원 | 미지원 |
| **스코프 계층** | Global + Project + Legacy | Global + Workspace + Component | Global + Workspace + Component | Workspace |
| **에디터 UI** | Settings에서 직접 편집 가능 | CLI 기반 | CLI 기반 | CLI 기반 |
| **세분화 수준** | 가장 높음 | 중간 | 중간 | 낮음 |

### 체크포인트

- [ ] `.cursorrules`(레거시)와 `.cursor/rules/`(현재)의 차이를 설명할 수 있다
- [ ] Cursor Rules의 4가지 규칙 유형을 나열할 수 있다
- [ ] CLAUDE.md, GEMINI.md와의 차이점을 이해했다

---

## Phase 2: 스코프 계층

### 목표
Cursor Rules의 세 가지 스코프 계층(Global, Project, Legacy)을 이해하고, 각 계층의 우선순위와 사용 시나리오를 파악한다.

### 단계별 구현

#### Step 2.1 — Global Rules (전역 규칙)

전역 규칙은 Cursor 에디터의 Settings UI에서 설정하며, **모든 프로젝트**에 공통으로 적용된다.

**설정 경로:**
```
Cursor Settings → General → Rules for AI
```

**적합한 내용:**
- 선호하는 응답 언어 (예: "항상 한국어로 응답해라")
- 개인 코딩 스타일 (예: "변수명은 snake_case를 사용해라")
- 범용적인 행동 규칙 (예: "코드 변경 시 설명을 먼저 제공해라")

```text
# Global Rules 예시 (Settings UI에 직접 입력)

- 모든 응답은 한국어로 작성한다
- 코드 변경 전에 변경 사항을 먼저 설명한다
- 불필요한 주석은 작성하지 않는다
- 변수명과 함수명은 영어로 작성하되, 주석은 한국어로 작성한다
```

> **💡 개념 설명: Global Rules 사용 팁**
>
> Global Rules는 **최소한으로 유지**하는 것이 좋다. 프로젝트마다 달라질 수 있는 내용은 Project Rules에 넣어야 한다.
> - 적합: "한국어로 응답", "코드 설명 먼저 제공"
> - 부적합: "Django 4.2 사용", "REST API 응답 형식" (프로젝트별로 다름)
>
> **한 줄 요약**: Global Rules는 "나의 개인 취향", Project Rules는 "프로젝트의 규칙".

#### Step 2.2 — Project Rules (프로젝트 규칙)

프로젝트 규칙은 `.cursor/rules/` 디렉토리에 `.mdc` 파일로 저장하며, **해당 프로젝트에만** 적용된다.

```bash
# 프로젝트 디렉토리 구조

my-project/
├── .cursor/
│   └── rules/
│       ├── general.mdc        # 프로젝트 공통 규칙 (Always)
│       ├── python-style.mdc   # Python 파일 규칙 (Auto-attached)
│       ├── testing.mdc        # 테스트 관련 규칙 (Auto-attached)
│       └── deployment.mdc     # 배포 관련 규칙 (Agent-requested)
├── src/
├── tests/
└── README.md
```

**핵심 특징:**
- Git으로 버전 관리 가능 → 팀원 간 규칙 공유
- 파일별로 독립적인 적용 조건 설정
- Cursor 에디터의 Rules 탭에서 UI로 생성/편집 가능

#### Step 2.3 — Legacy .cursorrules (레거시)

`.cursorrules` 파일은 프로젝트 루트에 위치하는 **1세대 규칙 파일**이다.

```bash
my-project/
├── .cursorrules      ← 레거시 (아직 동작하지만 권장하지 않음)
├── .cursor/
│   └── rules/        ← 현재 권장 방식
├── src/
└── README.md
```

> **💡 개념 설명: .cursorrules는 언제 사용하는가?**
>
> `.cursorrules`는 하위 호환성을 위해 아직 지원되지만, **향후 제거될 예정**이다.
> - 기존 프로젝트에서 이미 사용 중이라면 → 점진적으로 `.cursor/rules/`로 마이그레이션
> - 새 프로젝트를 시작한다면 → 처음부터 `.cursor/rules/` 사용
> - 간단한 개인 프로젝트라면 → `.cursorrules`도 무방 (단, 마이그레이션 권장)
>
> **한 줄 요약**: `.cursorrules`는 "아직 되지만 곧 사라질 레거시".

#### Step 2.4 — 스코프 우선순위

```
Global Rules (Settings UI)
    ↓ 기본 적용
Project Rules (.cursor/rules/*.mdc)
    ↓ 프로젝트별 적용 (Global 위에 오버레이)
Legacy .cursorrules
    ↓ 하위 호환 (Project Rules가 없을 때 폴백)
```

세 가지 스코프가 충돌할 경우, **Project Rules > Legacy > Global** 순으로 우선 적용된다.

### 체크포인트

- [ ] Global Rules의 설정 경로(Settings → General → Rules for AI)를 알고 있다
- [ ] Project Rules 파일의 위치(`.cursor/rules/*.mdc`)를 파악했다
- [ ] `.cursorrules`가 레거시임을 이해하고, 마이그레이션 필요성을 인지했다
- [ ] 세 가지 스코프의 우선순위를 설명할 수 있다

---

## Phase 3: MDC 규칙 파일 작성법

### 목표
MDC(Markdown Component) 형식의 구조를 이해하고, 4가지 규칙 유형별로 실제 `.mdc` 파일을 작성할 수 있다.

### 단계별 구현

#### Step 3.1 — MDC 파일의 기본 구조

MDC 파일은 **YAML 프론트매터 + 마크다운 본문**으로 구성된다.

```markdown
---
description: 이 규칙의 목적을 설명하는 문장
globs: 적용할 파일 패턴 (예: **/*.py)
alwaysApply: true 또는 false
---

# 규칙 제목

여기에 AI가 따라야 할 규칙을 마크다운으로 작성한다.

- 규칙 1: 함수에는 반드시 docstring을 작성한다
- 규칙 2: import는 그룹별로 정렬한다
```

#### Step 3.2 — 프론트매터 필드 상세

| 필드 | 타입 | 설명 | 예시 |
|------|------|------|------|
| `description` | string | 규칙의 목적 설명. Agent-requested 모드에서 AI가 이 설명을 읽고 적용 여부를 판단한다 | `"Django 뷰 작성 시 따라야 할 패턴"` |
| `globs` | string | 파일 매칭 패턴. Auto-attached 모드에서 참조된 파일이 이 패턴에 맞으면 자동 첨부된다 | `"**/*.py"`, `"src/**/*.ts, src/**/*.tsx"` |
| `alwaysApply` | boolean | `true`이면 모든 대화에 항상 포함. `false`이면 조건부 적용 | `true`, `false` |

> **💡 개념 설명: 프론트매터 조합이 규칙 유형을 결정한다**
>
> 프론트매터의 세 필드 조합에 따라 규칙의 동작 방식이 달라진다:
>
> | 규칙 유형 | alwaysApply | globs | description |
> |-----------|------------|-------|-------------|
> | Always | `true` | 비어 있음 | 선택 사항 |
> | Auto-attached | `false` | 패턴 지정 | 선택 사항 |
> | Agent-requested | `false` | 비어 있음 | **필수** |
> | Manual | `false` | 비어 있음 | 비어 있음 |
>
> **한 줄 요약**: 프론트매터 3개 필드의 조합이 곧 규칙의 유형이다.

#### Step 3.3 — 규칙 유형 1: Always (항상 적용)

`alwaysApply: true`로 설정하면, 모든 AI 대화에 항상 포함된다. CLAUDE.md나 GEMINI.md의 동작 방식과 동일하다.

```markdown
---
description: 프로젝트 공통 코딩 규칙
globs:
alwaysApply: true
---

# 프로젝트 공통 규칙

## 코드 스타일
- 모든 코드는 한국어 주석을 포함한다
- 함수와 클래스에는 반드시 docstring을 작성한다
- import문은 표준 라이브러리 → 서드파티 → 로컬 순으로 정렬한다

## 응답 형식
- 코드 변경 전에 변경 사항을 먼저 설명한다
- 파일 경로를 항상 명시한다
- 기존 코드 스타일과 일관성을 유지한다

## 기술 스택
- Python 3.12, Django 5.1
- PostgreSQL 16
- Django REST Framework 3.15
```

> **💡 개념 설명: Always 규칙의 적절한 사용**
>
> Always 규칙은 모든 대화의 컨텍스트에 포함되므로, **토큰을 항상 소비**한다. 따라서:
> - 적합: 프로젝트 공통 규칙, 기술 스택 정보, 응답 형식
> - 부적합: 특정 파일에만 해당하는 규칙 (→ Auto-attached 사용)
> - 권장: 500줄 이하로 간결하게 유지
>
> **한 줄 요약**: Always는 "모든 대화에 꼭 필요한 핵심 정보"만 넣는다.

#### Step 3.4 — 규칙 유형 2: Auto-attached (자동 첨부)

`globs` 패턴을 지정하면, 대화에서 해당 패턴에 매칭되는 파일이 참조될 때 자동으로 규칙이 첨부된다. Cursor만의 고유 기능이다.

```markdown
---
description: Python 파일 작성 시 따라야 할 스타일 가이드
globs: "**/*.py"
alwaysApply: false
---

# Python 스타일 가이드

## PEP 8 준수
- 들여쓰기는 공백 4칸
- 한 줄 최대 88자 (Black 포매터 기준)
- 클래스 사이에는 빈 줄 2개, 메서드 사이에는 빈 줄 1개

## Type Hints
- 모든 함수 파라미터와 반환값에 타입 힌트를 명시한다
- `from __future__ import annotations`를 파일 상단에 포함한다

## 예외 처리
- bare except 금지. 항상 구체적인 예외 클래스를 명시한다
- 커스텀 예외는 `exceptions.py`에 정의한다
```

**Glob 패턴 예시:**

| 패턴 | 매칭 대상 |
|------|----------|
| `**/*.py` | 모든 Python 파일 |
| `**/*.ts, **/*.tsx` | 모든 TypeScript/TSX 파일 |
| `**/views.py` | 모든 디렉토리의 views.py |
| `src/components/**/*.jsx` | src/components 하위 모든 JSX 파일 |
| `**/test_*.py, **/*_test.py` | 테스트 파일 (test_ 접두사 또는 _test 접미사) |
| `docker-compose*.yml` | Docker Compose 파일 |

#### Step 3.5 — 규칙 유형 3: Agent-requested (에이전트 요청)

`description`만 작성하고 `globs`와 `alwaysApply`를 비워두면, AI 에이전트가 `description`을 읽고 **현재 작업과 관련이 있다고 판단할 때** 스스로 규칙을 가져온다. Gemini CLI의 Skills와 유사한 개념이다.

```markdown
---
description: 배포 및 인프라 관련 작업 시 따라야 할 규칙. Docker, CI/CD, 서버 설정 등의 작업에 적용한다.
globs:
alwaysApply: false
---

# 배포 규칙

## Docker
- 멀티스테이지 빌드를 사용하여 이미지 크기를 최소화한다
- 프로덕션 이미지에 개발 의존성을 포함하지 않는다
- `.dockerignore`에 불필요한 파일을 등록한다

## CI/CD
- GitHub Actions를 사용한다
- main 브랜치에 직접 push하지 않는다
- PR 머지 전에 모든 테스트가 통과해야 한다

## 환경 변수
- 시크릿은 절대 코드에 하드코딩하지 않는다
- `.env.example` 파일에 모든 환경 변수의 예시를 유지한다
```

> **💡 개념 설명: Agent-requested의 description 작성 요령**
>
> AI가 description을 읽고 적용 여부를 판단하므로, 설명이 **구체적이고 명확**해야 한다:
> - 좋은 예: `"배포 및 인프라 관련 작업 시 따라야 할 규칙. Docker, CI/CD, 서버 설정 등의 작업에 적용한다."`
> - 나쁜 예: `"배포 규칙"` (너무 짧아서 AI가 언제 적용할지 판단하기 어려움)
>
> **한 줄 요약**: description이 곧 AI가 규칙을 선택하는 기준이다.

#### Step 3.6 — 규칙 유형 4: Manual (수동 참조)

`description`, `globs`, `alwaysApply` 모두 비워두면, 사용자가 **직접 `@rulename`으로 참조**해야만 적용되는 수동 규칙이 된다.

```markdown
---
description:
globs:
alwaysApply: false
---

# 데이터베이스 마이그레이션 체크리스트

마이그레이션 작업 시 다음 체크리스트를 따른다:

1. 마이그레이션 파일 생성 전에 모델 변경 사항을 검토한다
2. `makemigrations --dry-run`으로 변경 사항을 미리 확인한다
3. 마이그레이션 SQL을 `sqlmigrate`로 검토한다
4. 데이터 마이그레이션이 필요한 경우 별도 마이그레이션 파일을 작성한다
5. 롤백 가능 여부를 확인한다
6. 스테이징 환경에서 먼저 테스트한다
```

**사용 방법:**
```
# 채팅 또는 Composer에서 @규칙이름으로 참조
@db-migration 이번 모델 변경에 대한 마이그레이션을 작성해줘
```

#### Step 3.7 — 4가지 규칙 유형 요약

```
┌─────────────────────────────────────────────────────┐
│                  Cursor Rules 유형                    │
├──────────────────┬──────────────────────────────────┤
│  Always          │  모든 대화에 항상 포함             │
│  (alwaysApply)   │  → CLAUDE.md와 동일한 동작        │
├──────────────────┼──────────────────────────────────┤
│  Auto-attached   │  파일 패턴 매칭 시 자동 첨부       │
│  (globs)         │  → Cursor 고유 기능              │
├──────────────────┼──────────────────────────────────┤
│  Agent-requested │  AI가 설명을 읽고 필요 시 적용     │
│  (description)   │  → Gemini Skills와 유사          │
├──────────────────┼──────────────────────────────────┤
│  Manual          │  사용자가 @로 직접 참조            │
│  (@rulename)     │  → 필요할 때만 수동 호출          │
└──────────────────┴──────────────────────────────────┘
```

### 체크포인트

- [ ] MDC 파일의 프론트매터 구조(description, globs, alwaysApply)를 작성할 수 있다
- [ ] 4가지 규칙 유형(Always, Auto-attached, Agent-requested, Manual)의 차이를 설명할 수 있다
- [ ] Glob 패턴으로 파일 매칭 조건을 설정할 수 있다
- [ ] 각 규칙 유형에 적합한 사용 시나리오를 판단할 수 있다

---

## Phase 4: Notepads (보조 컨텍스트)

### 목표
Cursor Notepads의 개념, Rules와의 차이점, 주요 사용 사례를 이해한다.

### 단계별 구현

#### Step 4.1 — Notepads란?

Notepads는 Cursor에서 제공했던 **재사용 가능한 컨텍스트 저장소** 기능이다. Rules가 "AI의 행동 규칙"이라면, Notepads는 "AI가 참고할 배경 자료"에 가깝다.

> **💡 개념 설명: Notepads의 현재 상태**
>
> Notepads는 2025년 10월경 Cursor v2.0에서 **Deprecated(지원 중단)** 되었다. 그러나 Notepads가 제공했던 개념과 패턴은 여전히 유용하며, 현재는 다음과 같은 대안으로 대체할 수 있다:
> - **Agent-requested Rules**: description 기반으로 필요 시 AI가 자동으로 참조
> - **Manual Rules**: `@rulename`으로 필요할 때 직접 참조
> - **프로젝트 내 마크다운 파일**: `docs/` 디렉토리에 참고 자료를 저장하고 `@file`로 참조
>
> **한 줄 요약**: Notepads의 역할은 이제 Rules와 `@file` 참조로 대체되었다.

#### Step 4.2 — Notepads가 제공했던 기능

Notepads는 다음과 같은 보조 컨텍스트를 저장하는 데 사용되었다:

| 사용 사례 | 내용 예시 | 현재 대안 |
|-----------|----------|----------|
| API 문서 | 외부 API 스펙, 엔드포인트 목록 | Agent-requested Rule 또는 `@file` |
| 아키텍처 노트 | 시스템 설계 결정, 다이어그램 | Manual Rule |
| 참고 자료 | 라이브러리 사용법, 코드 패턴 | `docs/` 디렉토리 + `@file` |
| 팀 컨벤션 | 코드 리뷰 체크리스트, PR 템플릿 | Always Rule |

#### Step 4.3 — Notepads 패턴을 Rules로 전환하기

기존 Notepads 사용 패턴을 현재 Rules 시스템으로 전환하는 방법이다.

**기존 Notepad 방식 (Deprecated):**
```
# Notepad: API_Reference
채팅에서 @API_Reference로 참조

외부 결제 API 스펙:
- POST /payments/charge
- POST /payments/refund
...
```

**현재 권장 방식 - Agent-requested Rule:**
```markdown
---
description: 외부 결제 API 연동 시 참조할 API 스펙. 결제, 환불, 정기결제 관련 코드 작성 시 적용한다.
globs:
alwaysApply: false
---

# 외부 결제 API 참조

## 결제 요청
- 엔드포인트: `POST /payments/charge`
- 인증: Bearer Token (API Key)
- 요청 본문:
  ```json
  {
    "amount": 10000,
    "currency": "KRW",
    "order_id": "ORDER-123",
    "customer_id": "CUST-456"
  }
  ```

## 환불 요청
- 엔드포인트: `POST /payments/refund`
- 요청 본문:
  ```json
  {
    "payment_id": "PAY-789",
    "amount": 10000,
    "reason": "고객 요청"
  }
  ```
```

### 체크포인트

- [ ] Notepads가 Deprecated되었음을 이해했다
- [ ] Notepads의 역할을 Rules와 `@file`로 대체하는 방법을 파악했다
- [ ] Agent-requested Rule로 참고 자료를 제공하는 패턴을 이해했다

---

## Phase 5: 실전 예시 - Django 블로그 프로젝트

### 목표
실제 Django 블로그 프로젝트에 4가지 규칙 유형을 모두 적용한 완전한 `.cursor/rules/` 구성 예시를 작성한다.

### 프로젝트 구조

```bash
blog_project/
├── .cursor/
│   └── rules/
│       ├── general.mdc           # Always - 프로젝트 공통 규칙
│       ├── python-style.mdc      # Always - Python 스타일 가이드
│       ├── django-views.mdc      # Auto-attached - views.py 전용
│       ├── django-models.mdc     # Auto-attached - models.py 전용
│       ├── django-templates.mdc  # Auto-attached - 템플릿 전용
│       ├── testing.mdc           # Auto-attached - 테스트 파일 전용
│       ├── deployment.mdc        # Agent-requested - 배포 관련
│       ├── api-design.mdc        # Agent-requested - API 설계
│       └── db-migration.mdc      # Manual - DB 마이그레이션 체크리스트
├── .cursorrules                  # (레거시 - 마이그레이션 대상)
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   ├── urls.py
│   └── wsgi.py
├── posts/
│   ├── models.py
│   ├── views.py
│   ├── serializers.py
│   ├── urls.py
│   └── templates/posts/
├── comments/
├── users/
├── manage.py
└── requirements.txt
```

### 규칙 파일 작성

#### 1. general.mdc (Always - 프로젝트 공통)

```markdown
---
description: Django 블로그 프로젝트의 공통 규칙과 기술 스택 정보
globs:
alwaysApply: true
---

# Django 블로그 프로젝트 규칙

## 프로젝트 개요
Django 기반 개인 블로그 플랫폼이다. 사용자는 회원가입 후 게시글을 작성하고,
댓글을 달 수 있다. 태그 기반 검색과 카테고리 분류를 지원한다.

## 기술 스택
- Python 3.12, Django 5.1, Django REST Framework 3.15
- PostgreSQL 16, Redis 7.2 (캐시)
- pytest, pytest-django (테스트)
- Docker, GitHub Actions (CI/CD)

## 공통 규칙
- 모든 응답과 주석은 한국어로 작성한다
- 코드 변경 전에 변경 내용과 이유를 먼저 설명한다
- 기존 코드 패턴과 일관성을 유지한다
- 새 기능 추가 시 테스트 코드를 반드시 함께 작성한다

## 디렉토리 구조
- `config/` - Django 프로젝트 설정 (환경별 분리)
- `posts/` - 게시글 관리 앱 (Post, Category, Tag 모델)
- `comments/` - 댓글 관리 앱 (Comment 모델, 대댓글 지원)
- `users/` - 사용자 관리 앱 (CustomUser, 프로필)
- `core/` - 공통 유틸리티, 예외, 권한
```

#### 2. python-style.mdc (Always - Python 스타일)

```markdown
---
description: Python 코드 스타일 가이드
globs:
alwaysApply: true
---

# Python 스타일 가이드

## 포매팅
- PEP 8 준수, Black 포매터 기준 (최대 88자)
- 들여쓰기: 공백 4칸
- import 순서: 표준 라이브러리 → 서드파티 → 로컬 (isort 사용)

## 타입 힌트
- 모든 함수의 파라미터와 반환값에 타입 힌트를 명시한다
- `from __future__ import annotations` 포함

## Docstring
- Google 스타일 docstring을 사용한다
- 모든 공개 함수와 클래스에 docstring을 작성한다

## 네이밍
- 변수/함수: snake_case
- 클래스: PascalCase
- 상수: UPPER_SNAKE_CASE
- 비공개 멤버: _접두사
```

#### 3. django-views.mdc (Auto-attached - views.py 전용)

```markdown
---
description: Django 뷰 작성 시 따라야 할 패턴과 규칙
globs: "**/views.py, **/views/*.py"
alwaysApply: false
---

# Django 뷰 작성 규칙

## 기본 원칙
- 클래스 기반 뷰(CBV)를 우선 사용한다
- DRF ViewSet으로 CRUD API를 구현한다
- 비즈니스 로직은 뷰에 직접 작성하지 않고, 서비스 레이어나 모델 메서드로 분리한다

## API 응답 형식
모든 API 응답은 다음 형식을 따른다:
```json
{
  "status": "success",
  "data": {},
  "message": ""
}
```

## 에러 응답 형식
```json
{
  "status": "error",
  "error_code": "RESOURCE_NOT_FOUND",
  "message": "해당 게시글을 찾을 수 없습니다"
}
```

## 인증 및 권한
- 인증: JWT (djangorestframework-simplejwt)
- 읽기: 비인증 사용자 허용
- 쓰기: 인증 사용자만 허용
- 수정/삭제: 작성자만 허용 (IsOwnerOrReadOnly 커스텀 권한)

## 페이지네이션
- 기본 페이지 크기: 20
- 최대 페이지 크기: 100
- PageNumberPagination 사용
```

#### 4. django-models.mdc (Auto-attached - models.py 전용)

```markdown
---
description: Django 모델 작성 시 따라야 할 패턴과 규칙
globs: "**/models.py, **/models/*.py"
alwaysApply: false
---

# Django 모델 작성 규칙

## 공통 필드
- 모든 모델에 `created_at`, `updated_at` 필드를 포함한다
- `created_at`: `auto_now_add=True`
- `updated_at`: `auto_now=True`

## 외래 키
- `on_delete` 옵션을 명시적으로 지정한다
- `related_name`을 반드시 설정한다
- 예시: `author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')`

## Meta 클래스
- 모든 모델에 `ordering`을 지정한다
- `verbose_name`과 `verbose_name_plural`을 한국어로 작성한다
- 필요한 필드에 `db_index=True`를 설정한다

## 모델 메서드
- `__str__` 메서드를 반드시 정의한다
- 비즈니스 로직은 모델 메서드로 캡슐화한다
- 복잡한 쿼리는 커스텀 Manager에 정의한다

## 예시
```python
from django.db import models
from django.conf import settings


class Post(models.Model):
    """게시글 모델"""
    title = models.CharField("제목", max_length=200)
    content = models.TextField("내용")
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="posts",
        verbose_name="작성자",
    )
    status = models.CharField(
        "상태",
        max_length=10,
        choices=[("draft", "임시저장"), ("published", "발행")],
        default="draft",
    )
    created_at = models.DateTimeField("작성일", auto_now_add=True)
    updated_at = models.DateTimeField("수정일", auto_now=True)

    class Meta:
        ordering = ["-created_at"]
        verbose_name = "게시글"
        verbose_name_plural = "게시글 목록"

    def __str__(self) -> str:
        return self.title
```
```

#### 5. deployment.mdc (Agent-requested - 배포 관련)

```markdown
---
description: 배포, Docker, CI/CD, 서버 설정 관련 작업 시 참조할 규칙. Dockerfile, docker-compose, GitHub Actions 워크플로우, Nginx 설정 등의 작업에 적용한다.
globs:
alwaysApply: false
---

# 배포 규칙

## Docker
- 멀티스테이지 빌드를 사용한다
- 프로덕션 이미지: `python:3.12-slim` 기반
- `.dockerignore`에 venv, __pycache__, .git 등을 등록한다
- 보안을 위해 non-root 사용자로 실행한다

## Docker Compose
- 개발용: `docker-compose.yml`
- 프로덕션용: `docker-compose.prod.yml`
- 서비스: web (Django + Gunicorn), db (PostgreSQL), redis, nginx

## CI/CD (GitHub Actions)
- main 브랜치: 자동 배포
- develop 브랜치: 스테이징 배포
- PR: 린트 + 테스트 자동 실행
- 시크릿은 GitHub Secrets에 저장

## 환경 변수
- `.env.example`에 모든 환경 변수 예시를 유지한다
- 시크릿은 코드에 하드코딩하지 않는다
- 프로덕션에서 DEBUG=False를 반드시 확인한다
```

#### 6. .cursorrules 레거시 예시 (참고용)

아래는 마이그레이션 전의 레거시 `.cursorrules` 파일 예시이다. 모든 규칙이 단일 파일에 혼재되어 있어 관리가 어렵다.

```text
# .cursorrules (레거시 - 마이그레이션 권장)

You are an AI assistant for a Django blog project.

## General Rules
- Respond in Korean
- Use Python 3.12 and Django 5.1
- Follow PEP 8 style guide
- Write Google-style docstrings

## Django Conventions
- Use class-based views (CBV)
- Use DRF ViewSet for APIs
- JWT authentication with simplejwt
- All models must have created_at and updated_at

## Database Rules
- Always specify on_delete for ForeignKey
- Always specify related_name
- Use db_index for frequently queried fields

## Testing
- Write tests for all views and models
- Use pytest and pytest-django
- Minimum 80% coverage

## Deployment
- Use Docker with multi-stage builds
- Use GitHub Actions for CI/CD
- Never hardcode secrets
```

> **💡 개념 설명: .cursorrules에서 .cursor/rules/로 마이그레이션하기**
>
> 마이그레이션 순서:
> 1. `.cursorrules` 내용을 주제별로 분류한다
> 2. `.cursor/rules/` 디렉토리를 생성한다
> 3. 주제별로 `.mdc` 파일을 생성하고 적절한 프론트매터를 설정한다
> 4. 각 규칙 유형(Always/Auto/Agent/Manual)을 결정한다
> 5. 규칙이 정상 동작하는지 테스트한다
> 6. `.cursorrules` 파일을 삭제한다
>
> **한 줄 요약**: 하나의 큰 파일을 여러 개의 집중된 파일로 쪼개는 과정이다.

### 체크포인트

- [ ] 4가지 규칙 유형을 각각 `.mdc` 파일로 작성했다
- [ ] Always 규칙에 프로젝트 공통 정보를 포함했다
- [ ] Auto-attached 규칙에 적절한 globs 패턴을 설정했다
- [ ] Agent-requested 규칙에 구체적인 description을 작성했다
- [ ] `.cursorrules`에서 `.cursor/rules/`로의 마이그레이션 방법을 이해했다

---

## 마무리

### 학습한 내용 정리

Cursor Rules는 4가지 규칙 유형과 Glob 기반 자동 첨부로, Agentic Coding 도구 중 **가장 세분화된 규칙 시스템**을 제공한다.

1. **Always**: 모든 대화에 항상 포함 → 프로젝트 공통 규칙
2. **Auto-attached**: 파일 패턴 매칭 시 자동 첨부 → 파일 유형별 규칙
3. **Agent-requested**: AI가 description을 읽고 필요 시 적용 → 상황별 규칙
4. **Manual**: 사용자가 @로 직접 참조 → 특수 상황 체크리스트

### 멀티 도구 비교표

| 기능 | Cursor Rules | CLAUDE.md | GEMINI.md | AGENTS.md |
|------|-------------|-----------|-----------|-----------|
| 규칙 유형 수 | **4가지** | 1가지 | 1가지 | 1가지 |
| 파일별 자동 규칙 | **Glob 패턴** | 미지원 | 미지원 | 미지원 |
| AI 자동 선택 | **Agent-requested** | 미지원 | Skills | 미지원 |
| 수동 참조 | **@rulename** | 미지원 | 미지원 | 미지원 |
| 스코프 계층 | Global + Project | Global + Workspace + Component | Global + Workspace + Component | Workspace |
| 파일 형식 | MDC (YAML + MD) | Markdown | Markdown | Markdown |
| 에디터 UI | **있음** | CLI | CLI | CLI |
| Git 관리 | `.cursor/rules/` | 프로젝트 루트 | 프로젝트 루트 | 프로젝트 루트 |

### Cursor의 고유 장점

1. **가장 세분화된 규칙 시스템**: 4가지 규칙 유형으로 상황별 최적 적용
2. **Glob 기반 자동 첨부**: 파일 유형에 따라 관련 규칙이 자동으로 활성화
3. **컨텍스트 효율성**: 필요한 규칙만 포함하여 토큰 낭비 최소화
4. **에디터 UI 통합**: Settings에서 직접 규칙을 생성/편집/관리
5. **모듈화**: 주제별 파일 분리로 유지보수 용이

### .cursorrules에서 .cursor/rules/로 마이그레이션

```bash
# 1. 디렉토리 생성
mkdir -p .cursor/rules

# 2. 기존 규칙 분석
cat .cursorrules

# 3. 주제별 .mdc 파일 생성
# - 공통 규칙 → general.mdc (alwaysApply: true)
# - Python 스타일 → python-style.mdc (alwaysApply: true)
# - 뷰 규칙 → django-views.mdc (globs: "**/views.py")
# - 배포 규칙 → deployment.mdc (description 기반)

# 4. 동작 확인 후 레거시 파일 삭제
rm .cursorrules
```

### 자주 하는 실수

1. **Always 규칙을 너무 많이 만듦**
   - 문제: 모든 대화에 불필요한 규칙이 포함되어 컨텍스트 낭비
   - 해결: 정말 모든 대화에 필요한 정보만 Always로 설정. 나머지는 Auto-attached 또는 Agent-requested로 분리

2. **Agent-requested의 description이 불충분**
   - 문제: AI가 언제 규칙을 적용할지 판단하지 못함
   - 해결: "어떤 작업에, 어떤 상황에서 이 규칙을 적용하는지" 구체적으로 기술

3. **Glob 패턴 오류**
   - 문제: `*.py`만 쓰면 루트 디렉토리의 `.py` 파일만 매칭
   - 해결: 재귀 매칭은 반드시 `**/*.py` 사용

4. **프론트매터 YAML 문법 오류**
   - 문제: 들여쓰기, 따옴표 누락 등으로 규칙이 적용되지 않음
   - 해결: globs에 쉼표가 포함되면 따옴표로 감싸기 (예: `"**/*.ts, **/*.tsx"`)

5. **레거시와 신규 규칙 중복**
   - 문제: `.cursorrules`와 `.cursor/rules/`에 같은 내용이 중복
   - 해결: 마이그레이션 완료 후 `.cursorrules` 삭제

### 다음 단계

1. **자신의 프로젝트에 Rules 적용**
   - `.cursor/rules/` 디렉토리 생성
   - 프로젝트 공통 규칙을 Always로 작성
   - 파일 유형별 규칙을 Auto-attached로 분리

2. **팀과 규칙 공유**
   - `.cursor/rules/`를 Git에 커밋하여 팀 전체 일관성 확보
   - PR 리뷰 시 규칙 준수 여부 확인

3. **규칙 최적화**
   - 불필요한 Always 규칙을 Auto-attached로 전환
   - Agent-requested의 description을 지속적으로 개선
   - 규칙 파일 크기를 500줄 이하로 유지

4. **심화 학습**
   - CLAUDE.md, GEMINI.md와 Cursor Rules를 함께 관리하는 멀티 도구 전략
   - 커스텀 규칙 생성기 활용 (cursor.directory 등 커뮤니티 리소스)
