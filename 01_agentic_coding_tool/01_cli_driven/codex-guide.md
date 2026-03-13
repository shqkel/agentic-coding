# Codex CLI 학습 가이드 (GPT-5.4)

> OpenAI Codex CLI를 처음 쓰는 개발자를 위한 흐름 중심 가이드  
> 최신 모델 **GPT-5.4** 기준 (2025년 3월)

---

## 학습 목표

이 가이드를 마치면 Codex CLI를 설치하고 GPT-5.4 모델로 실제 프로젝트에 자연어 명령을 내려 코드를 작성·수정·리뷰하고, `config.toml`로 환경을 커스터마이징하며, `codex exec`으로 반복 작업을 자동화할 수 있다.

---

## 사전 준비

| 항목 | 요구 사항 |
|---|---|
| OS | macOS / Linux (Windows는 WSL 권장) |
| Node.js | v22.22.0 이상 |
| 계정 | ChatGPT Plus/Pro/Business/Edu/Enterprise 또는 OpenAI API Key |
| Git | 설치 권장 (undo/rollback 활용) |

```bash
# Node.js 버전 확인
node -v   # v22.22.0 이상이어야 함
```

---

## 전체 흐름 한눈에 보기

Codex CLI는 터미널에서 자연어로 명령을 내리면 GPT-5.4가 파일을 읽고·수정·실행까지 해주는 코딩 에이전트이다. 먼저 설치와 인증을 마치고, config.toml로 GPT-5.4를 기본 모델로 설정한다. 그다음 승인 모드(Approval Mode)와 샌드박스 개념을 이해하고, 다양한 슬래시 명령과 실전 패턴을 익히면 된다.

**전체 단계:**
1. Phase 1 — 설치 및 인증
2. Phase 2 — GPT-5.4 설정 (config.toml)
3. Phase 3 — 기본 대화 & 슬래시 명령
4. Phase 4 — 승인 모드 & 샌드박스
5. Phase 5 — 실전 패턴 & 자동화

---

## Phase 1: 설치 및 인증

### 목표
Codex CLI가 터미널에서 실행되고, ChatGPT 계정 또는 API Key로 인증이 완료된 상태가 된다.

### 단계별 구현

#### Step 1.1 — Codex CLI 설치하기

두 가지 방법 중 하나를 선택한다.

```bash
# ~/.codex/config.toml

# 방법 A: npm으로 전역 설치 (권장)
npm install -g @openai/codex

# 방법 B: Homebrew (macOS)
brew install --cask codex
```

설치 후 버전을 확인한다.

```bash
# 터미널

codex --version
```

> **💡 개념 설명: Codex CLI란?**
> 
> 터미널에서 "이 함수에 버그 있는 것 같으니 고쳐줘"라고 자연어로 입력하면, Codex CLI가 GPT-5.4를 호출해 코드를 실제로 수정해준다. 단순한 코드 제안(GitHub Copilot)과 달리, **파일을 직접 읽고 편집하며 명령까지 실행**하는 에이전트(agent)이다.
> 
> 핵심 요약: Codex CLI = 터미널 안의 자율 코딩 에이전트

#### Step 1.2 — 첫 실행 및 로그인

```bash
# 터미널

codex
```

처음 실행하면 인증 화면이 나타난다.

- **ChatGPT 계정으로 로그인** (Plus/Pro/Business 등): 브라우저가 열리며 OAuth 인증이 진행된다. API 키 없이 구독 크레딧을 그대로 사용한다.
- **API Key 사용**: OpenAI Platform에서 발급한 키를 붙여넣는다.

```bash
# 환경 변수로 API Key를 설정하는 방법 (선택)
# ~/.bashrc 또는 ~/.zshrc

export OPENAI_API_KEY="sk-..."
```

### 체크포인트 ✅

아래 명령 후 TUI(터미널 UI)가 열리고 `>` 프롬프트가 보이면 성공이다.

```bash
# 터미널

codex
```

---

## Phase 2: GPT-5.4 모델 설정 (config.toml)

### 목표
`~/.codex/config.toml`에 GPT-5.4를 기본 모델로 설정하고, 합리적인 추론 수준과 웹 검색 옵션을 적용한다.

### 단계별 구현

#### Step 2.1 — config.toml 생성 및 기본 설정

> **💡 개념 설명: config.toml이란?**
> 
> Codex CLI의 모든 동작 방식을 제어하는 설정 파일이다. 위치는 `~/.codex/config.toml`이며, 이 파일 하나로 사용 모델, 승인 정책, 샌드박스 수준을 결정할 수 있다. 특정 프로젝트에만 적용하고 싶다면 프로젝트 루트에 `.codex/config.toml`을 추가로 만들면 된다.
> 
> 우선순위: **CLI 플래그 > 프로젝트 config > 사용자 config**

```toml
# ~/.codex/config.toml

# ─── 모델 설정 ───────────────────────────────────────────────
model = "gpt-5.4"           # 대부분의 작업에 권장되는 최신 플래그십 모델
model_provider = "openai"

# ─── 추론(Reasoning) 수준 ────────────────────────────────────
# minimal | low | medium | high | xhigh
# 일반 작업은 medium, 복잡한 리팩토링·마이그레이션은 high 권장
model_reasoning_effort = "medium"

# ─── 승인 정책 ───────────────────────────────────────────────
# on-request(기본) | never | untrusted
approval_policy = "on-request"

# ─── 웹 검색 ─────────────────────────────────────────────────
# cached(기본, 안전) | live(최신 정보) | disabled
web_search = "cached"
```

> **💡 개념 설명: GPT-5.4란?**
>
> GPT-5.4는 OpenAI의 최신 플래그십 모델로, **GPT-5.3-Codex의 코딩 능력 + GPT-5의 범용 추론·도구 사용**을 결합한 모델이다. 일반 작업에서는 gpt-5.4가 기본 추천 모델이며, 극한의 코딩 작업에는 gpt-5.3-codex가, 초고속 반복 작업에는 gpt-5.3-codex-spark(ChatGPT Pro 한정)를 쓸 수 있다.
>
> 핵심 요약: GPT-5.4 = 코딩 + 추론 + 컴퓨터 활용을 하나의 모델에서

#### Step 2.2 — 프로필(Profile)로 상황별 설정 분리하기

무거운 리뷰가 필요할 때와 가벼운 질문을 할 때 모델을 수동으로 바꾸는 건 번거롭다. 프로필을 정의하면 `--profile` 하나로 전환된다.

```toml
# ~/.codex/config.toml (이어서 추가)

# ─── 프로필 정의 ──────────────────────────────────────────────
[profiles.deep-review]
model = "gpt-5.4"
model_reasoning_effort = "high"
approval_policy = "never"

[profiles.lightweight]
model = "gpt-5.4"
model_reasoning_effort = "low"
approval_policy = "on-request"
```

사용법:

```bash
# 터미널

# 심층 리뷰 모드로 실행
codex --profile deep-review

# 가벼운 질문 모드로 실행
codex --profile lightweight
```

### 체크포인트 ✅

TUI를 열었을 때 하단 상태 바에 `gpt-5.4`가 표시되면 정상이다.

```bash
# 터미널

codex
# 하단에 "gpt-5.4 | medium | ..." 표시 확인
```

---

## Phase 3: 기본 대화 & 슬래시 명령

### 목표
TUI 안에서 자연어로 코딩 작업을 지시하고, 주요 슬래시 명령을 사용해 세션을 제어한다.

### 단계별 구현

#### Step 3.1 — 첫 번째 프롬프트 보내기

```bash
# 터미널

codex "현재 디렉토리에 있는 Python 파일의 함수 목록을 알려줘"
```

> **💡 개념 설명: Codex CLI의 TUI 입력 방법**
>
> - **일반 텍스트 입력**: 자연어로 작업을 지시한다.
> - **`@` + 파일명**: 퍼지 검색으로 특정 파일을 참조한다. (예: `@src/main.py 이 파일 리팩토링해줘`)
> - **`!` + 명령**: 로컬 셸 명령을 실행하고 결과를 Codex에 전달한다. (예: `!npm test`)
> - **이미지 첨부**: 스크린샷, 디자인 목업을 붙여넣어 UI 구현을 요청할 수 있다.
> - **Enter (실행 중)**: 현재 진행 중인 턴에 추가 지시를 주입한다.
> - **Tab (실행 중)**: 다음 턴을 위한 프롬프트를 미리 대기열에 넣는다.

#### Step 3.2 — 주요 슬래시(/) 명령 익히기

| 슬래시 명령 | 기능 |
|---|---|
| `/model` | 모델 전환 (gpt-5.4 ↔ gpt-5.3-codex 등) |
| `/model gpt-5.4 high` | 모델 + 추론 레벨 즉시 지정 |
| `/permissions` | 승인 모드 변경 |
| `/review` | 변경사항을 별도 Codex 에이전트가 코드 리뷰 |
| `/clear` | 대화 기록 초기화 (새 채팅 시작) |
| `/copy` | 마지막 출력 클립보드 복사 |
| `/status` | 현재 모델·승인 모드·샌드박스 확인 |
| `/theme` | 색상 테마 미리보기 및 저장 |
| `/exit` | TUI 종료 |

#### Step 3.3 — 세션 이어받기 (resume)

Codex는 세션 기록을 로컬에 저장한다. 이전 작업을 이어서 하려면:

```bash
# 터미널

# 가장 최근 세션 재개
codex resume

# 특정 세션 ID로 재개
codex resume <session-id>
```

### 체크포인트 ✅

TUI에서 `/model`을 입력했을 때 모델 목록이 나타나고 선택할 수 있으면 정상이다.

---

## Phase 4: 승인 모드 & 샌드박스

### 목표
세 가지 승인 모드의 차이를 이해하고, 상황에 맞게 설정하여 안전하게 에이전트를 운영한다.

### 단계별 구현

#### Step 4.1 — 승인 모드(Approval Mode) 이해하기

> **💡 개념 설명: 왜 승인 모드가 필요한가?**
>
> Codex는 파일을 편집하고 셸 명령을 실행할 수 있다. 즉, 잘못하면 파일을 덮어쓰거나 의도치 않은 스크립트가 실행될 수 있다. 승인 모드는 "얼마나 자율적으로 실행할지"를 결정하는 안전장치이다.
>
> 핵심 요약: 모르는 코드베이스 = on-request / 검증된 환경 = never

| 모드 | 동작 방식 | 사용 시기 |
|---|---|---|
| **Auto** (기본) | 작업 디렉토리 내 편집·실행은 자동, 외부 접근은 물어봄 | 일반 개발 작업 |
| **Read-only** | 파일 읽기만 가능, 수정/실행 전에 계획 승인 필요 | 낯선 코드베이스 탐색 |
| **Full Access** | 머신 전체 + 네트워크 접근, 확인 없이 실행 | 완전히 신뢰하는 환경 |

```toml
# ~/.codex/config.toml

# on-request = Auto 모드
# never = Full Access 모드  
# untrusted = Read-only 모드
approval_policy = "on-request"
```

#### Step 4.2 — 네트워크 접근 설정

기본 샌드박스에서는 네트워크가 차단된다. npm install 같은 작업이 필요할 때:

```bash
# 터미널

# 일회성: 네트워크 허용하며 실행
codex -c 'sandbox_workspace_write.network_access=true' "npm install 하고 테스트 실행해줘"

# Full Access (가장 개방적, 주의해서 사용)
codex -s danger-full-access "gh pr create 해줘"
```

```toml
# .codex/config.toml (프로젝트 수준)

# 특정 프로젝트에서 항상 네트워크 허용
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
network_access = true
```

### 체크포인트 ✅

TUI에서 `/permissions`를 입력했을 때 현재 모드가 표시되고 변경할 수 있으면 정상이다.

---

## Phase 5: 실전 패턴 & 자동화

### 목표
일상적인 개발 워크플로우에 Codex를 녹여내고, `codex exec`으로 반복 작업을 자동화한다.

### 단계별 구현

#### Step 5.1 — AGENTS.md로 프로젝트 규칙 정의하기

> **💡 개념 설명: AGENTS.md란?**
>
> "Codex야, 이 프로젝트에서는 항상 TypeScript strict 모드를 써줘"처럼 프로젝트별 규칙을 파일로 정의하는 곳이다. Codex는 세션 시작 시 이 파일을 자동으로 읽는다. Git에 커밋해두면 팀 전체에 동일한 AI 가이드라인이 적용된다.

```markdown
<!-- AGENTS.md (프로젝트 루트에 생성) -->

## 코딩 규칙
- TypeScript strict 모드 사용
- 함수에는 항상 JSDoc 주석 추가
- 테스트 파일은 __tests__ 폴더에 작성

## 커밋 규칙
- conventional commits 형식 사용 (feat:, fix:, docs: 등)
- 커밋 전 반드시 npm test 실행

## 금지 사항
- console.log 남기지 말 것
- any 타입 사용 금지
```

#### Step 5.2 — 자주 쓰는 실전 프롬프트 패턴

```bash
# 터미널

# 1. 버그 수정
codex "src/api.ts에서 500 에러가 나는 부분 찾아서 고쳐줘"

# 2. 리팩토링
codex --profile deep-review "auth 모듈 전체를 async/await 패턴으로 리팩토링해줘"

# 3. 이미지로 UI 구현
# (TUI에서 스크린샷 붙여넣기 후)
# "이 디자인 목업대로 React 컴포넌트 만들어줘"

# 4. 코드 리뷰 요청
codex "변경된 파일들 코드 리뷰해줘"
# 또는 TUI에서 /review 슬래시 명령 사용

# 5. 특정 파일 참조
# TUI에서: @src/utils/parser.ts 이 파일에 엣지 케이스 처리 추가해줘
```

#### Step 5.3 — codex exec으로 CI/스크립트 자동화

> **💡 개념 설명: codex exec이란?**
>
> TUI 없이 명령 하나로 Codex를 실행하고 결과를 stdout으로 받는 비대화형 모드이다. GitHub Actions나 쉘 스크립트에 Codex를 통합할 때 사용한다.

```bash
# 터미널

# 기본 exec 사용법
codex exec --model gpt-5.4 "테스트 커버리지 리포트 생성하고 요약해줘"

# 승인 없이 자동 실행 (-a never)
codex exec -a never -m gpt-5.4 "lint 오류 모두 수정해줘"
```

```yaml
# .github/workflows/codex-review.yml (예시)

name: Codex Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Codex Review
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          npm install -g @openai/codex
          codex exec -a never -m gpt-5.4 "변경된 파일들의 보안 이슈와 버그를 검토하고 리포트해줘"
```

#### Step 5.4 — MCP 서버 연동 (선택)

Codex는 외부 도구를 MCP(Model Context Protocol)로 연결해 사용할 수 있다.

```toml
# ~/.codex/config.toml

# GitHub MCP 서버 예시
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_TOKEN = "ghp_..."
```

```bash
# 터미널

# MCP 서버 목록 관리
codex mcp list
codex mcp add <server-name>
```

### 체크포인트 ✅

```bash
# 터미널

# AGENTS.md가 있는 프로젝트 디렉토리에서 실행
codex "프로젝트 구조를 설명해줘"

# Codex가 AGENTS.md 규칙을 인식하고 적용하면 성공
```

---

## 마무리

### 이 가이드에서 배운 것

- Codex CLI를 npm/brew로 설치하고 ChatGPT 계정 또는 API Key로 인증한다.
- `~/.codex/config.toml`에 `model = "gpt-5.4"`를 설정하고, 프로필로 상황별 모드를 분리한다.
- TUI의 슬래시 명령(`/model`, `/permissions`, `/review` 등)으로 세션을 제어한다.
- 승인 모드 3가지(Auto / Read-only / Full Access)와 샌드박스 네트워크 설정을 이해한다.
- `AGENTS.md`로 프로젝트 규칙을 정의하고, `codex exec`으로 CI 자동화를 구성한다.

### 자주 발생하는 실수

| 실수 | 해결 방법 |
|---|---|
| 설정 초기화 후에도 구버전 모델 사용 | `config.toml`에 `model = "gpt-5.4"` 명시적으로 기입 |
| Full Access에서 의도치 않은 파일 삭제 | 항상 Git 커밋 후 Full Access 사용 |
| 네트워크 차단으로 npm install 실패 | `sandbox_workspace_write.network_access = true` 설정 |
| GPT-5.4 미표시 | `codex --version` 확인 후 최신 버전으로 업그레이드 |

### 업그레이드 방법

```bash
# 터미널

npm update -g @openai/codex
```

### 다음 학습 단계

- **Codex Skills**: 재사용 가능한 지시 번들(`$skill-name`)로 반복 작업 표준화
- **Codex Cloud**: 클라우드 에이전트로 장시간 작업을 백그라운드에서 병렬 처리
- **Multi-agent 실험**: `codex --enable multi_agent` 실험적 기능으로 에이전트 협업 구성
- **IDE Extension**: VS Code / Cursor에서 Codex를 직접 사용하는 IDE 확장

---

*작성 기준: OpenAI Codex CLI v0.112.0+, GPT-5.4, 2025년 3월*
