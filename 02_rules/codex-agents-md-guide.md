# AGENTS.md 작성 가이드 (Codex CLI)

## 학습 목표

OpenAI Codex CLI에서 사용하는 AGENTS.md 파일의 구조와 작성 방법을 이해하고, config.toml 설정과의 역할 분리를 파악하여, 실무 프로젝트에 체계적인 에이전트 지침을 구성할 수 있다.

## 사전 준비

- Codex CLI 설치 및 기본 사용법 숙지 (`npm install -g @openai/codex`)
- OpenAI API 키 설정 (`OPENAI_API_KEY` 환경 변수)
- Django 프로젝트 구조에 대한 기본 이해
- 마크다운 문법 기초 지식

---

## 전체 흐름 한눈에 보기

AGENTS.md는 Codex CLI가 작업을 시작하기 전에 읽는 "에이전트 지침서"이다. 자유 형식의 마크다운으로 작성하며, 프로젝트의 규칙과 맥락을 AI에게 전달한다. 설정(모델, 샌드박스 등)은 별도의 config.toml에서 관리한다.

**주요 단계:**
1. AGENTS.md의 역할과 다른 도구와의 차이점 이해
2. 스코프 계층(글로벌 → 프로젝트 루트 → 서브디렉토리) 파악
3. AGENTS.md 작성 모범 사례 학습
4. config.toml 설정 방법 숙지
5. Django 프로젝트 실전 적용

---

## Phase 1: AGENTS.md 이해하기

### 목표
AGENTS.md가 무엇인지, Codex CLI가 왜 이 파일명을 사용하는지, 다른 도구의 규칙 파일과 어떻게 다른지 명확히 이해한다.

### 단계별 구현

#### Step 1.1 — AGENTS.md란 무엇인가?

AGENTS.md는 Codex CLI가 모든 작업을 시작하기 전에 자동으로 읽는 마크다운 파일이다. 프로젝트의 코딩 규칙, 아키텍처 설명, 테스트 방법, 배포 절차 등을 자유롭게 기술한다.

> **💡 개념 설명: AGENTS.md는 "에이전트의 README"이다**
>
> README.md가 사람을 위한 프로젝트 설명서라면, AGENTS.md는 AI 에이전트를 위한 설명서이다.
> Codex CLI는 세션을 시작할 때 이 파일을 읽고, 모든 작업에 해당 지침을 반영한다.
>
> 핵심 특징:
> - **자유 형식**: 특별한 섹션 구분(Rule/Context) 없이 일반 마크다운으로 작성
> - **자동 로딩**: 별도 명령 없이 Codex가 자동으로 탐색하여 읽음
> - **캐스케이딩**: 디렉토리 계층을 따라 여러 파일을 연결하여 적용
>
> **한 줄 요약**: AGENTS.md는 세 도구 중 가장 단순한 규칙 시스템이다.

#### Step 1.2 — 왜 CODEX.md가 아니라 AGENTS.md인가?

Codex CLI는 도구 이름을 딴 `CODEX.md`가 아닌 `AGENTS.md`라는 범용적 이름을 사용한다. 이는 OpenAI의 설계 철학을 반영한다.

- **도구 독립적 이름**: 특정 제품에 종속되지 않는 범용 파일명
- **에이전트 중심 사고**: "이 코드베이스에서 일하는 모든 에이전트"를 위한 지침
- **생태계 호환성**: 다른 AI 도구들도 AGENTS.md를 인식할 수 있는 열린 구조

> **💡 개념 설명: 파일명 비교**
>
> | 도구 | 규칙 파일명 | 명명 방식 |
> |------|-----------|----------|
> | Codex CLI | `AGENTS.md` | 범용 (에이전트 중심) |
> | Claude Code | `CLAUDE.md` | 도구 이름 기반 |
> | Gemini CLI | `GEMINI.md` | 도구 이름 기반 |
>
> Codex만 유일하게 도구 이름이 아닌 역할 기반 파일명을 사용한다.
>
> **한 줄 요약**: AGENTS.md는 "어떤 에이전트든" 읽을 수 있는 범용 지침 파일이다.

#### Step 1.3 — AGENTS.md vs CLAUDE.md vs GEMINI.md

세 도구의 규칙 파일은 목적은 같지만 구조와 기능이 다르다.

| 비교 항목 | AGENTS.md (Codex) | CLAUDE.md (Claude Code) | GEMINI.md (Gemini CLI) |
|----------|-------------------|------------------------|----------------------|
| **형식** | 자유 마크다운 | 자유 마크다운 | Rule/Context 섹션 구분 |
| **글로벌 파일** | `~/.codex/AGENTS.md` | `~/.claude/CLAUDE.md` | `~/.gemini/GEMINI.md` |
| **프로젝트 파일** | 프로젝트 루트 `AGENTS.md` | 프로젝트 루트 `CLAUDE.md` | 프로젝트 루트 `GEMINI.md` |
| **서브디렉토리** | 중첩 `AGENTS.md` 지원 | 중첩 `CLAUDE.md` 지원 | 미지원 |
| **오버라이드** | `AGENTS.override.md` 지원 | 미지원 | 미지원 |
| **설정 파일** | `config.toml` (별도) | `settings.json` (별도) | `settings.json` (별도) |
| **폴백 파일명** | 설정 가능 | 미지원 | 미지원 |
| **복잡도** | 가장 단순 | 중간 | 가장 구조적 |

> **💡 개념 설명: Codex의 규칙 시스템이 가장 단순한 이유**
>
> Codex CLI는 "규칙은 그냥 마크다운으로 쓰면 된다"는 미니멀 철학을 따른다.
> - GEMINI.md처럼 Rule/Context 섹션을 구분하지 않는다
> - 특별한 지시어나 메타데이터 형식이 없다
> - 그냥 마크다운 파일에 원하는 내용을 적으면 된다
>
> 대신 **캐스케이딩**(디렉토리 계층별 병합)과 **오버라이드** 기능으로 유연성을 확보한다.
>
> **한 줄 요약**: 형식은 가장 단순하되, 계층 구조로 유연성을 보완한다.

### 체크포인트

- [ ] AGENTS.md가 Codex CLI의 에이전트 지침 파일임을 설명할 수 있다
- [ ] CODEX.md가 아닌 AGENTS.md를 사용하는 이유를 이해했다
- [ ] AGENTS.md, CLAUDE.md, GEMINI.md의 핵심 차이점을 3가지 이상 말할 수 있다

---

## Phase 2: 스코프 계층

### 목표
AGENTS.md의 탐색 경로와 적용 우선순위를 이해하고, config.toml과의 역할 분리를 명확히 파악한다.

### 단계별 구현

#### Step 2.1 — 탐색 순서 이해하기

Codex CLI는 세션을 시작할 때 다음 순서로 지침 파일을 탐색한다.

```
1. 글로벌 스코프: ~/.codex/
   └── AGENTS.override.md (있으면 사용) → 없으면 AGENTS.md

2. 프로젝트 스코프: Git 루트부터 현재 디렉토리까지 각 디렉토리에서
   └── AGENTS.override.md → AGENTS.md → 폴백 파일명 순서로 탐색
```

탐색된 모든 파일은 **루트에서 현재 디렉토리 순서로 연결(concatenate)** 된다. 더 깊은 디렉토리의 내용이 나중에 오므로, 사실상 **가까운 파일이 우선**한다.

```
# 실제 탐색 예시
# 현재 디렉토리: ~/project/services/payments/

탐색 순서:
1. ~/.codex/AGENTS.md              ← 글로벌
2. ~/project/AGENTS.md             ← 프로젝트 루트
3. ~/project/services/AGENTS.md    ← 중간 디렉토리
4. ~/project/services/payments/AGENTS.md  ← 현재 디렉토리 (최우선)
```

> **💡 개념 설명: 캐스케이딩(Cascading) 규칙**
>
> Codex는 발견한 모든 AGENTS.md를 빈 줄로 연결하여 하나의 프롬프트로 만든다.
> 나중에 나오는 내용(더 깊은 디렉토리)이 앞의 내용을 덮어쓰므로:
> - **글로벌**: 모든 프로젝트에 적용할 공통 규칙
> - **프로젝트 루트**: 해당 프로젝트 전체 규칙
> - **서브디렉토리**: 특정 모듈/팀만의 세부 규칙
>
> `project_doc_max_bytes` 설정(기본 32KiB)을 초과하면 파일 추가를 중단한다.
>
> **한 줄 요약**: 가까운 AGENTS.md가 먼 AGENTS.md를 덮어쓴다.

#### Step 2.2 — 프로젝트 루트 AGENTS.md

프로젝트 루트(보통 Git 루트)에 위치하는 핵심 지침 파일이다.

```bash
# 프로젝트 구조 예시

my-django-project/
├── AGENTS.md              ← 프로젝트 전체 규칙
├── manage.py
├── requirements.txt
├── apps/
│   ├── users/
│   │   └── AGENTS.md      ← users 앱 전용 규칙
│   └── payments/
│       └── AGENTS.md      ← payments 앱 전용 규칙
└── config/
    └── settings.py
```

#### Step 2.3 — 서브디렉토리 중첩 AGENTS.md

특정 앱이나 모듈에만 적용되는 규칙을 별도로 관리할 수 있다.

```markdown
# apps/payments/AGENTS.md

## 결제 모듈 규칙

- 모든 금액 계산은 Decimal 타입을 사용한다 (float 금지)
- 결제 관련 API는 반드시 트랜잭션으로 감싼다
- PCI DSS 준수를 위해 카드 정보를 직접 저장하지 않는다
- 결제 실패 시 반드시 롤백 로직을 포함한다
```

#### Step 2.4 — AGENTS.override.md

기본 AGENTS.md를 삭제하지 않고 임시로 덮어쓸 수 있는 오버라이드 파일이다.

```bash
# 디렉토리 내 탐색 우선순위
1. AGENTS.override.md    ← 있으면 이것만 사용
2. AGENTS.md             ← override가 없을 때 사용
3. 폴백 파일명           ← 둘 다 없을 때 사용

# 각 디렉토리에서 최대 1개 파일만 선택된다
```

> **💡 개념 설명: AGENTS.override.md 활용 시나리오**
>
> - **임시 실험**: 새로운 규칙을 테스트할 때 원본을 보존하면서 오버라이드
> - **개인 설정**: `.gitignore`에 추가하여 개인별 글로벌 규칙 적용
> - **긴급 수정**: 공유 AGENTS.md를 수정하지 않고 빠르게 지침 변경
>
> 오버라이드 파일을 삭제하면 기본 AGENTS.md가 자동으로 복원된다.
>
> **한 줄 요약**: override 파일은 원본을 건드리지 않는 임시 덮어쓰기이다.

#### Step 2.5 — config.toml은 규칙이 아니라 설정이다

Codex CLI에서 **규칙(지침)**과 **설정(환경)**은 완전히 분리되어 있다.

| 구분 | 파일 | 역할 | 예시 |
|------|------|------|------|
| **규칙** | `AGENTS.md` | AI가 따를 지침 | "PEP 8을 준수하라" |
| **설정** | `config.toml` | 도구 동작 환경 | 사용할 모델, 승인 정책 |

```bash
# config.toml 위치

~/.codex/
├── config.toml            ← 사용자 전역 설정
├── AGENTS.md              ← 사용자 전역 규칙 (별개!)
└── AGENTS.override.md     ← 전역 오버라이드 (선택)

my-project/
├── .codex/
│   └── config.toml        ← 프로젝트 설정
├── AGENTS.md              ← 프로젝트 규칙 (별개!)
└── ...
```

> **💡 개념 설명: 글로벌 지침 파일이 없는 것은 아니다**
>
> Codex에는 "글로벌 규칙 전용 설정"이 config.toml에 있는 것이 아니다.
> 글로벌 규칙은 `~/.codex/AGENTS.md`에 작성한다.
> config.toml의 `model_instructions_file` 필드로 별도 지침 파일을 지정할 수도 있다.
>
> **정리하면:**
> - `~/.codex/AGENTS.md` → 글로벌 에이전트 지침
> - `~/.codex/config.toml` → 글로벌 도구 설정
> - 프로젝트 `AGENTS.md` → 프로젝트 에이전트 지침
> - 프로젝트 `.codex/config.toml` → 프로젝트 도구 설정
>
> **한 줄 요약**: AGENTS.md는 "무엇을 하라", config.toml은 "어떻게 동작하라"이다.

#### Step 2.6 — config.toml의 instructions 관련 필드

config.toml에서 지침과 관련된 설정 필드도 존재한다.

```toml
# ~/.codex/config.toml

# 별도 지침 파일 경로 지정 (AGENTS.md 외 추가 지침)
model_instructions_file = "/path/to/custom-instructions.txt"

# AGENTS.md 외 다른 이름의 프로젝트 문서를 인식시키기
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]

# 프로젝트 문서 최대 크기 (기본 32KiB)
project_doc_max_bytes = 65536
```

### 체크포인트

- [ ] AGENTS.md의 4단계 탐색 순서(글로벌 → 루트 → 중간 → 현재)를 설명할 수 있다
- [ ] AGENTS.override.md의 역할과 활용 시나리오를 알고 있다
- [ ] config.toml과 AGENTS.md의 역할 차이를 명확히 구분할 수 있다
- [ ] `project_doc_fallback_filenames`로 폴백 파일명을 설정할 수 있다

---

## Phase 3: AGENTS.md 작성법

### 목표
효과적인 AGENTS.md를 작성하기 위한 모범 사례를 익히고, 실제 프로젝트에 적용할 수 있는 패턴을 학습한다.

### 단계별 구현

#### Step 3.1 — 자유 형식 마크다운

AGENTS.md는 특별한 포맷 규칙이 없다. 일반 마크다운으로 원하는 내용을 자유롭게 작성하면 된다.

```markdown
# AGENTS.md

이 프로젝트는 Django 기반 REST API 서버이다.

## 코딩 규칙
- PEP 8을 따른다
- 타입 힌트를 반드시 사용한다

## 테스트
테스트는 pytest로 실행한다: `pytest --cov`
```

> **💡 개념 설명: GEMINI.md와의 차이점**
>
> GEMINI.md는 `## Rule`과 `## Context` 섹션을 명시적으로 구분해야 한다.
> AGENTS.md는 그런 구분이 없다. 제목, 목록, 코드 블록 등 마크다운 문법만 지키면 된다.
>
> 이것이 Codex의 규칙 시스템이 "가장 단순하다"고 하는 이유이다.
>
> **한 줄 요약**: 형식 제약 없이, 마크다운으로 자유롭게 작성한다.

#### Step 3.2 — 권장 구성 요소

형식은 자유이지만, 다음 내용을 포함하면 AI가 프로젝트를 더 잘 이해한다.

**1. 코딩 컨벤션**
```markdown
## 코딩 컨벤션

- Python 3.12 이상을 대상으로 코드를 작성한다
- 모든 함수에 타입 힌트를 추가한다
- docstring은 Google 스타일을 따른다
- import 순서: 표준 라이브러리 → 서드파티 → 로컬
- 포매터: Black (line-length=88)
- 린터: Ruff
```

**2. 프로젝트 구조**
```markdown
## 프로젝트 구조

이 프로젝트는 Django 4.2 + DRF 기반의 블로그 API이다.

```
blog_project/
├── config/          # Django 설정
├── posts/           # 게시글 앱
├── comments/        # 댓글 앱
├── users/           # 사용자 앱
└── core/            # 공통 유틸리티
```
```

**3. 테스트 규칙**
```markdown
## 테스트

- pytest와 pytest-django를 사용한다
- 테스트 커버리지 최소 80%를 유지한다
- 테스트 실행: `pytest`
- 특정 앱 테스트: `pytest apps/posts/`
- 새로운 기능 추가 시 반드시 테스트를 함께 작성한다
```

**4. 배포 및 운영**
```markdown
## 배포

- Docker Compose로 로컬 환경을 구성한다
- main 브랜치에 병합 시 자동 배포된다
- 마이그레이션은 배포 전에 반드시 검증한다
```

**5. 보안 가이드라인**
```markdown
## 보안

- 시크릿 값은 절대 코드에 하드코딩하지 않는다
- .env 파일은 커밋하지 않는다
- SQL 쿼리는 ORM을 통해 작성한다 (Raw SQL 지양)
- 사용자 입력은 반드시 검증한다
```

#### Step 3.3 — /init 명령으로 초안 생성

Codex CLI의 `/init` 슬래시 명령으로 AGENTS.md 초안을 자동 생성할 수 있다.

```bash
# Codex CLI 실행 후
codex

# TUI 내에서 /init 명령 실행
> /init

# Codex가 프로젝트를 분석하여 AGENTS.md 초안 생성
# 생성된 파일을 검토하고 수정한다
```

> **💡 개념 설명: /init으로 생성 vs 수동 작성**
>
> `/init`은 프로젝트의 파일 구조를 분석하여 기본 골격을 만들어준다.
> 하지만 팀 컨벤션, 보안 규칙, 배포 절차 등은 수동으로 보강해야 한다.
>
> 추천 워크플로우:
> 1. `/init`으로 초안 생성
> 2. 프로젝트 특성에 맞게 수정
> 3. 팀 리뷰 후 Git에 커밋
>
> **한 줄 요약**: `/init`은 출발점이지, 완성본이 아니다.

### 체크포인트

- [ ] AGENTS.md가 자유 형식 마크다운임을 이해했다
- [ ] 권장 구성 요소 5가지(코딩 컨벤션, 구조, 테스트, 배포, 보안)를 파악했다
- [ ] `/init` 명령으로 초안을 생성하는 방법을 알고 있다

---

## Phase 4: config.toml 설정

### 목표
config.toml의 구조와 주요 설정 항목을 이해하고, 사용자 수준과 프로젝트 수준의 설정을 구성할 수 있다.

### 단계별 구현

#### Step 4.1 — config.toml 기본 구조

```bash
# 사용자 전역 설정
~/.codex/config.toml

# 프로젝트 수준 설정
my-project/.codex/config.toml
```

#### Step 4.2 — 모델 설정

```toml
# ~/.codex/config.toml

# 기본 모델 지정
model = "o4-mini"

# 모델 제공자 (기본: "openai")
model_provider = "openai"

# 추론 노력도 (low, medium, high)
# 지원 모델에서만 동작 (Responses API 기반)
model_reasoning_effort = "medium"
```

> **💡 개념 설명: 모델 선택 가이드**
>
> | 모델 | 특징 | 추천 용도 |
> |------|------|----------|
> | `o4-mini` | 빠르고 경제적 | 일상적인 코딩 작업 |
> | `o3` | 균형 잡힌 성능 | 중간 복잡도 작업 |
> | `codex-mini` | Codex 전용 최적화 | 코드 생성 특화 |
>
> `model_reasoning_effort`를 높이면 더 깊은 추론을 하지만 속도가 느려진다.
>
> **한 줄 요약**: 작업 복잡도에 따라 모델과 추론 노력도를 조절한다.

#### Step 4.3 — 승인 정책 (approval_policy)

Codex가 명령을 실행하기 전에 사용자 승인을 요구할지 결정한다.

```toml
# 승인 정책 설정
approval_policy = "on-request"
```

| 값 | 동작 | 안전도 |
|---|------|-------|
| `"untrusted"` | 알려진 읽기 전용 명령만 자동 실행, 나머지는 승인 요청 | 높음 |
| `"on-request"` | 모델이 필요할 때 승인 요청 (기본값) | 중간 |
| `"on-failure"` | 실패 시에만 승인 요청 | 낮음 |
| `"never"` | 모든 명령을 자동 실행 (위험) | 매우 낮음 |

> **💡 개념 설명: approval_policy와 보안**
>
> `"never"`는 모든 명령을 승인 없이 실행하므로 매우 위험하다.
> 프로덕션 환경이나 민감한 데이터가 있는 프로젝트에서는 `"untrusted"` 사용을 권장한다.
>
> 처음 사용할 때는 기본값인 `"on-request"`로 시작하고,
> 도구에 익숙해진 후 프로젝트 특성에 맞게 조절한다.
>
> **한 줄 요약**: 안전을 우선하려면 "untrusted", 속도를 원하면 "on-request"를 사용한다.

#### Step 4.4 — 샌드박스 설정

Codex가 파일 시스템과 네트워크에 접근하는 범위를 제한한다.

```toml
# 샌드박스 모드
sandbox_mode = "workspace-write"

# 세부 설정 (workspace-write 모드일 때)
[sandbox_workspace_write]
writable_roots = ["/home/user/.pyenv/shims"]
network_access = false
exclude_slash_tmp = false
```

| sandbox_mode | 설명 |
|-------------|------|
| `"read-only"` | 파일 읽기만 허용 |
| `"workspace-write"` | 프로젝트 디렉토리 내 쓰기 허용 |
| `"full-auto"` | 모든 접근 허용 (위험) |

#### Step 4.5 — 웹 검색 설정

```toml
# 웹 검색 설정
web_search = "cached"
```

| 값 | 동작 |
|---|------|
| `"cached"` | 웹 검색 캐시 사용 (기본값) |
| `"live"` | 실시간 웹 검색 |
| `"disabled"` | 웹 검색 비활성화 |

#### Step 4.6 — 프로필(Profiles)

작업 유형에 따라 다른 설정 조합을 프로필로 저장할 수 있다.

```toml
# ~/.codex/config.toml

# 기본 설정
model = "o4-mini"
approval_policy = "on-request"

# 심층 코드 리뷰 프로필
[profiles.deep-review]
model = "o3"
model_reasoning_effort = "high"
approval_policy = "never"

# 경량 작업 프로필
[profiles.lightweight]
model = "gpt-4.1-mini"
approval_policy = "untrusted"

# 전체 자동화 프로필
[profiles.full-auto]
model = "o4-mini"
approval_policy = "never"
sandbox_mode = "workspace-write"
```

```bash
# 프로필 사용
codex --profile deep-review
codex --profile lightweight
```

#### Step 4.7 — 프로젝트 수준 config.toml

프로젝트별 설정은 `.codex/config.toml`에 작성한다.

```bash
my-project/
├── .codex/
│   └── config.toml        ← 프로젝트 설정
├── AGENTS.md              ← 프로젝트 규칙
└── ...
```

```toml
# my-project/.codex/config.toml

model = "o4-mini"
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

#### Step 4.8 — 설정 우선순위

설정 값은 다음 우선순위로 결정된다 (위가 최우선).

```
1. CLI 플래그          codex --model o3
2. 프로젝트 config     .codex/config.toml (가장 가까운 디렉토리 우선)
3. 사용자 config       ~/.codex/config.toml
4. 기본값              Codex 내장 기본값
```

> **💡 개념 설명: 우선순위의 실용적 의미**
>
> - **팀 표준**: 프로젝트 `.codex/config.toml`을 Git에 커밋하여 팀 전체가 동일 설정 사용
> - **개인 커스텀**: `~/.codex/config.toml`에 개인 선호 설정
> - **일회성 변경**: CLI 플래그로 특정 세션만 다른 설정 적용
>
> **한 줄 요약**: CLI 플래그 > 프로젝트 설정 > 사용자 설정 > 기본값 순이다.

### 체크포인트

- [ ] config.toml의 주요 설정 항목(model, approval_policy, sandbox_mode)을 알고 있다
- [ ] 프로필을 정의하고 사용하는 방법을 이해했다
- [ ] 사용자 설정과 프로젝트 설정의 우선순위를 설명할 수 있다
- [ ] web_search 설정의 세 가지 옵션을 알고 있다

---

## Phase 5: 실전 예시 - Django 블로그 프로젝트

### 목표
실제 Django 블로그 프로젝트에 적용할 수 있는 완전한 AGENTS.md와 config.toml 예시를 작성한다.

### 완성된 AGENTS.md 예시

```markdown
# AGENTS.md

## 프로젝트 개요

Django 기반의 개인 블로그 플랫폼이다. 사용자는 회원가입 후 게시글을 작성하고,
다른 사용자의 게시글에 댓글을 달 수 있다. 태그 기반 검색과 카테고리 분류를 지원한다.

## 기술 스택

- **백엔드**: Django 4.2.7, Django REST Framework 3.14.0
- **데이터베이스**: PostgreSQL 15.3
- **인증**: djangorestframework-simplejwt 5.3.0
- **테스트**: pytest 7.4, pytest-django 4.5
- **배포**: Docker 24.0, Gunicorn 21.2, Nginx 1.25

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
│   ├── views.py           # ViewSet 기반 API
│   ├── serializers.py     # DRF 직렬화
│   ├── urls.py
│   └── tests/
├── comments/              # 댓글 관리 앱
├── users/                 # 사용자 관리 앱
├── core/                  # 공통 유틸리티
│   ├── exceptions.py
│   ├── permissions.py
│   └── utils.py
├── .codex/
│   └── config.toml        # Codex 프로젝트 설정
├── manage.py
├── requirements.txt
├── pytest.ini
└── AGENTS.md              # 이 파일
```

## 코딩 규칙

- 모든 Python 코드는 PEP 8을 따르고, Black(line-length=88)으로 포매팅한다
- Ruff를 린터로 사용한다
- 모든 함수와 클래스에 Google 스타일 docstring을 작성한다
- 타입 힌트를 반드시 사용한다
- import 순서: 표준 라이브러리 → 서드파티 → 로컬 (isort로 관리)

## 네이밍 규칙

- 변수/함수: snake_case (예: `user_name`, `get_post_list`)
- 클래스: PascalCase (예: `PostViewSet`, `UserSerializer`)
- 상수: UPPER_SNAKE_CASE (예: `MAX_PAGE_SIZE`)
- URL 패턴 이름: kebab-case (예: `post-detail`)

## Django 규칙

- 모든 뷰는 클래스 기반 뷰(CBV) 또는 ViewSet을 사용한다
- URL 패턴은 앱별로 urls.py를 분리하고 include()로 연결한다
- settings.py는 환경별로 분리한다 (base.py, dev.py, prod.py)
- 모델 변경 후 반드시 makemigrations와 migrate를 실행한다
- 외래 키는 on_delete 옵션을 명시적으로 지정한다
- created_at, updated_at 필드는 모든 모델에 포함한다

## API 설계

- Django REST Framework의 ViewSet + Router를 사용한다
- 인증은 JWT 토큰 방식 (djangorestframework-simplejwt)을 사용한다
- 페이지네이션: 기본 20개 항목
- 모든 API 응답은 다음 형식을 따른다:
  ```json
  {
    "status": "success",
    "data": {},
    "message": ""
  }
  ```

## 에러 처리

- try-except에서 구체적인 예외 클래스를 사용한다 (bare except 금지)
- 커스텀 예외는 core/exceptions.py에 정의한다
- 로깅은 Python logging 모듈을 사용한다 (print 금지)

## 테스트

- pytest와 pytest-django를 사용한다
- 테스트 커버리지 최소 80%를 유지한다
- 테스트 실행: `pytest`
- 새로운 기능 추가 시 반드시 테스트를 함께 작성한다
- 테스트 파일명: test_*.py

## Git 규칙

- 브랜치 전략: Git Flow (main, develop, feature/*, hotfix/*)
- 커밋 메시지 형식: `[타입] 제목`
- 타입: feat, fix, docs, style, refactor, test, chore
- 예시: `[feat] 게시글 작성 API 추가`

## 보안

- 시크릿 값은 환경 변수(.env)로 관리하고, 코드에 하드코딩하지 않는다
- .env 파일은 절대 Git에 커밋하지 않는다
- SQL 쿼리는 ORM을 통해 작성한다 (Raw SQL 지양)
- 사용자 입력은 Serializer를 통해 반드시 검증한다

## 로컬 개발 환경

1. 가상환경 생성: `python -m venv venv`
2. 패키지 설치: `pip install -r requirements.txt`
3. 환경 변수 설정: `cp .env.example .env`
4. DB 마이그레이션: `python manage.py migrate`
5. 개발 서버 실행: `python manage.py runserver`
```

### 서브디렉토리 AGENTS.md 예시

```markdown
# posts/AGENTS.md

## 게시글 앱 규칙

- Post 모델의 status 필드는 "draft"와 "published" 두 값만 허용한다
- 게시글 목록 API는 published 상태만 반환한다 (작성자 본인 제외)
- 게시글 삭제는 소프트 삭제(is_deleted 플래그)를 사용한다
- 썸네일 이미지는 최대 2MB, JPEG/PNG만 허용한다
- 태그는 최대 10개까지 허용한다
```

### 완성된 config.toml 예시

```toml
# .codex/config.toml (프로젝트 수준)

# 모델 설정
model = "o4-mini"
model_provider = "openai"
model_reasoning_effort = "medium"

# 승인 정책
approval_policy = "on-request"

# 샌드박스
sandbox_mode = "workspace-write"

# 웹 검색
web_search = "cached"

# 프로젝트 문서 설정
project_doc_max_bytes = 65536
```

```toml
# ~/.codex/config.toml (사용자 전역)

# 기본 모델
model = "o4-mini"
approval_policy = "on-request"

# 폴백 파일명
project_doc_fallback_filenames = ["TEAM_GUIDE.md"]

# 프로필: 심층 분석
[profiles.deep]
model = "o3"
model_reasoning_effort = "high"
approval_policy = "never"

# 프로필: 빠른 작업
[profiles.quick]
model = "gpt-4.1-mini"
approval_policy = "on-request"
model_reasoning_effort = "low"
```

### 체크포인트

- [ ] Django 블로그 프로젝트에 적합한 AGENTS.md를 확인했다
- [ ] 서브디렉토리 AGENTS.md의 활용 방법을 이해했다
- [ ] 프로젝트 수준과 사용자 수준 config.toml을 구성할 수 있다
- [ ] 프로필을 활용하여 작업별 설정을 전환할 수 있다

---

## 마무리

### 3-도구 규칙 시스템 비교표

| 비교 항목 | AGENTS.md (Codex CLI) | CLAUDE.md (Claude Code) | GEMINI.md (Gemini CLI) |
|----------|----------------------|------------------------|----------------------|
| **파일 형식** | 자유 마크다운 | 자유 마크다운 | Rule/Context 섹션 구분 |
| **파일명 방식** | 범용 (에이전트 중심) | 도구 이름 기반 | 도구 이름 기반 |
| **글로벌 위치** | `~/.codex/AGENTS.md` | `~/.claude/CLAUDE.md` | `~/.gemini/GEMINI.md` |
| **프로젝트 위치** | `프로젝트루트/AGENTS.md` | `프로젝트루트/CLAUDE.md` | `프로젝트루트/GEMINI.md` |
| **서브디렉토리 중첩** | 지원 (캐스케이딩) | 지원 | 미지원 |
| **오버라이드 파일** | `AGENTS.override.md` | 미지원 | 미지원 |
| **폴백 파일명** | 설정 가능 (`project_doc_fallback_filenames`) | 미지원 | 미지원 |
| **설정 파일** | `config.toml` (TOML) | `settings.json` (JSON) | `settings.json` (JSON) |
| **프로필 기능** | 지원 (`[profiles.*]`) | 미지원 | 미지원 |
| **초안 자동 생성** | `/init` 명령 | 미지원 | `gemini init` 명령 |
| **최대 크기 제한** | `project_doc_max_bytes` (32KiB 기본) | 없음 | 없음 |
| **규칙 복잡도** | 가장 단순 | 중간 | 가장 구조적 |

### 핵심 차이점 정리

1. **Codex는 가장 단순한 규칙 시스템을 갖는다**: 자유 마크다운, 특별한 섹션 구분 없음
2. **설정과 규칙이 완전히 분리되어 있다**: AGENTS.md(지침) vs config.toml(설정)
3. **캐스케이딩으로 유연성 확보**: 디렉토리 깊이에 따라 규칙을 계층적으로 적용
4. **오버라이드 메커니즘**: AGENTS.override.md로 원본 보존하면서 임시 변경 가능
5. **프로필 시스템**: config.toml에서 작업 유형별 설정 조합을 저장/전환

### 자주 하는 실수

1. **AGENTS.md와 config.toml 혼동**
   - ❌ config.toml에 코딩 규칙 작성 → AGENTS.md에 작성
   - ❌ AGENTS.md에 모델 설정 작성 → config.toml에 작성

2. **오버라이드 파일 남용**
   - ❌ AGENTS.override.md를 Git에 커밋하여 팀 전체에 적용
   - ✅ 개인 글로벌 오버라이드만 사용, 프로젝트 규칙은 AGENTS.md에

3. **서브디렉토리 규칙 과다**
   - ❌ 모든 디렉토리에 AGENTS.md 생성 → 관리 부담 증가
   - ✅ 규칙이 크게 다른 모듈에만 선택적으로 생성

4. **project_doc_max_bytes 초과**
   - ❌ 매우 긴 AGENTS.md 작성 → 32KiB 초과 시 잘림
   - ✅ 핵심 내용만 간결하게, 필요시 max_bytes 설정 증가

5. **approval_policy "never" 무분별 사용**
   - ❌ 모든 프로젝트에서 "never" 사용
   - ✅ 신뢰할 수 있는 환경에서만, 샌드박스와 병행 사용

### 다음 단계

1. **자신의 프로젝트에 AGENTS.md 작성**
   - Codex CLI에서 `/init`으로 초안 생성
   - 팀 컨벤션과 프로젝트 특성 반영

2. **config.toml 최적화**
   - 작업 유형별 프로필 설정
   - 프로젝트별 승인 정책과 샌드박스 조정

3. **팀과 공유**
   - AGENTS.md와 `.codex/config.toml`을 Git에 커밋
   - AGENTS.override.md는 `.gitignore`에 추가

4. **정기적인 업데이트**
   - 새로운 기능이나 규칙 변경 시 AGENTS.md 갱신
   - `project_doc_max_bytes` 내에서 핵심 내용 유지
