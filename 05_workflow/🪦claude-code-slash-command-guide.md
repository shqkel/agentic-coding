# Claude Code Slash Command 학습 가이드

> 작성일: 2026-03-12 | 난이도: 초급~중급 | 대상: Claude Code 사용자

---

## 개요

Slash Command(슬래시 커맨드)는 Claude Code에서 `/`로 시작하는 단축 명령어이다.
반복되는 프롬프트나 복잡한 워크플로우를 하나의 짧은 명령으로 실행할 수 있게 해준다.
이는 개발자의 집중력을 유지하면서 생산성을 크게 높여주는 핵심 기능이다.

---

## Phase 1 — 빌트인 명령어 (Built-in Commands)

Claude Code에 기본 내장된 명령어이다. 설치 즉시 사용할 수 있다.

### 핵심 세션 관리 명령어

| 명령어 | 설명 | 사용 시점 |
|--------|------|-----------|
| `/help` | 사용 가능한 모든 슬래시 커맨드 목록을 보여준다 | 처음 시작할 때, 커맨드가 기억나지 않을 때 |
| `/clear` | 대화 히스토리와 컨텍스트를 초기화한다 | 새 작업을 시작할 때, 컨텍스트가 오염됐을 때 |
| `/compact` | 대화를 요약하여 토큰을 절약한다 | 대화가 길어져 느려질 때 |
| `/exit` | Claude Code 세션을 종료한다 | 작업 완료 후 |

### 설정 및 환경 명령어

| 명령어 | 설명 | 사용 시점 |
|--------|------|-----------|
| `/config` | Claude Code 설정 인터페이스를 연다 | 설정을 변경할 때 |
| `/model` | 사용할 Claude 모델을 전환한다 | 복잡한 작업엔 Opus, 빠른 작업엔 Sonnet |
| `/context` | 현재 컨텍스트 사용량과 Skill 로드 상태를 확인한다 | 토큰 관리가 필요할 때 |
| `/doctor` | 설치 상태를 진단한다 | 문제가 발생했을 때 |

### 통합 및 자동화 명령어

| 명령어 | 설명 |
|--------|------|
| `/init` | 프로젝트에 CLAUDE.md를 자동 생성한다 |
| `/mcp` | MCP 서버를 추가/제거/설정한다 |
| `/hooks` | 파일 편집 후 자동 실행할 명령을 설정한다 |
| `/install-github-app` | GitHub PR 자동 리뷰를 설정한다 |

---

## Phase 2 — 커스텀 커맨드 (Custom Slash Commands)

### 2-1. 개념 이해

커스텀 커맨드의 작동 원리는 단순하다.

```
Markdown 파일 하나 = Slash Command 하나
```

파일명이 커맨드명이 된다. `commit.md` → `/commit`으로 호출된다.
파일 안의 내용이 Claude에게 전달되는 프롬프트이다.

### 2-2. 저장 위치와 범위

커스텀 커맨드는 저장 위치에 따라 적용 범위가 달라진다.

```
~/.claude/commands/         ← Personal Commands (전역 적용)
  └── commit.md             → 모든 프로젝트에서 /commit 사용 가능

프로젝트/.claude/commands/  ← Project Commands (해당 프로젝트만)
  └── review.md             → 이 프로젝트에서만 /review 사용 가능
```

**Personal Commands**: 자주 쓰는 개인 워크플로우에 적합하다.
**Project Commands**: 팀과 공유해야 하는 표준 프로세스에 적합하다. Git에 커밋되어 팀원 모두가 동일한 커맨드를 사용할 수 있다.

### 2-3. 기본 커맨드 만들기

```bash
# Personal 커맨드 디렉토리 생성
mkdir -p ~/.claude/commands

# 커맨드 파일 생성
touch ~/.claude/commands/commit.md
```

```markdown
<!-- 파일: ~/.claude/commands/commit.md -->

현재 staged된 변경사항을 분석하여 Conventional Commit 형식의 커밋 메시지를 작성해준다.

형식: <type>(<scope>): <subject>

type 종류: feat, fix, docs, style, refactor, test, chore
- 한국어로 subject를 작성한다
- 변경의 이유와 영향을 body에 간략히 설명한다
```

이제 Claude Code에서 `/commit`을 입력하면 이 프롬프트가 실행된다.

---

## Phase 3 — 고급 커맨드 문법

### 3-1. $ARGUMENTS — 동적 인자 전달

커맨드 실행 시 추가 인자를 받아 동적으로 동작하게 한다.

```markdown
<!-- 파일: .claude/commands/fix-issue.md -->

GitHub Issue #$ARGUMENTS를 분석하고 수정한다.

1. 이슈 내용을 파악한다
2. 관련 코드를 찾는다  
3. 수정사항을 적용한다
4. 테스트를 실행한다
```

```bash
# 사용법: 이슈 번호를 인자로 전달
/fix-issue 42
# → "$ARGUMENTS"가 "42"로 치환된다
```

### 3-2. 파일/명령어 참조

커맨드 내부에서 파일 내용이나 쉘 명령어 결과를 불러올 수 있다.

```markdown
<!-- 파일: .claude/commands/review.md -->

아래 변경사항을 코드 리뷰해준다:

## 변경된 파일 목록
!`git diff --name-only HEAD~1`

## 상세 변경사항
!`git diff HEAD~1`

## 리뷰 체크리스트
변경사항을 아래 기준으로 검토한다:
1. 코드 품질 및 가독성
2. 보안 취약점
3. 성능 영향
4. 테스트 커버리지
```

`!` + 백틱으로 감싼 명령어는 실행 결과가 프롬프트에 삽입된다.

### 3-3. Frontmatter — 커맨드 메타데이터

YAML 형식의 frontmatter로 커맨드 동작을 세밀하게 제어한다.

```markdown
<!-- 파일: .claude/commands/security-scan.md -->
---
description: 코드베이스 보안 취약점 스캔
allowed-tools: Read, Grep, Glob
model: claude-opus-4-6
disable-model-invocation: true
argument-hint: [파일경로]
---

$ARGUMENTS 파일에서 다음 보안 취약점을 분석한다:
- SQL Injection 위험
- XSS 취약점
- 노출된 자격증명(credentials)
- 안전하지 않은 설정값
```

**Frontmatter 주요 옵션:**

| 옵션 | 설명 |
|------|------|
| `description` | `/help`에 표시될 설명 |
| `allowed-tools` | 허용할 도구 목록 (Read, Write, Bash 등) |
| `model` | 이 커맨드에서 사용할 모델 |
| `disable-model-invocation: true` | 사용자만 수동 호출 가능 (Claude가 자동 실행 불가) |
| `argument-hint` | `/help`에 표시될 인자 힌트 |

> **🔒 `disable-model-invocation`을 꼭 써야 하는 경우:**
> `/deploy`, `/commit`, `/send-message` 처럼 **부작용이 있는 커맨드**는 반드시 이 옵션을 설정한다.
> Claude가 "코드가 완성된 것 같다"고 판단해 자동으로 배포를 실행하는 상황을 막아준다.

---

## Phase 4 — 실전 커맨드 레시피

### 레시피 1: 자동 커밋

```markdown
<!-- ~/.claude/commands/commit.md -->
---
description: Conventional Commit 메시지 자동 생성 및 커밋
allowed-tools: Bash
disable-model-invocation: true
---

현재 staged 변경사항을 확인한다:
!`git diff --cached`

위 변경사항을 분석하여 Conventional Commit 메시지를 생성하고 커밋을 실행한다.
```

### 레시피 2: 보안 리뷰

```markdown
<!-- .claude/commands/security-review.md -->
---
description: 보안 취약점 전체 스캔
allowed-tools: Read, Grep, Glob
---

프로젝트 전체를 스캔하여 다음 항목을 점검한다:
- Hardcoded secrets (API 키, 비밀번호)
- SQL Injection 패턴
- XSS 취약 코드
- 안전하지 않은 의존성

발견된 취약점을 심각도(Critical/High/Medium/Low)별로 분류하여 보고한다.
```

### 레시피 3: 테스트 자동 생성

```markdown
<!-- .claude/commands/test.md -->
---
description: 선택한 파일의 테스트 코드 자동 생성
allowed-tools: Read, Write, Bash
argument-hint: [파일경로]
---

$ARGUMENTS 파일을 분석하여:
1. 테스트 프레임워크를 자동 감지한다 (Jest, pytest 등)
2. 핵심 함수/메서드별 단위 테스트를 생성한다
3. 엣지 케이스를 포함한 테스트 케이스를 작성한다
4. 테스트를 실행하여 통과 여부를 확인한다
```

---

## Phase 5 — Skill과의 차이점

Slash Command와 Skill은 비슷해 보이지만 용도가 다르다.

| 구분 | Slash Command | Skill |
|------|---------------|-------|
| 호출 방식 | `/command-name` 직접 타이핑 | 자연어로 요청 시 Claude가 자동 판단 |
| 저장 위치 | `.claude/commands/` | `.claude/skills/` 또는 `~/.claude/skills/` |
| 적합한 용도 | 반복 워크플로우, 팀 표준 프로세스 | 배경 지식, 특정 작업 처리 방법 |
| 제어 주체 | 사용자가 명시적으로 실행 | Claude가 문맥에 따라 자동 실행 |
| 대표 예시 | `/commit`, `/deploy`, `/review` | pdf 처리 방법, docx 생성 방법 |

> **선택 기준**: 내가 "지금 이 작업을 실행해"라고 명령하고 싶으면 **Slash Command**,
> Claude가 "이런 상황에선 이렇게 해야 한다"는 것을 알게 하고 싶으면 **Skill**이다.

### 컨텍스트 로딩 방식의 차이

두 방식의 가장 중요한 내부 차이는 **언제 context에 올라가느냐**이다.

**Skill — 2단계 지연 로딩(Lazy Loading)**
```
세션 시작 → [name + description만 로드] → Claude가 관련성 판단 → [SKILL.md 전체 로드]
              ↑ 항상 (소량 토큰 소비)                               ↑ 필요할 때만
```

**Slash Command — 호출 시에만 로딩**
```
세션 시작 → [아무것도 로드 안 됨] → 사용자가 /commit 입력 → [commit.md 전체 로드]
```

| | Skill (자동 호출) | Skill (`disable-model-invocation`) | Slash Command |
|--|--|--|--|
| 세션 시작 시 | description 로드 (소량 소비) | 로드 안 됨 | 로드 안 됨 |
| 호출 시 | 전체 내용 로드 | 전체 내용 로드 | 전체 내용 로드 |
| 토큰 소비 패턴 | 상시 소량 + 호출 시 추가 | 호출 시에만 | 호출 시에만 |

`disable-model-invocation: true`를 설정한 Skill은 토큰 소비 측면에서 Slash Command와 동일하게 동작한다. 결국 부작용이 있는 커맨드에 이 옵션을 붙이면 **자동 실행 차단 + 불필요한 description 로딩 방지**라는 두 가지 효과를 동시에 얻는다.

---

## 자주 하는 실수

| 실수 | 원인 | 해결 |
|------|------|------|
| 커맨드가 인식 안 됨 | 파일이 잘못된 위치에 저장됨 | `.claude/commands/` 경로를 다시 확인한다 |
| 인자가 전달 안 됨 | `$ARGUMENTS` 오타 | 대문자로 정확히 `$ARGUMENTS` 작성 |
| 커맨드가 너무 많이 자동 실행됨 | `disable-model-invocation` 미설정 | 부작용 있는 커맨드에 반드시 추가 |
| Skill 목록 초과 경고 | Skills가 너무 많아 컨텍스트 예산 초과 | `/context`로 확인 후 불필요한 Skill 정리 |

---

## 핵심 정리

1. **빌트인 커맨드**는 세션 관리에 사용하고, `/help`로 전체 목록을 확인한다.
2. **커스텀 커맨드**는 `.md` 파일 하나로 만들며, 파일명이 커맨드명이 된다.
3. **저장 위치**가 적용 범위를 결정한다: `~/.claude/commands/`(전역) vs `.claude/commands/`(프로젝트).
4. **`$ARGUMENTS`**로 동적 인자를 받고, `!`(느낌표)로 쉘 명령어 결과를 삽입한다.
5. **`disable-model-invocation: true`**는 배포/커밋 같은 부작용 커맨드에 필수이다.

---

## 관련 개념

- `CLAUDE.md` — 프로젝트 컨텍스트 파일 (모든 세션에 자동 로드)
- `Hooks` — 도구 실행 전후에 자동으로 실행되는 쉘 명령어
- `MCP (Model Context Protocol)` — 외부 서비스와 Claude를 연결하는 프로토콜
- `Skills` — 자연어로 호출되는 Claude의 행동 지침

---

## 복습 리마인더 (에빙하우스 망각 곡선)

```
오늘 → 1일 후 → 3일 후 → 7일 후 → 14일 후 → 30일 후
```

**자가 점검 질문:**
- [ ] Personal Commands와 Project Commands의 차이를 설명할 수 있는가?
- [ ] `$ARGUMENTS`를 활용하는 커맨드를 직접 만들 수 있는가?
- [ ] `disable-model-invocation`이 필요한 상황을 예시로 들 수 있는가?
- [ ] Slash Command와 Skill의 차이를 비교 설명할 수 있는가?