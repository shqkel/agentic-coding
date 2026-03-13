# Antigravity Agent Skills 작성 가이드

## 학습 목표

Antigravity의 Agent Skills가 GEMINI.md Rule과 어떻게 다른지 이해하고, `.agent/skills/<skill-name>/SKILL.md` 구조를 처음부터 작성하여 점진적 공개(Progressive Disclosure) 방식으로 전문 지식을 필요할 때만 로드할 수 있다.

## 사전 준비

- Antigravity 기본 사용법 숙지 (`01_agentic_coding_tool/02_editor_native/antigravity-guide.md` 참고)
- 텍스트 에디터 (Gemini 플러그인이 설치된 편집기)
- 작업할 프로젝트 디렉토리

---

## 전체 흐름 한눈에 보기

GEMINI.md Rule은 세션 시작 시 항상 컨텍스트에 로드되어 모든 대화에 적용된다. 반면 Agent Skills는 사용자 요청이 SKILL.md의 `description`과 매칭될 때만 로드된다. 이를 점진적 공개(Progressive Disclosure) 방식이라 한다.

1. **개념 이해** — Rule vs Skill 차이, 점진적 공개 방식의 의미
2. **기본 Skill 작성** — SKILL.md frontmatter 구조와 파일 배치
3. **고급 기능** — 스크립트 번들링, 전역 vs 프로젝트 Skill, 우선순위
4. **실전 예시** — 실무에서 바로 쓸 수 있는 Skill 2-3개

---

## Phase 1: Rule vs Skill — 점진적 공개 방식

### 목표

GEMINI.md Rule과 Agent Skill의 핵심 차이인 "점진적 공개" 방식을 이해하고, 각 상황에 맞는 선택을 내릴 수 있다.

### 단계별 구현

#### Step 1.1 — Rule과 Skill 비교

> **💡 개념 설명: 점진적 공개(Progressive Disclosure)란?**
>
> 모든 지식을 한 번에 공개하는 대신, 필요한 순간에만 관련 지식을 꺼내는 방식이다. 도서관에 비유하면, GEMINI.md Rule은 항상 책상 위에 펼쳐져 있는 참고서이고, Skill은 필요할 때만 서가에서 꺼내는 전문 서적이다.
>
> Antigravity는 사용자 요청의 의도를 분석하고, 활성화된 Skill들의 `description`과 대조하여 가장 적합한 Skill을 자동으로 로드한다. 이 덕분에 수십 개의 전문 Skill을 등록해도 컨텍스트 창을 낭비하지 않는다.
>
> **핵심 한 줄:** Rule = 항상 켜진 기본 원칙 / Skill = 자동으로 꺼내는 전문 매뉴얼

| 구분 | GEMINI.md Rule | Agent Skill |
|------|----------------|-------------|
| 로드 시점 | 항상 (세션 시작 시) | description 매칭 시에만 |
| 적합한 내용 | 코딩 스타일, 커밋 규칙 등 보편적 원칙 | 배포 절차, 특정 도구 사용법 등 전문 지식 |
| 컨텍스트 영향 | 항상 소비 | 필요 시에만 소비 |
| 파일 형식 | Markdown | SKILL.md + 추가 파일들 (디렉토리) |
| 활성화 방식 | 자동 (파일 존재만으로) | description 키워드 매칭 |
| 파일 번들링 | 단일 파일 | 디렉토리 내 여러 파일 |

#### Step 1.2 — Skill이 적합한 상황 식별

다음 기준 중 하나라도 해당하면 Skill로 만드는 것이 좋다:

- 특정 작업(배포, 테스트, 리뷰 등)에만 필요한 전문 지식
- 단계별 절차가 명확한 워크플로우
- 실행 가능한 스크립트나 보조 파일이 필요한 작업
- "항상 켜두기엔 너무 긴" 전문 내용
- 프로젝트마다 다른 버전이 필요한 지침

### 체크포인트

팀의 코딩 컨벤션은 Rule, AWS 배포 절차는 Skill로 구분할 수 있는가?

---

## Phase 2: 첫 번째 Agent Skill 만들기

### 목표

Django 배포 Skill을 처음부터 작성하고 Antigravity에서 "배포해줘" 요청 시 자동으로 활성화되는지 확인한다.

### 단계별 구현

#### Step 2.1 — Skill 디렉토리 구조 이해

```
프로젝트 Skills (해당 프로젝트에서만 사용):
<project-root>/.agent/skills/<skill-name>/

전역 Skills (모든 프로젝트에서 사용 가능):
~/.gemini/antigravity/skills/<skill-name>/
```

> **💡 개념 설명: Antigravity의 디렉토리 구조**
>
> Antigravity의 설정 디렉토리는 `.agent/`이다 (Gemini CLI와 구분). 하지만 전역 Skills는 Gemini CLI와 동일한 `~/.gemini/` 경로를 공유한다. 주의: 일부 버전에서는 `~/.gemini/antigravity/skills/`를 사용한다. 현재 사용 버전의 공식 문서를 확인하자.
>
> **핵심 한 줄:** 프로젝트 전용 = `.agent/skills/`, 전역 = `~/.gemini/antigravity/skills/`

```bash
# 프로젝트 Skill 디렉토리 생성
mkdir -p .agent/skills/django-deploy

# 전역 Skill 디렉토리 생성
mkdir -p ~/.gemini/antigravity/skills/code-reviewer
```

#### Step 2.2 — SKILL.md frontmatter 작성

> **💡 개념 설명: SKILL.md frontmatter**
>
> SKILL.md 파일 최상단의 `---`로 감싸진 블록이 frontmatter이다. Antigravity는 이 메타데이터의 `description`을 읽어서 사용자 요청과 의미적으로 매칭한다. description을 잘 쓸수록 자동 활성화 정확도가 높아진다.
>
> 중요한 점은 단순 키워드 매칭이 아니라 의미적 유사도를 사용하기 때문에, 다양한 표현으로 설명하는 것이 효과적이다.
>
> **핵심 한 줄:** description = "이런 요청이 오면 나를 꺼내라"는 의미 신호

```markdown
<!-- .agent/skills/django-deploy/SKILL.md -->

---
name: django-deploy
description: >
  Django 웹 애플리케이션을 AWS Elastic Beanstalk에 배포한다.
  사용자가 "배포", "EB 배포", "AWS 배포", "프로덕션 올리기",
  "deploy", "release" 등을 요청할 때 활성화된다.
---

# Django Elastic Beanstalk 배포 가이드

당신은 Django + AWS EB 배포 전문가이다.
배포 요청이 오면 반드시 아래 순서를 따른다.

## 배포 전 체크리스트

### 1. 코드 상태 확인
- [ ] `git status` — 커밋되지 않은 변경사항이 없는가?
- [ ] `git log --oneline -5` — 최근 커밋 내용 확인

### 2. 테스트 실행
- [ ] `python manage.py test` — 모든 테스트 통과
- [ ] `python manage.py check --deploy` — 배포 관련 설정 검사

### 3. 마이그레이션 확인
- [ ] `python manage.py showmigrations` — 미적용 마이그레이션 없는가?

## 배포 실행 순서

```bash
# 1. 가상환경 활성화 확인
source venv/bin/activate

# 2. 의존성 업데이트
pip freeze > requirements.txt

# 3. EB 배포
eb deploy

# 4. 상태 확인
eb status
eb health
```

## 배포 후 확인

배포 완료 후 반드시 확인한다:
1. `eb open` — 애플리케이션이 정상 응답하는가?
2. 에러 로그: `eb logs --all`
3. 마이그레이션 실행 (필요 시): `eb ssh` → `python manage.py migrate`

## 롤백 방법

배포 실패 시 즉시 롤백:
```bash
eb deploy --version previous
```
```

#### Step 2.3 — Skill 동작 확인

Antigravity 대화에서 트리거 요청을 입력한다:

```
배포해줘
```

또는:

```
EB 배포 진행해줘
```

Skill이 활성화되면 에디터 하단에 "Skill: django-deploy 활성화됨" 표시가 나타난다.

### 체크포인트

"배포해줘"라고 요청했을 때 django-deploy Skill이 자동으로 활성화되고, 배포 체크리스트를 단계별로 안내해주면 성공이다.

---

## Phase 3: 고급 기능 — 스크립트 번들링과 Skill 관리

### 목표

단순한 지침 파일을 넘어 실행 스크립트, 참고 자료, 예시 파일을 번들링하여 더 강력한 Skill을 만든다.

### 단계별 구현

#### Step 3.1 — Skill 디렉토리 전체 구조

```
.agent/skills/django-deploy/
├── SKILL.md              ← 필수: Skill 정의 및 핵심 지침
├── instructions.md       ← 선택: 상세 지침 (SKILL.md가 너무 길어질 때)
├── scripts/              ← 선택: 실행 스크립트
│   ├── pre-deploy.sh     ← 배포 전 자동 검사
│   ├── deploy.sh         ← 실제 배포 실행
│   └── health-check.sh   ← 배포 후 상태 확인
├── resources/            ← 선택: 참고 자료
│   ├── troubleshooting.md
│   └── eb-config-template.yml
└── examples/             ← 선택: 예시 파일
    └── .ebextensions/
        └── django.config
```

#### Step 3.2 — 스크립트 작성 및 연결

```bash
#!/bin/bash
# .agent/skills/django-deploy/scripts/pre-deploy.sh
# 배포 전 자동 검사 스크립트

set -e  # 오류 발생 시 즉시 중단

echo "=== 배포 전 검사 시작 ==="

echo "1. 테스트 실행 중..."
python manage.py test --verbosity=0
echo "   ✅ 테스트 통과"

echo "2. 배포 설정 검사 중..."
python manage.py check --deploy
echo "   ✅ 설정 검사 통과"

echo "3. 미적용 마이그레이션 확인 중..."
PENDING=$(python manage.py showmigrations | grep "\[ \]" | wc -l)
if [ "$PENDING" -gt 0 ]; then
    echo "   ⚠️  미적용 마이그레이션 ${PENDING}개 발견"
    echo "   배포 후 수동으로 마이그레이션을 실행해야 합니다"
fi

echo "=== 모든 검사 완료 ==="
```

```bash
chmod +x .agent/skills/django-deploy/scripts/pre-deploy.sh
```

#### Step 3.3 — SKILL.md에서 스크립트 참조

스크립트를 SKILL.md 지침에서 참조하면 Antigravity가 자동으로 실행할 수 있다:

```markdown
<!-- .agent/skills/django-deploy/SKILL.md 일부 -->

## 배포 절차

배포 요청이 오면 반드시 다음 순서로 진행한다:

### Step 1: 자동 검사
`scripts/pre-deploy.sh`를 실행하여 배포 가능 여부를 확인한다.
검사를 통과하지 못하면 배포를 중단하고 사용자에게 알린다.

### Step 2: 배포 실행
검사 통과 후 `scripts/deploy.sh`를 실행하거나,
사용자에게 명령어를 제안한다:
```

#### Step 3.4 — Skill 우선순위

같은 이름의 Skill이 여러 위치에 있을 경우 프로젝트 Skill이 전역 Skill보다 우선한다:

```
프로젝트 Skill  >  전역 Skill  >  확장 패키지 Skill
.agent/skills/  >  ~/.gemini/antigravity/skills/  >  패키지 설치 Skill
```

활용 팁: 범용 `code-reviewer` Skill을 전역으로 등록하되, Python 프로젝트에서는 `.agent/skills/code-reviewer/`로 Python 특화 버전을 덮어쓸 수 있다.

### 체크포인트

스크립트가 있는 Skill을 만들고, 배포 요청 시 스크립트가 자동으로 실행되거나 실행 방법이 안내되면 성공이다.

---

## Phase 4: 실전 예시

### 목표

실무에서 즉시 활용할 수 있는 Skill 3개를 작성한다.

### 예시 1: 코드 리뷰 Skill

```markdown
<!-- ~/.gemini/antigravity/skills/code-reviewer/SKILL.md -->

---
name: code-reviewer
description: >
  코드 품질 분석 및 개선 제안 전문가.
  "코드 리뷰", "코드 검토", "리팩토링 제안",
  "PR 리뷰", "review" 요청 시 활성화된다.
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

### ✅ 잘된 점
- (구체적인 칭찬 — 반드시 1개 이상)

### 🔧 개선 제안
- **[파일명:라인번호]** 문제 설명
  - 현재: `기존 코드`
  - 제안: `개선된 코드`
  - 이유: 개선이 필요한 근거

### 🚨 즉시 수정 필요
- (보안 취약점, 크리티컬 버그 등 긴급 항목)
```

### 예시 2: 데이터베이스 마이그레이션 Skill

```markdown
<!-- .agent/skills/db-migration/SKILL.md -->

---
name: db-migration
description: >
  데이터베이스 스키마 마이그레이션 전문가.
  "마이그레이션", "DB 변경", "스키마 수정",
  "테이블 추가", "migration" 요청 시 활성화된다.
---

# DB Migration Skill

마이그레이션 작업은 데이터 손실 위험이 있으므로 신중하게 진행한다.

## 마이그레이션 전 필수 확인

1. **백업 확인**: 최근 백업이 존재하는가?
2. **롤백 계획**: 마이그레이션 실패 시 되돌리는 방법이 있는가?
3. **영향 분석**: 변경이 기존 코드에 영향을 주는가?

## 마이그레이션 유형별 주의사항

### 컬럼 추가 (안전)
- NULL 허용 또는 기본값을 반드시 지정한다
- 예: `ALTER TABLE users ADD COLUMN phone VARCHAR(20) DEFAULT NULL;`

### 컬럼 삭제 (위험 🚨)
- 반드시 2단계로 진행:
  1. 코드에서 해당 컬럼 참조를 먼저 제거 → 배포
  2. 일정 기간 모니터링 후 컬럼 삭제

### 컬럼명 변경 (위험 🚨)
- 새 컬럼 추가 → 데이터 복사 → 코드 변경 → 배포 → 구 컬럼 삭제

## Django 마이그레이션 절차

```bash
# 1. 마이그레이션 파일 생성
python manage.py makemigrations --name 변경내용_설명

# 2. SQL 미리 확인 (매우 중요!)
python manage.py sqlmigrate app_name 0001

# 3. 개발 DB에서 테스트
python manage.py migrate

# 4. 프로덕션 배포 전 dry-run 확인
python manage.py migrate --plan
```

`resources/migration-checklist.md`의 체크리스트를 반드시 완료 후 진행한다.
```

### 예시 3: API 설계 Skill

```markdown
<!-- ~/.gemini/antigravity/skills/api-designer/SKILL.md -->

---
name: api-designer
description: >
  RESTful API 설계 전문가.
  "API 만들어줘", "엔드포인트 설계", "REST API",
  "API 스펙 작성" 요청 시 활성화된다.
---

# API Designer Skill

당신은 RESTful API 설계 전문가이다.

## 설계 원칙

### 1. 리소스 중심 URL
- 동사 대신 명사 사용: `/users` (✅) vs `/getUsers` (❌)
- 복수형 사용: `/articles` (✅) vs `/article` (❌)
- 중첩은 최대 2단계: `/users/{id}/posts` (✅) vs `/users/{id}/posts/{id}/comments/{id}` (❌)

### 2. HTTP 메서드 의미
- GET: 조회 (부작용 없음)
- POST: 생성
- PUT: 전체 수정
- PATCH: 부분 수정
- DELETE: 삭제

### 3. 응답 형식 표준화
```json
{
  "data": { ... },
  "meta": {
    "total": 100,
    "page": 1,
    "per_page": 20
  },
  "error": null
}
```

### 4. 에러 응답
```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "이메일 형식이 올바르지 않습니다",
    "details": [{ "field": "email", "message": "..." }]
  }
}
```

## API 명세 작성 형식

각 엔드포인트에 대해 다음을 명시한다:
- **URL**: 경로 및 경로 파라미터
- **Method**: HTTP 메서드
- **Auth**: 인증 필요 여부
- **Request**: Body/Query 파라미터 (타입, 필수 여부, 설명)
- **Response**: 성공/실패 응답 예시
- **Errors**: 발생 가능한 에러 코드 목록
```

### 체크포인트

3개의 Skill을 작성하고, 각각 트리거 키워드로 요청했을 때 자동 활성화되는지 확인했는가?

---

## 다른 도구와의 비교

| 기능 | Antigravity | Gemini CLI | Claude Code | Cursor |
|------|-------------|------------|-------------|--------|
| Skills 위치 | `.agent/skills/<name>/` | `.gemini/skills/<name>/` | `.claude/commands/` | Notepads + `.cursor/rules/` |
| 전역 위치 | `~/.gemini/antigravity/skills/` | `~/.gemini/skills/` | `~/.claude/commands/` | 없음 |
| 활성화 방식 | description 키워드 매칭 (자동) | description 키워드 매칭 (자동) | 사용자가 `/명령어` 입력 | 에이전트 자동 판단 |
| 파일 구조 | 디렉토리 (SKILL.md + 추가 파일) | 디렉토리 (SKILL.md + 추가 파일) | 단일 .md 파일 | 단일 .mdc 파일 |
| 스크립트 번들 | ✅ (scripts/ 서브디렉토리) | ✅ (scripts/ 서브디렉토리) | ❌ (단일 파일만) | ❌ |
| 우선순위 | 프로젝트 > 전역 | 프로젝트 > 전역 | 프로젝트 > 전역 | 항상 동일 |
| 규칙 파일 | `GEMINI.md` | `GEMINI.md` | `CLAUDE.md` | `.cursorrules` |
| 인자 전달 | 없음 (컨텍스트 기반) | 없음 (컨텍스트 기반) | `$ARGUMENTS` 변수 | 없음 |

---

## 마무리

### 이 가이드에서 배운 것

- Rule vs Skill: 항상 로드 vs description 매칭 시 로드 (점진적 공개)
- SKILL.md frontmatter의 `description`이 자동 매칭의 핵심
- 디렉토리 구조로 SKILL.md, 스크립트, 참고 자료를 번들링
- 프로젝트 Skill이 전역 Skill보다 우선 적용됨

### 효과적인 description 작성 팁

```markdown
# ❌ 나쁜 예 (너무 모호함)
description: 코드와 관련된 도움을 제공한다.

# ✅ 좋은 예 (구체적인 트리거 키워드 + 동의어 포함)
description: >
  코드 품질 분석 전문가.
  "코드 리뷰", "PR 리뷰", "리팩토링 제안",
  "코드 검토", "review", "코드 개선" 요청 시 활성화.
```

### 다음 단계

- `05_workflow` — 여러 Skill을 연결하는 복합 워크플로우 구성
- `02_rule/antigravity-rule-guide.md` — GEMINI.md Rule과 Skill의 최적 조합 전략
