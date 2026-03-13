# Google Antigravity — Hooks 부재와 대체 메커니즘 가이드

## 학습 목표

Google Antigravity에는 Claude Code 방식의 이벤트 기반 Hooks 시스템이 **존재하지 않는다**는 것을 이해하고, Antigravity가 제공하는 대체 메커니즘(Denylist/Allowlist, Workflows, Rules, Skills)을 활용하여 유사한 자동화를 구성할 수 있다.

> **⚠️ 명확한 전제**: Antigravity는 `PreToolUse`, `PostToolUse`와 같은 프로그래밍 가능한 이벤트 훅을 지원하지 않는다. 이 가이드는 그 사실을 전제로, 가능한 대안을 설명한다.

## 사전 준비

- Google Antigravity 계정 및 접근 권한 (2026년 3월 기준: 공개 프리뷰)
- Claude Code 또는 Gemini CLI hooks 개념 이해 (비교 기준)
- Markdown 기초 지식 (Rules/Workflows는 Markdown으로 작성)

> **현황 주의**: Antigravity는 2025년 11월 공개 프리뷰로 출시되었으며, 2026년 3월 기준으로 여전히 프리뷰 단계다. 대용량 사용 시 계정 제한이 발생한 사례가 있다.

---

## 전체 흐름 한눈에 보기

Antigravity는 hooks 대신 세 가지 정적 메커니즘으로 에이전트 동작을 제어한다: **Rules**(항상 적용되는 지침), **Workflows**(사용자가 트리거하는 순서 작업), **Skills**(재사용 가능한 도구 패키지). 터미널 명령 실행은 **Allowlist/Denylist**로 별도 관리된다. 이 메커니즘들은 이벤트 기반이 아니라 프롬프트 주입 기반이다.

1. **개념 이해** — Hooks 없음, 어떻게 제어하는가?
2. **Allowlist/Denylist** — 터미널 명령 실행 정책
3. **Rules** — 항상 적용되는 지침
4. **Workflows와 Skills** — 구조화된 작업 자동화

---

## Phase 1: Antigravity의 에이전트 제어 방식

### 목표

왜 Antigravity에 hooks가 없는지 이해하고, 대체 메커니즘의 전체 구조를 파악한다.

### 단계별 구현

#### Step 1.1 — hooks가 없는 이유

> **💡 개념 설명: Antigravity의 설계 철학**
>
> Antigravity는 "LLM이 올바른 판단을 한다"는 전제로 설계되었다. 이벤트 훅 같은 하드웨어적 제어보다 Rules/Workflows를 통한 소프트 가이드라인을 선호한다. 이 접근법은 유연하지만, 보안 연구자들이 "LLM에 과도하게 의존한다"고 지적한 부분이기도 하다.
>
> **핵심 한 줄:** Antigravity = 프롬프트 기반 소프트 제어 (이벤트 기반 하드 제어 없음)

#### Step 1.2 — Claude Code hooks와의 기능 대응

| Claude Code | Antigravity 대응 | 비고 |
|-------------|-----------------|------|
| `PreToolUse` (차단) | Denylist | 터미널 명령만 해당, MCP 도구 차단 불가 |
| `PostToolUse` (사후 처리) | **없음** | 파일 수정 후 자동 포매팅 불가 |
| `Stop` (세션 종료) | **없음** | 세션 종료 자동화 불가 |
| `Notification` (알림) | **없음** | 에이전트 알림 기능 없음 |
| Rules (CLAUDE.md) | `.agent/rules/` + GEMINI.md | 기능 유사 |
| Skills (커스텀 명령) | `.agent/workflows/` | 방식이 다름 (슬래시 명령) |
| http 훅 | **없음** | 외부 Webhook 트리거 불가 |

#### Step 1.3 — 디렉토리 구조

```
.agent/
├── rules/          ← 항상 적용되는 지침 (Rules)
│   ├── coding-standards.md
│   └── security-policy.md
├── workflows/      ← 슬래시 명령으로 실행 (/deploy, /review)
│   ├── deploy.md
│   └── code-review.md
└── skills/         ← 재사용 가능한 도구 패키지
    └── my-tool/
        ├── SKILL.md
        └── scripts/

전역 설정:
~/.gemini/GEMINI.md                      ← 전역 Rules
~/.gemini/antigravity/global_workflows/  ← 전역 Workflows
~/.gemini/antigravity/skills/            ← 전역 Skills
```

### 체크포인트

Antigravity에 hooks가 없으며, Rules/Workflows/Skills/Denylist가 부분적인 대체 수단임을 설명할 수 있는가?

---

## Phase 2: Allowlist/Denylist — 터미널 명령 실행 정책

### 목표

Antigravity의 4가지 실행 모드와 Allowlist/Denylist를 구성하여 터미널 명령을 제어한다.

### 단계별 구현

#### Step 2.1 — 4가지 실행 모드

> **💡 개념 설명: 실행 모드**
>
> Antigravity는 터미널 명령 자동 실행 여부를 4가지 모드로 제어한다. Claude Code의 `PreToolUse` hooks처럼 개별 명령을 검사하는 것이 아니라, 전체 실행 정책을 전환하는 방식이다.
>
> **핵심 한 줄:** 실행 모드 = 에이전트가 터미널을 얼마나 자율적으로 사용할 수 있는지 결정

| 모드 | 동작 | 사용 시점 |
|------|------|-----------|
| **Off** | Allowlist에 있는 명령만 실행 | 가장 안전한 환경 |
| **Auto** | LLM이 판단하여 확인 요청 | 일반 사용 권장 |
| **Turbo** | Denylist에 없는 모든 명령 자동 실행 | 신뢰된 환경 |
| **Custom** | 세밀한 수동 구성 | 고급 사용자 |

#### Step 2.2 — Denylist 설정 (Turbo 모드)

```
설정 경로: Antigravity 설정 → Terminal Execution → Turbo mode → Deny List
```

Denylist에 추가할 일반적인 위험 명령:

```
rm
sudo
curl
wget
chmod
chown
dd
mkfs
```

> **⚠️ 보안 한계**: Denylist는 터미널 명령에만 적용된다. MCP 도구나 내장 파일 도구에는 적용되지 않는다. 보안 연구자들은 MCP 도구를 통한 프롬프트 인젝션이 이 제한을 우회할 수 있다고 지적한다.

#### Step 2.3 — Allowlist 설정 (Off 모드)

"Off" 모드에서 허용할 명령을 Allowlist에 추가한다:

```
git status
git diff
git log
npm test
npm run lint
npx prettier --check
```

브라우저 URL Allowlist는 별도 파일로 관리된다:

```
# ~/.gemini/antigravity/browserAllowlist.txt

https://docs.anthropic.com
https://developer.mozilla.org
https://github.com
```

### 체크포인트

Turbo 모드에서 `rm` 명령이 차단되고, Off 모드에서 Allowlist에 없는 명령이 실행되지 않으면 성공이다.

---

## Phase 3: Rules — 항상 적용되는 지침

### 목표

Claude Code의 CLAUDE.md에 대응하는 Antigravity Rules를 작성하여 에이전트 행동을 일관되게 제어한다.

### 단계별 구현

#### Step 3.1 — Rules란?

> **💡 개념 설명: Rules**
>
> Rules는 모든 에이전트 응답에 시스템 프롬프트로 자동 주입되는 지침이다. Claude Code의 CLAUDE.md와 기능적으로 동일하다. PostToolUse hooks처럼 이벤트 발생 시 실행되는 것이 아니라, **항상** 에이전트의 컨텍스트에 포함된다.
>
> **핵심 한 줄:** Rules = 에이전트에게 항상 주입되는 시스템 프롬프트

```markdown
<!-- .agent/rules/coding-standards.md -->

# 코딩 표준

## 언어
- 모든 응답과 주석은 한국어로 작성한다

## 코드 품질
- 파일 수정 후 반드시 린트 오류를 확인한다
- 새 파일 생성 시 단위 테스트도 함께 생성한다

## 보안
- 시크릿 키나 비밀번호를 코드에 하드코딩하지 않는다
- 환경 변수나 설정 파일을 사용한다
```

> **한계**: Rules는 에이전트에게 "해야 한다"고 요청하는 것이지, 프로그래밍적으로 강제하는 것이 아니다. Claude Code의 PostToolUse hooks처럼 파일 수정 후 자동으로 린터를 실행하는 것은 불가능하다.

#### Step 3.2 — 스코프 계층

```
전역 (항상 적용):     ~/.gemini/GEMINI.md
워크스페이스 (프로젝트): .agent/rules/*.md
컴포넌트 (서브디렉토리): <subdir>/.agent/rules/*.md
```

### 체크포인트

에이전트가 Rules에 정의된 지침을 응답에 반영하는지 확인할 수 있는가?

---

## Phase 4: Workflows와 Skills — 구조화된 자동화

### 목표

Workflows와 Skills로 Claude Code hooks에 없는 Antigravity 고유의 자동화 패턴을 구현한다.

### 단계별 구현

#### Step 4.1 — Workflows: 슬래시 명령으로 실행하는 절차

> **💡 개념 설명: Workflows**
>
> Workflows는 사용자가 `/deploy`, `/review` 같은 슬래시 명령으로 직접 실행하는 절차 문서다. Claude Code hooks와 달리 자동으로 실행되지 않고, 사용자가 명시적으로 트리거해야 한다. Markdown 파일에 단계를 서술하면 에이전트가 그 절차를 따른다.
>
> **핵심 한 줄:** Workflows = 사용자 트리거 방식의 자동화 (이벤트 트리거 방식이 아님)

```markdown
<!-- .agent/workflows/code-review.md -->

---
name: code-review
description: 현재 브랜치의 변경사항을 코드 리뷰한다
---

# 코드 리뷰 워크플로우

## 단계

1. `git diff main...HEAD` 로 변경사항을 확인한다
2. 각 파일에 대해:
   - 코딩 표준 준수 여부 확인
   - 잠재적 버그 식별
   - 보안 이슈 검토
3. 심각도별 이슈 목록을 작성한다 (Critical/High/Medium/Low)
4. 개선 제안을 포함한 리뷰 보고서를 마크다운으로 작성한다
```

사용 방법:
```
/code-review
```

#### Step 4.2 — Workflows의 `// turbo` 애노테이션

```markdown
<!-- .agent/workflows/format.md -->

---
name: format
description: 프로젝트 전체 코드를 포매팅한다
---

# 포매팅 워크플로우

1. JS/TS 파일 포매팅: // turbo
   ```
   npx prettier --write "**/*.{js,ts,jsx,tsx}"
   ```

2. Python 파일 포매팅: // turbo
   ```
   black .
   ```

3. 포매팅 결과 확인
```

`// turbo` 애노테이션이 붙은 단계는 사용자 확인 없이 자동 실행된다.

> **참고**: Claude Code hooks처럼 파일 저장 시 자동으로 포매팅이 실행되지는 않는다. `/format` 슬래시 명령을 직접 실행해야 한다.

#### Step 4.3 — Skills: ADK 기반 커스텀 도구

Skills는 에이전트가 사용할 수 있는 커스텀 도구를 패키징하는 구조다. Google ADK(Agent Development Kit)의 `BaseTool`을 구현하여 복잡한 동작을 캡슐화할 수 있다.

```markdown
<!-- .agent/skills/lint-checker/SKILL.md -->

---
name: lint-checker
description: 현재 파일의 린트 오류를 검사하고 자동 수정을 제안한다
---

# Lint Checker Skill

## 사용 방법

파일 경로를 지정하여 린트 검사를 실행한다:
```
lint-checker check <file-path>
```

## 동작

1. 파일 확장자에 따라 적절한 린터 선택
2. 린트 오류 목록 반환
3. 자동 수정 가능한 항목 식별
```

### 체크포인트

`/code-review` 워크플로우가 실행되어 변경사항 분석 보고서가 생성되면 성공이다.

---

## 마무리

### Antigravity vs Claude Code: 제어 방식 비교

| 목표 | Claude Code | Antigravity |
|------|-------------|-------------|
| 파일 수정 후 자동 포매팅 | PostToolUse hooks | **불가** (Workflows로 수동 실행) |
| 위험 명령 자동 차단 | PreToolUse hooks | Denylist (터미널만, MCP 제외) |
| 항상 지켜야 할 규칙 | CLAUDE.md | `.agent/rules/` |
| 반복 작업 자동화 | skills/workflows | Workflows (수동 트리거) |
| 세션 종료 후 보고서 | Stop hooks | **불가** |
| LLM 응답에 컨텍스트 주입 | prompt 타입 hooks | Rules (항상 주입) |
| MCP 도구 차단 | PreToolUse hooks | **불가** |

### Antigravity의 보안 한계

보안 연구자들이 지적한 주요 한계:

1. **MCP 도구 무통제**: Denylist는 터미널 명령에만 적용, MCP 도구 호출에는 Human-in-the-loop 없음
2. **LLM 과의존**: 보안 결정을 LLM 판단에 의존 → 프롬프트 인젝션으로 우회 가능
3. **하드 차단 없음**: PreToolUse hooks 같은 프로그래밍적 강제 수단이 없어 에이전트가 Rules를 "무시"할 수 있음

### 권장 사용 패턴

| 시나리오 | 권장 설정 |
|----------|-----------|
| 탐색/리뷰 작업 | Off 모드 + 최소 Allowlist |
| 일반 개발 작업 | Auto 모드 |
| 자동화 파이프라인 | Turbo 모드 + 엄격한 Denylist |
| 보안 민감 프로젝트 | Off 모드 + Rules 강화 + MCP 도구 최소화 |
