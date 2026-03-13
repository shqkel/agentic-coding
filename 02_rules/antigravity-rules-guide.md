# Antigravity Rules 작성 가이드

## 학습 목표

Google Antigravity에서 사용하는 Rules 시스템의 구조와 작성 방법을 이해하고, Django 프로젝트에 적용할 수 있는 실무 중심의 Rules 파일을 직접 작성할 수 있다.

## 사전 준비

- Google Antigravity 설치 및 기본 사용법 숙지 (`agy` 명령어로 실행 가능)
- Antigravity의 Agent Manager 개념 이해
- Django 프로젝트 구조에 대한 기본 이해
- 마크다운 문법 기초 지식

---

## 전체 흐름 한눈에 보기

Rules는 Antigravity 에이전트가 코드를 생성하고 작업을 수행할 때 **항상** 참고하는 "행동 지침서"이다. Claude Code의 CLAUDE.md, Gemini CLI의 GEMINI.md와 동일한 역할을 하지만, 파일 위치와 스코프 구조가 다르다.

**주요 단계:**
1. Antigravity Rules의 역할과 다른 도구와의 차이 이해
2. 글로벌/워크스페이스 스코프 계층 파악
3. 효과적인 Rules 작성법 습득
4. Rules로 Hooks 패턴 구현하기
5. 실전 프로젝트에 적용

---

## Phase 1: Antigravity Rules 이해하기

### 목표
Rules가 무엇인지, 왜 필요한지, 다른 도구의 유사 기능과 어떻게 다른지 명확히 이해한다.

### 단계별 구현

#### Step 1.1 — Rules의 역할 파악하기

Rules는 Antigravity 에이전트의 **시스템 지시(System Instruction)** 역할을 한다. 에이전트가 코드를 생성하거나, 파일을 수정하거나, 테스트를 실행할 때 항상 Rules에 명시된 지침을 따른다.

> **💡 개념 설명: Rules = 에이전트의 행동 헌법**
>
> Rules는 에이전트에게 "항상 이렇게 행동하라"고 지시하는 **수동적이고 영속적인 가이드라인**이다. 사용자가 트리거하지 않아도 항상 "켜져 있는" 상태로 동작한다.
>
> - **Workflows**: 사용자가 `/명령어`로 **선택적**으로 실행하는 저장 프롬프트
> - **Skills**: 특정 키워드가 매칭될 때 **자동 로드**되는 전문 지식 패키지
> - **Rules**: 모든 대화에서 **항상 적용**되는 시스템 지시
>
> **한 줄 요약**: Rules는 에이전트가 매 작업마다 반드시 따르는 행동 헌법이다.

#### Step 1.2 — Rules vs CLAUDE.md vs GEMINI.md 비교

세 도구 모두 "에이전트에게 프로젝트 규칙을 알려주는 파일"이라는 점에서 동일한 목적을 가진다. 하지만 구현 방식에서 차이가 있다.

| 특성 | Antigravity Rules | Claude Code (CLAUDE.md) | Gemini CLI (GEMINI.md) |
|------|------------------|------------------------|----------------------|
| **파일명** | `rules.md` | `CLAUDE.md` | `GEMINI.md` |
| **워크스페이스 경로** | `<project>/.agent/rules.md` | `<project>/CLAUDE.md` | `<project>/GEMINI.md` |
| **글로벌 경로** | `~/.gemini/antigravity/rules.md` | `~/.claude/CLAUDE.md` | `~/.gemini/GEMINI.md` |
| **스코프 단계** | 2단계 (글로벌 + 워크스페이스) | 3단계 (글로벌 + 워크스페이스 + 컴포넌트) | 3단계 (글로벌 + 워크스페이스 + 컴포넌트) |
| **항상 로드** | O | O | O |
| **Hooks 대체** | Rules에 통합 | 별도 hooks 시스템 존재 | 없음 |
| **UI 편집** | O (Customizations 메뉴) | X (파일 직접 편집) | X (파일 직접 편집) |
| **AGENTS.md 지원** | O (v1.20.3+) | X | X |

> **💡 개념 설명: 왜 파일 위치가 다른가?**
>
> Claude Code와 Gemini CLI는 프로젝트 루트에 직접 설정 파일을 놓는 반면, Antigravity는 `.agent/` 디렉토리 안에 모든 에이전트 관련 파일을 모아둔다.
>
> ```
> # Claude Code / Gemini CLI 방식
> my-project/
> ├── CLAUDE.md          ← 프로젝트 루트
> ├── GEMINI.md          ← 프로젝트 루트
> └── src/
>
> # Antigravity 방식
> my-project/
> ├── .agent/
> │   ├── rules.md       ← .agent 디렉토리 내부
> │   ├── workflows/
> │   └── skills/
> └── src/
> ```
>
> Antigravity는 Rules, Workflows, Skills를 하나의 `.agent/` 디렉토리에 통합 관리한다.
>
> **한 줄 요약**: Antigravity는 `.agent/` 디렉토리로 에이전트 설정을 캡슐화한다.

#### Step 1.3 — AGENTS.md 지원 (v1.20.3+)

2026년 3월 5일 릴리스된 Antigravity v1.20.3부터 `AGENTS.md` 파일도 Rules 소스로 인식한다. 이는 GEMINI.md와 동일한 위치(프로젝트 루트)에 놓을 수 있으며, 멀티 도구 환경에서 하나의 파일로 여러 에이전트 도구를 지원하려는 표준화 움직임의 일환이다.

```bash
# AGENTS.md도 Rules로 인식됨 (v1.20.3+)
my-project/
├── AGENTS.md           ← Antigravity가 자동 인식
├── GEMINI.md           ← Gemini CLI + Antigravity 모두 인식
├── .agent/
│   └── rules.md        ← Antigravity 전용
└── src/
```

### 체크포인트

- [ ] Rules, Workflows, Skills의 차이를 설명할 수 있다
- [ ] Antigravity Rules와 CLAUDE.md/GEMINI.md의 경로 차이를 알고 있다
- [ ] `.agent/` 디렉토리의 역할을 이해했다

---

## Phase 2: 스코프 계층

### 목표
Rules의 글로벌 스코프와 워크스페이스 스코프를 이해하고, 적절한 위치에 규칙을 배치할 수 있다.

### 단계별 구현

#### Step 2.1 — 글로벌 Rules

글로벌 Rules는 모든 프로젝트에 공통으로 적용되는 개인 코딩 철학이나 조직 정책을 정의한다.

**경로:** `~/.gemini/antigravity/rules.md`

```markdown
# ~/.gemini/antigravity/rules.md (글로벌)

## 코딩 철학
- 모든 코드는 "읽는 사람"을 최우선으로 고려하여 작성할 것
- DRY(Don't Repeat Yourself) 원칙을 준수할 것
- 함수는 하나의 역할만 수행할 것 (Single Responsibility)

## 언어 공통 규칙
- 주석은 한국어로 작성할 것
- 변수명과 함수명은 영문으로 작성할 것
- 모든 함수에 type hints를 포함할 것

## 보안
- 하드코딩된 비밀번호, API 키, 토큰을 절대 코드에 포함하지 말 것
- 환경 변수를 통해 민감 정보를 관리할 것
- `.env` 파일은 절대 Git에 커밋하지 말 것

## 커밋
- 커밋 메시지는 Conventional Commits 형식을 따를 것
- 한 커밋에는 하나의 논리적 변경만 포함할 것
```

> **💡 개념 설명: 글로벌 Rules에 넣어야 할 것**
>
> 글로벌 Rules는 **프로젝트에 관계없이 항상 적용**되는 규칙이다.
>
> **적합한 내용:**
> - 개인 코딩 스타일 (들여쓰기, 네이밍, 주석 언어)
> - 보안 정책 (비밀 키 관리, 환경 변수 사용)
> - Git 커밋 규칙
> - 코드 리뷰 기준
>
> **부적합한 내용:**
> - 특정 프레임워크 규칙 (Django, React 등) → 워크스페이스 Rules
> - 프로젝트별 디렉토리 구조 → 워크스페이스 Rules
> - 특정 API 설계 규칙 → 워크스페이스 Rules
>
> **한 줄 요약**: 글로벌 = "나는 어떤 프로젝트든 이렇게 코딩한다."

#### Step 2.2 — 워크스페이스 Rules

워크스페이스 Rules는 특정 프로젝트에만 적용되는 규칙이다. 프로젝트의 기술 스택, 아키텍처, 팀 컨벤션을 정의한다.

**경로:** `<project>/.agent/rules.md`

```markdown
# .agent/rules.md (워크스페이스)

## 프로젝트 컨텍스트
- 이 프로젝트는 Django 4.2 + DRF 기반 REST API 서버이다
- 데이터베이스는 PostgreSQL 15를 사용한다
- Python 3.12 환경이다

## Django 규칙
- 모든 뷰는 클래스 기반 뷰(CBV)를 우선 사용할 것
- URL 패턴은 앱별 urls.py에 분리하고 include()로 연결할 것
- settings.py는 config/settings/ 아래 base.py, dev.py, prod.py로 분리할 것

## 테스트
- pytest와 pytest-django를 사용할 것
- 테스트 커버리지 80% 이상을 유지할 것
- 테스트 파일은 tests/ 디렉토리에 test_*.py 형식으로 작성할 것
```

#### Step 2.3 — 스코프 병합 규칙

글로벌 Rules와 워크스페이스 Rules가 모두 존재할 경우, Antigravity는 **두 파일을 병합**하여 에이전트에 주입한다. 워크스페이스 Rules가 글로벌 Rules보다 우선한다.

```
┌─────────────────────────────────────┐
│         에이전트 컨텍스트             │
│                                     │
│  ┌───────────────────────────────┐  │
│  │  글로벌 Rules (기본값)          │  │
│  │  ~/.gemini/antigravity/       │  │
│  │  rules.md                     │  │
│  └───────────────────────────────┘  │
│            ▼ 병합 (워크스페이스 우선)  │
│  ┌───────────────────────────────┐  │
│  │  워크스페이스 Rules (오버라이드)  │  │
│  │  <project>/.agent/rules.md    │  │
│  └───────────────────────────────┘  │
│                                     │
│  + AGENTS.md / GEMINI.md (있으면)    │
└─────────────────────────────────────┘
```

> **💡 개념 설명: 2단계 스코프의 장단점**
>
> Claude Code와 GEMINI.md는 서브디렉토리마다 별도 규칙 파일을 둘 수 있는 **3단계 스코프**를 지원한다. Antigravity는 **2단계만** 지원한다.
>
> | 비교 항목 | 2단계 (Antigravity) | 3단계 (Claude Code) |
> |----------|-------------------|-------------------|
> | 구성 복잡도 | 낮음 | 높음 |
> | 세밀한 제어 | 제한적 | 디렉토리별 가능 |
> | 유지보수 부담 | 적음 | 파일 증가 시 관리 부담 |
> | 모노레포 지원 | 워크스페이스 단일 규칙 | 패키지별 규칙 가능 |
>
> Antigravity의 접근은 **단순하지만 덜 세밀하다**. 모노레포처럼 서브 프로젝트 간 규칙이 크게 다른 경우에는 한계가 있다.
>
> **한 줄 요약**: Antigravity는 단순함을 택했고, Claude Code는 세밀함을 택했다.

#### Step 2.4 — UI를 통한 Rules 편집

Antigravity는 파일 직접 편집 외에도 **UI 기반 편집**을 제공한다.

1. Editor 모드에서 우측 상단 **`...`** (점 세 개) 클릭
2. **Customizations** 선택
3. **Rules** 탭에서 글로벌/워크스페이스 Rules 편집
4. 저장 시 해당 경로의 `rules.md` 파일에 자동 반영

```
┌──────────────────────────────────┐
│  Antigravity → ... → Customizations  │
│  ┌────────┬──────────┐           │
│  │ Rules  │ Workflows│           │
│  ├────────┴──────────┤           │
│  │ Scope: [Global ▼] │           │
│  ├───────────────────┤           │
│  │ # 코딩 스타일      │           │
│  │ - PEP 8 준수       │           │
│  │ - type hints 필수  │           │
│  │ ...                │           │
│  └───────────────────┘           │
└──────────────────────────────────┘
```

> **💡 개념 설명: UI vs 파일 직접 편집**
>
> Claude Code와 Gemini CLI는 `CLAUDE.md` / `GEMINI.md` 파일을 텍스트 에디터로 직접 수정해야 한다. Antigravity는 **UI 편집기**를 추가로 제공하여 진입 장벽을 낮추었다.
>
> 단, UI로 편집한 내용은 결국 `rules.md` 파일에 저장되므로, **Git으로 버전 관리할 수 있다**는 점은 동일하다.
>
> **한 줄 요약**: UI로 편집하든 파일로 편집하든, 결과는 동일한 rules.md 파일이다.

### 체크포인트

- [ ] 글로벌 Rules와 워크스페이스 Rules의 경로를 정확히 알고 있다
- [ ] 두 스코프가 병합될 때 우선순위를 설명할 수 있다
- [ ] UI Customizations 메뉴에서 Rules를 편집하는 방법을 알고 있다
- [ ] 2단계 스코프의 장단점을 다른 도구와 비교하여 설명할 수 있다

---

## Phase 3: Rules 작성법

### 목표
효과적인 Rules를 카테고리별로 작성하고, 에이전트가 정확하게 따르는 명확한 지침을 만들 수 있다.

### 단계별 구현

#### Step 3.1 — Rules 작성 형식

Rules는 **마크다운 형식**으로 작성한다. 헤딩(`#`, `##`, `###`)으로 카테고리를 분류하고, 리스트(`-`)로 개별 규칙을 나열한다.

```markdown
# 카테고리명

## 하위 카테고리

- 규칙 1: 명확하고 구체적인 지시문
- 규칙 2: 에이전트가 바로 실행할 수 있는 명령
```

**좋은 규칙 vs 나쁜 규칙:**

```markdown
# 나쁜 예시 ❌
- 코드를 잘 작성할 것
- 보안에 신경 쓸 것
- 테스트를 적절히 할 것

# 좋은 예시 ✅
- 함수는 최대 30줄을 넘지 않을 것
- API 키는 반드시 환경 변수로 관리하고, 코드에 하드코딩하지 말 것
- 새 기능 구현 시 반드시 pytest 기반 단위 테스트를 함께 작성할 것
```

#### Step 3.2 — 효과적인 카테고리 분류

실무에서 자주 사용하는 Rules 카테고리를 정리하면 다음과 같다.

| 카테고리 | 설명 | 예시 |
|---------|------|------|
| 코딩 스타일 | 코드 포매팅, 네이밍 규칙 | PEP 8 준수, Black 포매터 사용 |
| 테스트 | 테스트 프레임워크, 커버리지 | pytest 사용, 커버리지 80% |
| 커밋/Git | 커밋 메시지, 브랜치 전략 | Conventional Commits |
| 보안 | 비밀 관리, 취약점 방지 | 환경 변수 사용, SQL injection 방지 |
| 프레임워크 | 특정 프레임워크 규칙 | Django CBV 우선, DRF ViewSet |
| 문서화 | docstring, README | Google 스타일 docstring |
| 에러 처리 | 예외, 로깅 | 구체적 예외 사용, logger 활용 |
| 아키텍처 | 프로젝트 구조, 패턴 | 앱별 분리, 서비스 레이어 |

#### Step 3.3 — Django 프로젝트 Rules 예시

```markdown
# .agent/rules.md

## 프로젝트 정보
- 이 프로젝트는 Django 4.2 + DRF 3.14 기반 온라인 서점 API이다
- Python 3.12, PostgreSQL 15를 사용한다
- 패키지 관리는 pip + requirements.txt를 사용한다

## 코딩 스타일
- 모든 Python 코드는 PEP 8을 따른다
- 한 줄 최대 길이는 88자이다 (Black 포매터 기준)
- import 순서: 표준 라이브러리 → 서드파티 → 로컬 앱
- 모든 함수와 클래스에 Google 스타일 docstring을 작성할 것

## Django 규칙
- 뷰는 클래스 기반 뷰(CBV)를 우선 사용할 것
- URL은 앱별로 urls.py를 분리하고 include()로 연결할 것
- settings.py는 config/settings/ 아래 base.py, dev.py, prod.py로 분리할 것
- 모델 변경 후 반드시 makemigrations와 migrate를 안내할 것
- 외래 키는 on_delete 옵션을 반드시 명시할 것
- created_at, updated_at 필드를 모든 모델에 포함할 것

## API 설계
- DRF ViewSet을 사용하여 CRUD API를 구현할 것
- API 응답은 다음 형식을 따를 것:
  ```json
  {"status": "success", "data": {}, "message": ""}
  ```
- 인증은 djangorestframework-simplejwt를 사용할 것
- 페이지네이션은 기본 20개 항목으로 설정할 것

## 테스트
- pytest + pytest-django를 기본 테스트 프레임워크로 사용할 것
- 모든 뷰, 모델, 시리얼라이저에 대한 테스트를 작성할 것
- 테스트 파일은 각 앱의 tests/ 디렉토리에 test_*.py 형식으로 작성할 것
- Factory Boy를 사용하여 테스트 데이터를 생성할 것

## 에러 처리
- try-except에서 구체적인 예외 클래스를 사용할 것 (bare except 금지)
- 커스텀 예외는 core/exceptions.py에 정의할 것
- 로깅은 Python logging 모듈을 사용할 것 (print 금지)

## 보안
- SECRET_KEY, DB 비밀번호 등은 반드시 환경 변수로 관리할 것
- .env 파일은 절대 Git에 커밋하지 말 것
- SQL 쿼리는 반드시 ORM 또는 파라미터 바인딩을 사용할 것
- CORS 설정을 명시적으로 관리할 것
```

> **💡 개념 설명: Rules에 프로젝트 정보를 넣는 이유**
>
> GEMINI.md는 Rule과 Context 섹션을 명시적으로 분리한다. 반면 Antigravity Rules는 **규칙과 컨텍스트를 하나의 파일에 혼합**하는 것이 일반적이다.
>
> "이 프로젝트는 Django 4.2를 사용한다"는 규칙이 아닌 컨텍스트이지만, Rules 파일에 포함함으로써 에이전트가 항상 올바른 버전의 코드를 생성하도록 유도한다.
>
> **한 줄 요약**: Antigravity Rules = 규칙 + 컨텍스트를 하나로 통합.

#### Step 3.4 — 팀 컨벤션 반영하기

```markdown
## 네이밍 규칙
- 변수명: snake_case (예: user_name)
- 클래스명: PascalCase (예: UserProfile)
- 상수명: UPPER_SNAKE_CASE (예: MAX_RETRY_COUNT)
- URL 이름: kebab-case (예: user-detail)
- Django 앱 이름: snake_case 단수형 (예: book, user_profile)

## Git 커밋 규칙
- 커밋 메시지 형식: `[타입] 제목`
- 타입: feat, fix, docs, style, refactor, test, chore
- 예시: `[feat] 도서 검색 API 추가`
- 한 커밋에는 하나의 논리적 변경만 포함할 것

## 코드 리뷰
- PR 설명에 변경 목적과 테스트 방법을 명시할 것
- 새로운 의존성 추가 시 requirements.txt를 갱신할 것
```

### 체크포인트

- [ ] 마크다운 형식으로 카테고리별 Rules를 작성할 수 있다
- [ ] 구체적이고 실행 가능한 규칙 문장을 작성할 수 있다
- [ ] Django 프로젝트에 필요한 주요 카테고리를 파악했다
- [ ] 규칙과 컨텍스트를 Rules 파일에 통합하는 방법을 이해했다

---

## Phase 4: Rules로 Hooks 패턴 구현

### 목표
Claude Code의 Hooks 시스템(PreToolUse, PostToolUse)을 Antigravity Rules로 대체하는 패턴을 이해하고 적용할 수 있다.

### 단계별 구현

#### Step 4.1 — Hooks vs Rules 근본 차이

Claude Code의 Hooks는 특정 이벤트(파일 수정 전, 명령 실행 후 등)에 **프로그래밍적으로** 자동 실행되는 스크립트이다. Antigravity에는 이런 명시적 훅 시스템이 **없다**. 대신 Rules에 "~전에 반드시 ~를 수행할 것"이라고 명시하면, 에이전트가 이를 **자발적으로 준수**한다.

```
# Claude Code Hooks (프로그래밍적 강제)
PreToolUse("file_edit") → 자동으로 lint 실행 → 통과해야 편집 가능

# Antigravity Rules (에이전트 자율 준수)
"파일 수정 전 반드시 lint를 실행할 것" → 에이전트가 스스로 lint 실행
```

> **💡 개념 설명: 강제 실행 vs 자율 준수**
>
> | 비교 | Claude Code Hooks | Antigravity Rules |
> |------|------------------|------------------|
> | 실행 방식 | 이벤트 기반 자동 실행 | 에이전트 자율 준수 |
> | 보장 수준 | 100% 실행 보장 | 에이전트 판단에 의존 |
> | 설정 복잡도 | JSON 설정 + 스크립트 | 마크다운 문장 |
> | 유연성 | 특정 이벤트에 바인딩 | 자연어로 자유롭게 기술 |
> | 디버깅 | 로그로 실행 여부 확인 | 에이전트 행동 관찰 |
>
> Antigravity의 접근은 **설정이 간단**하지만, 에이전트가 규칙을 무시할 수 있다는 위험이 있다. 중요한 규칙은 **굵게 강조**하거나 **"반드시", "절대"** 같은 강한 어조를 사용하면 준수율이 높아진다.
>
> **한 줄 요약**: Hooks는 기계적 강제, Rules는 에이전트에 대한 강한 요청이다.

#### Step 4.2 — Pre-action 패턴 (작업 전 규칙)

Claude Code의 `PreToolUse` hook에 해당하는 Rules 패턴이다.

```markdown
## 파일 수정 전 반드시 수행할 것

- 수정 대상 파일의 기존 테스트를 먼저 실행하여 현재 상태를 확인할 것
- `git status`를 실행하여 현재 브랜치와 변경 사항을 확인할 것
- 변경 범위가 3개 파일 이상이면 구현 계획서(Artifact)를 먼저 생성할 것
- 기존 코드의 import 구조와 네이밍 패턴을 먼저 분석한 뒤 동일한 스타일을 따를 것

## 새 파일 생성 전 반드시 수행할 것

- 동일하거나 유사한 기능의 파일이 이미 존재하는지 확인할 것
- 프로젝트의 디렉토리 구조 규칙에 맞는 위치에 생성할 것
- Django 앱이면 `python manage.py startapp`으로 생성할 것

## 외부 패키지 설치 전 반드시 수행할 것

- 현재 requirements.txt에 이미 포함되어 있는지 확인할 것
- 해당 패키지의 라이선스가 프로젝트와 호환되는지 확인할 것
- 설치 후 requirements.txt를 갱신할 것
```

#### Step 4.3 — Post-action 패턴 (작업 후 규칙)

Claude Code의 `PostToolUse` hook에 해당하는 Rules 패턴이다.

```markdown
## 파일 수정 후 반드시 수행할 것

- 수정된 함수에 대한 단위 테스트를 실행할 것
- 린트 오류가 없는지 확인할 것 (flake8 또는 ruff)
- 변경 사항을 간결한 bullet point로 요약 보고할 것
- 타입 체크(mypy)를 실행하여 타입 오류가 없는지 확인할 것

## 모델 변경 후 반드시 수행할 것

- `python manage.py makemigrations`를 실행할 것
- 마이그레이션 파일 내용을 검토하여 데이터 손실 위험이 없는지 확인할 것
- 관련된 시리얼라이저와 뷰에 변경이 필요한지 안내할 것

## 기능 구현 완료 후 반드시 수행할 것

- 해당 기능에 대한 pytest 테스트를 작성할 것
- 정상 케이스, 경계값, 예외 케이스를 모두 포함할 것
- API 엔드포인트의 경우 요청/응답 예시를 docstring에 추가할 것
```

#### Step 4.4 — Claude Code Hooks를 Rules로 변환하는 실전 예시

Claude Code의 `settings.json` hooks 설정을 Antigravity Rules로 변환하는 예시이다.

**Claude Code (hooks 설정):**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "file_edit",
        "command": "flake8 $FILE_PATH"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "file_edit",
        "command": "pytest tests/ -x --tb=short"
      }
    ]
  }
}
```

**Antigravity Rules (동일 효과):**
```markdown
## 코드 품질 보장 규칙

### 파일 수정 전
- 수정 대상 파일에 대해 `flake8`을 실행하여 기존 린트 상태를 확인할 것
- 린트 오류가 있으면 수정 내용에 함께 해결할 것

### 파일 수정 후
- 수정과 관련된 테스트를 `pytest tests/ -x --tb=short`로 실행할 것
- 테스트 실패 시 즉시 원인을 분석하고 수정할 것
- 모든 테스트가 통과한 후에만 작업 완료를 보고할 것
```

> **💡 개념 설명: 변환 시 핵심 포인트**
>
> 1. **명시적 명령어 포함**: "린트를 실행할 것"보다 "`flake8`을 실행할 것"이 더 정확하다
> 2. **실패 시 행동 정의**: Hooks는 실패 시 자동 중단하지만, Rules는 "실패 시 어떻게 할 것"까지 명시해야 한다
> 3. **순서 명시**: "수정 전", "수정 후"로 시점을 명확히 구분한다
>
> **한 줄 요약**: Hooks의 자동화를 Rules의 자연어로 옮길 때, 도구명과 실패 처리를 반드시 명시한다.

### 체크포인트

- [ ] Claude Code Hooks와 Antigravity Rules의 근본적 차이를 이해했다
- [ ] Pre-action / Post-action 패턴을 Rules로 작성할 수 있다
- [ ] Hooks 설정을 Rules로 변환하는 방법을 실습했다
- [ ] Rules의 한계(자율 준수 방식)를 인지하고 대응 전략을 알고 있다

---

## Phase 5: 실전 예시

### 목표
완전한 `.agent/` 디렉토리 구조와 `rules.md` 예시를 확인하고, 자신의 프로젝트에 즉시 적용할 수 있다.

### 단계별 구현

#### Step 5.1 — .agent/ 디렉토리 전체 구조

```
my-django-project/
├── .agent/
│   ├── rules.md                    ← Rules (항상 로드)
│   ├── workflows/
│   │   ├── generate-tests.md       ← /generate-tests 명령
│   │   ├── deploy-staging.md       ← /deploy-staging 명령
│   │   └── code-review.md          ← /code-review 명령
│   └── skills/
│       ├── django-deploy/
│       │   ├── SKILL.md
│       │   └── instructions.md
│       └── db-migration/
│           ├── SKILL.md
│           └── instructions.md
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   ├── urls.py
│   └── wsgi.py
├── books/
│   ├── models.py
│   ├── views.py
│   ├── serializers.py
│   ├── urls.py
│   └── tests/
│       ├── test_models.py
│       └── test_views.py
├── users/
│   ├── models.py
│   ├── views.py
│   └── urls.py
├── core/
│   ├── exceptions.py
│   ├── permissions.py
│   └── utils.py
├── manage.py
├── requirements.txt
├── .env.example
├── .gitignore
└── AGENTS.md                       ← v1.20.3+에서 추가 인식
```

#### Step 5.2 — 완성된 rules.md 전체 예시

```markdown
# Django 온라인 서점 프로젝트 Rules

## 프로젝트 개요
- Django 4.2 + DRF 3.14 기반 온라인 서점 REST API
- Python 3.12, PostgreSQL 15, Redis 7 사용
- JWT 인증 (djangorestframework-simplejwt)
- Docker 기반 배포

## 디렉토리 구조 규칙
- Django 앱: books/, users/, orders/, core/
- 설정 파일: config/settings/ (base.py, dev.py, prod.py)
- 테스트: 각 앱의 tests/ 디렉토리
- 정적 파일: static/, 업로드 파일: media/

## 코딩 스타일
- PEP 8 준수, Black 포매터 기준 88자 제한
- import 순서: stdlib → third-party → local (isort 사용)
- 모든 함수/클래스에 Google 스타일 docstring 필수
- type hints를 모든 함수 시그니처에 포함할 것
- 변수: snake_case, 클래스: PascalCase, 상수: UPPER_SNAKE_CASE

## Django 규칙
- 뷰: 클래스 기반 뷰(CBV) 우선, 필요 시 함수 기반 뷰 허용
- URL: 앱별 urls.py 분리 + include() 연결
- 모델: created_at(auto_now_add), updated_at(auto_now) 필수 포함
- 외래 키: on_delete 옵션 반드시 명시
- QuerySet: select_related/prefetch_related로 N+1 문제 방지

## API 설계
- ViewSet + Router 조합으로 RESTful 엔드포인트 구성
- 응답 형식:
  ```json
  {"status": "success", "data": {}, "message": ""}
  ```
- 에러 응답:
  ```json
  {"status": "error", "code": "ERROR_CODE", "message": "설명"}
  ```
- 페이지네이션: PageNumberPagination, 기본 20개
- 필터링: django-filter 사용, 검색: SearchFilter

## 파일 수정 전 반드시 수행
- 수정 대상 파일의 기존 테스트를 실행하여 현재 통과 상태 확인
- git status로 현재 브랜치와 미커밋 변경 사항 확인
- 변경 범위가 3개 파일 이상이면 계획서를 먼저 작성

## 파일 수정 후 반드시 수행
- 수정된 함수에 대한 단위 테스트 실행 (pytest -x --tb=short)
- flake8 또는 ruff로 린트 오류 확인
- 변경 사항을 bullet point로 요약 보고

## 모델 변경 후 반드시 수행
- makemigrations 실행 안내
- 마이그레이션 파일 검토 (데이터 손실 위험 확인)
- 관련 시리얼라이저/뷰 변경 필요 여부 안내

## 테스트
- pytest + pytest-django + factory_boy 사용
- 커버리지 80% 이상 유지 (pytest-cov)
- 정상 케이스, 경계값, 예외 케이스 모두 포함
- API 테스트: APIClient를 사용한 통합 테스트 작성

## 에러 처리
- 구체적 예외 클래스 사용 (bare except 금지)
- 커스텀 예외: core/exceptions.py에 정의
- 로깅: Python logging 모듈 사용 (print 금지)
- DRF exception_handler 커스터마이징하여 일관된 에러 응답

## 보안
- SECRET_KEY, DB 비밀번호 등은 환경 변수로 관리
- .env 파일은 절대 Git에 커밋하지 말 것
- SQL: ORM 또는 파라미터 바인딩만 사용 (raw SQL 지양)
- CORS: django-cors-headers로 명시적 관리
- 인증이 필요한 엔드포인트에는 permission_classes 반드시 명시

## Git 커밋
- 형식: [타입] 제목 (예: [feat] 도서 검색 API 추가)
- 타입: feat, fix, docs, style, refactor, test, chore
- 한 커밋 = 하나의 논리적 변경
- 브랜치: main, develop, feature/*, hotfix/*
```

#### Step 5.3 — .gitignore에 추가할 항목

`.agent/` 디렉토리는 **팀과 공유**해야 하므로 `.gitignore`에 포함하지 않는다. 단, 개인 설정이 포함된 글로벌 Rules는 Git 관리 대상이 아니다.

```bash
# .gitignore

# Antigravity 관련
# .agent/ 디렉토리는 커밋한다 (팀 공유 목적)
# 글로벌 rules는 ~/.gemini/ 에 있으므로 관리 불필요

# 환경 변수
.env
.env.local
.env.production

# Python
__pycache__/
*.pyc
venv/
```

### 체크포인트

- [ ] `.agent/` 디렉토리의 전체 구조를 이해했다
- [ ] 완성된 `rules.md` 예시를 확인하고 자신의 프로젝트에 맞게 수정할 수 있다
- [ ] `.agent/` 디렉토리를 Git으로 버전 관리하는 것이 권장됨을 알고 있다

---

## 마무리

### 학습한 내용 정리

Antigravity Rules는 에이전트의 행동을 가이드하는 **시스템 지시** 역할을 한다.

1. **스코프**: 글로벌(`~/.gemini/antigravity/rules.md`) + 워크스페이스(`<project>/.agent/rules.md`)의 2단계
2. **형식**: 마크다운으로 작성, 카테고리별 분류
3. **역할 통합**: 규칙 + 컨텍스트 + Hooks 패턴을 하나의 파일에서 관리
4. **편집 방법**: 파일 직접 편집 또는 UI(Customizations 메뉴)
5. **버전 관리**: `.agent/` 디렉토리를 Git으로 커밋하여 팀과 공유

### 멀티 도구 비교표

| 기능 | Antigravity | Claude Code | Gemini CLI |
|------|-------------|-------------|------------|
| 규칙 파일 | `.agent/rules.md` | `CLAUDE.md` | `GEMINI.md` |
| 글로벌 경로 | `~/.gemini/antigravity/rules.md` | `~/.claude/CLAUDE.md` | `~/.gemini/GEMINI.md` |
| 스코프 단계 | 2단계 | 3단계 | 3단계 |
| 컴포넌트 레벨 | 미지원 | 서브디렉토리별 | 서브디렉토리별 |
| UI 편집 | O (Customizations) | X | X |
| Hooks 시스템 | Rules에 통합 | 별도 Hooks | 없음 |
| AGENTS.md 지원 | O (v1.20.3+) | X | X |
| 규칙/컨텍스트 분리 | 통합 | 통합 | Rule/Context 분리 |
| 자동 생성 | X | X | `gemini init` |
| 저장 프롬프트 | Workflows | 없음 (수동) | 없음 (수동) |
| 조건부 지식 | Skills | Subagents | 없음 |
| 병렬 에이전트 | Agent Manager | Task() | 없음 |

### Antigravity만의 차별점

1. **단순함**: 2단계 스코프로 관리 부담이 적다
2. **통합 관리**: `.agent/` 디렉토리에 Rules, Workflows, Skills를 모두 모아둔다
3. **UI 편집**: Customizations 메뉴로 비개발자도 규칙을 편집할 수 있다
4. **Rules = Hooks**: 별도 훅 시스템 없이 자연어 규칙으로 전/후 처리를 정의한다
5. **AGENTS.md 호환**: v1.20.3부터 표준화된 `AGENTS.md` 파일도 인식한다

### 자주 하는 실수

1. **rules.md를 프로젝트 루트에 놓는 실수**
   - ❌ `my-project/rules.md`
   - ✅ `my-project/.agent/rules.md`

2. **글로벌 Rules에 프로젝트별 규칙을 넣는 실수**
   - ❌ 글로벌: "Django CBV를 사용할 것"
   - ✅ 글로벌: "함수에 type hints를 포함할 것"

3. **너무 추상적인 규칙**
   - ❌ "코드를 깔끔하게 작성할 것"
   - ✅ "함수는 최대 30줄, 한 줄은 최대 88자를 넘지 않을 것"

4. **Hooks 패턴에서 실패 처리를 빠뜨리는 실수**
   - ❌ "테스트를 실행할 것"
   - ✅ "테스트를 실행하고, 실패 시 원인을 분석하여 수정할 것"

5. **버전 정보 누락**
   - ❌ "Django와 PostgreSQL을 사용한다"
   - ✅ "Django 4.2, PostgreSQL 15를 사용한다"

6. **.agent/ 디렉토리를 .gitignore에 추가하는 실수**
   - `.agent/`는 팀 공유 목적이므로 Git에 커밋해야 한다
   - 글로벌 Rules(`~/.gemini/`)만 개인 설정이다

### 다음 단계

1. **자신의 프로젝트에 `.agent/rules.md` 작성**
   - 이 가이드의 Django 예시를 참고하여 프로젝트에 맞게 수정

2. **Workflows와 Skills 연동**
   - Rules를 기반으로 반복 작업을 Workflows로 자동화
   - 전문 지식을 Skills로 패키징

3. **팀과 공유 및 반복 개선**
   - `.agent/` 디렉토리를 Git에 커밋
   - 코드 리뷰 시 Rules 준수 여부 확인
   - 프로젝트 변경에 따라 Rules를 주기적으로 업데이트

4. **AGENTS.md 활용 검토**
   - 멀티 도구 환경(Antigravity + Gemini CLI)에서는 AGENTS.md로 통합 관리 고려
