# Gemini CLI 서브에이전트 가이드

## 학습 목표

- Gemini CLI의 서브에이전트 지원 현황을 정확히 이해한다
- 네이티브 서브에이전트가 없는 상황에서 현실적인 대안 패턴을 익힌다
- 멀티 터미널, 셸 스크립팅, tmux를 활용해 병렬 작업을 구성한다
- 다른 도구(Claude Code 등)와 병용하는 하이브리드 전략을 습득한다

---

## 전체 흐름 한눈에 보기

```
Gemini CLI 서브에이전트 대안 전략
─────────────────────────────────────────────────────────────────
[ 네이티브 서브에이전트 없음 ]
            │
            ▼
┌─────────────────────────────────────────────┐
│  대안 1: 여러 터미널에서 병렬 세션 실행     │
│  대안 2: 셸 스크립팅으로 시퀀셜 자동화     │
│  대안 3: tmux 활용 병렬 관리               │
│  대안 4: 다른 도구(Claude Code)와 병용     │
└─────────────────────────────────────────────┘
            │
            ▼
       수동 결과 취합 및 통합
```

---

## Phase 1: 개념 이해

### Gemini CLI의 서브에이전트 지원 수준

| 항목 | 내용 |
|------|------|
| 네이티브 서브에이전트 | 없음 |
| 병렬 실행 방법 | 여러 터미널 세션으로 수동 관리 |
| 자동 태스크 분배 | 없음 (수동 분배) |
| 결과 자동 취합 | 없음 (수동 취합) |
| 향후 계획 | 공식 로드맵에 멀티에이전트 기능 추가 예정 가능성 있음 |

Gemini CLI는 2024~2025년 기준으로 단일 에이전트 세션 중심으로 설계되어 있다. 메인 에이전트가 서브에이전트를 자동으로 스폰하거나, 여러 에이전트가 서로 메시지를 주고받는 기능은 아직 내장되어 있지 않다.

그러나 Gemini CLI의 강력한 코드 실행 능력(`--sandbox`, `-y` 플래그)과 멀티모달 입력을 활용하면, 외부 자동화 도구와 조합하여 유사한 병렬 작업 흐름을 구성할 수 있다.

### 서브에이전트가 필요한 이유

복잡한 소프트웨어 작업은 단일 에이전트로 처리하기에 한계가 있다:

- **컨텍스트 오염**: 여러 관련 없는 작업이 하나의 컨텍스트에 섞이면 품질 저하
- **시간 비효율**: 순차 실행은 병렬 실행보다 총 소요 시간이 길다
- **책임 분리**: 각 작업 단위를 독립적으로 검토하고 롤백할 수 있어야 한다

### 체크포인트 1

- [ ] Gemini CLI에 네이티브 서브에이전트가 없다는 것을 이해했다
- [ ] 대안 패턴 4가지를 파악했다
- [ ] 서브에이전트가 필요한 상황을 구별할 수 있다

---

## Phase 2: 서브에이전트 설정 및 실행

### 대안 1: 여러 터미널에서 병렬 세션 실행

가장 간단한 방법이다. 터미널을 여러 개 열고 각각 독립적인 Gemini CLI 세션을 실행한다.

```bash
# 터미널 1 — 백엔드 작업
cd /path/to/project
gemini -p "auth 모듈의 JWT 검증 로직을 리팩토링해줘. 보안 취약점도 함께 점검해줘"

# 터미널 2 — 프론트엔드 작업
cd /path/to/project
gemini -p "로그인 컴포넌트의 단위 테스트를 Vitest로 작성해줘"

# 터미널 3 — 문서화 작업
cd /path/to/project
gemini -p "API 엔드포인트 목록을 분석하고 OpenAPI 3.0 스펙 초안을 작성해줘"
```

**장점**: 설정 없이 즉시 실행 가능
**단점**: 각 터미널의 결과를 사람이 직접 취합해야 한다

### 대안 2: 셸 스크립팅으로 시퀀셜 자동화

앞 단계의 출력을 다음 단계의 입력으로 넘기는 파이프라인을 구성한다.

```bash
#!/bin/bash
# sequential-agent.sh

set -e

PROJECT_DIR="/path/to/project"
cd "$PROJECT_DIR"

echo "=== Step 1: 코드 분석 ==="
gemini -p "프로젝트의 주요 모듈 구조를 JSON 형식으로 분석해줘" \
  --output-format json > /tmp/analysis.json

echo "=== Step 2: 리팩토링 계획 수립 ==="
ANALYSIS=$(cat /tmp/analysis.json)
gemini -p "다음 분석 결과를 바탕으로 리팩토링 우선순위를 정해줘: $ANALYSIS" \
  > /tmp/refactor_plan.txt

echo "=== Step 3: 실행 ==="
PLAN=$(cat /tmp/refactor_plan.txt)
gemini -p "다음 계획대로 auth 모듈부터 리팩토링을 시작해줘: $PLAN" -y

echo "=== 완료 ==="
```

```bash
chmod +x sequential-agent.sh
./sequential-agent.sh
```

**장점**: 단계 간 컨텍스트를 전달할 수 있다
**단점**: 순차 실행이므로 병렬 처리 이점 없음

### 대안 3: tmux를 활용한 병렬 세션 관리

tmux를 사용하면 하나의 터미널에서 여러 세션을 관리할 수 있다.

```bash
#!/bin/bash
# parallel-gemini-tmux.sh

SESSION="gemini-agents"

# tmux 세션 생성
tmux new-session -d -s "$SESSION" -n "agent1"
tmux new-window -t "$SESSION" -n "agent2"
tmux new-window -t "$SESSION" -n "agent3"

# 각 윈도우에 작업 전송
tmux send-keys -t "$SESSION:agent1" \
  'gemini -p "백엔드 API 에러 핸들링 로직 개선해줘" -y' Enter

tmux send-keys -t "$SESSION:agent2" \
  'gemini -p "프론트엔드 성능 최적화 포인트를 분석해줘"' Enter

tmux send-keys -t "$SESSION:agent3" \
  'gemini -p "데이터베이스 쿼리 슬로우 쿼리를 찾아 최적화 제안해줘"' Enter

# 세션 연결
tmux attach-session -t "$SESSION"
```

tmux 기본 단축키:
- `Ctrl+b, n`: 다음 윈도우로 이동
- `Ctrl+b, p`: 이전 윈도우로 이동
- `Ctrl+b, w`: 전체 윈도우 목록
- `Ctrl+b, d`: 세션에서 분리 (detach)

### 대안 4: Gemini CLI + Claude Code 병용

Gemini CLI를 탐색/분석 단계에 사용하고, 구현 단계는 Task() 기반 서브에이전트가 있는 Claude Code에 위임하는 하이브리드 전략이다.

```bash
#!/bin/bash
# hybrid-workflow.sh

echo "=== Gemini CLI: 초기 분석 ==="
gemini -p "이 프로젝트에서 가장 복잡한 모듈 3개를 선별하고 리팩토링 방향을 제안해줘" \
  > /tmp/gemini_analysis.txt

echo "=== Claude Code: 병렬 구현 ==="
# Claude Code의 Task() 서브에이전트를 활용해 병렬 실행
ANALYSIS=$(cat /tmp/gemini_analysis.txt)
claude -p "다음 Gemini 분석 결과를 기반으로 3개 모듈을 병렬로 리팩토링해줘: $ANALYSIS"
```

### 체크포인트 2

- [ ] 여러 터미널을 열어 독립적인 Gemini CLI 세션을 실행해봤다
- [ ] 셸 스크립트로 2단계 이상의 시퀀셜 파이프라인을 구성해봤다
- [ ] tmux로 멀티 세션을 동시에 관리해봤다

---

## Phase 3: 병렬 처리 패턴

### 패턴 1: 독립 작업 병렬화

서로 의존성이 없는 작업들은 완전한 병렬 실행이 가능하다.

```bash
#!/bin/bash
# independent-parallel.sh

PROJECT="/path/to/project"

# 백그라운드로 병렬 실행
gemini -p "src/auth/ 디렉토리의 보안 취약점 점검" \
  > /tmp/result_auth.txt 2>&1 &
PID1=$!

gemini -p "src/api/ 디렉토리의 에러 핸들링 개선점 분석" \
  > /tmp/result_api.txt 2>&1 &
PID2=$!

gemini -p "src/db/ 디렉토리의 N+1 쿼리 문제 탐지" \
  > /tmp/result_db.txt 2>&1 &
PID3=$!

# 모든 작업 완료 대기
echo "작업 실행 중..."
wait $PID1 && echo "auth 분석 완료"
wait $PID2 && echo "api 분석 완료"
wait $PID3 && echo "db 분석 완료"

echo ""
echo "=== 통합 결과 ==="
echo "--- AUTH ---"
cat /tmp/result_auth.txt
echo "--- API ---"
cat /tmp/result_api.txt
echo "--- DB ---"
cat /tmp/result_db.txt
```

### 패턴 2: 의존 작업 시퀀스

이전 작업의 결과가 다음 작업의 입력이 되는 경우다.

```bash
#!/bin/bash
# dependent-sequence.sh

# 1단계: 전체 구조 파악
gemini -p "프로젝트 전체 모듈 의존성 그래프를 텍스트로 표현해줘" \
  > /tmp/deps.txt

# 2단계: 병목 지점 식별
gemini -p "다음 의존성 그래프에서 순환 의존이나 과도한 결합이 있는 모듈을 찾아줘:
$(cat /tmp/deps.txt)" > /tmp/bottlenecks.txt

# 3단계: 개선 계획 수립
gemini -p "다음 병목 지점들을 해결하기 위한 단계별 리팩토링 계획을 세워줘:
$(cat /tmp/bottlenecks.txt)" > /tmp/plan.txt

cat /tmp/plan.txt
```

### 패턴 3: 결과 취합 및 충돌 해결

여러 에이전트의 결과를 하나로 통합할 때 충돌을 처리한다.

```bash
#!/bin/bash
# merge-results.sh

# 병렬 분석 실행
gemini -p "auth 모듈 리팩토링 방안 제시" > /tmp/plan_auth.txt &
gemini -p "api 모듈 리팩토링 방안 제시" > /tmp/plan_api.txt &
wait

# 통합 및 충돌 해결
PLAN_AUTH=$(cat /tmp/plan_auth.txt)
PLAN_API=$(cat /tmp/plan_api.txt)

gemini -p "다음 두 리팩토링 계획을 통합하고 충돌이 있으면 해결해줘:
=== AUTH 계획 ===
$PLAN_AUTH

=== API 계획 ===
$PLAN_API"
```

### 체크포인트 3

- [ ] `&`와 `wait`를 활용한 백그라운드 병렬 실행을 구현했다
- [ ] 의존 작업을 순서대로 연결하는 파이프라인을 구성했다
- [ ] 여러 결과를 통합하는 취합 단계를 추가했다

---

## Phase 4: 실전 시나리오

### 시나리오 1: 레거시 코드 분석 및 리팩토링 계획

```bash
#!/bin/bash
# legacy-analysis.sh

echo "레거시 코드 분석 시작..."

# 병렬 분석
gemini -p "이 코드베이스에서 10년 이상 된 레거시 패턴을 찾아줘" \
  > /tmp/legacy.txt &

gemini -p "이 코드베이스의 테스트 커버리지가 낮은 모듈을 파악해줘" \
  > /tmp/coverage.txt &

gemini -p "이 코드베이스에서 deprecated API나 라이브러리를 사용하는 곳을 찾아줘" \
  > /tmp/deprecated.txt &

wait

# 종합 보고서 생성
gemini -p "다음 세 분석 결과를 바탕으로 리팩토링 우선순위 보고서를 작성해줘:

레거시 패턴: $(cat /tmp/legacy.txt)
테스트 커버리지: $(cat /tmp/coverage.txt)
Deprecated 사용: $(cat /tmp/deprecated.txt)

각 항목을 위험도(높음/중간/낮음)와 작업 규모(대/중/소)로 분류해줘." \
  > /tmp/report.txt

echo "분석 완료. 보고서: /tmp/report.txt"
cat /tmp/report.txt
```

### 시나리오 2: 다중 언어 프로젝트 병렬 처리

```bash
#!/bin/bash
# multilang-project.sh

# Python 백엔드 분석
gemini -p "Python FastAPI 백엔드의 타입 힌트 누락과 비동기 처리 문제를 찾아줘" \
  > /tmp/python_issues.txt &

# TypeScript 프론트엔드 분석
gemini -p "TypeScript React 프론트엔드의 타입 오류와 성능 이슈를 찾아줘" \
  > /tmp/ts_issues.txt &

# SQL 스키마 분석
gemini -p "PostgreSQL 스키마에서 인덱스 최적화 기회를 찾아줘" \
  > /tmp/sql_issues.txt &

wait

echo "Python 이슈:" && cat /tmp/python_issues.txt
echo "TypeScript 이슈:" && cat /tmp/ts_issues.txt
echo "SQL 이슈:" && cat /tmp/sql_issues.txt
```

### 시나리오 3: CI/CD 파이프라인 통합

```bash
#!/bin/bash
# ci-gemini-check.sh
# .github/workflows/gemini-review.yml에서 호출

CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)

# 변경된 파일 유형별로 Gemini CLI 검토 병렬 실행
if echo "$CHANGED_FILES" | grep -q "\.py$"; then
  gemini -p "다음 Python 파일들의 변경사항을 코드 리뷰해줘: $CHANGED_FILES" \
    > /tmp/py_review.txt &
fi

if echo "$CHANGED_FILES" | grep -q "\.ts$\|\.tsx$"; then
  gemini -p "다음 TypeScript 파일들의 변경사항을 코드 리뷰해줘: $CHANGED_FILES" \
    > /tmp/ts_review.txt &
fi

wait

# 리뷰 결과를 PR 코멘트로 등록 (GitHub CLI 활용)
if [ -f /tmp/py_review.txt ]; then
  gh pr comment --body "$(cat /tmp/py_review.txt)"
fi
```

---

## 다른 도구와의 비교표

| 항목 | Gemini CLI | Claude Code | Codex CLI | Antigravity | Cursor | VS Code Copilot |
|------|-----------|-------------|-----------|-------------|--------|-----------------|
| 네이티브 서브에이전트 | 없음 | Task() 기반 | 실험적 | Agent Manager | Background Agent | Explore + Coding Agent |
| 병렬 실행 수 | 수동 제한 없음 | 최대 7개 | 제한적 | 제한 없음 | 최대 8개 | 제한적 |
| 자동 태스크 분배 | 없음 | 있음 | 부분적 | 있음 | 있음 | 있음 |
| UI | CLI | CLI | CLI | GUI | GUI + CLI | GUI |
| 결과 자동 취합 | 없음 | 있음 | 없음 | 있음 | 있음 | 있음 |
| 설정 난이도 | 낮음 (없음) | 중간 | 낮음 | 낮음 | 낮음 | 낮음 |
| 비용 효율 | 높음 | 모델 선택 가능 | 높음 | 중간 | 중간 | GitHub 구독 포함 |

---

## 주의사항 및 모범 사례

### 비용 관리

```bash
# 비용 절감: 짧고 명확한 프롬프트 사용
# 비추천
gemini -p "이 전체 프로젝트를 모두 분석하고 개선해줘"

# 추천: 범위를 명확히 지정
gemini -p "src/auth/jwt.py 파일만 보안 취약점 점검해줘"
```

- 여러 터미널 세션을 동시에 실행하면 API 비용이 비례해서 증가한다
- 분석 범위를 구체적인 파일이나 디렉토리로 좁혀서 불필요한 토큰 소비를 줄인다
- `--model gemini-flash` 옵션으로 경량 모델을 사용하면 비용을 크게 줄일 수 있다

### 보안 고려사항

```bash
# 민감한 정보가 포함된 파일을 프롬프트에 직접 포함하지 않는다
# 비추천
gemini -p "다음 코드 분석해줘: $(cat config/secrets.yaml)"

# 추천: 민감 정보를 제거하고 구조만 공유
gemini -p "config/ 디렉토리의 설정 파일 구조를 분석해줘 (실제 값은 제외)"
```

- API 키, 비밀번호, 토큰 등이 포함된 파일을 직접 프롬프트에 포함하지 않는다
- `gemini -y` 플래그(자동 승인)는 신뢰할 수 있는 환경에서만 사용한다
- CI/CD 환경에서는 실행 권한을 최소화한다

### 리소스 관리

```bash
# 병렬 세션 수를 제한하여 시스템 부하를 관리한다
MAX_PARALLEL=3
CURRENT=0

for task in "${TASKS[@]}"; do
  if [ $CURRENT -ge $MAX_PARALLEL ]; then
    wait -n  # 하나가 끝날 때까지 대기
    CURRENT=$((CURRENT - 1))
  fi
  gemini -p "$task" > "/tmp/result_${CURRENT}.txt" &
  CURRENT=$((CURRENT + 1))
done

wait
```

### 향후 전망

Gemini CLI는 2025~2026년 사이에 다음 기능이 추가될 가능성이 있다:

- **네이티브 멀티에이전트 지원**: Gemini 2.x 모델의 멀티에이전트 API가 CLI에 통합될 수 있다
- **Gemini Workspace Agents**: Google Workspace와 연동된 에이전트 자동화
- **Gemini API의 Agentic 기능 확장**: `google-adk` (Agent Development Kit)의 기능이 CLI에 반영될 가능성

현재 시점에서는 복잡한 멀티에이전트 워크플로우가 필요하다면 Claude Code의 Task() 기반 서브에이전트와 병용하는 것이 가장 현실적인 전략이다.

### 모범 사례 요약

1. **단순 작업**: 단일 Gemini CLI 세션으로 충분하다
2. **독립적 병렬 작업**: 여러 터미널 + 백그라운드 프로세스(`&`)
3. **순차 의존 작업**: 셸 스크립트 파이프라인
4. **복잡한 멀티에이전트 작업**: Claude Code로 위임하거나, Antigravity 사용
5. **장시간 백그라운드 작업**: Cursor Background Agent 또는 Copilot Coding Agent 사용

---

## 참고 자료

- [Gemini CLI 공식 문서](https://github.com/google-gemini/gemini-cli)
- [Gemini API 멀티에이전트 가이드](https://ai.google.dev/gemini-api/docs/agents)
- [tmux 공식 위키](https://github.com/tmux/tmux/wiki)
- [Google Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
