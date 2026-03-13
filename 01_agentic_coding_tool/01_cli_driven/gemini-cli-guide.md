# Gemini CLI 실습 가이드

## 개요

Gemini CLI는 Google이 개발한 오픈소스 AI 에이전트로, 터미널에서 직접 Gemini 모델의 강력한 기능을 활용할 수 있게 해주는 도구이다. ReAct(Reason and Act) 루프를 통해 복잡한 작업을 수행하며, 코드 생성, 버그 수정, 문서 작성 등 다양한 개발 작업을 자동화할 수 있다.

### 주요 특징

- **무료 사용**: 개인 Google 계정으로 로그인 시 분당 60회, 일일 1,000회 요청 무료 제공
- **강력한 AI 모델**: Gemini 3 Pro/Flash 모델 지원 (100만 토큰 컨텍스트 윈도우)
- **내장 도구**: Google Search, 파일 작업, 셸 명령, 웹 페이지 가져오기 등
- **확장성**: MCP(Model Context Protocol) 서버 지원으로 외부 통합 가능
- **터미널 중심**: 명령줄 환경에서 작업하는 개발자를 위한 설계

---

## 1. 설치

### 시스템 요구사항

- Node.js 18 이상
- npm 9 이상 또는 호환 가능한 패키지 매니저
- 운영체제: macOS, Linux, Windows 10/11

### 설치 방법

#### 방법 1: npm 전역 설치 (권장)

```bash
# Gemini CLI 전역 설치
npm install -g @google/gemini-cli

# 설치 확인
gemini --version
```

#### 방법 2: npx로 즉시 실행 (설치 없이)

```bash
# 영구 설치 없이 바로 실행
npx @google/gemini-cli
```

#### 방법 3: Homebrew (macOS)

```bash
# Homebrew가 설치되어 있는 경우
brew install gemini-cli
```

#### 방법 4: Conda 환경

```bash
# 새 Conda 환경 생성 및 활성화
conda create -y -n gemini_env -c conda-forge nodejs
conda activate gemini_env

# 환경 내에서 Gemini CLI 설치
npm install -g @google/gemini-cli
```

### 초기 실행 및 인증

설치 후 처음 실행 시:

```bash
gemini
```

1. **테마 선택**: 원하는 UI 테마를 선택한다
2. **인증 방법 선택**: 다음 세 가지 중 선택
   - **Login with Google** (권장): 개인 개발자 및 무료 티어 사용자
   - **API Key**: Google AI Studio에서 발급받은 API 키 사용
   - **Vertex AI**: 기업 사용자용 Google Cloud 인증

3. **Google 로그인 인증**:
   - 브라우저가 열리며 Google 계정으로 로그인
   - 권한 승인 후 터미널로 돌아와 사용 시작

---

## 2. Gemini CLI 옵션

### 기본 실행 옵션

```bash
# 기본 실행
gemini

# 특정 모델 지정
gemini --model gemini-3-pro-preview
gemini --model gemini-3-flash-preview

# 프롬프트와 함께 실행 (헤드리스 모드)
gemini -p "코드를 분석해줘"

# JSON 형식 출력 (스크립팅용)
gemini -p "프로젝트 구조 설명해줘" --output-format json

# 여러 디렉토리 포함
gemini --include-directories ./src,./docs

# 샌드박스 모드 (보안 강화)
gemini --sandbox
```

### 주요 커맨드라인 플래그

| 플래그 | 설명 | 예시 |
|--------|------|------|
| `-m, --model` | 사용할 모델 선택 | `--model gemini-3-flash-preview` |
| `-p, --prompt` | 프롬프트와 함께 실행 | `-p "README 생성해줘"` |
| `--output-format` | 출력 형식 지정 (text/json) | `--output-format json` |
| `--include-directories` | 작업 디렉토리 추가 | `--include-directories ./api,./web` |
| `--sandbox` | 샌드박스 컨테이너에서 실행 | `--sandbox` |
| `-v, --version` | 버전 확인 | `--version` |
| `--help` | 도움말 표시 | `--help` |

### 대화형 세션 내 명령어

대화형 세션에서는 `/` 접두사로 시작하는 슬래시 명령어를 사용할 수 있다:

```bash
/help          # 도움말 표시
/quit          # 세션 종료
/clear         # 화면 지우기
/model         # 모델 변경
/stats         # 토큰 사용량 통계
/settings      # 설정 편집기 열기
```

### YOLO 모드 (자동 승인 모드)

#### YOLO 모드란?

YOLO 모드("You Only Live Once"의 약자)는 Gemini CLI가 파일 쓰기, 셸 명령 실행 등 모든 도구 작업을 사용자 확인 없이 자동으로 승인하는 기능이다. 기본적으로 Gemini CLI는 파일 수정이나 명령 실행 전에 항상 사용자에게 승인을 요청하지만, YOLO 모드에서는 이 단계를 건너뛰고 즉시 실행한다.

#### YOLO 모드 활성화 방법

**방법 1: 시작 시 플래그로 활성화**

```bash
# 전체 표기
gemini --yolo

# 단축 표기
gemini -y
```

**방법 2: 대화형 세션 중 토글**

```bash
# 대화형 세션에서 Ctrl+Y를 누르면 YOLO 모드 ON/OFF
# 화면 상단에 "YOLO mode enabled (all actions auto-approved)" 표시됨
```

**방법 3: 환경 변수 설정**

```bash
export GEMINI_YOLO_MODE=true
gemini
```

#### YOLO 모드의 특징

1. **자동 샌드박스 활성화**
   - YOLO 모드 활성화 시 안전을 위해 샌드박스 모드가 자동으로 활성화된다
   - Docker 컨테이너 내부에서 격리된 환경으로 실행됨

2. **즉시 실행**
   - AI가 도구 사용을 결정하면 "승인하시겠습니까? (y/n)" 프롬프트 없이 즉시 실행
   - 반복적인 작업 수행 시 워크플로우 속도가 크게 향상됨

#### 적합한 사용 사례

**✅ YOLO 모드 사용이 적절한 경우:**

```bash
# CI/CD 파이프라인에서 자동화된 코드 리뷰
# .github/workflows/ai-review.yml
- name: AI Code Review
  run: |
    npm install -g @google/gemini-cli
    export GEMINI_API_KEY=${{ secrets.GEMINI_API_KEY }}
    gemini --yolo "분석 결과를 code-review.md에 작성해줘"
```

```bash
# Docker 컨테이너 개발 환경 자동 설정
docker run -it --rm mydev-image bash -c \
  "gemini --yolo '개발 환경 셋업하고 필요한 패키지 설치해줘'"
```

```bash
# 반복적인 파일 생성 작업
gemini --yolo "src/utils/ 디렉토리의 모든 함수에 대해 유닛 테스트 파일 생성해줘"
```

```bash
# 격리된 테스트 환경에서 실험
# (손상되어도 재생성 가능한 환경)
gemini --yolo "다양한 성능 최적화 방법을 시도해보고 벤치마크 결과 기록해줘"
```

**❌ YOLO 모드 사용을 피해야 하는 경우:**

- **프로덕션 시스템**: 실제 운영 중인 서버나 데이터베이스
- **민감한 데이터 환경**: 고객 정보, 개인정보, 기밀 문서가 있는 곳
- **익숙하지 않은 코드베이스**: 처음 보는 프로젝트나 레거시 코드
- **되돌릴 수 없는 작업**: Git 커밋 이력, 중요한 설정 파일 등

#### 안전 장치

**관리자 설정으로 YOLO 모드 차단**

일부 조직에서는 보안을 위해 YOLO 모드를 차단할 수 있다:

```json
// ~/.gemini/settings.json
{
  "security": {
    "disallowYolo": true
  }
}
```

이 경우 `gemini --yolo` 실행 시 다음 에러가 표시된다:
```
Cannot start in YOLO mode when it is disabled by settings
```

**특정 명령만 자동 승인 (대안)**

전체 YOLO 모드가 부담스러울 경우, 특정 명령만 자동 승인하도록 설정할 수 있다:

```json
// ~/.gemini/settings.json
{
  "sandbox": {
    "allowedCommands": [
      "git add *",
      "git commit *",
      "npm run *",
      "npx prettier *"
    ]
  }
}
```

#### 사고 발생 시 복구 방법

YOLO 모드 사용 중 의도하지 않은 변경이 발생한 경우:

```bash
# 모든 커밋되지 않은 변경사항 폐기
git checkout -- .

# 특정 커밋으로 되돌리기
git reset --hard HEAD~1

# 변경 내용 확인
git diff HEAD~1

# 파일 복원 (git 사용 안 하는 경우)
# 반드시 백업이 있어야 함!
```

#### YOLO 모드 모범 사례

1. **항상 Git 버전 관리 사용**
   ```bash
   # YOLO 모드 사용 전 현재 상태 커밋
   git add .
   git commit -m "YOLO 모드 사용 전 체크포인트"
   
   # YOLO 모드로 작업 수행
   gemini --yolo "리팩토링 수행"
   
   # 결과 확인
   git diff
   ```

2. **격리된 환경에서 테스트**
   ```bash
   # 브랜치 생성하여 YOLO 모드 테스트
   git checkout -b yolo-experiment
   gemini --yolo "실험적인 변경 수행"
   
   # 결과가 좋으면 머지, 나쁘면 폐기
   git checkout main
   git branch -D yolo-experiment
   ```

3. **작업 범위 명확히 지정**
   ```bash
   # ❌ 모호한 요청
   gemini --yolo "코드 개선해줘"
   
   # ✅ 구체적인 요청
   gemini --yolo "src/auth/ 디렉토리의 TypeScript 파일에만 ESLint 규칙 적용해줘"
   ```

4. **정기적으로 결과 검토**
   ```bash
   # YOLO 모드로 여러 작업 수행 후
   git log --oneline -10  # 최근 커밋 확인
   git diff HEAD~5        # 전체 변경사항 리뷰
   ```

#### 유용한 팁

**1. Dry-run과 함께 사용**
```bash
# 실제 변경하지 않고 계획만 확인
gemini --plan "리팩토링 계획 세워줘"

# 계획 확인 후 YOLO 모드로 실행
gemini --yolo "위 계획대로 실행해줘"
```

**2. 제한된 권한으로 실행**
```bash
# Docker 컨테이너에서 읽기 전용 볼륨 마운트
docker run -v $(pwd):/workspace:ro gemini-cli \
  gemini --yolo "코드 분석 리포트 작성"
```

**3. 키보드 단축키 활용**
- `Ctrl+Y`: YOLO 모드 즉시 토글
- `Ctrl+C` (2번): 긴급 종료
- `Ctrl+L`: 화면 지우기

---

## 3. 프로젝트/사용자 스코프 구분

Gemini CLI는 계층적 컨텍스트 시스템을 사용하여 전역, 프로젝트, 컴포넌트 레벨에서 설정과 컨텍스트를 관리한다.

### 스코프 계층 구조

```
전역(Global) > 워크스페이스(Workspace) > 컴포넌트(Component)
```

### 3.1 설정 파일 위치

#### 전역 설정 (User Scope)

사용자의 모든 프로젝트에 적용되는 설정이다.

```bash
~/.gemini/
├── settings.json       # 전역 설정
├── GEMINI.md          # 전역 컨텍스트/메모리
├── commands/          # 사용자 정의 슬래시 명령어
├── skills/            # 사용자 전역 Agent Skills
└── extensions/        # 설치된 확장 프로그램
```

**예시: 전역 GEMINI.md**

```bash
# ~/.gemini/GEMINI.md 생성
cat > ~/.gemini/GEMINI.md << 'EOF'
# 전역 설정

나는 풀스택 개발자이다.
선호하는 프로그래밍 언어는 Python과 TypeScript이다.
코드 작성 시 항상 타입 힌트를 포함해줘.
주석은 한국어로 작성한다.
EOF
```

#### 프로젝트 설정 (Project/Workspace Scope)

특정 프로젝트 디렉토리에만 적용되는 설정이다.

```bash
<project-root>/
├── .gemini/
│   ├── settings.json      # 프로젝트별 설정
│   ├── GEMINI.md         # 프로젝트 컨텍스트
│   ├── commands/         # 프로젝트 전용 슬래시 명령어
│   └── skills/           # 프로젝트 전용 Agent Skills
└── .geminiignore         # 제외할 파일 패턴
```

**예시: 프로젝트 GEMINI.md**

```markdown
<!-- <project-root>/.gemini/GEMINI.md -->
# 프로젝트: E-commerce 백엔드 API

## 기술 스택
- Node.js + Express
- PostgreSQL
- Redis
- JWT 인증

## 코딩 규칙
- ESLint 규칙 준수
- 모든 API 엔드포인트에 OpenAPI 문서 작성
- 에러 핸들링은 middleware/errorHandler.js 사용
```

#### 컴포넌트 설정 (Component Scope)

프로젝트 내 특정 디렉토리나 모듈에 대한 세부 지침이다.

```bash
<project-root>/
└── src/
    └── auth/
        └── GEMINI.md     # auth 모듈 전용 컨텍스트
```

**예시: 컴포넌트 GEMINI.md**

```markdown
<!-- src/auth/GEMINI.md -->
# 인증 모듈

이 디렉토리는 JWT 기반 인증을 처리한다.

- passport-jwt 라이브러리 사용
- 토큰 만료 시간: 1시간
- 리프레시 토큰: 7일
- 비밀번호 해싱: bcrypt (saltRounds=10)
```

### 3.2 컨텍스트 로딩 우선순위

Gemini CLI는 다음 순서로 컨텍스트를 로드하고 병합한다:

1. **전역 컨텍스트** (`~/.gemini/GEMINI.md`)
2. **워크스페이스 컨텍스트** (프로젝트 루트의 `.gemini/GEMINI.md`)
3. **컴포넌트 컨텍스트** (현재 작업 중인 파일의 상위 디렉토리 내 `GEMINI.md`)

더 구체적인 스코프가 더 일반적인 스코프보다 우선한다.

### 3.3 컨텍스트 관리 명령어

```bash
# 현재 로드된 모든 컨텍스트 확인
/memory show

# 컨텍스트 다시 로드
/memory reload

# 전역 메모리에 내용 추가
/memory add "React 프로젝트에서는 함수형 컴포넌트를 사용한다"
```

### 3.4 .geminiignore 파일

특정 파일이나 디렉토리를 컨텍스트에서 제외한다 (`.gitignore`와 유사한 문법).

```bash
# .geminiignore 예시
node_modules/
dist/
*.log
.env
test/
coverage/
```

---

## 4. Rule / Slash Command / Skill 등 부가기능

### 4.1 GEMINI.md를 통한 Rule 정의

GEMINI.md 파일은 AI가 따라야 할 규칙과 컨텍스트를 제공한다.

#### Rule 작성 예시

```markdown
<!-- .gemini/GEMINI.md -->
# 프로젝트 규칙

## 코드 작성 규칙
- TypeScript로 작성하되 `any` 타입 사용 금지
- 모든 함수에 JSDoc 주석 필수
- 2칸 들여쓰기 사용
- 인터페이스 이름은 `I` 접두사 사용 (예: `IUserService`)

## Git 커밋 규칙
- Conventional Commits 형식 준수
- 커밋 메시지는 한국어로 작성

## 보안 규칙
- 환경 변수는 절대 코드에 직접 하드코딩하지 않는다
- 모든 외부 입력은 검증 후 사용한다
```

#### 파일 임포트 기능

큰 GEMINI.md 파일을 작은 파일로 분할할 수 있다:

```markdown
<!-- .gemini/GEMINI.md -->
# 메인 컨텍스트

@./components/coding-style.md
@./components/architecture.md
@../shared/security-guidelines.md
```

### 4.2 Custom Slash Commands

#### Slash Command란?

슬래시 명령어는 자주 사용하는 프롬프트를 재사용 가능한 단축 명령어로 만드는 기능이다. `.toml` 파일 형식으로 정의한다.

#### 명령어 위치와 우선순위

```
프로젝트 명령어 > 사용자 명령어 > 확장 명령어
```

- **프로젝트**: `<project>/.gemini/commands/`
- **사용자**: `~/.gemini/commands/`

#### 기본 Slash Command 작성

**예시 1: /plan 명령어 (계획만 세우고 실행하지 않음)**

```toml
# ~/.gemini/commands/plan.toml

prompt = """
다음 작업에 대해 자세한 실행 계획을 단계별로 작성해줘.
코드를 직접 작성하지 말고, 전략과 구현 단계만 제시해줘.

작업 내용: {{args}}
"""

description = "코드를 작성하지 않고 계획만 세운다"
```

**사용 예시**:
```bash
> /plan JWT 인증 시스템 구축하기
```

**예시 2: /review 명령어 (GitHub PR 리뷰)**

```toml
# ~/.gemini/commands/review.toml

prompt = """
다음 GitHub PR을 검토해줘:
!{gh pr view {{args}} --json title,body,files}

코드 품질, 보안 이슈, 성능 문제를 중점적으로 확인하고
개선 제안을 구체적으로 제시해줘.
"""

description = "GitHub PR을 검토하고 피드백을 제공한다"
```

**사용 예시**:
```bash
> /review 123
```

#### 네임스페이스 명령어

디렉토리 구조로 명령어를 그룹화할 수 있다:

```bash
~/.gemini/commands/
├── git/
│   ├── commit.toml      # /git:commit
│   └── summary.toml     # /git:summary
└── docs/
    └── api.toml         # /docs:api
```

#### 고급 기능

**1. 인자 치환 (`{{args}}`)**

```toml
prompt = "다음 파일을 분석해줘: {{args}}"
```

**2. 셸 명령 실행 (`!{...}`)**

```toml
prompt = """
현재 Git 상태:
!{git status}

변경 사항 설명:
!{git diff --staged}

위 내용을 바탕으로 커밋 메시지를 작성해줘.
"""
```

**3. 파일 내용 주입 (`@{...}`)**

```toml
prompt = """
다음 코드를 리팩토링해줘:
@{src/main.ts}

성능과 가독성을 개선하고, 타입 안정성을 강화해줘.
"""
```

**4. 이미지/PDF 포함**

```toml
prompt = """
이 디자인 목업을 분석해줘:
@{designs/mockup.png}

HTML/CSS로 구현 가능한지 평가하고 구조를 제안해줘.
"""
```

#### 명령어 관리

```bash
# 명령어 목록 확인
/commands list

# 명령어 다시 로드 (파일 수정 후)
/commands reload
```

### 4.3 Agent Skills

#### Agent Skill이란?

Agent Skill은 전문화된 지식과 절차적 워크플로우를 패키징한 재사용 가능한 모듈이다. GEMINI.md가 항상 로드되는 것과 달리, Skill은 필요할 때만 활성화되어 컨텍스트 윈도우를 절약한다.

#### Skill 구조

```bash
.gemini/skills/api-auditor/
├── SKILL.md           # 스킬 정의 및 지침
├── scripts/
│   └── audit.js       # 번들된 스크립트
├── resources/
│   └── checklist.md   # 참고 자료
└── assets/
    └── example.json   # 예시 데이터
```

#### SKILL.md 형식

```markdown
---
name: api-auditor
description: >
  API 엔드포인트의 응답성과 신뢰성을 검증하는 QA 전문가.
  사용자가 "API 테스트", "엔드포인트 검증"을 요청할 때 사용한다.
---

# API Auditor Skill

당신은 API 신뢰성 전문 QA 엔지니어이다.

## 역할
- API 엔드포인트의 응답 상태 검증
- 응답 시간 측정 및 성능 분석
- 에러 핸들링 확인

## 절차
1. 엔드포인트 목록 수집
2. `scripts/audit.js` 실행하여 상태 코드 확인
3. 결과를 표 형식으로 정리
4. 이슈가 있는 엔드포인트 강조 표시

## 번들된 도구
- `scripts/audit.js`: HTTP 요청을 보내고 응답을 검증하는 Node.js 스크립트
```

#### Skill 작성 예시

**파일: `~/.gemini/skills/adk-expert/SKILL.md`**

```markdown
---
name: adk-expert
description: >
  Agent Development Kit (ADK) SDK에 대한 깊이 있는 지식을 가진 AI 개발 전문가.
  ADK를 사용한 에이전트 개발 시 활성화된다.
---

# ADK Expert

당신은 Google의 Agent Development Kit (ADK) SDK 전문가이다.

## 전문 지식
- ADK의 아키텍처와 핵심 컴포넌트
- 에이전트 라이프사이클 관리
- 도구 통합 및 MCP 서버 연동
- 성능 최적화 및 디버깅

## 작업 시 원칙
1. 항상 최신 ADK API를 참조한다
2. 보안 모범 사례를 따른다
3. 코드 예시는 TypeScript로 작성한다
4. 에러 핸들링을 빠짐없이 구현한다

## 리소스
- `resources/adk-api-reference.md`: 최신 API 문서
- `assets/examples/`: 검증된 예제 코드
```

#### Skill 관리 명령어

```bash
# 사용 가능한 스킬 목록 확인
/skills list

# 특정 스킬 활성화
/skills enable api-auditor

# 스킬 비활성화
/skills disable api-auditor

# 스킬 상세 정보 확인
/skills show api-auditor
```

#### Skill 우선순위

같은 이름의 스킬이 여러 위치에 있을 경우:

```
워크스페이스 스킬 > 사용자 스킬 > 확장 스킬
```

- 워크스페이스: `<project>/.gemini/skills/`
- 사용자: `~/.gemini/skills/`
- 확장: 설치된 확장 프로그램 내부

#### 공식 Skill 설치 예시

```bash
# Firebase용 Agent Skills 설치
npx skills.sh install firebase-basics
npx skills.sh install firebase-auth-basics
npx skills.sh install firebase-firestore-basics

# 설치된 스킬 확인
/skills firebase
```

### 4.4 MCP (Model Context Protocol) 서버

#### MCP란?

MCP는 AI 에이전트가 외부 시스템(GitHub, Slack, 데이터베이스 등)과 상호작용할 수 있게 해주는 표준 프로토콜이다.

#### MCP 서버 설정

**파일: `~/.gemini/settings.json`**

```json
{
  "security": {
    "auth": {
      "selectedType": "oauth-personal"
    }
  },
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@github/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "ghp_your_token_here"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@slack/mcp-server"],
      "env": {
        "SLACK_TOKEN": "xoxb-your-token"
      }
    }
  }
}
```

#### MCP 사용 예시

```bash
# GitHub MCP 사용
> @github 내 오픈 PR 목록 보여줘
> @github 이슈 #123 닫아줘

# Slack MCP 사용
> @slack #dev 채널에 오늘의 커밋 요약 전송해줘

# 데이터베이스 MCP 사용
> @database 비활성 사용자 조회 쿼리 실행해줘
```

#### MCP 서버 관리

```bash
# 설정된 MCP 서버 목록 확인
/mcp list

# 도구 설명 표시/숨기기
/mcp show
/mcp hide

# 도구의 JSON 스키마 확인
/mcp schema
```

### 4.5 Hooks (후킹 시스템)

Hooks는 Gemini CLI의 특정 라이프사이클 이벤트에 맞춰 커스텀 스크립트를 실행할 수 있게 한다.

#### 사용 가능한 Hook 이벤트

- `pre-prompt`: 프롬프트 전송 전
- `post-prompt`: 응답 수신 후
- `pre-tool`: 도구 실행 전
- `post-tool`: 도구 실행 후

#### Hooks 관리

```bash
# 등록된 훅 목록 확인
/hooks list

# 특정 훅 활성화
/hooks enable pre-prompt

# 특정 훅 비활성화
/hooks disable pre-prompt

# 모든 훅 활성화/비활성화
/hooks enable all
/hooks disable all
```

---

## 5. 고급 활용

### 5.1 Headless 모드 (스크립팅)

```bash
# 한 줄 명령으로 결과만 받기
gemini -p "README.md 파일을 분석하고 요약해줘" --output-format json

# 결과를 파일로 저장
gemini -p "프로젝트 아키텍처 다이어그램 생성해줘" > architecture.md
```

### 5.2 세션 관리 (Checkpointing)

```bash
# 현재 대화 저장
/checkpoint save feature-development

# 저장된 대화 목록 확인
/history list

# 저장된 대화 재개
/history resume feature-development
```

### 5.3 Rewind (대화 되감기)

```bash
# 이전 메시지로 되돌아가기
/rewind 3    # 3개 메시지 되감기
```

### 5.4 셸 명령어 통합

```bash
# 셸 명령 실행
> !git status
> !npm run test

# 셸 모드 토글
> !
# (셸 모드 활성화 - 모든 입력이 셸 명령으로 처리됨)
> exit  # 셸 모드 종료
```

### 5.5 IDE 통합

```bash
# VS Code 통합 활성화
/ide enable

# VS Code companion 설치
/ide install

# 통합 상태 확인
/ide status
```

---

## 6. 모범 사례

### 프로젝트 초기 설정 체크리스트

1. **프로젝트 루트에 `.gemini/GEMINI.md` 생성**
   - 기술 스택, 코딩 규칙, 아키텍처 설명 포함

2. **`.geminiignore` 설정**
   - `node_modules/`, `dist/`, `.env` 등 제외

3. **프로젝트 전용 슬래시 명령어 정의**
   - 자주 사용하는 작업을 명령어로 만들기

4. **필요한 MCP 서버 설정**
   - GitHub, Slack 등 팀에서 사용하는 도구 연동

5. **팀원과 설정 공유**
   - `.gemini/` 디렉토리를 Git에 커밋

### 효과적인 프롬프트 작성

```bash
# ❌ 나쁜 예
> 코드 고쳐줘

# ✅ 좋은 예
> src/auth/login.ts 파일에서 JWT 토큰 생성 로직을 수정해줘.
> 현재는 만료 시간이 하드코딩되어 있는데, 환경 변수에서 읽어오도록 변경하고,
> 리프레시 토큰 발급 로직도 추가해줘.
```

### 컨텍스트 관리 팁

- 전역 GEMINI.md는 최소화하고 프로젝트별로 구체화
- 민감한 정보는 GEMINI.md에 절대 포함하지 않기
- 컴포넌트별 GEMINI.md는 해당 모듈의 특수 규칙만 포함

---

## 7. 문제 해결

### 인증 실패

```bash
# 인증 토큰 초기화
rm -rf ~/.gemini/auth

# 재인증
gemini
```

### MCP 서버 연결 실패

```bash
# 설정 파일 확인
cat ~/.gemini/settings.json

# MCP 서버 로그 확인
/mcp list
```

### 높은 토큰 사용량

```bash
# 컨텍스트 요약으로 토큰 절약
/compress

# 통계 확인
/stats model
```

### 명령어가 인식되지 않을 때

```bash
# 명령어 다시 로드
/commands reload

# 스킬 다시 로드
/skills reload
```

---

## 8. 추가 자료

### 공식 문서

- [Gemini CLI 공식 문서](https://geminicli.com/docs/)
- [GitHub 저장소](https://github.com/google-gemini/gemini-cli)
- [Agent Skills 표준](https://github.com/anthropics/agent-skills)

### 커뮤니티

- [Google Cloud Blog - Gemini CLI](https://cloud.google.com/blog/topics/developers-practitioners/gemini-cli-custom-slash-commands)
- [Medium - Gemini CLI 튜토리얼](https://medium.com/google-cloud/gemini-cli-tutorial-series-part-7-custom-slash-commands-64c06195294b)

### 업데이트

```bash
# 최신 버전으로 업데이트
npm update -g @google/gemini-cli

# 버전 확인
gemini --version
```

---

## 마치며

Gemini CLI는 터미널에서 강력한 AI 에이전트를 활용할 수 있는 혁신적인 도구이다. GEMINI.md를 통한 컨텍스트 관리, 커스텀 슬래시 명령어, Agent Skills, MCP 서버 통합 등의 기능을 적절히 조합하면 개발 생산성을 크게 향상시킬 수 있다.

초기 학습 곡선이 있지만, 프로젝트 요구사항에 맞게 설정을 구성하고 팀 워크플로우에 통합하면 반복 작업을 자동화하고 코드 품질을 개선하는 강력한 도구가 된다.
