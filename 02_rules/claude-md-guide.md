# CLAUDE.md 작성 가이드

## 학습 목표

Claude Code에서 사용하는 CLAUDE.md 파일의 구조와 작성 방법을 이해하고, Django 프로젝트에 적용할 수 있는 실무 중심의 프로젝트 지침 파일을 직접 작성할 수 있다.

## 사전 준비

- Claude Code 설치 및 기본 사용법 숙지
- Django 프로젝트 구조에 대한 기본 이해
- 마크다운 문법 기초 지식

---

## 전체 흐름 한눈에 보기

CLAUDE.md는 Claude Code가 프로젝트를 이해하고 작업할 때 참고하는 "프로젝트 지침서"이다. 이 파일은 프로젝트 루트에 위치하며, **자유 형식(free-form) 마크다운**으로 작성한다. GEMINI.md와 달리 별도의 섹션 규격(Rule/Context)이 없고, 자연어로 원하는 지시 사항을 자유롭게 기술한다.

**주요 단계:**
1. CLAUDE.md의 역할과 필요성 이해
2. 스코프 계층(글로벌/프로젝트/컴포넌트) 파악
3. CLAUDE.md 작성법 및 메모리 시스템 활용
4. settings.json과의 관계 이해
5. .claudeignore 설정
6. Django 프로젝트 예시로 실전 적용

---

## Phase 1: CLAUDE.md 이해하기

### 목표
CLAUDE.md가 무엇인지, 왜 필요한지, 다른 도구의 규칙 파일과 어떻게 다른지 명확히 이해한다.

### 단계별 구현

#### Step 1.1 -- CLAUDE.md의 역할 파악하기

CLAUDE.md는 Claude Code가 **모든 대화 시작 시 자동으로 읽어들이는** 프로젝트 지침 파일이다. 이 파일에 작성된 내용은 시스템 프롬프트보다 강하게 적용되어, Claude가 코드를 생성하거나 수정할 때 반드시 따르는 규칙이 된다.

> **💡 개념 설명: 왜 CLAUDE.md가 필요한가?**
>
> Claude Code는 대화마다 매번 초기화된다. 프로젝트 규칙을 매번 설명하는 대신, CLAUDE.md에 한 번 작성해두면:
> - **일관성**: 모든 대화에서 동일한 코딩 스타일 유지
> - **효율성**: 매번 "우리 프로젝트는 DRF 사용해"라고 말할 필요 없음
> - **정확성**: 프로젝트 구조를 미리 알려줘서 잘못된 경로나 파일명 생성 방지
>
> **한 줄 요약**: CLAUDE.md는 Claude Code의 프로젝트별 작업 지침서다.

#### Step 1.2 -- GEMINI.md와의 핵심 차이점

| 항목 | CLAUDE.md | GEMINI.md |
|------|-----------|-----------|
| **파일 위치** | 프로젝트 루트 (`./CLAUDE.md`) | 프로젝트 루트 (`./GEMINI.md`) |
| **파일 형식** | 자유 형식 마크다운 (섹션 규격 없음) | Rule/Context 섹션 구분 필요 |
| **프론트매터** | 불필요 | 불필요 |
| **권한 설정** | `settings.json`에서 별도 관리 | GEMINI.md 안에서 혼합 관리 |
| **메모리 시스템** | `/memory` 명령 + 자동 메모리 지원 | 해당 없음 |
| **서브디렉토리** | 컴포넌트별 CLAUDE.md 지원 | 미지원 |
| **로딩 방식** | 대화 시작 시 자동 로드 | CLI 실행 시 자동 로드 |

> **💡 개념 설명: 자유 형식 마크다운이란?**
>
> GEMINI.md는 `## Rule`과 `## Context` 섹션을 구분해야 하지만, CLAUDE.md는 이런 구분이 없다.
> 마크다운 헤더, 리스트, 코드 블록 등을 자유롭게 사용하여 지침을 작성하면 된다.
> Claude는 마크다운의 **의미**를 이해하므로, 자연어로 명확하게 쓰면 그대로 따른다.
>
> **한 줄 요약**: CLAUDE.md는 정해진 형식 없이 마크다운으로 자유롭게 작성한다.

#### Step 1.3 -- 기본 파일 위치 확인

CLAUDE.md는 **프로젝트 루트 디렉토리**에 위치한다. GEMINI.md와 달리 `.claude/` 디렉토리 안이 아닌, 프로젝트 최상위에 둔다.

```bash
# 파일 위치 예시

my-django-project/
├── CLAUDE.md          # 여기! (프로젝트 루트)
├── .claude/           # settings.json은 이 안에 위치
│   ├── settings.json
│   └── settings.local.json
├── manage.py
├── requirements.txt
├── myapp/
│   ├── CLAUDE.md      # 컴포넌트별 CLAUDE.md (선택)
│   ├── models.py
│   └── views.py
└── config/
    └── settings.py
```

### 체크포인트

- [ ] CLAUDE.md가 자유 형식 마크다운임을 이해했다
- [ ] GEMINI.md와의 핵심 차이점(형식, 권한 분리, 메모리)을 설명할 수 있다
- [ ] 프로젝트 루트에 CLAUDE.md 파일을 생성할 위치를 알고 있다

---

## Phase 2: 스코프 계층 (3단계)

### 목표
CLAUDE.md의 글로벌/프로젝트/컴포넌트 3단계 스코프 계층과 로딩 순서를 이해한다.

### 단계별 구현

#### Step 2.1 -- 3단계 스코프 계층 이해하기

Claude Code는 CLAUDE.md를 **세 단계 계층**으로 관리한다. 더 구체적인(좁은 범위의) 파일이 더 넓은 범위의 파일보다 우선한다.

```
스코프 계층 (넓음 → 좁음)

┌─────────────────────────────────────────────┐
│ 1. 글로벌: ~/.claude/CLAUDE.md              │
│    - 모든 프로젝트에 공통 적용              │
│    - 개인 코딩 스타일, 선호 언어 등         │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │ 2. 프로젝트: {project-root}/CLAUDE.md  │ │
│  │    - Git에 커밋하여 팀원과 공유         │ │
│  │    - 프로젝트 기술 스택, 규칙 등        │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐  │ │
│  │  │ 3. 컴포넌트: {subdir}/CLAUDE.md  │  │ │
│  │  │    - 특정 디렉토리에만 적용       │  │ │
│  │  │    - 모듈별 고유 규칙             │  │ │
│  │  └──────────────────────────────────┘  │ │
│  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

#### Step 2.2 -- 글로벌 CLAUDE.md

```bash
# 위치: ~/.claude/CLAUDE.md
# 적용 범위: 모든 Claude Code 세션

# 글로벌 CLAUDE.md 생성
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md << 'EOF'
# 글로벌 코딩 규칙

- 모든 응답은 한국어로 작성한다
- 코드 주석은 한국어로 작성한다
- Git 커밋 메시지는 한국어로 작성한다
- 함수와 클래스에는 반드시 docstring을 포함한다
EOF
```

> **💡 개념 설명: 글로벌 CLAUDE.md의 용도**
>
> 글로벌 CLAUDE.md는 **개인적인** 작업 선호도를 설정하는 곳이다.
> 모든 프로젝트에서 공통으로 적용하고 싶은 규칙(응답 언어, 코딩 스타일 등)을 여기에 둔다.
> 이 파일은 Git에 커밋되지 않으므로, 팀원에게 영향을 주지 않는다.
>
> **한 줄 요약**: 글로벌 CLAUDE.md는 "나만의" 작업 규칙이다.

#### Step 2.3 -- 프로젝트 CLAUDE.md

```bash
# 위치: {project-root}/CLAUDE.md
# 적용 범위: 해당 프로젝트의 모든 대화

# 프로젝트 CLAUDE.md 생성 (또는 /init 명령 사용)
cat > ./CLAUDE.md << 'EOF'
# Django 블로그 프로젝트

## 기술 스택
- 백엔드: Django 4.2, Django REST Framework 3.14
- 데이터베이스: PostgreSQL 15
- 테스트: pytest, pytest-django

## 코딩 규칙
- PEP 8 스타일 가이드를 따른다
- import 순서: 표준 라이브러리 > 서드파티 > 로컬
- 모든 뷰는 클래스 기반 뷰(CBV)를 우선 사용한다
EOF
```

프로젝트 CLAUDE.md는 **Git에 커밋하여 팀원과 공유**한다. 팀 전체가 동일한 규칙으로 Claude Code를 사용할 수 있다.

#### Step 2.4 -- 컴포넌트(서브디렉토리) CLAUDE.md

```bash
# 위치: {project-root}/{subdirectory}/CLAUDE.md
# 적용 범위: 해당 디렉토리 내 파일 작업 시

# 예: API 앱 전용 규칙
cat > ./api/CLAUDE.md << 'EOF'
# API 앱 규칙

- 모든 API 뷰는 ViewSet을 사용한다
- Serializer는 반드시 validation 로직을 포함한다
- 페이지네이션은 CursorPagination을 사용한다
EOF

# 예: 프론트엔드 디렉토리 전용 규칙
cat > ./frontend/CLAUDE.md << 'EOF'
# 프론트엔드 규칙

- React 18 + TypeScript를 사용한다
- 컴포넌트는 함수형 컴포넌트로 작성한다
- 상태 관리는 Zustand를 사용한다
EOF
```

> **💡 개념 설명: 컴포넌트 CLAUDE.md가 유용한 경우**
>
> 모노레포나 프론트엔드/백엔드가 분리된 프로젝트에서 특히 유용하다.
> Claude가 특정 디렉토리의 파일을 읽거나 수정할 때, 해당 디렉토리의 CLAUDE.md가 자동으로 로드된다.
> 예를 들어 `/api` 폴더 작업 시 API 전용 규칙이, `/frontend` 폴더 작업 시 프론트엔드 규칙이 적용된다.
>
> **한 줄 요약**: 서브디렉토리 CLAUDE.md로 모듈별 맞춤 규칙을 적용한다.

#### Step 2.5 -- 로컬 오버라이드: CLAUDE.local.md

```bash
# 위치: {project-root}/CLAUDE.local.md
# 용도: 개인 설정 (Git에 커밋하지 않음, .gitignore에 추가)

cat > ./CLAUDE.local.md << 'EOF'
# 개인 작업 설정 (팀 공유 안 함)

- 디버깅 시 print 문 사용을 허용한다
- 테스트 실행 시 verbose 모드를 사용한다
- 로컬 개발 서버는 포트 8080을 사용한다
EOF
```

#### Step 2.6 -- 로딩 순서와 우선순위

Claude Code가 시작되면 다음 순서로 CLAUDE.md를 로드한다:

```
로딩 순서 (먼저 → 나중에)
────────────────────────────

1. ~/.claude/CLAUDE.md             (글로벌)
2. {project-root}/CLAUDE.md        (프로젝트, Git 커밋됨)
3. {project-root}/CLAUDE.local.md  (로컬 오버라이드, .gitignore)
4. {subdirectory}/CLAUDE.md        (컴포넌트, 해당 디렉토리 작업 시)

우선순위: 더 구체적인(좁은 범위) 파일이 더 넓은 범위의 파일보다 우선한다.
```

모든 파일의 내용은 **병합(merge)** 된다. 서로 충돌하는 지시가 있으면, 더 구체적인 스코프의 내용이 우선한다.

### 체크포인트

- [ ] 글로벌/프로젝트/컴포넌트 3단계 스코프를 설명할 수 있다
- [ ] CLAUDE.local.md의 용도와 .gitignore 설정을 이해했다
- [ ] 로딩 순서와 우선순위 규칙을 파악했다

---

## Phase 3: CLAUDE.md 작성법

### 목표
효과적인 CLAUDE.md를 작성하는 구체적인 방법과 메모리 시스템을 활용하는 방법을 익힌다.

### 단계별 구현

#### Step 3.1 -- CLAUDE.md의 권장 구성 요소

CLAUDE.md는 자유 형식이지만, 다음 구성 요소를 포함하면 효과적이다:

```markdown
# 프로젝트명

## 프로젝트 개요
이 프로젝트가 무엇인지 한두 문장으로 설명

## 기술 스택
사용 중인 프레임워크, 라이브러리, 버전 정보

## 디렉토리 구조
주요 폴더와 파일의 역할

## 코딩 규칙
코드 스타일, 네이밍 컨벤션, 아키텍처 패턴

## 주요 명령어
빌드, 테스트, 배포 등 자주 쓰는 명령

## 아키텍처 결정 사항
기술 선택의 이유, 설계 원칙
```

> **💡 개념 설명: CLAUDE.md 작성 원칙**
>
> 좋은 CLAUDE.md는:
> 1. **간결하다**: 각 줄에 대해 "이걸 빼면 Claude가 실수할까?" 질문한다. 아니면 삭제한다
> 2. **구체적이다**: "코드를 깔끔하게" (X) --> "함수는 최대 50줄을 넘지 않는다" (O)
> 3. **실행 가능하다**: Claude가 즉시 적용할 수 있는 명확한 지시문
>
> 너무 길면(200줄 이상) Claude가 지시를 무시할 수 있다. **핵심만 남기는 것**이 중요하다.
>
> **한 줄 요약**: CLAUDE.md는 짧고 구체적일수록 효과적이다.

#### Step 3.2 -- /init으로 초기 CLAUDE.md 생성하기

Claude Code는 프로젝트를 분석하여 초기 CLAUDE.md를 자동 생성할 수 있다.

```bash
# 프로젝트 루트에서 Claude Code 실행 후
/init
```

`/init` 명령은:
1. 프로젝트의 파일 구조를 스캔한다
2. `requirements.txt`, `package.json` 등에서 기술 스택을 추출한다
3. 주요 파일(`models.py`, `views.py` 등)을 분석한다
4. 기본 CLAUDE.md 초안을 생성한다

생성된 초안을 검토하고, 팀 규칙과 프로젝트 특성에 맞게 수정한다.

#### Step 3.3 -- 효과적인 섹션별 작성 예시

**프로젝트 개요 및 기술 스택:**

```markdown
# Django 블로그 프로젝트

Django 기반의 블로그 플랫폼이다. 사용자는 게시글을 작성하고 댓글을 달 수 있다.

## 기술 스택
- 백엔드: Django 4.2.7, Django REST Framework 3.14.0
- 데이터베이스: PostgreSQL 15.3
- 인증: djangorestframework-simplejwt 5.3.0
- 테스트: pytest 7.4, pytest-django 4.5
- 배포: Docker, Gunicorn, Nginx
```

**코딩 규칙:**

```markdown
## 코딩 규칙

- 모든 Python 코드는 PEP 8을 따른다
- Black 포매터 사용 (최대 줄 길이 88자)
- import 순서: 표준 라이브러리 > 서드파티 > 로컬 (isort 적용)
- 함수와 클래스에는 Google 스타일 docstring을 작성한다
- try-except에서 구체적인 예외 클래스를 사용한다 (bare except 금지)
- 로깅은 Python logging 모듈을 사용한다 (print 금지)
```

**주요 명령어:**

```markdown
## 주요 명령어

- 개발 서버 실행: `python manage.py runserver`
- 테스트 실행: `pytest`
- 테스트 커버리지: `pytest --cov`
- 마이그레이션 생성: `python manage.py makemigrations`
- 마이그레이션 적용: `python manage.py migrate`
- 린트 검사: `ruff check .`
- 포맷팅: `black .`
```

> **💡 개념 설명: 명령어 섹션이 중요한 이유**
>
> Claude Code는 CLAUDE.md에 명시된 명령어를 보고:
> - 테스트 실행 방법을 즉시 파악한다
> - 빌드 도구를 올바르게 사용한다
> - 린트/포맷팅 도구를 자동으로 적용한다
>
> 특히 `pytest` vs `python manage.py test` 같은 선택이 프로젝트마다 다르므로 명시가 필요하다.
>
> **한 줄 요약**: 명령어를 명시하면 Claude가 올바른 도구를 사용한다.

#### Step 3.4 -- @import 구문으로 외부 파일 참조하기

CLAUDE.md에서 `@경로` 구문을 사용하면 다른 파일의 내용을 참조할 수 있다.

```markdown
# 프로젝트 지침

프로젝트 개요는 @README.md 를 참고하라.
사용 가능한 스크립트는 @package.json 을 확인하라.

## 추가 지침
- Git 워크플로우: @docs/git-workflow.md
- API 설계 가이드: @docs/api-guide.md
- 개인 설정: @~/.claude/my-project-overrides.md
```

> **💡 개념 설명: @import의 장점**
>
> CLAUDE.md를 200줄 이내로 유지하면서도, 상세 문서를 별도 파일로 분리하여 참조할 수 있다.
> 이미 README.md나 CONTRIBUTING.md에 작성된 내용을 반복하지 않아도 된다.
>
> **한 줄 요약**: @경로 구문으로 외부 파일을 참조하여 CLAUDE.md를 간결하게 유지한다.

#### Step 3.5 -- 메모리 시스템 활용하기

Claude Code에는 CLAUDE.md 외에도 **자동 메모리(Auto Memory)** 시스템이 있다. Claude가 작업하면서 스스로 배운 내용을 자동으로 저장한다.

**메모리 관련 명령어:**

```bash
# 메모리 상태 확인
/memory

# 출력 예시:
# CLAUDE.md files loaded:
#   - ~/.claude/CLAUDE.md (global)
#   - ./CLAUDE.md (project)
#   - ./api/CLAUDE.md (component)
#
# Auto memory: enabled
# Memory location: ~/.claude/projects/<project>/memory/
```

**자동 메모리 저장 위치:**

```
~/.claude/
├── CLAUDE.md                           # 글로벌 규칙
└── projects/
    └── <project-hash>/
        └── memory/                     # 자동 메모리 저장소
            ├── MEMORY.md               # 메인 메모리 파일
            ├── debugging-notes.md      # 디버깅 인사이트
            └── architecture-notes.md   # 아키텍처 메모
```

**자동 메모리가 저장하는 내용:**

- 빌드/테스트 명령어와 그 결과
- 디버깅 중 발견한 패턴
- 아키텍처 관련 메모
- 코드 스타일 선호도
- 프로젝트별 워크플로우 습관

> **💡 개념 설명: CLAUDE.md vs 자동 메모리**
>
> | 항목 | CLAUDE.md | 자동 메모리 |
> |------|-----------|-------------|
> | 작성자 | 사용자(개발자) | Claude가 자동 생성 |
> | 위치 | 프로젝트 루트 | `~/.claude/projects/` |
> | Git 커밋 | 가능 (팀 공유) | 불가 (개인 로컬) |
> | 용도 | 명시적 규칙, 프로젝트 정보 | 학습된 패턴, 디버깅 노트 |
> | 관리 | 수동 편집 | 자동 (수동 편집도 가능) |
>
> **한 줄 요약**: CLAUDE.md는 "내가 알려주는 규칙", 자동 메모리는 "Claude가 스스로 배운 것"이다.

**자동 메모리 비활성화 (필요 시):**

```bash
# 환경 변수로 비활성화
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1

# 또는 /memory 명령에서 토글
/memory
# → auto memory toggle 사용
```

### 체크포인트

- [ ] CLAUDE.md의 권장 구성 요소(개요, 기술 스택, 규칙, 명령어 등)를 파악했다
- [ ] `/init`으로 초기 CLAUDE.md를 생성하는 방법을 알고 있다
- [ ] @import 구문으로 외부 파일을 참조하는 방법을 이해했다
- [ ] `/memory` 명령과 자동 메모리 시스템을 이해했다

---

## Phase 4: settings.json과의 관계

### 목표
CLAUDE.md와 settings.json의 역할 차이를 이해하고, 권한 설정을 올바르게 구성할 수 있다.

### 단계별 구현

#### Step 4.1 -- CLAUDE.md vs settings.json 구분하기

Claude Code는 두 종류의 설정 파일을 사용한다:

| 항목 | CLAUDE.md | settings.json |
|------|-----------|---------------|
| **형식** | 마크다운 (자연어) | JSON (구조화된 데이터) |
| **용도** | 프로젝트 지침, 코딩 규칙 | 권한, 도구 허용, MCP 설정 |
| **내용** | "Django CBV를 사용하라" | `{"allowedTools": ["Bash"]}` |
| **위치** | 프로젝트 루트 | `.claude/` 디렉토리 내 |

> **💡 개념 설명: 왜 분리되어 있는가?**
>
> GEMINI.md는 규칙과 설정이 하나의 파일에 혼합되어 있지만,
> Claude Code는 **"무엇을 해야 하는가"(CLAUDE.md)** 와 **"무엇을 할 수 있는가"(settings.json)** 를 분리한다.
>
> 이는 보안 관점에서 중요하다. CLAUDE.md는 누구나 편집할 수 있지만,
> settings.json의 deny 규칙은 다른 스코프에서 절대 덮어쓸 수 없기 때문이다.
>
> **한 줄 요약**: CLAUDE.md는 "지침", settings.json은 "권한"이다.

#### Step 4.2 -- settings.json 3단계 스코프

```
settings.json 스코프 계층
─────────────────────────

1. ~/.claude/settings.json          (사용자 전역)
   - 모든 프로젝트에 적용
   - 개인 도구 권한, 글로벌 MCP 서버 설정

2. .claude/settings.json            (프로젝트, Git 커밋됨)
   - 팀원과 공유되는 프로젝트 권한
   - 프로젝트 MCP 서버 설정

3. .claude/settings.local.json      (로컬 오버라이드, .gitignore)
   - 개인별 권한 오버라이드
   - Git에 포함하지 않는 로컬 설정
```

#### Step 4.3 -- settings.json 작성 예시

**사용자 전역 설정 (`~/.claude/settings.json`):**

```json
{
  "permissions": {
    "allow": [
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git status)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)"
    ]
  },
  "env": {
    "CLAUDE_CODE_DISABLE_AUTO_MEMORY": "0"
  }
}
```

**프로젝트 설정 (`.claude/settings.json`):**

```json
{
  "permissions": {
    "allow": [
      "Bash(python manage.py test*)",
      "Bash(pytest*)",
      "Bash(python manage.py makemigrations*)",
      "Bash(python manage.py migrate*)"
    ],
    "deny": [
      "Bash(python manage.py flush*)",
      "Bash(python manage.py dbshell*)"
    ]
  },
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/blog_db"
      }
    }
  }
}
```

**로컬 오버라이드 (`.claude/settings.local.json`):**

```json
{
  "permissions": {
    "allow": [
      "Bash(python manage.py runserver*)"
    ]
  }
}
```

#### Step 4.4 -- "Deny가 항상 이긴다" 규칙

Claude Code의 권한 평가는 다음 순서를 따른다:

```
권한 평가 순서
──────────────

1. deny 규칙 확인 → 매치되면 즉시 차단 (어떤 스코프든)
2. ask 규칙 확인 → 매치되면 사용자에게 확인 요청
3. allow 규칙 확인 → 매치되면 자동 허용
```

**핵심 원칙: deny 규칙은 어떤 스코프에서든 한 번 설정되면 다른 스코프에서 절대 덮어쓸 수 없다.**

```json
// ~/.claude/settings.json (사용자 전역)
{
  "permissions": {
    "deny": ["Bash(rm -rf*)"]   // 여기서 deny하면
  }
}

// .claude/settings.json (프로젝트)
{
  "permissions": {
    "allow": ["Bash(rm -rf*)"]  // 여기서 allow해도 무효!
  }
}
// → deny가 항상 우선. rm -rf는 영원히 차단됨
```

> **💡 개념 설명: 보안 심층 방어(Defense in Depth)**
>
> "deny가 항상 이긴다" 원칙은 보안 관점에서 중요하다.
> - 관리자가 전역 deny를 설정하면, 개별 프로젝트에서 이를 우회할 수 없다
> - 실수로 위험한 명령을 allow하더라도, 상위 deny가 보호한다
> - 이는 GEMINI.md에는 없는 Claude Code만의 보안 기능이다
>
> **한 줄 요약**: deny 규칙은 절대적이다. 어디서든 deny하면 어디서도 allow할 수 없다.

### 체크포인트

- [ ] CLAUDE.md와 settings.json의 역할 차이를 명확히 구분할 수 있다
- [ ] settings.json의 3단계 스코프를 이해했다
- [ ] "deny가 항상 이긴다" 원칙을 설명할 수 있다
- [ ] 프로젝트에 맞는 permissions를 작성할 수 있다

---

## Phase 5: .claudeignore 설정

### 목표
.claudeignore를 활용하여 Claude Code의 파일 접근을 제어하는 방법을 익힌다.

### 단계별 구현

#### Step 5.1 -- .claudeignore 기본 이해

`.claudeignore` 파일은 `.gitignore`와 동일한 문법을 사용하여, Claude Code의 컨텍스트 윈도우에서 특정 파일을 제외한다.

```bash
# 위치: 프로젝트 루트
my-django-project/
├── .claudeignore      # 여기!
├── .gitignore
├── CLAUDE.md
└── ...
```

#### Step 5.2 -- .claudeignore 작성 예시

```gitignore
# .claudeignore

# 환경 변수 및 비밀 파일
.env
.env.*
*.pem
*.key
credentials.json

# 빌드 아티팩트
__pycache__/
*.pyc
*.pyo
dist/
build/
*.egg-info/

# 대용량 데이터 파일
*.sql
*.dump
data/fixtures/large_*.json

# 미디어 및 정적 파일 (수집된)
media/uploads/
staticfiles/

# IDE 설정
.vscode/
.idea/

# 로그 파일
*.log
logs/

# 노드 모듈 (프론트엔드 포함 시)
node_modules/

# Docker 관련
docker-compose.override.yml
```

#### Step 5.3 -- .claudeignore의 한계와 보완

> **💡 개념 설명: .claudeignore만으로는 보안이 불완전하다**
>
> `.claudeignore`는 Claude의 **컨텍스트 윈도우**에서 파일을 제외하지만,
> 완벽한 접근 차단을 보장하지는 않는다. 민감한 파일(`.env`, 비밀키 등)의 접근을 확실히 차단하려면
> **settings.json의 deny 규칙**을 함께 사용해야 한다.
>
> ```json
> // .claude/settings.json
> {
>   "permissions": {
>     "deny": [
>       "Read(.env*)",
>       "Read(*.pem)",
>       "Read(credentials.json)"
>     ]
>   }
> }
> ```
>
> **한 줄 요약**: .claudeignore는 컨텍스트 제외, settings.json deny는 접근 차단이다. 둘 다 쓰자.

### 체크포인트

- [ ] .claudeignore의 문법(.gitignore와 동일)을 이해했다
- [ ] 프로젝트에 맞는 .claudeignore 패턴을 작성할 수 있다
- [ ] .claudeignore의 한계를 알고, settings.json deny로 보완할 수 있다

---

## Phase 6: 실전 예시 - Django 블로그 프로젝트

### 목표
실제 Django 블로그 프로젝트에 대한 완전한 CLAUDE.md 예시를 보고, GEMINI.md와의 차이를 이해하며, 자신의 프로젝트에 적용할 수 있다.

### 완성된 CLAUDE.md 예시

```markdown
# Django 블로그 프로젝트

Django 기반의 개인 블로그 플랫폼이다. 사용자는 회원가입 후 게시글을 작성하고,
다른 사용자의 게시글에 댓글을 달 수 있다. 태그 기반 검색과 카테고리 분류를 지원한다.

## 기술 스택

- **백엔드**: Django 4.2.7, Django REST Framework 3.14.0
- **데이터베이스**: PostgreSQL 15.3
- **인증**: djangorestframework-simplejwt 5.3.0
- **이미지 처리**: Pillow 10.1.0
- **배포**: Docker 24.0, Gunicorn 21.2, Nginx 1.25
- **테스트**: pytest 7.4, pytest-django 4.5
- **버전 관리**: Git, GitHub

## 디렉토리 구조

```
blog_project/
├── config/                 # Django 프로젝트 설정
│   ├── settings/
│   │   ├── base.py        # 공통 설정
│   │   ├── dev.py         # 개발 환경
│   │   └── prod.py        # 운영 환경
│   ├── urls.py            # 루트 URL
│   └── wsgi.py
├── posts/                 # 게시글 관리 앱
│   ├── models.py          # Post, Category, Tag 모델
│   ├── views.py           # CBV 뷰
│   ├── serializers.py     # DRF 직렬화
│   ├── urls.py
│   └── tests/
│       ├── test_models.py
│       └── test_views.py
├── comments/              # 댓글 관리 앱
│   ├── models.py          # Comment 모델
│   ├── views.py
│   ├── serializers.py
│   └── urls.py
├── users/                 # 사용자 관리 앱
│   ├── models.py          # CustomUser 모델
│   ├── views.py
│   ├── serializers.py
│   └── urls.py
├── core/                  # 공통 유틸리티
│   ├── exceptions.py      # 커스텀 예외
│   ├── permissions.py     # 커스텀 권한
│   └── utils.py           # 헬퍼 함수
├── media/                 # 업로드 파일
├── static/                # 정적 파일
├── templates/             # 템플릿
├── manage.py
├── requirements.txt
├── pytest.ini
├── .env.example
├── CLAUDE.md
└── .claude/
    └── settings.json
```

## 코딩 규칙

- 모든 Python 코드는 PEP 8 스타일 가이드를 따른다
- Black 포매터 사용 (최대 줄 길이 88자)
- import 순서: 표준 라이브러리 > 서드파티 > 로컬 (isort 적용)
- 함수와 클래스에는 Google 스타일 docstring을 작성한다
- 모든 뷰는 클래스 기반 뷰(CBV)를 우선 사용한다
- 템플릿 파일은 `<앱명>/templates/<앱명>/` 구조를 따른다

## Django 규칙

- 새 앱 생성: `python manage.py startapp <앱명>`
- URL 패턴은 앱별 `urls.py` 분리 후 `include()`로 연결
- settings는 환경별 분리: `config/settings/base.py`, `dev.py`, `prod.py`
- 외래 키는 `on_delete` 옵션을 명시적으로 지정
- `created_at`, `updated_at` 필드는 모든 모델에 포함
- 인덱스 필요 필드는 `db_index=True` 설정

## API 설계

- ViewSet을 사용하여 CRUD API 구현
- Serializer는 `serializers.py`에 정의
- 인증: JWT (djangorestframework-simplejwt)
- 페이지네이션: 기본 20개 항목
- API 응답 형식:
  ```json
  {
    "status": "success",
    "data": {},
    "message": ""
  }
  ```

## 에러 처리

- try-except에서 구체적인 예외 클래스 사용 (bare except 금지)
- 커스텀 예외는 `core/exceptions.py`에 정의
- 로깅은 Python logging 모듈 사용 (print 금지)

## 테스트

- pytest와 pytest-django 사용
- 모든 모델, 뷰, API에 대한 테스트 작성
- 테스트 커버리지 최소 80% 유지

## 주요 명령어

- 개발 서버: `python manage.py runserver`
- 테스트: `pytest`
- 테스트 커버리지: `pytest --cov`
- 마이그레이션 생성: `python manage.py makemigrations`
- 마이그레이션 적용: `python manage.py migrate`
- 린트: `ruff check .`
- 포맷팅: `black .`
- 슈퍼유저 생성: `python manage.py createsuperuser`

## 주요 모델

### CustomUser (users/models.py)
- AbstractUser 상속
- 추가 필드: bio, profile_image, website

### Post (posts/models.py)
- 필드: title, content, author(FK), category(FK), tags(M2M), thumbnail, status, created_at, updated_at
- 관계: User(1)-Post(N), Category(1)-Post(N), Tag(N)-Post(M)

### Comment (comments/models.py)
- 필드: post(FK), author(FK), content, parent(자기참조 FK), created_at, updated_at
- 대댓글 지원 (계층 구조)

## API 엔드포인트

### 인증
- `POST /api/auth/register/` - 회원가입
- `POST /api/auth/login/` - 로그인 (JWT 발급)
- `POST /api/auth/refresh/` - 토큰 갱신

### 게시글
- `GET /api/posts/` - 목록 (필터: category, tag, search)
- `GET /api/posts/{id}/` - 상세
- `POST /api/posts/` - 작성 (인증 필요)
- `PUT /api/posts/{id}/` - 수정 (작성자만)
- `DELETE /api/posts/{id}/` - 삭제 (작성자만)

### 댓글
- `GET /api/posts/{post_id}/comments/` - 댓글 목록
- `POST /api/posts/{post_id}/comments/` - 댓글 작성
- `PUT /api/comments/{id}/` - 수정 (작성자만)
- `DELETE /api/comments/{id}/` - 삭제 (작성자만)

## 환경 변수

`.env.example` 참조. 주요 변수:
- `SECRET_KEY` - Django 비밀키
- `DEBUG` - 디버그 모드 (True/False)
- `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT` - DB 연결
- `JWT_SECRET_KEY` - JWT 서명키

## Git 규칙

- 브랜치 전략: Git Flow (main, develop, feature/*, hotfix/*)
- 커밋 메시지: `[타입] 제목` (예: `[feat] 게시글 작성 API 추가`)
- 타입: feat, fix, docs, style, refactor, test, chore
```

### GEMINI.md와의 구조 비교

동일한 Django 블로그 프로젝트를 두 파일로 작성할 때의 차이:

| 항목 | CLAUDE.md | GEMINI.md |
|------|-----------|-----------|
| **섹션 구분** | 자유 형식 (## 헤더로 자유 구분) | `## Rule` + `## Context` 필수 |
| **규칙 작성** | 코딩 규칙 섹션에 자연어로 작성 | Rule 섹션에 명령형 문장으로 작성 |
| **프로젝트 정보** | 기술 스택/디렉토리 섹션에 작성 | Context 섹션에 작성 |
| **명령어** | `## 주요 명령어` 섹션에 명시 | Context 또는 Rule에 포함 |
| **권한 설정** | settings.json으로 분리 | GEMINI.md 안에서 처리 |
| **파일 위치** | 프로젝트 루트 (`./CLAUDE.md`) | 프로젝트 루트 (`./GEMINI.md`) |

```
# GEMINI.md 구조 (비교용)        # CLAUDE.md 구조
─────────────────────            ─────────────────────
## Rule                          # 프로젝트명
### 코드 스타일                   ## 프로젝트 개요
  - PEP 8 따른다                  ## 기술 스택
  - docstring 작성한다             ## 코딩 규칙
### Django 규칙                    - PEP 8 따른다
  - CBV 우선 사용                  - docstring 작성한다
---                               ## Django 규칙
## Context                         - CBV 우선 사용
### 프로젝트 개요                  ## 주요 명령어
### 기술 스택                      ## API 엔드포인트
### 디렉토리 구조                  ## 환경 변수
```

### 체크포인트

- [ ] Django 블로그 프로젝트의 완성된 CLAUDE.md 예시를 확인했다
- [ ] GEMINI.md와의 구조적 차이를 이해했다
- [ ] 자신의 프로젝트에 적용할 수 있는 패턴을 파악했다

---

## 마무리

### 학습한 내용 정리

CLAUDE.md는 Claude Code의 프로젝트 지침 파일로, 다음과 같은 특징을 가진다:

1. **자유 형식 마크다운**: 정해진 섹션 규격 없이 자연어로 작성
2. **3단계 스코프**: 글로벌 > 프로젝트 > 컴포넌트 계층 지원
3. **권한 분리**: 지침(CLAUDE.md)과 권한(settings.json)을 분리 관리
4. **메모리 시스템**: `/memory` 명령 + 자동 메모리로 세션 간 학습 지원
5. **자동 로드**: 대화 시작 시 자동으로 읽어들임

### CLAUDE.md vs GEMINI.md vs AGENTS.md 비교표

| 항목 | CLAUDE.md | GEMINI.md | AGENTS.md |
|------|-----------|-----------|-----------|
| **도구** | Claude Code | Gemini CLI | OpenAI Codex 등 (벤더 중립) |
| **파일 형식** | 자유 형식 마크다운 | Rule/Context 섹션 구분 | 벤더 중립 마크다운 |
| **스코프 계층** | 글로벌/프로젝트/컴포넌트 | 프로젝트 레벨 | 프로젝트 레벨 |
| **권한 설정** | settings.json으로 분리 | 파일 내 혼합 | 파일 내 기술 |
| **메모리 시스템** | /memory + 자동 메모리 지원 | 미지원 | 미지원 |
| **로컬 오버라이드** | CLAUDE.local.md | 미지원 | 미지원 |
| **@import** | @경로 구문 지원 | 미지원 | 미지원 |
| **초기 생성** | `/init` 명령 | `gemini init` | 수동 작성 |
| **표준화** | Anthropic 전용 | Google 전용 | 벤더 중립 표준 (Sourcegraph 주도) |
| **호환성** | AGENTS.md 심링크 가능 | AGENTS.md 심링크 가능 | 여러 도구 지원 |

> **💡 개념 설명: AGENTS.md와의 관계**
>
> AGENTS.md는 Sourcegraph가 주도하고 OpenAI, Google이 지지하는 **벤더 중립 표준**이다.
> 기존 CLAUDE.md나 GEMINI.md를 AGENTS.md로 이름을 바꾸고 심볼릭 링크를 만들면 하위 호환성을 유지할 수 있다.
>
> ```bash
> # AGENTS.md 표준으로 전환 (하위 호환 유지)
> mv CLAUDE.md AGENTS.md
> ln -s AGENTS.md CLAUDE.md
> ln -s AGENTS.md GEMINI.md
> ```
>
> **한 줄 요약**: AGENTS.md는 벤더 중립 표준이며, 심링크로 기존 파일과 공존할 수 있다.

### 자주 하는 실수

1. **CLAUDE.md를 너무 길게 작성**
   - 200줄을 넘으면 Claude가 지시를 무시할 수 있다
   - 상세 내용은 @import로 외부 파일에 분리한다

2. **settings.json과 역할 혼동**
   - CLAUDE.md에 권한 설정을 쓰면 안 된다 (효과 없음)
   - 도구 허용/차단은 반드시 settings.json에 작성한다

3. **버전 정보 누락**
   - "Django 사용" (X) --> "Django 4.2.7 사용" (O)
   - 버전을 명시해야 해당 버전에 맞는 코드가 생성된다

4. **.claudeignore만으로 보안 의존**
   - .claudeignore는 컨텍스트 제외일 뿐, 완벽한 차단이 아니다
   - 민감 파일은 settings.json deny 규칙으로 차단해야 한다

5. **CLAUDE.local.md를 Git에 커밋**
   - 개인 설정 파일이므로 `.gitignore`에 반드시 추가한다
   - `echo "CLAUDE.local.md" >> .gitignore`

6. **추상적인 규칙 작성**
   - "코드를 깔끔하게 작성한다" (X)
   - "함수는 최대 50줄, 클래스는 최대 300줄을 넘지 않는다" (O)

### 빠른 참조 (Quick Reference)

```bash
# CLAUDE.md 초기 생성
/init

# 메모리 상태 확인
/memory

# 파일 위치 요약
~/.claude/CLAUDE.md              # 글로벌 (모든 프로젝트)
./CLAUDE.md                      # 프로젝트 (Git 커밋, 팀 공유)
./CLAUDE.local.md                # 로컬 오버라이드 (.gitignore)
./{subdir}/CLAUDE.md             # 컴포넌트 (서브디렉토리)

~/.claude/settings.json          # 글로벌 권한/MCP
.claude/settings.json            # 프로젝트 권한/MCP (Git 커밋)
.claude/settings.local.json      # 로컬 권한 오버라이드 (.gitignore)

.claudeignore                    # 컨텍스트 제외 패턴

~/.claude/projects/<project>/memory/  # 자동 메모리 저장소
```

```bash
# .gitignore에 추가할 항목
CLAUDE.local.md
.claude/settings.local.json
```

### 다음 단계

1. **자신의 프로젝트에 CLAUDE.md 작성**
   - `/init`으로 초안 생성
   - 팀 컨벤션과 프로젝트 특성 반영
   - 200줄 이내로 유지

2. **settings.json 권한 설정**
   - 프로젝트에 맞는 allow/deny 규칙 작성
   - 민감 파일 접근 차단 확인

3. **팀과 공유**
   - CLAUDE.md와 `.claude/settings.json`을 Git에 커밋
   - CLAUDE.local.md는 `.gitignore`에 추가

4. **심화 학습**
   - MCP 서버 설정으로 외부 도구 연동
   - 커스텀 슬래시 명령어 작성
   - AGENTS.md 표준으로의 전환 검토
