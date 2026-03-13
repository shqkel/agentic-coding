# VS Code Copilot Instructions 작성 가이드

## 학습 목표

GitHub Copilot의 커스텀 지시(Custom Instructions) 시스템을 이해하고, `.github/copilot-instructions.md` 파일과 VS Code `settings.json` 설정을 활용하여 프로젝트에 맞는 AI 코딩 환경을 구성할 수 있다.

## 사전 준비

- VS Code 설치 및 GitHub Copilot 확장 프로그램 활성화
- GitHub Copilot 구독 (Individual, Business, 또는 Enterprise)
- Django 프로젝트 구조에 대한 기본 이해
- 마크다운 문법 기초 지식

---

## 전체 흐름 한눈에 보기

Copilot Instructions는 GitHub Copilot이 코드를 생성하거나 채팅에 응답할 때 참고하는 "프로젝트 지침서"이다. CLAUDE.md나 GEMINI.md와 유사한 역할을 하지만, VS Code 설정 시스템과 긴밀하게 연동되는 것이 특징이다.

**주요 단계:**
1. Copilot Instructions의 역할과 구조 이해
2. 스코프 계층(글로벌, 프로젝트, 파일별) 파악
3. `copilot-instructions.md` 작성법 습득
4. VS Code `settings.json` 연동 설정
5. 콘텐츠 제외(Content Exclusion) 설정
6. Django 프로젝트 실전 적용

---

## Phase 1: Copilot Instructions 이해하기

### 목표
Copilot Instructions가 무엇인지, 다른 AI 도구의 규칙 파일과 어떻게 다른지 이해한다.

### 단계별 구현

#### Step 1.1 --- copilot-instructions.md의 역할 파악하기

`copilot-instructions.md`는 GitHub Copilot이 코드 생성, 채팅 응답, 코드 리뷰 등을 수행할 때 참고하는 프로젝트 수준의 지침 파일이다. 이 파일은 `.github/` 디렉토리 안에 위치한다.

> **💡 개념 설명: copilot-instructions.md란?**
>
> Copilot은 대화마다 프로젝트의 맥락을 모른다. `copilot-instructions.md`에 프로젝트 규칙을 작성해두면:
> - **일관성**: 모든 Copilot 응답에서 동일한 코딩 스타일 유지
> - **효율성**: 매번 "우리 프로젝트는 DRF를 사용해"라고 말할 필요 없음
> - **팀 협업**: 저장소에 커밋하여 팀 전체가 동일한 지침 공유
>
> **핵심 차이점**: Copilot의 지침은 "규칙(Rule)"이 아니라 "제안(Suggestion)"에 가깝다. Copilot이 지침을 항상 100% 따르지는 않을 수 있다.
>
> **한 줄 요약**: `copilot-instructions.md`는 Copilot에게 전달하는 프로젝트 맥락 파일이다.

#### Step 1.2 --- Copilot이 지침을 읽는 방식

VS Code는 워크스페이스 루트의 `.github/copilot-instructions.md` 파일을 자동으로 감지하여 해당 워크스페이스 내 모든 Copilot 요청에 지침을 적용한다.

```
my-django-project/
├── .github/
│   └── copilot-instructions.md   ← Copilot이 자동으로 읽는 파일
├── manage.py
├── requirements.txt
└── myapp/
    ├── models.py
    └── views.py
```

> **💡 개념 설명: 자동 감지 vs 명시적 설정**
>
> - `.github/copilot-instructions.md`는 VS Code가 **자동으로 감지**한다
> - 추가적인 지침 파일은 `settings.json`에서 **명시적으로 경로를 지정**해야 한다
> - `.instructions.md` 확장자를 가진 파일은 YAML 프론트매터의 `applyTo` 패턴에 따라 **동적으로 적용**된다
>
> **한 줄 요약**: 기본 파일은 자동 감지, 추가 파일은 설정이 필요하다.

#### Step 1.3 --- 다른 AI 도구와의 비교

| 항목 | Copilot Instructions | CLAUDE.md | GEMINI.md |
|------|---------------------|-----------|-----------|
| **파일 위치** | `.github/copilot-instructions.md` | 프로젝트 루트 `CLAUDE.md` | 프로젝트 루트 `GEMINI.md` |
| **자동 감지** | VS Code가 자동 감지 | Claude Code가 자동 감지 | gemini-cli가 자동 감지 |
| **스코프 계층** | 글로벌 + 프로젝트 + 파일별 | 글로벌 + 프로젝트 + 서브디렉토리 | 글로벌 + 프로젝트 + 서브디렉토리 |
| **지침 강제성** | 제안 수준 (미준수 가능) | 강한 규칙 | 강한 규칙 |
| **설정 연동** | VS Code settings.json | 독립 실행형 | 독립 실행형 |
| **형식** | Markdown | Markdown | Markdown (Rule/Context 구분) |
| **파일별 지침** | `.instructions.md` + applyTo | 서브디렉토리 CLAUDE.md | 서브디렉토리 GEMINI.md |
| **초기화 명령** | `/generateInstructions` | 수동 작성 | `gemini init` |

> **💡 개념 설명: "제안" vs "규칙"**
>
> CLAUDE.md나 GEMINI.md의 지침은 대체로 강하게 준수된다. 반면 Copilot의 지침은:
> - 복잡한 다단계 절차는 무시될 수 있다
> - 외부 리소스 참조 요청은 작동하지 않는다
> - 간결하고 구체적인 지침일수록 잘 따른다
>
> **한 줄 요약**: Copilot 지침은 "강제 규칙"보다는 "강한 힌트"에 가깝다.

### 체크포인트

- [ ] `copilot-instructions.md`의 위치(`.github/` 디렉토리)를 알고 있다
- [ ] Copilot Instructions가 "규칙"이 아닌 "제안" 성격임을 이해했다
- [ ] CLAUDE.md, GEMINI.md와의 핵심 차이점을 설명할 수 있다

---

## Phase 2: 스코프 계층

### 목표
Copilot Instructions의 세 가지 스코프(글로벌, 프로젝트, 파일별)를 이해하고 적절히 활용한다.

### 단계별 구현

#### Step 2.1 --- 글로벌 스코프: VS Code settings.json

사용자 수준의 `settings.json`에 지침을 설정하면 모든 프로젝트에 공통 적용된다.

```jsonc
// 사용자 settings.json (글로벌)
// Windows: %APPDATA%\Code\User\settings.json
// macOS: ~/Library/Application Support/Code/User/settings.json
// Linux: ~/.config/Code/User/settings.json

{
    "github.copilot.chat.codeGeneration.instructions": [
        {
            "text": "모든 코드에 한국어 주석을 작성한다."
        },
        {
            "text": "변수명과 함수명은 영어로 작성하되, snake_case를 사용한다."
        }
    ]
}
```

#### Step 2.2 --- 프로젝트 스코프: .github/copilot-instructions.md

프로젝트 루트의 `.github/copilot-instructions.md` 파일은 해당 워크스페이스에서만 적용된다.

```
my-project/
├── .github/
│   └── copilot-instructions.md   ← 프로젝트 전체에 적용
├── .vscode/
│   └── settings.json             ← 워크스페이스 설정 (추가 지침 경로 지정 가능)
└── src/
```

> **💡 개념 설명: 스코프 우선순위**
>
> Copilot은 여러 스코프의 지침을 **결합(merge)**하여 사용한다:
> 1. **글로벌** (사용자 settings.json) --- 모든 프로젝트에 적용
> 2. **프로젝트** (.github/copilot-instructions.md) --- 현재 워크스페이스에 적용
> 3. **파일별** (.instructions.md 파일) --- 특정 파일 패턴에 적용
>
> CLAUDE.md의 경우 서브디렉토리마다 별도 파일을 둘 수 있지만, Copilot은 `.instructions.md` 파일의 `applyTo` 패턴으로 파일별 지침을 관리한다.
>
> **한 줄 요약**: 글로벌 + 프로젝트 + 파일별 지침이 결합되어 적용된다.

#### Step 2.3 --- 파일별 스코프: .instructions.md 파일

`.github/instructions/` 디렉토리에 `*.instructions.md` 파일을 만들면, YAML 프론트매터의 `applyTo` 패턴에 따라 특정 파일에만 지침이 적용된다.

```
my-project/
├── .github/
│   ├── copilot-instructions.md              ← 프로젝트 전체
│   └── instructions/
│       ├── python-style.instructions.md     ← Python 파일에만 적용
│       ├── typescript.instructions.md       ← TypeScript 파일에만 적용
│       └── django-models.instructions.md    ← 모델 파일에만 적용
```

파일별 지침 파일의 구조:

```markdown
---
applyTo: "**/*.py"
description: "Python 코드 스타일 가이드"
---

- 모든 함수에 타입 힌트를 사용한다.
- docstring은 Google 스타일을 따른다.
- import 순서: 표준 라이브러리 > 서드파티 > 로컬.
```

여러 패턴을 쉼표로 구분하여 지정할 수도 있다:

```markdown
---
applyTo: "**/*.ts,**/*.tsx"
description: "TypeScript/React 코드 가이드"
---

- 컴포넌트는 함수형으로 작성한다.
- Props 인터페이스를 반드시 정의한다.
```

#### Step 2.4 --- 스코프 계층 비교표

| 스코프 | 설정 위치 | 적용 범위 | 사용 사례 |
|--------|-----------|-----------|-----------|
| 글로벌 | 사용자 `settings.json` | 모든 프로젝트 | 개인 코딩 스타일, 언어 선호 |
| 프로젝트 | `.github/copilot-instructions.md` | 현재 워크스페이스 | 프로젝트 규칙, 기술 스택 |
| 파일별 | `.github/instructions/*.instructions.md` | `applyTo` 패턴 매칭 파일 | 언어별/프레임워크별 규칙 |
| 워크스페이스 설정 | `.vscode/settings.json` | 현재 워크스페이스 | 추가 지침 파일 경로 |

### 체크포인트

- [ ] 세 가지 스코프(글로벌, 프로젝트, 파일별)의 차이를 설명할 수 있다
- [ ] `.instructions.md` 파일의 `applyTo` YAML 프론트매터를 작성할 수 있다
- [ ] 스코프가 결합(merge)되어 적용됨을 이해했다

---

## Phase 3: copilot-instructions.md 작성법

### 목표
효과적인 `copilot-instructions.md`를 작성하는 방법과 주의사항을 익힌다.

### 단계별 구현

#### Step 3.1 --- 기본 형식

`copilot-instructions.md`는 일반 Markdown 파일이다. 특별한 헤더나 섹션 구조가 강제되지 않지만, 간결하고 명확한 문장이 효과적이다.

```markdown
# Project Instructions

이 프로젝트는 Django 4.2 기반의 블로그 플랫폼이다.

## 코드 스타일
- PEP 8을 따른다.
- 함수와 클래스에 docstring을 작성한다 (Google 스타일).
- 한 줄 최대 88자 (Black 포매터 기준).

## 프레임워크
- Django REST Framework로 API를 구현한다.
- 클래스 기반 뷰(CBV)를 우선 사용한다.
- 인증은 SimpleJWT를 사용한다.

## 네이밍
- 변수/함수: snake_case
- 클래스: PascalCase
- 상수: UPPER_SNAKE_CASE
```

> **💡 개념 설명: 효과적인 지침 작성의 핵심 원칙**
>
> GitHub 공식 문서에서 권장하는 5가지 팁:
> 1. **짧고 독립적으로**: 각 지침은 한 문장으로 완결되어야 한다
> 2. **이유를 포함**: "왜" 그런 규칙인지 설명하면 AI가 엣지 케이스에서 더 나은 판단을 한다
> 3. **코드 예시 제공**: 추상적 설명보다 구체적 코드 패턴이 효과적이다
> 4. **비자명한 규칙 위주**: 린터가 이미 잡아주는 것은 제외한다
> 5. **외부 참조 금지**: "공식 문서를 참고해라"는 작동하지 않는다
>
> **한 줄 요약**: 짧고, 구체적이며, 이유가 포함된 지침이 가장 효과적이다.

#### Step 3.2 --- 잘 작동하는 지침

Copilot이 잘 따르는 지침 유형:

```markdown
## 잘 작동하는 지침 예시

### 코딩 스타일
- 모든 함수에 타입 힌트를 사용한다.
- f-string을 사용한다 (.format() 대신).

### 프레임워크 선호
- ORM 쿼리에서 .select_related()와 .prefetch_related()를 적극 사용한다.
  (이유: N+1 쿼리 문제를 방지하기 위해서이다)

### 네이밍 컨벤션
- Boolean 변수는 is_, has_, can_ 접두사를 사용한다.
  예: is_active, has_permission, can_edit

### 선호/비선호 패턴
- 선호: `from django.db import models` 후 `models.CharField`
- 비선호: `from django.db.models import CharField` 후 `CharField`
```

#### Step 3.3 --- 잘 작동하지 않는 지침

다음과 같은 지침은 Copilot이 무시하거나 잘못 해석할 가능성이 높다:

```markdown
## 잘 작동하지 않는 지침 예시 (피해야 할 것)

### 복잡한 다단계 절차 (비효과적)
- 새 API를 만들 때는 먼저 serializer를 작성하고,
  그 다음 viewset을 만들고,
  urls.py에 등록한 후,
  테스트를 작성하고,
  마지막으로 문서를 업데이트한다.

### 외부 리소스 참조 (작동 안 함)
- Django 공식 문서(https://docs.djangoproject.com)를 참고하여 코드를 작성한다.

### 너무 추상적인 지침 (비효과적)
- 코드를 깔끔하고 유지보수하기 좋게 작성한다.
- 보안에 신경 쓴다.

### 모순되는 지침 (혼란 유발)
- 함수는 최대한 짧게 작성한다.
- 모든 에러 케이스를 함수 내에서 처리한다.
```

> **💡 개념 설명: 지침 작성의 안티패턴**
>
> | 안티패턴 | 이유 | 개선 방법 |
> |---------|------|-----------|
> | 다단계 절차 | Copilot은 상태를 기억하지 않음 | 각 단계를 독립된 지침으로 분리 |
> | 외부 URL 참조 | Copilot은 URL에 접근하지 않음 | 필요한 내용을 직접 기술 |
> | 추상적 표현 | 해석의 여지가 넓음 | 구체적 수치나 패턴 제시 |
> | 모순 지침 | AI가 어느 쪽을 따를지 예측 불가 | 우선순위를 명시하거나 하나만 유지 |
>
> **한 줄 요약**: 한 문장으로 완결되고, 외부 참조 없이, 구체적으로 작성한다.

#### Step 3.4 --- /generateInstructions 명령으로 자동 생성

VS Code의 Copilot Chat에서 `/generateInstructions` 명령을 사용하면 프로젝트를 분석하여 `copilot-instructions.md` 초안을 자동 생성할 수 있다.

```
# Copilot Chat에서 입력
/generateInstructions
```

이 명령은 프로젝트의 코딩 스타일, 사용 중인 프레임워크, 파일 구조 등을 분석하여 기본 지침을 생성해준다. 생성된 초안을 검토하고 팀 규칙에 맞게 수정하면 된다.

### 체크포인트

- [ ] 효과적인 지침의 5가지 원칙을 이해했다
- [ ] 잘 작동하는 지침과 작동하지 않는 지침의 차이를 구분할 수 있다
- [ ] `/generateInstructions`로 초안을 생성할 수 있다

---

## Phase 4: VS Code settings.json 연동

### 목표
`settings.json`을 통해 다양한 유형의 Copilot 지침을 설정하고 관리한다.

### 단계별 구현

#### Step 4.1 --- codeGeneration.instructions 설정

코드 생성 시 적용할 지침을 배열 형태로 설정한다. 텍스트 기반과 파일 기반 두 가지 방식을 지원한다.

```jsonc
// .vscode/settings.json (워크스페이스)

{
    "github.copilot.chat.codeGeneration.instructions": [
        // 방법 1: 텍스트 기반 (인라인 지침)
        {
            "text": "Python 코드에 타입 힌트를 항상 사용한다."
        },
        {
            "text": "Django 모델에는 __str__ 메서드를 반드시 정의한다."
        },

        // 방법 2: 파일 기반 (외부 마크다운 파일 참조)
        {
            "file": "./docs/coding-standards.md"
        },
        {
            "file": "./docs/api-conventions.md"
        }
    ]
}
```

> **💡 개념 설명: 텍스트 vs 파일 기반 지침**
>
> | 방식 | 장점 | 단점 | 사용 시기 |
> |------|------|------|-----------|
> | 텍스트 (`text`) | 설정에서 바로 확인 가능 | 길어지면 관리 어려움 | 1-2줄의 간단한 규칙 |
> | 파일 (`file`) | 체계적 관리, Git 추적 용이 | 별도 파일 관리 필요 | 상세한 코딩 가이드라인 |
>
> 두 방식을 혼합하여 사용할 수 있다. 간단한 규칙은 텍스트로, 상세 가이드는 파일로 관리하는 것이 효과적이다.
>
> **한 줄 요약**: 짧은 규칙은 `text`, 긴 가이드는 `file`로 관리한다.

#### Step 4.2 --- 작업 유형별 지침 설정

Copilot은 코드 생성 외에도 다양한 작업에 대한 지침을 지원한다.

```jsonc
// .vscode/settings.json

{
    // 코드 생성 지침
    "github.copilot.chat.codeGeneration.instructions": [
        { "text": "Django REST Framework의 ViewSet을 사용한다." },
        { "text": "모든 API 응답은 {status, data, message} 형식이다." }
    ],

    // 테스트 생성 지침
    "github.copilot.chat.testGeneration.instructions": [
        { "text": "pytest와 pytest-django를 사용한다." },
        { "text": "각 테스트 함수는 test_ 접두사로 시작한다." },
        { "text": "테스트 클래스명은 Test 접두사를 사용한다 (예: TestPostModel)." },
        { "text": "factory_boy를 사용하여 테스트 데이터를 생성한다." }
    ],

    // 커밋 메시지 생성 지침
    "github.copilot.chat.commitMessageGeneration.instructions": [
        { "text": "Conventional Commit 형식을 사용한다: type(scope): description" },
        { "text": "type: feat, fix, docs, style, refactor, test, chore" },
        { "text": "한국어로 description을 작성한다." },
        { "text": "제목은 50자 이내로 작성한다." }
    ],

    // 코드 리뷰 지침
    "github.copilot.chat.reviewSelection.instructions": [
        { "text": "보안 취약점을 우선 검토한다." },
        { "text": "N+1 쿼리 문제가 있는지 확인한다." },
        { "text": "타입 힌트 누락 여부를 확인한다." }
    ]
}
```

#### Step 4.3 --- 사용자 설정 vs 워크스페이스 설정

```
설정 우선순위 (높은 것이 낮은 것을 덮어씀):

┌─────────────────────────────────────────┐
│  워크스페이스 설정                        │  ← 최우선 (프로젝트별)
│  .vscode/settings.json                  │
├─────────────────────────────────────────┤
│  사용자 설정                             │  ← 차순위 (글로벌)
│  ~/Library/.../settings.json            │
└─────────────────────────────────────────┘
```

```jsonc
// 사용자 settings.json (모든 프로젝트에 적용)
{
    "github.copilot.chat.codeGeneration.instructions": [
        { "text": "한국어 주석을 작성한다." }
    ]
}

// .vscode/settings.json (현재 프로젝트에만 적용)
{
    "github.copilot.chat.codeGeneration.instructions": [
        { "text": "Django REST Framework를 사용한다." },
        { "file": "./docs/api-guide.md" }
    ]
}
```

#### Step 4.4 --- Instruction Files 자동 사용 설정

`.instructions.md` 파일의 자동 적용 여부를 제어하는 설정:

```jsonc
{
    // .instructions.md 파일 자동 사용 (기본값: true)
    "github.copilot.chat.codeGeneration.useInstructionFiles": true
}
```

이 설정이 `true`이면, `.github/instructions/` 디렉토리의 `*.instructions.md` 파일이 `applyTo` 패턴에 따라 자동으로 적용된다.

### 체크포인트

- [ ] `codeGeneration`, `testGeneration`, `commitMessageGeneration` 설정의 차이를 이해했다
- [ ] 텍스트 기반과 파일 기반 지침을 혼합하여 설정할 수 있다
- [ ] 사용자 설정과 워크스페이스 설정의 우선순위를 파악했다

---

## Phase 5: 콘텐츠 제외 설정

### 목표
Copilot이 접근하거나 참고하지 않아야 할 파일과 디렉토리를 설정한다.

### 단계별 구현

#### Step 5.1 --- 콘텐츠 제외의 필요성

프로젝트에는 Copilot이 참고하면 안 되는 민감한 파일이 있을 수 있다:
- 환경 변수 파일 (`.env`)
- 비밀 키나 인증 정보
- 라이선스 제한이 있는 코드
- 레거시 코드 (잘못된 패턴 학습 방지)

#### Step 5.2 --- GitHub 조직/저장소 수준의 콘텐츠 제외

GitHub.com의 조직 또는 저장소 설정에서 Copilot이 접근하지 않을 경로를 지정할 수 있다.

```
# GitHub.com > Settings > Copilot > Content exclusion

# 제외 경로 예시
- "**/.env"
- "**/secrets/**"
- "legacy_code/**"
- "vendor/**"
```

> **💡 개념 설명: 콘텐츠 제외가 적용되면**
>
> 제외된 파일에 대해:
> - 해당 파일에서 인라인 제안이 비활성화된다
> - 해당 파일의 내용이 다른 파일의 제안에 영향을 주지 않는다
> - Copilot Chat에서 해당 파일 내용을 참조하지 않는다
>
> **주의**: 콘텐츠 제외는 GitHub 조직/저장소 관리자가 설정한다. 개인 사용자가 로컬에서 설정하는 것과는 다르다.
>
> **한 줄 요약**: 민감한 파일은 조직 수준에서 Copilot 접근을 차단한다.

#### Step 5.3 --- VS Code에서의 파일 제외

VS Code 설정에서 Copilot이 특정 파일을 무시하도록 할 수 있다:

```jsonc
// .vscode/settings.json

{
    // Copilot 인라인 제안에서 특정 언어 비활성화
    "github.copilot.enable": {
        "*": true,
        "plaintext": false,
        "markdown": false,
        "yaml": false
    }
}
```

#### Step 5.4 --- .copilotignore 확장 프로그램 활용

공식 기능은 아니지만, VS Code 마켓플레이스의 "Copilot Ignore" 확장 프로그램을 사용하면 `.gitignore`와 유사한 구문으로 파일을 제외할 수 있다.

```bash
# .copilotignore (확장 프로그램 사용 시)

# 환경 변수 파일
.env
.env.*

# 비밀 정보
secrets/
**/credentials.*

# 빌드 결과물
dist/
build/
node_modules/

# 레거시 코드
legacy/
old_code/

# 테스트 픽스처 (대량 데이터)
**/fixtures/*.json
```

> **💡 개념 설명: .copilotignore vs .gitignore**
>
> | 파일 | 용도 | 공식 지원 |
> |------|------|-----------|
> | `.gitignore` | Git 추적에서 제외 | Git 공식 |
> | `.copilotignore` | Copilot 컨텍스트에서 제외 | 확장 프로그램 (비공식) |
>
> `.copilotignore`는 아직 GitHub 공식 기능이 아니다. 공식적인 콘텐츠 제외는 GitHub.com의 조직/저장소 설정 또는 VS Code의 `github.copilot.enable` 설정을 사용해야 한다.
>
> **한 줄 요약**: 공식 콘텐츠 제외는 GitHub 설정에서, 로컬은 확장 프로그램으로 보완한다.

### 체크포인트

- [ ] 콘텐츠 제외가 필요한 파일 유형을 파악했다
- [ ] GitHub 조직 수준과 VS Code 로컬 수준의 제외 방법 차이를 이해했다
- [ ] `.copilotignore`가 비공식 확장 기능임을 인지했다

---

## Phase 6: 실전 예시 - Django 블로그 프로젝트

### 목표
실제 Django 블로그 프로젝트에 Copilot Instructions를 완전하게 적용한 예시를 보고, 자신의 프로젝트에 활용할 수 있다.

### 프로젝트 구조

```
blog-project/
├── .github/
│   ├── copilot-instructions.md          ← 프로젝트 전체 지침
│   └── instructions/
│       ├── python.instructions.md       ← Python 파일 지침
│       ├── django-models.instructions.md ← 모델 파일 지침
│       └── tests.instructions.md        ← 테스트 파일 지침
├── .vscode/
│   └── settings.json                    ← Copilot 설정
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
│   └── tests/
├── users/
│   ├── models.py
│   ├── views.py
│   └── serializers.py
├── manage.py
├── requirements.txt
└── pytest.ini
```

### 완성된 copilot-instructions.md

```markdown
# Blog Project Instructions

이 프로젝트는 Django 4.2 기반의 블로그 플랫폼이다.
사용자는 게시글 작성, 댓글, 태그 기반 검색을 할 수 있다.

## 기술 스택

- 백엔드: Django 4.2, Django REST Framework 3.14
- 데이터베이스: PostgreSQL 15
- 인증: djangorestframework-simplejwt
- 테스트: pytest, pytest-django, factory_boy
- 포매터: Black (88자), isort

## 코드 스타일

- PEP 8을 따르되 Black 포매터 기준 88자 제한을 사용한다.
- 모든 함수와 클래스에 Google 스타일 docstring을 작성한다.
- 타입 힌트를 사용한다. 반환 타입도 명시한다.
- f-string을 사용한다 (.format()이나 % 포매팅 대신).

## Django 규칙

- 클래스 기반 뷰(CBV)를 우선 사용한다.
- URL은 앱별로 urls.py를 분리하고 include()로 연결한다.
- 모든 모델에 created_at(auto_now_add=True)과 updated_at(auto_now=True)를 포함한다.
- 모든 모델에 __str__ 메서드를 정의한다.
- 외래 키에는 on_delete를 명시한다. 대부분 CASCADE를 사용한다.
- ORM 쿼리에서 select_related()와 prefetch_related()를 사용하여 N+1 문제를 방지한다.

## API 규칙

- ViewSet과 Router를 사용하여 RESTful API를 구현한다.
- API 응답 형식:
  ```json
  {"status": "success", "data": {}, "message": ""}
  ```
- 페이지네이션: PageNumberPagination, 기본 20개.
- 필터링: django-filter를 사용한다.

## 네이밍

- 변수/함수: snake_case (예: get_published_posts)
- 클래스: PascalCase (예: PostViewSet)
- 상수: UPPER_SNAKE_CASE (예: MAX_POST_LENGTH)
- URL 이름: kebab-case (예: post-detail)
- Boolean 변수: is_, has_, can_ 접두사 (예: is_published)

## 에러 처리

- 구체적인 예외 클래스를 사용한다 (bare except 금지).
- 커스텀 예외는 core/exceptions.py에 정의한다.
- 로깅은 logging 모듈을 사용한다 (print 금지).

## 테스트

- pytest와 pytest-django를 사용한다.
- 테스트 데이터는 factory_boy로 생성한다.
- 각 앱의 tests/ 디렉토리에 test_models.py, test_views.py로 분리한다.
```

### 파일별 지침 예시

#### python.instructions.md

```markdown
---
applyTo: "**/*.py"
description: "Python 코드 공통 규칙"
---

- import 순서: 표준 라이브러리 > 서드파티 > Django > 로컬 앱. isort로 자동 정렬한다.
- 모든 함수에 타입 힌트를 사용한다.
- 매직 넘버 대신 상수를 정의한다.
```

#### django-models.instructions.md

```markdown
---
applyTo: "**/models.py"
description: "Django 모델 작성 규칙"
---

- 모델 필드 순서: PK > FK/관계 > 일반 필드 > created_at > updated_at.
- 모든 모델에 class Meta를 정의한다 (ordering, verbose_name 포함).
- CharField에는 max_length를 항상 명시한다.
- blank=True와 null=True의 차이를 구분하여 사용한다.
  (문자열 필드는 null=True 대신 blank=True, default=""를 사용한다)
```

#### tests.instructions.md

```markdown
---
applyTo: "**/tests/**,**/test_*.py"
description: "테스트 코드 작성 규칙"
---

- Arrange-Act-Assert (AAA) 패턴으로 테스트를 구성한다.
- 테스트 함수명: test_<기능>_<조건>_<기대결과> 형식.
  예: test_create_post_without_title_returns_400
- factory_boy의 Factory 클래스를 사용하여 테스트 데이터를 생성한다.
- API 테스트에는 APITestCase와 APIClient를 사용한다.
```

### settings.json 설정 예시

```jsonc
// .vscode/settings.json

{
    // 코드 생성 지침
    "github.copilot.chat.codeGeneration.instructions": [
        { "text": "Django 4.2와 DRF 3.14 문법을 사용한다." },
        { "text": "Python 3.11 이상의 기능을 활용한다." }
    ],

    // 테스트 생성 지침
    "github.copilot.chat.testGeneration.instructions": [
        { "text": "pytest를 사용한다 (unittest 대신)." },
        { "text": "factory_boy로 테스트 데이터를 생성한다." },
        { "text": "API 테스트에는 rest_framework.test.APIClient를 사용한다." }
    ],

    // 커밋 메시지 지침
    "github.copilot.chat.commitMessageGeneration.instructions": [
        { "text": "Conventional Commit 형식: type(scope): 한국어 설명" },
        { "text": "scope는 앱 이름: posts, users, comments, config" },
        { "text": "예: feat(posts): 게시글 목록 페이지네이션 추가" }
    ],

    // 코드 리뷰 지침
    "github.copilot.chat.reviewSelection.instructions": [
        { "text": "N+1 쿼리 문제를 확인한다." },
        { "text": "타입 힌트 누락을 확인한다." },
        { "text": "보안 취약점(SQL 인젝션, XSS 등)을 확인한다." }
    ],

    // .instructions.md 파일 자동 사용
    "github.copilot.chat.codeGeneration.useInstructionFiles": true,

    // 특정 파일 유형에서 Copilot 비활성화
    "github.copilot.enable": {
        "*": true,
        "env": false
    }
}
```

### 체크포인트

- [ ] `.github/copilot-instructions.md`와 `.instructions.md` 파일을 모두 작성할 수 있다
- [ ] `settings.json`에서 작업 유형별 지침을 설정할 수 있다
- [ ] Django 프로젝트에 적합한 Copilot 설정을 구성했다

---

## 마무리

### 학습한 내용 정리

Copilot Instructions는 세 가지 계층으로 구성된다:

1. **글로벌** (사용자 settings.json): 모든 프로젝트에 적용되는 개인 코딩 스타일
2. **프로젝트** (.github/copilot-instructions.md): 팀이 공유하는 프로젝트 규칙
3. **파일별** (.instructions.md + applyTo): 특정 파일 유형에 적용되는 세부 규칙

### 멀티 도구 비교표

| 기능 | Copilot Instructions | CLAUDE.md | GEMINI.md | AGENTS.md |
|------|---------------------|-----------|-----------|-----------|
| **도구** | GitHub Copilot (VS Code) | Claude Code (CLI) | Gemini CLI | 범용 표준 |
| **파일 위치** | `.github/copilot-instructions.md` | 프로젝트 루트 | 프로젝트 루트 | 프로젝트 루트 |
| **글로벌 설정** | settings.json | `~/.claude/CLAUDE.md` | `~/.gemini/GEMINI.md` | 없음 |
| **서브디렉토리** | `.instructions.md` (applyTo) | 서브디렉토리 CLAUDE.md | 서브디렉토리 GEMINI.md | 서브디렉토리 AGENTS.md |
| **형식** | 자유 Markdown | 자유 Markdown | Rule + Context 구분 | 자유 Markdown |
| **지침 강제성** | 약함 (제안 수준) | 강함 | 강함 | 도구에 따라 다름 |
| **설정 연동** | VS Code settings.json | 없음 (독립형) | 없음 (독립형) | 없음 (독립형) |
| **자동 생성** | `/generateInstructions` | 수동 | `gemini init` | 수동 |
| **콘텐츠 제외** | GitHub 설정 / 확장 프로그램 | `.claudeignore` | 없음 | 없음 |
| **작업별 지침** | 코드생성/테스트/커밋/리뷰 분리 | 통합 | 통합 | 통합 |

### Copilot Instructions의 한계

CLI 도구(Claude Code, Gemini CLI)와 비교했을 때의 제약:

1. **지침 준수 강도**: Copilot은 지침을 "제안"으로 처리하여 항상 따르지는 않는다
2. **외부 참조 불가**: URL이나 외부 문서를 참조하라는 지침이 작동하지 않는다
3. **복잡한 로직 한계**: 다단계 절차나 조건부 규칙은 무시될 가능성이 높다
4. **서브디렉토리 스코프 없음**: CLAUDE.md처럼 디렉토리마다 파일을 두는 것이 아닌, `applyTo` glob 패턴으로만 분기 가능하다
5. **검증 도구 부재**: `gemini validate` 같은 지침 검증 명령이 없다

### 자주 하는 실수

1. **파일 위치 오류**
   - `.github/copilot-instructions.md`가 아닌 프로젝트 루트에 파일 생성
   - `.github` 디렉토리를 만들지 않음

2. **과도한 지침 작성**
   - 수십 개의 규칙을 나열하면 오히려 효과가 떨어진다
   - 핵심 규칙 10-20개를 간결하게 유지하는 것이 효과적이다

3. **외부 리소스 참조**
   - "Django 공식 문서를 참고해라"와 같은 지침은 작동하지 않는다
   - 필요한 내용은 지침 파일에 직접 기술한다

4. **린터와 중복되는 규칙**
   - "들여쓰기를 4칸으로 한다" 같은 규칙은 린터/포매터가 처리하므로 불필요하다
   - 도구가 자동으로 잡지 못하는 규칙에 집중한다

5. **settings.json과 copilot-instructions.md 혼동**
   - `settings.json`은 Copilot의 동작 방식 설정 (작업별 지침 분리 가능)
   - `copilot-instructions.md`는 프로젝트 맥락 전달 (모든 Copilot 요청에 적용)
   - 두 가지를 보완적으로 사용하는 것이 가장 효과적이다

### 다음 단계

1. **프로젝트에 적용**: `.github/copilot-instructions.md` 파일을 생성하고 핵심 규칙을 작성한다
2. **settings.json 구성**: 작업 유형별 지침을 설정하여 코드 생성, 테스트, 커밋 메시지를 커스터마이즈한다
3. **파일별 지침 추가**: `.instructions.md` 파일로 언어/프레임워크별 세부 규칙을 적용한다
4. **팀 공유**: `.github/copilot-instructions.md`와 `.vscode/settings.json`을 Git에 커밋하여 팀원과 공유한다
5. **AGENTS.md 탐색**: 여러 AI 도구를 함께 사용한다면 범용 표준인 AGENTS.md도 검토한다
