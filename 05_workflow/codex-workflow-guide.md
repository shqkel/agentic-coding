# Codex CLI 자동화 워크플로우 가이드

## 학습 목표

Codex CLI의 **비대화형 실행 모드**와 **프로파일 시스템**을 이해하고, `codex exec` 명령과 `config.toml` 기반 프로파일을 조합하여 CI/CD 파이프라인에서 AI 자동화를 구현할 수 있다.

> **다른 도구와의 대응**: Claude Code의 **Hooks**, Antigravity의 **Workflows**, Cursor의 **Automations**에 해당하는 Codex CLI 기능이다. Codex CLI는 명시적 이벤트 훅 대신 **비대화형 실행**과 **프로파일 분기**로 자동화를 구성한다.

## 사전 준비

- Node.js 18 이상 설치
- OpenAI API 키 발급
- Git 기본 사용법 숙지

---

## 전체 흐름 한눈에 보기

Codex CLI 자동화의 핵심은 두 가지다. 첫째, `codex exec` 명령으로 스크립트나 CI 파이프라인에서 대화 없이 AI를 실행한다. 둘째, `config.toml` 프로파일로 작업 유형별 설정(모델, 승인 정책, 추론 강도)을 분리 관리한다.

1. **개념 이해** — 비대화형 실행과 프로파일의 역할
2. **기본 자동화** — `codex exec`와 파이프라인 구성
3. **고급 설정** — 프로파일 정의와 CI/CD 통합
4. **실전 예시** — 코드 리뷰, 테스트 생성, 문서화 자동화

---

## Phase 1: Codex CLI 자동화 개념

### 목표

비대화형 실행 모드가 해결하는 문제와 프로파일 기반 분기의 원리를 이해한다.

### 단계별 구현

#### Step 1.1 — 비대화형 실행이 해결하는 문제

> **💡 개념 설명: 왜 비대화형인가?**
>
> Codex CLI는 기본적으로 대화형 REPL이다. 하지만 CI 서버나 자동화 스크립트에서는 사람이 개입할 수 없다. `codex exec`는 단일 작업을 대화 없이 수행하고 종료한다. 프롬프트를 전달하면 코드를 작성하거나 파일을 수정하고, 완료되면 자동으로 종료된다.
>
> **핵심 한 줄:** `codex exec` = 대화 없는 1회용 AI 작업 실행기

#### Step 1.2 — 기본 실행 구조

```bash
# 기본 설치
npm install -g @openai/codex

# 환경 변수 설정
export OPENAI_API_KEY="sk-..."

# 비대화형 실행 기본 문법
codex exec [옵션] "작업 설명"
```

#### Step 1.3 — 주요 옵션 정리

| 옵션 | 의미 | 예시 |
|------|------|------|
| `--model (-m)` | 사용할 모델 지정 | `-m gpt-5.4` |
| `--approval (-a)` | 승인 정책 | `-a never` (자동 승인) |
| `--profile (-p)` | 프로파일 이름 | `-p deep-review` |
| `--quiet (-q)` | 출력 최소화 | `-q` |
| `--output (-o)` | 출력 형식 | `-o json` |

#### Step 1.4 — 승인 정책(Approval Policy)

| 정책 | 설명 | 적합한 상황 |
|------|------|-------------|
| `suggest` | 모든 행동에 승인 요청 (기본값) | 대화형 작업 |
| `auto-edit` | 파일 수정만 자동 승인 | 코드 수정 자동화 |
| `full-auto` | 모든 행동 자동 승인 | 신뢰할 수 있는 환경 |
| `never` | 승인 없이 전부 실행 | CI/CD 파이프라인 |

### 체크포인트

`codex exec -a never "현재 디렉토리의 파일 목록을 보여줘"`를 실행하여 응답이 출력되면 성공이다.

---

## Phase 2: 기본 자동화 파이프라인

### 목표

`codex exec`를 셸 스크립트에 통합하고 여러 작업을 순서대로 실행하는 파이프라인을 구성한다.

### 단계별 구현

#### Step 2.1 — 단일 작업 자동화

```bash
# 린트 오류 자동 수정
codex exec -a never -m gpt-5.4 "이 프로젝트의 ESLint 오류를 모두 수정해줘"

# 테스트 커버리지 분석
codex exec -a never -m gpt-5.4 "테스트 커버리지 리포트를 생성하고 커버리지가 낮은 파일을 알려줘"

# 의존성 취약점 분석
codex exec -a never -m gpt-5.4 "npm audit 결과를 분석하고 취약한 패키지 목록을 작성해줘"
```

#### Step 2.2 — 파이프라인 스크립트

여러 AI 작업을 순서대로 연결하는 셸 스크립트를 작성한다:

```bash
#!/bin/bash
# scripts/ai-code-pipeline.sh

set -e  # 오류 발생 시 중단

echo "=== AI 코드 품질 파이프라인 시작 ==="

# 1단계: 보안 취약점 검사
echo "[1/4] 보안 취약점 검사 중..."
codex exec -a never -q -m gpt-5.4 \
  "변경된 파일들의 SQL Injection, XSS, 인증 우회 등 보안 취약점을 검사하고 severity별로 보고해줘" \
  > reports/security-report.md

# 2단계: 코드 품질 분석
echo "[2/4] 코드 품질 분석 중..."
codex exec -a never -q -m gpt-5.4 \
  "SOLID 원칙 위반, 과도한 복잡도, 중복 코드를 찾아서 리팩토링 제안을 작성해줘" \
  > reports/quality-report.md

# 3단계: 누락된 테스트 보완
echo "[3/4] 테스트 커버리지 개선 중..."
codex exec -a auto-edit -m gpt-5.4 \
  "테스트가 없는 함수에 대한 단위 테스트를 자동으로 생성해줘"

# 4단계: 최종 요약
echo "[4/4] 요약 보고서 생성 중..."
cat reports/security-report.md reports/quality-report.md | \
  codex exec -a never -q -m gpt-5.4 \
  "위 보고서들을 통합하여 3줄 이내 요약과 즉시 조치 항목을 정리해줘" \
  > reports/summary.md

echo "=== 파이프라인 완료. reports/ 디렉토리 확인 ==="
```

#### Step 2.3 — 입력 파이프라인 (stdin 활용)

```bash
# git diff를 codex에 전달
git diff HEAD~1 | codex exec -a never "이 변경사항에서 버그 가능성이 있는 부분을 찾아줘"

# 파일 내용을 직접 전달
cat src/auth.ts | codex exec -a never "이 파일의 보안 취약점을 분석해줘"

# 여러 파일 내용 통합 전달
find src -name "*.ts" -exec cat {} \; | \
  codex exec -a never "이 TypeScript 코드베이스의 타입 안전성 문제를 찾아줘"
```

### 체크포인트

파이프라인 스크립트 실행 후 `reports/` 디렉토리에 3개의 보고서 파일이 생성되면 성공이다.

---

## Phase 3: 프로파일 기반 고급 설정

### 목표

`config.toml`에 작업 유형별 프로파일을 정의하고, 상황에 맞는 모델과 정책을 자동 선택하도록 구성한다.

### 단계별 구현

#### Step 3.1 — config.toml 위치와 기본 구조

```toml
# ~/.codex/config.toml (전역 설정)
# 또는 <project-root>/.codex/config.toml (프로젝트 설정)

# 기본 설정
model = "gpt-4o"
approval_policy = "suggest"
```

#### Step 3.2 — 작업별 프로파일 정의

```toml
# ~/.codex/config.toml

# 기본 설정
model = "gpt-4o"
approval_policy = "suggest"

# 심층 코드 리뷰 프로파일
[profiles.deep-review]
model = "gpt-5.4"
model_reasoning_effort = "high"
approval_policy = "never"
description = "보안, 성능, 품질을 종합적으로 분석하는 심층 리뷰"

# 빠른 수정 프로파일
[profiles.quick-fix]
model = "gpt-4o-mini"
model_reasoning_effort = "low"
approval_policy = "auto-edit"
description = "린트 오류, 타입 오류 등 단순 수정 작업"

# CI 자동화 프로파일
[profiles.ci]
model = "gpt-4o"
model_reasoning_effort = "medium"
approval_policy = "never"
description = "CI 파이프라인에서 사람 개입 없이 실행"

# 문서화 프로파일
[profiles.docs]
model = "gpt-4o"
model_reasoning_effort = "medium"
approval_policy = "auto-edit"
description = "JSDoc, docstring, README 자동 생성"

# 테스트 생성 프로파일
[profiles.test-gen]
model = "gpt-5.4"
model_reasoning_effort = "high"
approval_policy = "auto-edit"
description = "엣지 케이스를 포함한 포괄적인 테스트 자동 생성"
```

#### Step 3.3 — 프로파일을 활용한 실행

```bash
# 심층 리뷰 (고성능 모델, 자동 실행)
codex exec -p deep-review "이 PR의 모든 변경 파일을 보안 관점에서 검토해줘"

# 빠른 수정 (경량 모델, 파일 수정 자동 승인)
codex exec -p quick-fix "TypeScript 컴파일 오류를 모두 수정해줘"

# CI 모드 (자동 승인, 출력 최소화)
codex exec -p ci -q "테스트를 실행하고 실패한 테스트의 원인을 분석해줘"

# 문서화
codex exec -p docs "src/api/ 디렉토리의 모든 함수에 JSDoc 주석을 추가해줘"
```

#### Step 3.4 — GitHub Actions 통합

```yaml
# .github/workflows/codex-review.yml
name: Codex 자동 코드 리뷰

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  security-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Codex CLI 설치
        run: npm install -g @openai/codex

      - name: 변경 파일 보안 검토
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          CHANGED_FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          echo "변경된 파일: $CHANGED_FILES"

          git diff origin/${{ github.base_ref }}...HEAD > /tmp/pr-diff.txt

          codex exec -p ci \
            "다음 PR diff를 검토하고 보안 이슈와 버그를 찾아줘. 심각도를 HIGH/MEDIUM/LOW로 분류해줘:" \
            < /tmp/pr-diff.txt \
            > /tmp/review-result.txt

      - name: PR에 리뷰 댓글 작성
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          REVIEW=$(cat /tmp/review-result.txt)
          gh pr comment ${{ github.event.pull_request.number }} \
            --body "## 🤖 Codex 자동 코드 리뷰

          $REVIEW

          ---
          *이 리뷰는 Codex CLI에 의해 자동 생성되었습니다.*"

  test-generation:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - name: 새 파일에 대한 테스트 자동 생성
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          npm install -g @openai/codex
          NEW_FILES=$(git diff --name-only --diff-filter=A origin/${{ github.base_ref }}...HEAD | grep -E '\.(ts|js|py)$')

          if [ -n "$NEW_FILES" ]; then
            codex exec -p test-gen \
              "다음 새 파일들에 대한 단위 테스트를 자동 생성해줘: $NEW_FILES"
          fi
```

### 체크포인트

`codex exec -p deep-review "..."` 실행 시 `config.toml`의 프로파일 설정이 적용되어 고성능 모델로 실행되면 성공이다.

---

## Phase 4: 실전 예시

### 목표

개발 팀에서 바로 사용할 수 있는 완성된 자동화 워크플로우를 구축한다.

### 단계별 구현

#### Step 4.1 — 사전 커밋 훅(pre-commit)과 연동

```bash
#!/bin/bash
# .git/hooks/pre-commit (또는 .husky/pre-commit)

# 스테이징된 파일만 대상으로 빠른 검사
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM | grep -E '\.(ts|js|py)$')

if [ -z "$STAGED_FILES" ]; then
  exit 0
fi

echo "Codex 빠른 검사 실행 중..."
RESULT=$(git diff --cached | codex exec -p quick-fix -q \
  "이 변경사항에 명백한 버그, 콘솔 로그 미제거, 하드코딩된 비밀번호가 있으면 알려줘. 없으면 'OK'만 출력해줘")

if [ "$RESULT" != "OK" ]; then
  echo "⚠️  Codex 경고:"
  echo "$RESULT"
  echo ""
  echo "계속 커밋하려면 'git commit --no-verify' 사용"
  exit 1
fi
```

#### Step 4.2 — 주간 코드 품질 보고서 자동화

```bash
#!/bin/bash
# scripts/weekly-report.sh
# cron: 0 9 * * 1 (매주 월요일 오전 9시 실행)

WEEK_START=$(date -d "last monday" +%Y-%m-%d)
WEEK_END=$(date +%Y-%m-%d)

echo "주간 코드 품질 보고서 생성 중 ($WEEK_START ~ $WEEK_END)..."

# 이번 주 변경사항 수집
GIT_LOG=$(git log --since="$WEEK_START" --until="$WEEK_END" --oneline)
GIT_STATS=$(git log --since="$WEEK_START" --until="$WEEK_END" --stat | tail -20)

# AI 분석
REPORT=$(echo "$GIT_LOG
$GIT_STATS" | codex exec -p deep-review -q \
  "이번 주 개발 활동을 분석하여 주요 변경사항, 잠재적 기술 부채, 다음 주 개선 제안을 작성해줘")

# Slack 전송
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"*주간 코드 품질 보고서*\n\`\`\`$REPORT\`\`\`\"}"
```

#### Step 4.3 — 자주 발생하는 문제와 해결

| 증상 | 원인 | 해결 |
|------|------|------|
| `Rate limit exceeded` | API 호출 한도 초과 | `--model gpt-4o-mini`로 변경하거나 요청 간격 조절 |
| 응답이 너무 길어 잘림 | 컨텍스트 윈도우 초과 | 파일을 분할하여 처리 |
| 프로파일을 못 찾음 | config.toml 경로 오류 | `~/.codex/config.toml` 파일 존재 여부 확인 |
| CI에서 OPENAI_API_KEY 오류 | 시크릿 미설정 | GitHub Secrets에 키 등록 |
| 자동 수정이 원치 않은 파일 변경 | approval_policy 설정 | `auto-edit` 대신 `never`로 변경 후 출력만 검토 |

### 체크포인트

사전 커밋 훅이 문제 있는 코드 커밋을 차단하고, GitHub Actions에서 PR 자동 리뷰가 동작하면 성공이다.

---

## 마무리

### 이 가이드에서 배운 것

- `codex exec` = 스크립트/CI에서 사용하는 비대화형 AI 실행
- 승인 정책(`never`, `auto-edit`, `full-auto`)으로 자동화 수준 조절
- `config.toml` 프로파일로 작업 유형별 모델/정책 분리
- stdin 파이프라인으로 기존 CLI 도구와 통합

### 실용적인 자동화 아이디어

| 시나리오 | 프로파일 | 설명 |
|----------|----------|------|
| PR 보안 리뷰 | `deep-review` | PR diff 분석 후 이슈 댓글 |
| 사전 커밋 검사 | `quick-fix` | 명백한 오류만 빠르게 검사 |
| API 문서 생성 | `docs` | 전체 코드베이스 JSDoc 보완 |
| 테스트 자동 생성 | `test-gen` | 새 파일 감지 후 테스트 생성 |
| 주간 품질 보고서 | `deep-review` | Cron 기반 자동 분석 및 Slack 전송 |

---

## 다른 도구와의 비교표

| 기능 | Codex CLI | Claude Code Hooks | Antigravity Workflows | Cursor Automations |
|------|-----------|-------------------|-----------------------|--------------------|
| 이벤트 트리거 | CI/CD 이벤트, 스크립트 | 도구 실행 전/후 | 채팅 명령어 | Schedule, GitHub, Slack |
| 자동화 방식 | 비대화형 단일 실행 | 이벤트 기반 훅 | 저장 프롬프트 파이프라인 | 이벤트 기반 에이전트 |
| 설정 형식 | TOML 프로파일 | JSON hooks | YAML 프론트매터 | GUI 설정 |
| 모델 선택 | 프로파일별 지정 | 세션 설정 따름 | Gemini 고정 | 에이전트 모델 설정 |
| 승인 정책 | never/auto-edit/suggest | 훅 차단으로 간접 구현 | 미지원 | 에이전트 설정 |
| stdin 파이프 | 지원 | 미지원 | 미지원 | 미지원 |
| GitHub Actions | 직접 통합 | 직접 통합 | 제한적 | GitHub 이벤트 트리거 |
