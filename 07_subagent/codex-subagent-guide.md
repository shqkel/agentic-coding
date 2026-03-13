# Codex CLI 서브에이전트 가이드

## 학습 목표

- Codex CLI의 실험적 멀티에이전트 기능(`--enable multi_agent`)의 현황과 한계를 이해한다
- Codex Cloud를 활용해 장시간 작업을 백그라운드에서 처리하는 방법을 익힌다
- 네이티브 서브에이전트가 제한적인 상황에서 현실적인 병렬 처리 대안을 구성한다
- 프로파일 기반 세션 관리로 다양한 작업 유형을 효과적으로 처리한다

---

## 전체 흐름 한눈에 보기

```
Codex CLI 서브에이전트 전략
─────────────────────────────────────────────────────────────────
[ 실험적 멀티에이전트 ]          [ 현실적 대안 ]
        │                               │
        ▼                               ▼
--enable multi_agent 플래그    여러 터미널 + 프로파일 세션
        │                               │
        ▼                               ▼
  자동 태스크 분배 (제한적)     수동 분배 + 결과 취합
        │
        ▼
[ Codex Cloud (실험적) ]
  클라우드 에이전트로 백그라운드 장시간 처리
  로컬 터미널 불필요
```

---

## Phase 1: 개념 이해

### Codex CLI의 서브에이전트 지원 수준

| 항목 | 내용 |
|------|------|
| 네이티브 서브에이전트 | 실험적 (`--enable multi_agent` 플래그) |
| 병렬 실행 수 | 제한적 (2~3개, 실험적) |
| 클라우드 에이전트 | Codex Cloud (별도 서비스, 실험적) |
| 자동 태스크 분배 | 제한적 |
| 결과 자동 취합 | 제한적 |
| 성숙도 | 낮음 (2025~2026년 기준 초기 단계) |

Codex CLI는 OpenAI가 제공하는 터미널 기반 에이전트 도구로, `codex-1` 모델(GPT-4o 기반 에이전트 특화)을 사용한다. 멀티에이전트 기능은 실험적 단계이며, 실무에서는 여러 터미널 세션과 프로파일 설정을 조합하는 것이 현재 가장 안정적인 방법이다.

### Codex CLI의 구조적 특징

Codex CLI는 세 가지 실행 모드를 지원한다:

**1. 인터랙티브 모드 (기본)**
```bash
codex
# 대화형으로 작업 진행
```

**2. 자동 승인 모드 (에이전트 모드)**
```bash
codex --approval-mode full-auto "auth 모듈 리팩토링해줘"
# 모든 파일 수정과 명령 실행을 자동 승인
```

**3. 샌드박스 모드 (안전 실행)**
```bash
codex --sandbox "테스트 실행하고 결과 알려줘"
# 격리된 환경에서 실행
```

### 멀티에이전트 실험적 기능의 현실

`--enable multi_agent` 플래그는 Codex CLI가 내부적으로 하위 태스크를 별도 에이전트에 위임하는 동작을 활성화한다. 그러나 2025~2026년 기준으로 다음과 같은 제한이 있다:

- 서브에이전트 수: 자동으로 2~3개 수준
- 안정성: 간헐적 오류 발생
- 투명성: 어떤 서브에이전트가 어떤 작업을 하는지 확인 어려움
- 결과 취합: 자동이지만 품질이 일정하지 않음

### 체크포인트 1

- [ ] Codex CLI의 세 가지 실행 모드 차이를 이해했다
- [ ] `--enable multi_agent`가 실험적 기능임을 인지했다
- [ ] Codex Cloud와 로컬 CLI의 차이를 파악했다

---

## Phase 2: 서브에이전트 설정 및 실행

### 실험적 멀티에이전트 활성화

```bash
# 기본 멀티에이전트 실험 활성화
codex --enable multi_agent "복잡한 리팩토링 작업 수행해줘"

# 자동 승인 모드와 함께 사용
codex --enable multi_agent --approval-mode full-auto \
  "auth 모듈을 분석하고 리팩토링해줘. 테스트도 함께 업데이트해줘"

# 특정 디렉토리로 범위 제한
codex --enable multi_agent --workdir src/auth \
  "이 모듈의 모든 함수에 JSDoc 추가해줘"
```

실험적 기능 사용 시 주의사항:
- 중요한 작업 전에 반드시 git commit으로 스냅샷을 남긴다
- `--dry-run` 플래그로 먼저 계획만 확인한다
- 결과를 주의 깊게 검토한다

### 프로파일 기반 세션 구성

Codex CLI는 `~/.codex/config.yaml`을 통해 프로파일을 설정할 수 있다.

```yaml
# ~/.codex/config.yaml
profiles:
  deep-review:
    model: codex-1
    approval_mode: suggest
    context: |
      보안 취약점, 성능 이슈, 코드 품질 문제를 심층 분석한다.
      각 문제를 심각도(critical/high/medium/low)로 분류한다.

  quick-fix:
    model: codex-1-mini
    approval_mode: full-auto
    context: |
      명확한 버그 수정과 간단한 개선 작업을 빠르게 처리한다.
      수정 후 반드시 관련 테스트를 실행한다.

  test-writer:
    model: codex-1
    approval_mode: full-auto
    context: |
      기존 코드를 분석하고 Jest/Vitest 기반 테스트를 작성한다.
      엣지 케이스와 오류 경로를 포함한다.

  doc-writer:
    model: codex-1-mini
    approval_mode: full-auto
    context: |
      코드를 읽고 JSDoc, README, OpenAPI 스펙을 작성한다.
      기술 문서는 영문으로, 사용자 가이드는 한국어로 작성한다.
```

```bash
# 프로파일을 지정하여 실행
codex --profile deep-review "auth 모듈 보안 감사"
codex --profile quick-fix "로그인 버튼 클릭 시 500 에러 수정"
codex --profile test-writer "payment 서비스 통합 테스트 작성"
```

### Codex Cloud 활용

Codex Cloud는 로컬 터미널 없이 클라우드에서 에이전트가 작업을 처리하는 실험적 서비스다.

```bash
# Codex Cloud 에이전트 시작
codex cloud run "전체 API 문서를 OpenAPI 3.0으로 작성해줘"

# 실행 상태 확인
codex cloud status [job-id]

# 결과 다운로드
codex cloud pull [job-id]

# 실행 중인 모든 클라우드 작업 목록
codex cloud list
```

Codex Cloud 사용 시나리오:
- 수 시간이 걸리는 대규모 마이그레이션 작업
- 로컬 머신이 꺼져 있어도 계속 실행해야 하는 작업
- 여러 클라우드 에이전트를 병렬로 시작해야 하는 경우

### 체크포인트 2

- [ ] `--enable multi_agent` 플래그를 실험적 환경에서 테스트해봤다
- [ ] 작업 유형별 프로파일을 `~/.codex/config.yaml`에 설정했다
- [ ] Codex Cloud로 간단한 백그라운드 작업을 시작해봤다

---

## Phase 3: 병렬 처리 패턴

### 패턴 1: 프로파일 기반 병렬 터미널

서로 다른 프로파일로 여러 터미널 세션을 동시에 실행한다.

```bash
# 터미널 1 — 심층 리뷰
codex --profile deep-review --approval-mode suggest \
  "src/auth/ 전체 보안 리뷰"

# 터미널 2 — 빠른 수정
codex --profile quick-fix --approval-mode full-auto \
  "eslint 오류 모두 수정해줘"

# 터미널 3 — 테스트 작성
codex --profile test-writer --approval-mode full-auto \
  "커버리지 없는 파일들에 단위 테스트 추가해줘"
```

### 패턴 2: 실험적 멀티에이전트 + 모니터링

```bash
#!/bin/bash
# codex-multi-agent.sh

echo "멀티에이전트 실험 시작 (실험적 기능)"
echo "git 스냅샷 생성 중..."
git stash

# 멀티에이전트 실행
codex --enable multi_agent \
  --approval-mode full-auto \
  --output-log /tmp/codex_log.txt \
  "다음 세 작업을 분배해서 처리해줘:
   1. src/auth/ 보안 취약점 수정
   2. src/api/ 에러 핸들링 표준화
   3. tests/ 누락된 테스트 추가" \
  2>&1 | tee /tmp/codex_output.txt

echo "실행 완료. 결과 확인:"
cat /tmp/codex_output.txt

echo "변경 사항 검토:"
git diff --stat
```

### 패턴 3: Codex Cloud 병렬 작업

```bash
#!/bin/bash
# codex-cloud-parallel.sh

# 여러 클라우드 작업 동시 시작
echo "클라우드 에이전트 1: 백엔드 마이그레이션"
JOB1=$(codex cloud run --async \
  "Express에서 Fastify로 마이그레이션 - src/routes/ 처리" \
  --output json | jq -r '.job_id')

echo "클라우드 에이전트 2: 프론트엔드 타입 추가"
JOB2=$(codex cloud run --async \
  "PropTypes를 TypeScript 타입으로 변환 - src/components/" \
  --output json | jq -r '.job_id')

echo "클라우드 에이전트 3: 테스트 업데이트"
JOB3=$(codex cloud run --async \
  "Jest에서 Vitest로 마이그레이션 - tests/" \
  --output json | jq -r '.job_id')

echo "실행 중인 작업: $JOB1, $JOB2, $JOB3"
echo "상태 모니터링:"

while true; do
  S1=$(codex cloud status $JOB1 --output json | jq -r '.status')
  S2=$(codex cloud status $JOB2 --output json | jq -r '.status')
  S3=$(codex cloud status $JOB3 --output json | jq -r '.status')

  echo "$(date): [$S1] [$S2] [$S3]"

  if [ "$S1" = "done" ] && [ "$S2" = "done" ] && [ "$S3" = "done" ]; then
    echo "모든 작업 완료!"
    break
  fi

  sleep 30
done

# 결과 가져오기
codex cloud pull $JOB1 --dest ./results/backend/
codex cloud pull $JOB2 --dest ./results/frontend/
codex cloud pull $JOB3 --dest ./results/tests/
```

### 패턴 4: 시퀀셜 파이프라인 (안정적 대안)

멀티에이전트 기능이 불안정할 때 사용하는 안정적인 순차 처리 패턴이다.

```bash
#!/bin/bash
# stable-pipeline.sh

STEP=1

run_step() {
  local desc="$1"
  local cmd="$2"

  echo "=== Step $STEP: $desc ==="
  eval "$cmd"

  if [ $? -ne 0 ]; then
    echo "Step $STEP 실패. 중단."
    exit 1
  fi

  echo "Step $STEP 완료."
  STEP=$((STEP + 1))
}

run_step "코드 분석" \
  'codex --profile deep-review "전체 코드베이스 이슈 분석 후 /tmp/issues.md에 저장해줘"'

run_step "우선순위 결정" \
  'codex "$(cat /tmp/issues.md)를 바탕으로 수정 우선순위를 /tmp/priority.md에 정리해줘"'

run_step "핵심 수정" \
  'codex --profile quick-fix --approval-mode full-auto \
   "$(cat /tmp/priority.md)에서 critical 항목부터 수정해줘"'

run_step "테스트 실행" \
  'codex "수정된 코드에 대해 테스트 실행하고 결과 보고해줘"'

echo "파이프라인 완료."
```

### 체크포인트 3

- [ ] 프로파일 기반 멀티 터미널 실행을 구성했다
- [ ] 실험적 멀티에이전트 실행 시 git stash로 스냅샷을 남기는 습관을 들였다
- [ ] Codex Cloud로 비동기 병렬 작업을 시작하고 모니터링했다

---

## Phase 4: 실전 시나리오

### 시나리오 1: 오픈소스 프로젝트 기여 준비

```bash
#!/bin/bash
# oss-contribution-prep.sh

echo "오픈소스 기여 준비 중..."

# 1. 기여 가이드 분석
codex "CONTRIBUTING.md와 코드 스타일 가이드를 읽고 핵심 규칙을 /tmp/rules.md에 정리해줘"

# 2. 병렬: 내 변경사항 검토
codex --profile deep-review \
  "git diff main HEAD | 기여 규칙 $(cat /tmp/rules.md)에 맞는지 검토해줘" \
  > /tmp/review.txt &

# 3. 병렬: 테스트 추가
codex --profile test-writer --approval-mode full-auto \
  "내가 수정한 함수들에 대한 테스트가 없으면 추가해줘" &

wait

# 4. PR 설명 작성
codex "$(cat /tmp/review.txt)를 바탕으로 GitHub PR 설명 초안을 작성해줘"
```

### 시나리오 2: 레거시 코드 현대화

```bash
#!/bin/bash
# modernize-legacy.sh

# Codex Cloud로 장시간 작업 백그라운드 처리
echo "레거시 코드 현대화 - 클라우드 에이전트 시작"

# CommonJS → ES Modules 변환 (수백 개 파일)
JOB1=$(codex cloud run --async \
  "모든 require()를 ES import로 변환. package.json에 type:module 추가" \
  --output json | jq -r '.job_id')

echo "클라우드 에이전트 실행 중: $JOB1"
echo "로컬에서 다른 작업 계속 가능"

# 로컬에서 별도 작업 진행 (클라우드 에이전트 실행 중)
codex --profile doc-writer --approval-mode full-auto \
  "기존 JSDoc이 없는 공개 API 함수에 JSDoc 추가해줘"

# 클라우드 작업 완료 확인
codex cloud wait $JOB1
echo "ES Modules 변환 완료"
codex cloud pull $JOB1
```

### 시나리오 3: 다국어 지원 추가

```bash
# 터미널 1: 한국어 번역 파일 생성
codex --profile doc-writer --approval-mode full-auto \
  "en.json의 모든 키를 한국어로 번역한 ko.json을 생성해줘"

# 터미널 2: 일본어 번역 파일 생성
codex --profile doc-writer --approval-mode full-auto \
  "en.json의 모든 키를 일본어로 번역한 ja.json을 생성해줘"

# 터미널 3: i18n 컴포넌트 업데이트
codex --approval-mode full-auto \
  "하드코딩된 한국어/일본어 텍스트를 i18n 함수로 교체해줘"
```

---

## 다른 도구와의 비교표

| 항목 | Gemini CLI | Claude Code | Codex CLI | Antigravity | Cursor | VS Code Copilot |
|------|-----------|-------------|-----------|-------------|--------|-----------------|
| 서브에이전트 방식 | 없음 (수동) | Task() 내장 | 실험적 플래그 | Agent Manager | Background VM | Explore + Coding Agent |
| 최대 병렬 수 | 수동 무제한 | 7개 | 제한적 | 제한 없음 | 8개 | 제한적 |
| 클라우드 에이전트 | 없음 | 없음 | Codex Cloud (실험적) | 없음 | Background Agent | Copilot Coding Agent |
| 프로파일/설정 | 있음 | CLAUDE.md | config.yaml | 설정 패널 | 설정 | settings.json |
| 성숙도 | 보통 | 높음 | 낮음 | 높음 | 높음 | 중간 |
| 오픈소스 | 예 (일부) | 아니오 | 예 | 아니오 | 아니오 | 아니오 |
| 로컬 모델 지원 | 예 (일부) | 아니오 | 예 (일부) | 예 | 예 | 아니오 |

---

## 주의사항 및 모범 사례

### 실험적 기능 사용 시 안전 수칙

```bash
# 실험적 멀티에이전트 실행 전 필수 체크리스트

# 1. 현재 상태 저장
git add -A && git commit -m "chore: codex multi-agent 실험 전 스냅샷"

# 2. 변경 범위 확인
git status

# 3. dry-run으로 계획 확인
codex --enable multi_agent --dry-run "리팩토링 계획 확인"

# 4. 실제 실행
codex --enable multi_agent --approval-mode full-auto "리팩토링 실행"

# 5. 결과 검토
git diff --stat
```

### 비용 관리

- Codex Cloud는 실행 시간 기반 과금이므로 장시간 작업은 명확한 범위를 지정한다
- `codex-1-mini` 모델은 간단한 작업(린팅, 포매팅, 문서화)에 사용해 비용을 줄인다
- 실험 단계에서는 `--max-tokens`로 토큰 사용량을 제한한다

```bash
# 비용 절감: 간단한 작업에 mini 모델 사용
codex --model codex-1-mini --approval-mode full-auto \
  "prettier 포매팅 적용해줘"

# 비용 절감: 범위 제한
codex --workdir src/specific-module --enable multi_agent \
  "이 모듈만 리팩토링해줘"
```

### 보안 고려사항

```bash
# 민감한 환경변수가 있는 파일을 .codexignore에 추가
cat >> .codexignore << EOF
.env
.env.local
config/secrets.yaml
credentials/
EOF
```

- `--approval-mode full-auto`는 신뢰할 수 있는 작업에만 사용한다
- CI/CD 환경에서는 `--sandbox` 모드를 기본으로 한다
- 클라우드 에이전트(Codex Cloud)에 전송하는 코드에 API 키가 포함되어 있지 않은지 확인한다

### Codex CLI vs Claude Code 선택 기준

| 상황 | 추천 도구 |
|------|-----------|
| 안정적인 서브에이전트 워크플로우 | Claude Code |
| OpenAI 생태계를 선호 | Codex CLI |
| 로컬 모델과 함께 사용 | Codex CLI |
| 장시간 클라우드 백그라운드 작업 | Codex Cloud |
| 최대 병렬 처리 성능 필요 | Claude Code / Cursor |

### 모범 사례 요약

1. **실험적 기능 = 항상 git 스냅샷 먼저**: `--enable multi_agent` 전에 반드시 커밋
2. **프로파일 활용**: 작업 유형별 설정을 미리 준비
3. **안정성 우선 시**: 멀티에이전트보다 순차 파이프라인이 더 신뢰할 수 있음
4. **클라우드는 장시간 작업에**: 짧은 작업은 로컬 터미널이 더 빠름
5. **로컬 모델 실험**: 민감한 코드는 로컬 모델(llama, deepseek-coder)과 함께 사용

---

## 참고 자료

- [Codex CLI GitHub](https://github.com/openai/codex)
- [Codex CLI 공식 문서](https://platform.openai.com/docs/codex)
- [OpenAI API 멀티에이전트 가이드](https://platform.openai.com/docs/guides/agents)
- [Codex Cloud 베타 페이지](https://platform.openai.com/codex)
