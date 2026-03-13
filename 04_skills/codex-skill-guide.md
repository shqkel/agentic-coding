# Codex CLI Skills 작성 가이드

## 학습 목표

Codex CLI에서 Skills에 해당하는 기능이 무엇인지 이해하고, AGENTS.md를 활용한 재사용 가능한 작업 패턴 정의와 `~/.codex/skills/` 디렉토리를 통한 실험적 Skills 기능을 활용할 수 있다.

## 사전 준비

- Codex CLI 설치 및 기본 사용법 숙지
- OpenAI API 키 설정
- 텍스트 에디터

> **⚠️ 참고**: Codex CLI의 Skills 기능은 2025년 기준 초기 단계이며 일부 기능은 실험적(experimental)이다. 공식 문서와 함께 이 가이드를 참고하되, 세부 동작은 버전에 따라 달라질 수 있다.

---

## 전체 흐름 한눈에 보기

Codex CLI는 명시적인 Skills 시스템 대신, AGENTS.md를 통한 작업 패턴 정의와 `$skill-name` 참조 방식이라는 두 가지 접근법을 제공한다. 또한 실험적 기능으로 `~/.codex/skills/` 디렉토리를 지원한다.

1. **개념 이해** — Codex CLI에서 Skill의 의미와 AGENTS.md와의 관계
2. **AGENTS.md 기반 패턴 정의** — 재사용 가능한 작업 절차 작성
3. **실험적 Skills 기능** — `~/.codex/skills/` 디렉토리 활용
4. **실전 예시** — 실무에서 바로 쓸 수 있는 패턴 작성

---

## Phase 1: Codex CLI에서 Skill의 개념

### 목표

Codex CLI의 Skills 개념을 이해하고 AGENTS.md와 어떻게 연계되는지 파악한다.

### 단계별 구현

#### Step 1.1 — Codex CLI의 구성 파일 비교

> **💡 개념 설명: Codex CLI의 컨텍스트 제공 방식**
>
> Codex CLI는 OpenAI의 Codex 모델(또는 GPT-4 계열)을 터미널에서 사용하는 도구이다. Claude Code의 CLAUDE.md에 해당하는 파일이 AGENTS.md이다. Codex CLI의 Skills는 아직 초기 단계로, Claude Code의 `.claude/commands/`처럼 완성된 시스템은 아니다.
>
> 현재 가장 안정적으로 사용할 수 있는 "Skill 유사 기능"은 두 가지이다:
> 1. AGENTS.md에 작업 패턴을 섹션으로 정의하는 방식
> 2. `~/.codex/skills/` 디렉토리를 활용하는 실험적 방식
>
> **핵심 한 줄:** Codex CLI의 Skills = AGENTS.md 섹션 패턴 + 실험적 skills 디렉토리

| 구분 | AGENTS.md | Skills (`~/.codex/skills/`) |
|------|-----------|---------------------------|
| 상태 | 안정적 | 실험적 |
| 로드 시점 | 항상 | `$skill-name` 참조 시 |
| 범위 | 프로젝트 단위 | 전역 또는 프로젝트 |
| 인자 전달 | 없음 | 제한적 |
| 팀 공유 | 버전 관리 가능 | 개인 설정 |

#### Step 1.2 — Skills 적합 상황 식별

AGENTS.md 섹션으로 정의하기 좋은 케이스:

- 프로젝트 특화 빌드/테스트/배포 절차
- 팀이 공유해야 하는 표준 작업 패턴
- 코드베이스 구조 설명과 함께 제공할 작업 지침

`~/.codex/skills/` 로 정의하기 좋은 케이스:

- 여러 프로젝트에서 재사용하는 개인 작업 패턴
- 도구 특화 설정이 필요 없는 범용 프로세스

### 체크포인트

AGENTS.md와 `~/.codex/skills/`의 차이점을 설명할 수 있는가?

---

## Phase 2: AGENTS.md 기반 작업 패턴 정의

### 목표

AGENTS.md에 구조화된 작업 패턴 섹션을 작성하여 Codex CLI가 일관된 방식으로 반복 작업을 수행하도록 한다.

### 단계별 구현

#### Step 2.1 — AGENTS.md 기본 구조

```markdown
<!-- AGENTS.md (프로젝트 루트) -->

# 프로젝트 에이전트 지침

## 기본 규칙
(항상 적용되는 코딩 스타일, 커밋 규칙 등)

## 작업 패턴

### 패턴: 코드 리뷰
사용자가 "코드 리뷰", "리뷰해줘"라고 요청하면 다음 절차를 따른다:
...

### 패턴: 배포
사용자가 "배포해줘", "deploy"라고 요청하면 다음 절차를 따른다:
...
```

#### Step 2.2 — 작업 패턴 섹션 작성 방법

> **💡 개념 설명: AGENTS.md 작업 패턴의 구조**
>
> Codex CLI는 대화 시작 시 AGENTS.md 전체를 컨텍스트로 로드한다. 따라서 "작업 패턴" 섹션은 항상 로드되지만, 그 내용이 "특정 요청이 왔을 때 이 절차를 따르라"는 조건부 지침이기 때문에 Skill과 유사한 효과를 낸다.
>
> **핵심 한 줄:** AGENTS.md 패턴 = "만약 ~를 요청하면 이렇게 해라"는 조건부 절차 정의

```markdown
<!-- AGENTS.md 내 작업 패턴 섹션 예시 -->

## 작업 패턴

### 패턴: PR 코드 리뷰
트리거: "코드 리뷰", "PR 리뷰", "검토해줘"

절차:
1. 변경된 파일 목록을 확인한다 (`git diff --name-only HEAD~1`)
2. 각 파일의 변경사항을 분석한다
3. 다음 항목을 순서대로 검토한다:
   - 코드 품질 (가독성, 중복, 단일 책임)
   - 보안 (입력 검증, 인증/인가)
   - 성능 (알고리즘 복잡도, DB 쿼리)
   - 테스트 커버리지
4. 결과를 다음 형식으로 출력한다:
   **잘된 점**: (구체적 칭찬)
   **개선 제안**: [파일:라인] 문제 + 해결책
   **즉시 수정**: (보안/버그 등 긴급 항목)

---

### 패턴: 기능 구현
트리거: "만들어줘", "구현해줘", "추가해줘"

절차:
1. 요구사항을 구체화한다 (불명확한 부분은 질문)
2. 구현 계획을 먼저 제시하고 승인을 받는다
3. 테스트 파일을 먼저 작성한다 (TDD)
4. 구현 코드를 작성한다
5. 테스트를 실행하고 결과를 확인한다
6. 필요 시 문서를 업데이트한다

---

### 패턴: 버그 수정
트리거: "버그", "오류", "안 돼", "에러"

절차:
1. 오류 메시지나 증상을 분석한다
2. 재현 가능한 최소 케이스를 작성한다
3. 근본 원인을 찾는다 (로그, 스택 트레이스 분석)
4. 수정사항을 제안하고 사이드 이펙트를 검토한다
5. 수정 후 회귀 테스트를 추가한다
```

#### Step 2.3 — 프로젝트별 AGENTS.md 스코프

```bash
# 프로젝트 루트에 AGENTS.md 생성
touch AGENTS.md

# 서브디렉토리별 특화 지침 (선택)
# src/api/AGENTS.md — API 모듈 전용 지침
# src/tests/AGENTS.md — 테스트 디렉토리 전용 지침
```

Codex CLI는 현재 디렉토리에서 상위로 올라가며 AGENTS.md를 탐색하므로, 서브디렉토리에 특화 지침을 넣을 수 있다:

```markdown
<!-- src/api/AGENTS.md -->

# API 모듈 지침

이 디렉토리의 모든 작업에 추가로 적용된다:

- RESTful 설계 원칙을 준수한다
- 모든 엔드포인트에 OpenAPI 주석을 추가한다
- 에러 응답은 `{ error: string, code: number }` 형식을 사용한다
- 인증이 필요한 엔드포인트는 반드시 JWT 미들웨어를 적용한다
```

### 체크포인트

AGENTS.md에 작업 패턴 섹션을 추가하고, 해당 트리거 키워드로 요청했을 때 지정한 절차를 따르는지 확인했는가?

---

## Phase 3: 실험적 Skills 기능 (`~/.codex/skills/`)

### 목표

Codex CLI의 실험적 Skills 디렉토리 기능을 이해하고, `$skill-name` 참조 방식을 활용한다.

### 단계별 구현

#### Step 3.1 — Skills 디렉토리 구조

> **💡 개념 설명: 실험적 Skills 기능**
>
> `~/.codex/skills/` 디렉토리는 Codex CLI의 초기 버전에서 실험적으로 제공되는 기능이다. 각 Skill은 하나의 Markdown 파일로 정의되며, 대화 중에 `$skill-name` 형식으로 참조할 수 있다. Claude Code의 슬래시 명령처럼 완성된 시스템은 아니지만, 재사용 가능한 지침 번들 개념을 지원한다.
>
> **핵심 한 줄:** `$skill-name` = 미리 정의된 지침 번들을 대화에 주입하는 참조 기호

```bash
# Skills 디렉토리 생성
mkdir -p ~/.codex/skills

# Skills 파일 목록 예시
~/.codex/skills/
├── code-review.md        # $code-review 로 참조
├── performance.md        # $performance 로 참조
├── security-audit.md     # $security-audit 로 참조
└── test-generator.md     # $test-generator 로 참조
```

#### Step 3.2 — Skill 파일 작성

```markdown
<!-- ~/.codex/skills/code-review.md -->

# 코드 리뷰 전문가

당신은 10년 경력의 시니어 소프트웨어 엔지니어이다.
코드를 리뷰할 때 다음 관점에서 분석한다.

## 분석 관점

### 코드 품질
- 가독성: 변수/함수명이 의도를 전달하는가?
- 복잡도: 함수가 50줄을 넘는가? 중첩이 3단계를 초과하는가?
- DRY 원칙: 중복 코드가 있는가?

### 보안
- 입력 검증: 모든 외부 입력이 검증되는가?
- 의존성: 알려진 취약점이 있는 패키지를 사용하는가?
- 민감 정보: API 키, 비밀번호가 하드코딩되어 있는가?

### 성능
- 알고리즘: 더 효율적인 방법이 있는가?
- 쿼리: N+1 문제, 불필요한 전체 탐색이 있는가?

## 피드백 형식

항상 다음 구조로 피드백을 제공한다:
1. 전체 평점 (1-5)
2. 잘된 점 (최소 1개)
3. 개선 제안 (파일명:라인 + 구체적 제안)
4. 즉시 수정 필요 항목 (있는 경우)
```

#### Step 3.3 — Skill 참조 방식

대화 중에 Skill을 참조하는 두 가지 방법:

**방법 1: 명시적 참조**

```
$code-review 이 파일의 코드를 리뷰해줘
```

**방법 2: AGENTS.md에서 참조**

```markdown
<!-- AGENTS.md -->

## 작업 패턴

### 패턴: 코드 리뷰 요청 시
사용자가 코드 리뷰를 요청하면 $code-review skill을 활용하여 분석한다.
```

#### Step 3.4 — Skill 버전 관리

Skills 파일을 dotfiles 저장소에 포함시켜 여러 환경에서 동기화할 수 있다:

```bash
# dotfiles 구조 예시
~/dotfiles/
├── .codex/
│   └── skills/
│       ├── code-review.md
│       └── security-audit.md
└── setup.sh   # 심볼릭 링크 설정 스크립트
```

```bash
# setup.sh 내 심볼릭 링크 생성
ln -sf ~/dotfiles/.codex ~/.codex
```

### 체크포인트

`~/.codex/skills/` 디렉토리에 Skill 파일을 만들고 `$skill-name`으로 참조했을 때 해당 지침이 적용되는지 확인했는가?

---

## Phase 4: 실전 예시

### 목표

실무에서 즉시 활용할 수 있는 AGENTS.md 패턴 2개와 Skills 파일 1개를 작성한다.

### 예시 1: 테스트 생성 패턴 (AGENTS.md)

```markdown
<!-- AGENTS.md 내 패턴 섹션 -->

### 패턴: 테스트 작성
트리거: "테스트 만들어줘", "테스트 추가", "test 작성"

절차:
1. 대상 코드의 public API(함수/메서드 시그니처)를 분석한다
2. 테스트 케이스를 다음 순서로 작성한다:
   a. 정상 케이스 (Happy path) — 예상되는 주요 사용법
   b. 경계값 케이스 — null, undefined, 빈 배열, 최솟값/최댓값
   c. 에러 케이스 — 잘못된 입력, 네트워크 오류, 타임아웃
3. 외부 의존성(DB, API, 파일시스템)은 Mock으로 처리한다
4. 테스트 파일명은 `{원본파일명}.test.{확장자}` 형식을 사용한다
5. 테스트를 실행하여 모두 통과하는지 확인한다

참고: 프로젝트의 테스트 프레임워크(Jest, pytest, Go test 등)를 자동 감지하여 사용한다.
```

### 예시 2: 의존성 업그레이드 패턴 (AGENTS.md)

```markdown
<!-- AGENTS.md 내 패턴 섹션 -->

### 패턴: 의존성 업그레이드
트리거: "의존성 업데이트", "패키지 업그레이드", "dependency update"

절차:
1. 현재 의존성 상태를 확인한다:
   - npm: `npm outdated`
   - pip: `pip list --outdated`
2. 업그레이드 우선순위를 결정한다:
   - 🔴 보안 취약점이 있는 패키지 (즉시)
   - 🟡 메이저 버전 업그레이드 (사이드 이펙트 검토 필요)
   - 🟢 마이너/패치 버전 업그레이드 (안전)
3. 보안 패키지부터 업그레이드하고 테스트를 실행한다
4. 메이저 버전은 CHANGELOG를 확인하고 Breaking Change를 정리한다
5. 전체 테스트 통과 후 커밋한다: `chore: 의존성 업그레이드 (목록)`
```

### 예시 3: 보안 감사 Skill (`~/.codex/skills/security-audit.md`)

```markdown
<!-- ~/.codex/skills/security-audit.md -->

# 보안 감사 전문가

당신은 애플리케이션 보안 전문가이다.
코드 보안 감사를 요청받으면 다음 체크리스트를 기준으로 분석한다.

## OWASP Top 10 체크리스트

### A01 - 접근 제어 취약점
- [ ] 수평적 권한 상승 가능성 (다른 사용자 데이터 접근)
- [ ] 수직적 권한 상승 가능성 (관리자 권한 우회)
- [ ] JWT 토큰 검증이 올바르게 이루어지는가?

### A02 - 암호화 실패
- [ ] 민감 데이터가 평문으로 저장되거나 전송되는가?
- [ ] 강력한 해시 알고리즘 사용 (bcrypt, argon2)?
- [ ] HTTPS 강제 적용이 되어 있는가?

### A03 - 인젝션
- [ ] SQL 쿼리에 사용자 입력이 직접 포함되는가?
- [ ] ORM Parameterized Query 사용 여부
- [ ] Command Injection 가능성

### A05 - 보안 설정 오류
- [ ] 디버그 모드가 프로덕션에서 활성화되어 있는가?
- [ ] 불필요한 HTTP 헤더가 노출되는가?
- [ ] 에러 메시지에 내부 정보가 포함되는가?

## 보고서 형식

발견된 취약점을 다음 형식으로 정리한다:
- **심각도**: Critical / High / Medium / Low
- **위치**: 파일명:라인번호
- **설명**: 취약점 내용
- **PoC**: 공격 시나리오 (개념 증명)
- **수정 방법**: 구체적인 코드 수정 방안
```

### 체크포인트

3가지 예시 패턴/Skill을 실제로 작성하고 각각 호출하여 기대한 결과가 나오는지 확인했는가?

---

## 다른 도구와의 비교

| 기능 | Codex CLI | Claude Code | Gemini CLI (Antigravity) | Cursor |
|------|-----------|-------------|--------------------------|--------|
| Skills 파일 형식 | `.md` | `.md` | `SKILL.md` + 추가 파일 | `.mdc` |
| Skills 위치 | `~/.codex/skills/` (실험적) | `.claude/commands/` | `.agent/skills/<name>/` | `.cursor/rules/` |
| 호출 방식 | `$skill-name` (실험적) | `/명령어` (안정적) | 키워드 자동 매칭 | 에이전트 자동 판단 |
| 규칙 파일 | `AGENTS.md` | `CLAUDE.md` | `GEMINI.md` | `.cursorrules` |
| 안정성 | Skills: 실험적 | Skills: 안정적 | Skills: 안정적 | Agent Rules: 안정적 |
| 팀 공유 | AGENTS.md (버전 관리) | `.claude/commands/` | `.agent/skills/` | `.cursor/rules/` |
| 전역 지원 | `~/.codex/skills/` | `~/.claude/commands/` | `~/.gemini/skills/` | 없음 |

---

## 마무리

### 이 가이드에서 배운 것

- Codex CLI의 Skills는 초기 단계이며 두 가지 접근법이 있음
- AGENTS.md 작업 패턴 섹션으로 안정적인 "Skill 유사 기능" 구현 가능
- `~/.codex/skills/` + `$skill-name` 은 실험적이지만 재사용 가능한 지침 번들 제공
- 팀 공유가 필요한 패턴은 AGENTS.md에, 개인 패턴은 `~/.codex/skills/`에 관리

### 현재 한계와 대안

```markdown
# Codex CLI Skills의 현재 한계
- $skill-name 참조 기능이 모든 버전에서 동작하지 않을 수 있음
- Claude Code의 /명령어 시스템처럼 자동완성이 없음
- 인자 전달 메커니즘이 제한적

# 실용적 대안
1. AGENTS.md의 작업 패턴 섹션 → 가장 안정적
2. 자주 쓰는 프롬프트를 .md 파일로 저장하고 수동으로 붙여넣기
3. Shell alias를 활용한 프롬프트 템플릿
```

### 다음 단계

- `05_workflow` — AGENTS.md 패턴을 활용한 자동화 파이프라인 구성
- Codex CLI 공식 저장소에서 Skills 기능 업데이트 추적
