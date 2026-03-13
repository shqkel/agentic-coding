# Gemini CLI 커스텀 슬래시 명령어 가이드

## 학습 목표

자주 반복하는 복잡한 프롬프트를 `.toml` 형식의 커스텀 슬래시 명령어로 만들어 Gemini CLI에서 `/명령어`로 즉시 재사용할 수 있다.

> **다른 도구와의 대응**: Antigravity의 **Workflows**, Claude Code의 **Skills**에 해당하는 Gemini CLI 기능이다.

## 사전 준비

- Gemini CLI 기본 사용법 숙지 (`01_agentic_coding_tool/01_cli_driven/gemini-cli-guide.md` 참고)
- 텍스트 에디터

---

## 전체 흐름 한눈에 보기

매번 긴 프롬프트를 타이핑하거나 복사-붙여넣기 하는 대신, 한 번 `.toml` 파일로 저장해두면 `/명령어`로 즉시 실행할 수 있다. 인자 전달, 셸 명령 실행, 파일 내용 주입 등 고급 기능을 조합하면 복잡한 작업도 한 줄 명령으로 처리된다.

1. **개념 이해** — 슬래시 명령어가 해결하는 문제
2. **첫 명령어 만들기** — 기본 .toml 구조
3. **고급 기능** — 인자, 셸 명령, 파일 주입
4. **네임스페이스와 관리** — 명령어 체계적으로 구성하기

---

## Phase 1: 커스텀 슬래시 명령어란?

### 목표

슬래시 명령어가 해결하는 문제를 이해하고 세 가지 핵심 기능을 파악한다.

### 단계별 구현

#### Step 1.1 — 해결하는 문제 파악하기

> **💡 개념 설명: 왜 슬래시 명령어인가?**
>
> PR을 리뷰할 때마다 "GitHub에서 PR #번호를 불러와서, 코드 품질·보안·성능 관점으로 검토하고, 각 파일별 피드백을 한국어로 작성해줘"라는 긴 프롬프트를 반복 입력하게 된다. 슬래시 명령어는 이런 반복 프롬프트를 재사용 가능한 단위로 저장한다.
>
> **핵심 한 줄:** 슬래시 명령어 = 자주 쓰는 프롬프트의 단축키

| 기능 | 문법 | 설명 |
|------|------|------|
| 인자 치환 | `{{args}}` | 명령어 호출 시 입력한 텍스트 삽입 |
| 셸 명령 실행 | `!{셸 명령}` | 명령 실행 결과를 프롬프트에 주입 |
| 파일 내용 주입 | `@{파일 경로}` | 파일 내용을 프롬프트에 삽입 |

#### Step 1.2 — 명령어 파일 위치

```
전역 명령어:    ~/.gemini/commands/
프로젝트 명령어: <project-root>/.gemini/commands/
```

우선순위: `프로젝트 > 전역 > 확장`

### 체크포인트

슬래시 명령어가 저장되는 위치와 세 가지 템플릿 기능(`{{args}}`, `!{...}`, `@{...}`)을 설명할 수 있는가?

---

## Phase 2: 첫 번째 슬래시 명령어 만들기

### 목표

커밋 메시지 생성 명령어를 만들고 `/commit`으로 실행되는지 확인한다.

### 단계별 구현

#### Step 2.1 — commands 디렉토리 생성

```bash
# 전역 명령어 디렉토리 생성
mkdir -p ~/.gemini/commands
```

#### Step 2.2 — 기본 .toml 파일 작성

> **💡 개념 설명: TOML 형식**
>
> TOML(Tom's Obvious, Minimal Language)은 설정 파일을 위한 간단한 형식이다. `key = "value"` 형태로 작성하며, 여러 줄 문자열은 `"""..."""`로 감싼다.
>
> **핵심 한 줄:** TOML = JSON보다 읽기 쉬운 설정 파일 형식

```toml
# ~/.gemini/commands/commit.toml

description = "현재 스테이징된 변경사항으로 커밋 메시지를 작성한다"

prompt = """
다음은 현재 스테이징된 Git 변경사항이다:

!{git diff --staged}

위 변경사항을 바탕으로 Conventional Commits 형식의 커밋 메시지를 작성해줘.
- 형식: type(scope): 한국어 제목
- type: feat / fix / docs / refactor / test / chore
- 제목은 50자 이내
- 필요 시 본문(body)도 작성
"""
```

#### Step 2.3 — 명령어 확인 및 실행

Gemini CLI에서 명령어 목록을 확인한다:

```bash
/commands list
```

실행:

```bash
> /commit
```

인자 없이 호출하면 현재 `git diff --staged` 결과를 가져와 커밋 메시지를 자동 생성한다.

### 체크포인트

`/commit` 실행 시 스테이징된 변경사항을 분석하여 커밋 메시지가 생성되면 성공이다.

---

## Phase 3: 고급 기능 — 인자, 셸 명령, 파일 주입

### 목표

`{{args}}`, `!{...}`, `@{...}` 세 가지 템플릿 기능을 활용하여 더 유연한 명령어를 만든다.

### 단계별 구현

#### Step 3.1 — 인자 치환: `{{args}}`

사용자가 명령어와 함께 입력한 텍스트를 프롬프트에 삽입한다:

```toml
# ~/.gemini/commands/explain.toml

description = "특정 개념이나 코드를 초보자도 이해할 수 있게 설명한다"

prompt = """
다음에 대해 초보자도 이해할 수 있도록 쉽게 설명해줘.
비유와 예시를 반드시 포함할 것.

설명 대상: {{args}}
"""
```

사용:

```bash
> /explain async/await
> /explain 클로저
> /explain src/auth/jwt.py 파일의 decode_token 함수
```

#### Step 3.2 — 셸 명령 실행: `!{...}`

프롬프트 생성 시 셸 명령을 실행하고 그 결과를 삽입한다:

```toml
# ~/.gemini/commands/review.toml

description = "GitHub PR을 불러와 코드 리뷰를 수행한다"

prompt = """
다음 GitHub Pull Request를 리뷰해줘:

PR 정보:
!{gh pr view {{args}} --json title,body,files,additions,deletions}

변경된 코드:
!{gh pr diff {{args}}}

코드 품질, 보안, 성능 관점에서 검토하고
각 파일별로 구체적인 피드백을 한국어로 작성해줘.
"""
```

사용:

```bash
> /review 42
```

> **💡 개념 설명: `!{...}` 실행 타이밍**
>
> `!{...}` 안의 셸 명령은 프롬프트가 Gemini에게 전송되기 **전에** 실행된다. 실행 결과가 프롬프트 텍스트에 삽입된 후 전송된다. `gh`, `git`, `curl` 등 모든 CLI 도구를 사용할 수 있다.
>
> **핵심 한 줄:** `!{명령}` = 실시간 데이터를 프롬프트에 주입하는 방법

#### Step 3.3 — 파일 내용 주입: `@{...}`

특정 파일의 내용을 프롬프트에 삽입한다:

```toml
# ~/.gemini/commands/refactor.toml

description = "지정한 파일을 리팩토링한다"

prompt = """
다음 파일을 리팩토링해줘:

@{{{args}}}

다음 원칙을 따를 것:
- 함수는 하나의 역할만 수행
- 변수명은 의도가 명확하게
- 중복 코드 제거
- 타입 안정성 강화

리팩토링 전후 비교와 변경 이유를 함께 설명해줘.
"""
```

사용:

```bash
> /refactor src/utils/auth.py
```

#### Step 3.4 — 세 가지 기능 조합

실제 개발에서 자주 쓰이는 스탠드업 자동화 예시이다:

```toml
# ~/.gemini/commands/standup.toml

description = "오늘의 스탠드업 미팅 내용을 자동으로 작성한다"

prompt = """
오늘의 스탠드업 미팅 보고서를 작성해줘.

오늘 날짜: !{date +"%Y-%m-%d"}

오늘 내가 커밋한 내용:
!{git log --since="yesterday" --author="$(git config user.email)" --oneline}

현재 브랜치 상태:
!{git status --short}

프로젝트 README:
@{README.md}

위 내용을 바탕으로 다음 형식으로 보고서를 작성해줘:
- 어제 한 일
- 오늘 할 일
- 블로커 (있다면)
"""
```

### 체크포인트

`/review 42` 실행 시 GitHub에서 PR 데이터를 가져와 리뷰가 생성되면 성공이다.

---

## Phase 4: 네임스페이스와 명령어 관리

### 목표

관련 명령어를 그룹으로 묶어 네임스페이스를 구성하고, 명령어를 효율적으로 관리한다.

### 단계별 구현

#### Step 4.1 — 디렉토리로 네임스페이스 구성

디렉토리를 만들면 자동으로 네임스페이스가 된다:

```bash
mkdir -p ~/.gemini/commands/git
mkdir -p ~/.gemini/commands/docs
mkdir -p ~/.gemini/commands/test
```

```
~/.gemini/commands/
├── commit.toml          # /commit
├── git/
│   ├── summary.toml     # /git:summary
│   └── cleanup.toml     # /git:cleanup
├── docs/
│   └── api.toml         # /docs:api
└── test/
    └── coverage.toml    # /test:coverage
```

#### Step 4.2 — 네임스페이스 명령어 예시

```toml
# ~/.gemini/commands/git/summary.toml

description = "이번 주 Git 커밋 내용을 요약한다"

prompt = """
이번 주 Git 커밋을 분석해줘:

!{git log --since="1 week ago" --oneline --graph}

팀원별 기여도와 주요 변경사항을 요약 보고서로 작성해줘.
"""
```

사용:

```bash
> /git:summary
```

#### Step 4.3 — 명령어 관리 명령어

```bash
/commands list      # 모든 슬래시 명령어 목록 확인
/commands reload    # 파일 수정 후 명령어 다시 로드
```

#### Step 4.4 — 이미지/PDF 첨부 활용

`@{...}`는 텍스트 파일뿐만 아니라 이미지와 PDF도 지원한다:

```toml
# ~/.gemini/commands/design-review.toml

description = "디자인 목업 파일을 분석하여 구현 방안을 제시한다"

prompt = """
다음 디자인 파일을 분석해줘:

@{{{args}}}

이 디자인을 HTML/CSS로 구현하기 위한:
1. 컴포넌트 구조 제안
2. 사용할 CSS 기법 (Flexbox/Grid 등)
3. 예상 구현 난이도와 주의사항

을 정리해줘.
"""
```

사용:

```bash
> /design-review designs/landing-page-v2.png
```

### 체크포인트

`/commands list`에서 네임스페이스 그룹과 명령어 목록이 정상 표시되면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- 커스텀 슬래시 명령어 = 반복 프롬프트의 재사용 단위
- 세 가지 템플릿 기능: `{{args}}` (인자), `!{...}` (셸 실행), `@{...}` (파일 주입)
- 디렉토리 구조로 네임스페이스를 자동 구성
- 전역과 프로젝트 범위의 명령어 분리 관리

### 실용적인 명령어 아이디어

| 명령어 | 설명 |
|--------|------|
| `/standup` | 오늘의 커밋 기반 스탠드업 보고 자동 생성 |
| `/review [PR번호]` | GitHub PR 불러와 코드 리뷰 |
| `/refactor [파일명]` | 파일 리팩토링 |
| `/docs:api` | 현재 코드 기반 API 문서 생성 |
| `/test:coverage` | 테스트 커버리지 분석 및 부족한 케이스 추천 |
| `/git:summary` | 주간 커밋 요약 보고서 |

### 자주 발생하는 문제

| 증상 | 해결 |
|------|------|
| `!{git diff --staged}` 가 비어 있음 | `git add` 먼저 실행 |
| `@{파일명}` 오류 | 현재 작업 디렉토리 기준 상대경로 확인 |
| 명령어가 목록에 안 보임 | `/commands reload` 실행 |

### 다음 단계

- MCP 서버와 슬래시 명령어를 조합하여 더 강력한 자동화 구성
- 팀 전체가 공유하는 프로젝트 명령어를 `.gemini/commands/`에 커밋
