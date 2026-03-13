# Google Antigravity 실습 가이드

## 1. Antigravity란 무엇인가

Google Antigravity는 에디터, 터미널, 브라우저에 걸쳐 복잡한 작업을 자율적으로 계획·실행·검증할 수 있는 에이전트 기반 개발 플랫폼이다. 2025년 11월 Gemini 3 출시와 함께 공개된 무료 공개 프리뷰 도구로, macOS / Windows / Linux를 모두 지원한다.

기존 AI 도구들(Cursor, Windsurf)은 개발자가 주도적으로 코드를 작성하며 AI를 보조 수단으로 사용하는 구조이다. Antigravity는 이 모델을 뒤집어, AI 에이전트가 주도적으로 기능 전체를 계획하고 실행하며 검증하는 에이전트 우선(Agent-First) 구조를 취한다.

**지원 모델:** Gemini 3 Pro (기본값) / Claude Sonnet 4.5 / OpenAI 모델

---

## 2. 핵심 인터페이스 구조

### 2-1. Agent Manager (Mission Control)

Agent Manager는 여러 워크스페이스나 태스크에 걸쳐 비동기로 동작하는 에이전트들을 스폰(spawn)하고 모니터링하는 Mission Control 대시보드 역할을 한다. 각 요청은 전용 에이전트 인스턴스를 스폰하며, UI는 병렬 작업 스트림을 시각화하여 각 에이전트의 상태, 생성한 아티팩트(계획서, 결과물, diff), 보류 중인 승인 요청을 표시한다.

### 2-2. Planning 모드 vs Fast 모드

| 모드 | 설명 | 적합한 상황 |
|------|------|-------------|
| Planning | 실행 전 계획 수립, Task Groups 구성, 아티팩트 생성 | 복잡한 기능 구현, 리서치 |
| Fast | 계획 없이 직접 실행 | 변수명 변경, bash 명령 등 단순 작업 |

---

## 3. 병렬 처리 (Parallel Agent Execution)

### 3-1. 병렬 처리 원리

VS Code에서 긴 빌드 프로세스를 실행하면 터미널이 블로킹된다. 반면 Antigravity의 Agent Manager는 진정한 병렬성을 제공한다. 프론트엔드 코드를 에디터에서 작업하는 동안, 별도의 에이전트를 스폰하여 백엔드 API 리팩터링을 백그라운드에서 독립적으로 처리할 수 있다. 작업이 완료되면 에이전트는 diff와 테스트 결과가 담긴 아티팩트로 알림을 보낸다.

### 3-2. 병렬 처리 실전 시나리오

**시나리오 A: 아이디어 팩토리 방식**
```
에이전트 1: 랜딩 페이지 디자인 시안
에이전트 2: 백엔드 API 구조 설계
에이전트 3: 로고 및 브랜딩 에셋 생성
```

**시나리오 B: 시니어/주니어 역할 분담**
```
메인 에이전트: 복잡한 비즈니스 로직 구현
백그라운드 에이전트: 반복적인 리팩터링 / 의존성 업데이트
```

**시나리오 C: 멀티 버그 동시 수정**
```
에이전트 1: 로그인 버그 수정
에이전트 2: API 응답 속도 최적화
에이전트 3: 테스트 커버리지 확장
```

### 3-3. 주의사항

Antigravity는 유휴 상태에서도 1GB 이상의 메모리를 사용하며 에이전트 작업 중에는 더 높아진다. 에이전트 수는 시스템 사양에 맞게 조절하고, 동일 파일을 여러 에이전트가 동시에 수정하는 상황은 피하는 것이 좋다.

---

## 4. 프롬프트 재사용 시스템 (Rules / Workflows / Skills)

### 4-1. Rules — Claude Code의 CLAUDE.md에 해당

Rules는 에이전트의 행동을 가이드하는 지침이다. 시스템 지시(system instruction)와 유사하며, 코딩 스타일·테스트 정책·커밋 규칙 등을 사전 정의하면 에이전트가 항상 그 기준을 따른다.

| 범위 | 경로 |
|------|------|
| 글로벌 | `~/.gemini/antigravity/rules.md` |
| 워크스페이스 | `<project>/.agent/rules.md` |

**Rules 예시:**
```markdown
# 코딩 스타일
- Python 코드는 항상 type hints를 포함할 것
- 모든 함수에 docstring을 작성할 것

# 테스트
- 새 기능 구현 시 반드시 unit test를 함께 생성할 것
- pytest를 기본 테스트 프레임워크로 사용할 것

# 커밋
- 커밋 메시지는 Conventional Commits 형식을 따를 것
```

### 4-2. Workflows — 저장된 프롬프트 온디맨드 실행

Workflows는 사용자가 `/` 명령을 통해 온디맨드로 트리거할 수 있는 저장된 프롬프트이다. Rules가 항상 적용되는 시스템 지시라면, Workflows는 필요할 때 선택적으로 실행하는 저장 프롬프트이다.

**사용법:** 채팅창에서 `/generate` 입력 → 등록된 workflow 목록 제안 → 선택 후 실행

**Workflow 파일 예시** (`.agent/workflows/generate-unit-tests.md`)
```yaml
---
name: generate-unit-tests
description: 현재 워크스페이스의 모든 함수에 대한 pytest 단위 테스트를 생성한다
---

1. src/ 디렉토리의 모든 Python 파일을 스캔한다
2. 각 함수의 시그니처와 docstring을 분석한다
3. tests/ 디렉토리에 대응하는 테스트 파일을 생성한다
4. 경계값, 정상값, 예외 케이스를 모두 포함한다
```

### 4-3. Skills — 전문화된 지식 패키지 (점진적 공개)

Skills는 특정 요청이 스킬 설명과 매칭될 때만 에이전트 컨텍스트에 로드되는 전문화된 지식 패키지이다. 이 점진적 공개(Progressive Disclosure) 방식은 컨텍스트 과부하와 비용 증가를 방지한다.

**Skills 디렉토리 구조:**
```
.agent/
└── skills/
    └── django-deploy/
        ├── SKILL.md      # 스킬 설명 및 트리거 조건
        ├── instructions.md
        └── scripts/
            └── deploy.sh
```

**SKILL.md 예시:**
```yaml
---
name: django-deploy
description: Django 앱을 AWS Elastic Beanstalk에 배포한다.
             '배포', 'EB 배포', 'AWS 배포' 키워드로 트리거된다.
---

1. requirements.txt 의존성 확인
2. 환경변수 .env.production 검증
3. eb deploy 명령 실행
4. 배포 상태 모니터링
```

| 스킬 범위 | 경로 |
|----------|------|
| 글로벌 | `~/.gemini/antigravity/skills/` |
| 워크스페이스 | `<project>/.agent/skills/` |

---

## 5. Claude Code Hooks와의 비교 및 대응 전략

Antigravity에는 Claude Code의 hooks처럼 특정 이벤트에 자동 반응하는 명시적 훅 시스템은 없다. 하지만 **Rules + Workflows + Skills 조합**으로 유사한 효과를 구현할 수 있다.

### 5-1. 비교표

| 기능 | Claude Code Hooks | Antigravity 대응 방법 |
|------|-------------------|-----------------------|
| 작업 전 자동 실행 | PreToolUse hook | Rules에 '작업 시작 전 체크리스트' 명시 |
| 작업 후 자동 실행 | PostToolUse hook | Rules에 '작업 완료 후 항상 ~을 수행' 명시 |
| 저장 프롬프트 재사용 | 없음 (수동 관리) | Workflows (/명령어)로 즉시 재사용 |
| 컨텍스트 자동 주입 | CLAUDE.md | Rules (global/workspace 분리) |
| 조건부 실행 | hooks 조건 필터링 | Skills (트리거 키워드 기반 자동 로드) |
| 알림 | terminal-notifier 연동 | Agent Manager Inbox 알림 |
| 병렬 실행 | 서브에이전트 Task() | Agent Manager 멀티 에이전트 스폰 |

### 5-2. Hooks를 Rules로 구현하는 방법

Claude Code의 PreToolUse / PostToolUse 패턴을 Rules로 변환한 예시이다.

```markdown
# .agent/rules.md

## 파일 수정 전 반드시 수행
- 수정 대상 파일의 기존 테스트 실행 결과를 먼저 확인할 것
- Git 상태(git status)를 확인하고 현재 브랜치를 보고할 것
- 변경 범위가 3개 파일 이상이면 구현 계획서(Artifact)를 먼저 생성할 것

## 파일 수정 후 반드시 수행
- 수정된 함수에 대한 단위 테스트를 실행할 것
- 린트 오류가 없는지 확인할 것 (flake8 / eslint)
- 변경 사항을 간결한 bullet point로 요약 보고할 것
```

### 5-3. Workflow로 Skills를 연결하는 자동화 파이프라인

Workflow는 여러 스킬을 순서대로 연결하는 파이프라인으로 구성할 수 있다. /deploy 워크플로우가 build → test → push-to-cloud 스킬을 순서대로 호출하는 방식이 그 예시이다.

```yaml
# .agent/workflows/full-cycle.md
---
name: full-cycle
description: 기능 구현부터 배포까지 전체 사이클을 실행한다
---

1. @skill:code-review 를 통해 기존 코드 품질 점검
2. 요청된 기능을 구현한다
3. @skill:generate-tests 를 통해 테스트 자동 생성
4. 테스트 통과 확인 후 @skill:django-deploy 실행
```

---

## 6. 보안 설정

| 모드 | 설명 | 권장 대상 |
|------|------|----------|
| Agent-driven | AI가 모든 것을 자율 실행 | 빠른 MVP 프로토타이핑 |
| Review-driven (권장) | 대부분의 작업 전 승인 요청 | 일반 개발자 |
| Secure Mode | 외부 리소스 접근 및 민감 작업 제한 | 기업/팀 환경 |

`rm -rf`, `sudo`, `pip install --global` 등의 위험 명령은 'Request Human Approval' 설정을 통해 실행 전 반드시 승인을 받도록 구성할 것을 권장한다.

---

## 7. 핵심 단축키 & 명령어

| 동작 | 방법 |
|------|------|
| Antigravity 열기 (CLI) | `agy` |
| 새 에이전트 스폰 | Agent Manager → Start Conversation |
| Workflow 실행 | 채팅창에서 `/workflow명` 입력 |
| 에디터 전환 | 좌측 하단 Editor View 아이콘 |
| 설정 변경 | `Cmd + ,` |
| 컨텍스트 추가 | 채팅창에서 `@` 입력 |

---

## 참고 링크

- 공식 다운로드: https://antigravity.google/download
- 공식 문서: https://antigravity.google/docs/agent-modes-settings
- 구글 코드랩: https://codelabs.developers.google.com/getting-started-google-antigravity
