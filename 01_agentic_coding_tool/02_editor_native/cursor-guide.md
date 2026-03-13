# Cursor AI 에디터 실습 가이드

> 작성일: 2026-03-12

## 개요

### Cursor란 무엇인가

Cursor는 AI-First 철학으로 설계된 코드 에디터이다. VS Code를 포크하여 만들어졌기 때문에 기존 VS Code 확장과 설정을 그대로 사용할 수 있으면서도, AI 기능이 에디터의 핵심 아키텍처에 깊이 통합되어 있다. 단순히 AI 플러그인을 추가한 것이 아니라, 편집 경험 자체가 AI를 중심으로 재설계된 도구이다.

### AI-First 에디터 철학

기존 도구들(VS Code + GitHub Copilot 등)은 기존 에디터에 AI를 "부착"하는 방식이다. Cursor는 이를 뒤집어, AI가 코드 편집의 1급 시민(first-class citizen)이 되도록 설계했다. Tab 자동완성, 인라인 편집, 멀티파일 에이전트까지 모든 워크플로우가 AI와의 협업을 전제로 구성된다.

### VS Code + Copilot과의 핵심 차별점

| 항목 | VS Code + Copilot | Cursor |
|------|-------------------|--------|
| AI 통합 방식 | 플러그인 기반 (부착형) | 에디터 코어에 내장 |
| 자동완성 | 단일 라인 중심 | 멀티라인, 편집 의도 예측 (Cursor Tab) |
| 멀티파일 편집 | 수동으로 파일 전환 | Composer가 여러 파일을 동시 편집 |
| 에이전트 모드 | 제한적 | Agent 모드로 자율적 계획·실행·검증 |
| 코드베이스 이해 | 열린 파일 중심 | `@codebase`로 전체 인덱싱 및 시맨틱 검색 |
| 모델 선택 | GitHub Copilot 모델 고정 | Claude, GPT, Gemini, 커스텀 모델 자유 선택 |
| 백그라운드 에이전트 | 미지원 | 격리된 VM에서 비동기 작업 수행 |
| VS Code 호환성 | 네이티브 | 확장·설정·키바인딩 100% 호환 |

---

## Phase 1: 설치 및 설정

### 1-1. 다운로드 및 설치

공식 사이트에서 운영체제에 맞는 설치 파일을 다운로드한다.

```
https://cursor.com/download
```

- **지원 OS**: macOS, Windows, Linux
- **요구사항**: 64비트 OS, 4GB 이상 RAM (8GB 권장)

설치 후 처음 실행하면 VS Code 설정 가져오기 옵션이 표시된다.

### 1-2. VS Code 설정 및 확장 가져오기

Cursor는 VS Code 포크이므로, 기존 환경을 그대로 이전할 수 있다.

```
Cursor 첫 실행 → "Import VS Code Settings" 선택
```

가져올 수 있는 항목:
- 확장(Extensions) 전체
- 키바인딩(Keybindings)
- 사용자 설정(Settings)
- 테마(Theme)
- 스니펫(Snippets)

> 💡 **개념 설명**: Cursor는 VS Code와 동일한 확장 마켓플레이스를 사용한다. 기존에 사용하던 ESLint, Prettier, GitLens 등 모든 확장이 그대로 동작한다. VS Code와 Cursor를 동시에 설치하여 운영할 수도 있다.

### 1-3. 계정 설정 및 구독

Cursor는 무료 플랜과 유료 플랜을 제공한다.

| 플랜 | 가격 | 주요 제공 |
|------|------|----------|
| **Free** | 무료 | 자동완성 2,000회 + 프리미엄 요청 50회 |
| **Pro** | $20/월 | 무제한 Tab 완성, 확장 에이전트 요청, 백그라운드 에이전트 |
| **Pro+** | $60/월 | 더 많은 크레딧, 우선 처리 |
| **Ultra** | $200/월 | 최대 크레딧 풀, 최고 우선순위 |
| **Teams** | $40/유저/월 | 팀 관리, 중앙 설정, 관리자 대시보드 |
| **Enterprise** | 별도 문의 | SSO, 감사 로그, 전용 인프라 |

> 💡 **개념 설명**: 2025년 6월부터 Cursor는 요청 기반에서 **크레딧 기반 과금**으로 전환했다. 유료 플랜의 월 가격(달러)이 곧 월간 크레딧 풀이 되며, 사용하는 AI 모델에 따라 크레딧 소모량이 달라진다. **Auto 모드**를 활성화하면 작업 난이도에 따라 가장 비용 효율적인 모델을 자동 선택한다.

### 1-4. 모델 설정

Cursor Settings > Models에서 사용할 AI 모델을 선택한다.

**기본 제공 모델:**
- Claude Sonnet 4.5 / Claude Opus
- GPT-5.3 Codex / GPT-4o
- Gemini 3 Pro
- Cursor Composer (자체 모델)

**Auto 모드:**
```
Settings → Models → Auto-select 활성화
```

Auto 모드를 켜면 Cursor가 작업 복잡도에 따라 최적의 모델을 자동 선택한다. 간단한 버그 수정에는 경량 모델을, 복잡한 아키텍처 설계에는 고성능 모델을 배정한다.

**커스텀 모델 연동:**

OpenAI 호환 API 엔드포인트가 있으면 어떤 모델이든 연결할 수 있다.

```
Settings → Models → Add Custom Model
→ API Base URL 입력
→ API Key 입력
→ Model Name 지정
```

> 💡 **개념 설명**: 커스텀 모델은 Cursor의 크레딧 풀을 소모하지 않고 직접 API 비용을 부담한다. 단, Tab 자동완성 등 일부 기능은 커스텀 모델에서 지원되지 않을 수 있다.

### 1-5. Privacy Mode (프라이버시 모드)

민감한 코드를 다루는 경우 프라이버시 모드를 활성화한다.

```
Settings → Privacy Mode → 활성화
```

프라이버시 모드가 켜지면:
- Cursor와 모델 제공사 모두 코드를 저장하지 않는다
- 요청 처리 후 모든 데이터가 즉시 삭제된다
- 의료, 금융, 기업 환경에서 권장된다

---

### Phase 1 체크포인트

- [ ] Cursor를 설치하고 VS Code 설정을 가져왔는가?
- [ ] 계정을 생성하고 원하는 플랜을 선택했는가?
- [ ] 선호하는 AI 모델을 설정했는가?
- [ ] 프라이버시 모드 필요 여부를 검토했는가?

---

## Phase 2: 기본 사용법

### 2-1. Cursor Tab — 지능형 자동완성

Cursor Tab은 일반적인 코드 자동완성을 넘어서는 핵심 차별 기능이다.

**특징:**
- **멀티라인 예측**: 한 줄이 아닌 여러 줄의 코드를 한번에 제안
- **편집 의도 파악**: 커서 위치와 최근 변경 내역을 분석하여 다음 편집을 예측
- **오류 자동 수정**: 타이포나 경미한 문법 오류를 감지하고 자동 교정 제안
- **컨텍스트 인식**: 현재 파일뿐 아니라 관련 파일들의 패턴도 반영

**사용법:**

```
코드 작성 중 → 회색 제안 텍스트 표시 → Tab 키로 수락
```

```python
# 예: 함수 시그니처만 작성하면 Cursor Tab이 전체 구현을 제안한다
def calculate_monthly_interest(principal: float, annual_rate: float, months: int) -> float:
    # ← 여기서 Tab을 누르면 전체 함수 본문이 자동 생성됨
```

> 💡 **개념 설명**: Cursor Tab은 GitHub Copilot의 자동완성과 다르다. Copilot이 주로 "다음에 올 코드"를 예측하는 반면, Cursor Tab은 "개발자가 하려는 편집 행위"를 예측한다. 예를 들어 변수명을 하나 변경하면, 관련된 다른 위치의 변수명 변경까지 제안한다.

### 2-2. Cmd+K — 인라인 편집

코드를 선택하고 `Cmd+K` (Windows: `Ctrl+K`)를 누르면 인라인 편집 프롬프트가 열린다.

**사용법:**

```
1. 코드 블록 선택 (또는 커서만 위치)
2. Cmd+K 입력
3. 프롬프트 창에 변경 요청 작성
4. 결과를 diff로 확인 후 수락/거절
```

**활용 예시:**

```python
# 선택 후 Cmd+K → "에러 핸들링 추가해줘"

# 변경 전
def fetch_data(url):
    response = requests.get(url)
    return response.json()

# 변경 후 (diff로 표시됨)
def fetch_data(url):
    try:
        response = requests.get(url, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        logger.error(f"Failed to fetch data from {url}: {e}")
        raise
```

> 💡 **개념 설명**: Cmd+K는 소규모 편집에 최적화되어 있다. 한 파일 내에서 빠르게 코드를 수정하거나 생성할 때 사용한다. 여러 파일에 걸친 대규모 변경은 Composer(Cmd+I)를 사용하는 것이 효과적이다.

### 2-3. Chat 패널 (Cmd+L)

`Cmd+L` (Windows: `Ctrl+L`)로 사이드 Chat 패널을 열 수 있다.

**주요 용도:**
- 코드에 대한 질문 및 설명 요청
- 디버깅 도움
- 아키텍처 설계 토론
- 문서 참조 및 학습

```
Cmd+L → "이 함수의 시간 복잡도를 분석해줘" 입력
```

Chat은 현재 열린 파일과 선택된 코드를 자동으로 컨텍스트에 포함한다. `@` 멘션으로 추가 컨텍스트를 명시적으로 지정할 수도 있다.

### 2-4. Composer (Cmd+I) — 멀티파일 편집

`Cmd+I` (Windows: `Ctrl+I`)로 Composer를 열면 여러 파일에 걸친 대규모 편집을 수행할 수 있다.

**Composer의 핵심 강점:**
- 여러 파일을 동시에 생성/수정/삭제
- 각 파일의 변경사항을 시각적 diff로 표시
- 변경사항을 파일별로 수락/거절 가능
- 터미널 명령 실행 가능

**사용 예시:**

```
Cmd+I → "Express API에 사용자 인증 엔드포인트를 추가해줘.
         JWT 토큰 발급, 검증 미들웨어, 로그인/회원가입 라우트를 포함하고
         해당 테스트 파일도 생성해줘."
```

Composer가 수행하는 작업:
```
✅ src/middleware/auth.ts    (신규 생성)
✅ src/routes/auth.ts        (신규 생성)
✅ src/services/authService.ts (신규 생성)
📝 src/app.ts               (라우트 등록 추가)
✅ tests/auth.test.ts        (신규 생성)
```

각 파일의 변경사항이 diff 형태로 표시되며, 파일별로 개별 수락/거절을 결정할 수 있다.

> 💡 **개념 설명**: Chat(Cmd+L)은 "대화와 질문"에, Cmd+K는 "인라인 빠른 수정"에, Composer(Cmd+I)는 "멀티파일 대규모 편집"에 각각 최적화되어 있다. 작업 규모에 맞는 도구를 선택하면 효율이 극대화된다.

### 2-5. @ 멘션 — 컨텍스트 지정

Chat과 Composer의 프롬프트에서 `@`를 입력하면 다양한 컨텍스트 소스를 지정할 수 있다.

| 멘션 | 설명 | 사용 예시 |
|------|------|----------|
| `@file` | 특정 파일 참조 | `@src/utils/auth.ts 이 파일 리팩터링해줘` |
| `@codebase` | 전체 코드베이스 시맨틱 검색 | `@codebase 인증 관련 코드 모두 찾아줘` |
| `@web` | 웹 검색 결과 참조 | `@web React 19의 새로운 기능 알려줘` |
| `@docs` | 외부 문서 참조 | `@docs Next.js 라우팅 API 사용법` |
| `@git` | Git 히스토리 참조 | `@git 최근 커밋에서 변경된 내용 설명해줘` |
| `@folder` | 특정 폴더 전체 참조 | `@src/components 이 폴더 구조 분석해줘` |
| `@definition` | 심볼 정의 참조 | `@definition UserService 클래스 구조 보여줘` |
| `@notepad` | 노트패드 참조 | `@notepad:api-spec 이 스펙대로 구현해줘` |

> 💡 **개념 설명**: `@codebase`는 Cursor의 가장 강력한 차별 기능 중 하나이다. 프로젝트를 인덱싱하여 시맨틱 검색을 수행하므로, 파일 이름을 모르더라도 "인증 관련 코드" 같은 의미 기반 검색이 가능하다. 대규모 코드베이스에서 관련 코드를 빠르게 찾을 때 매우 유용하다.

### 2-6. Notepads — 영구 컨텍스트 저장

Notepads는 자주 재사용하는 프롬프트, API 스펙, 프로젝트 규칙 등을 영구적으로 저장하고 `@notepad`로 참조할 수 있는 기능이다.

```
Cursor 사이드바 → Notepads → "+" 버튼 → 노트패드 생성
```

**활용 예시:**

```markdown
# 노트패드 이름: api-spec
# API 응답 형식 규칙

모든 API 응답은 다음 형식을 따른다:

{
  "success": boolean,
  "data": object | null,
  "error": { "code": string, "message": string } | null,
  "metadata": { "timestamp": string, "requestId": string }
}
```

프롬프트에서 `@notepad:api-spec`으로 참조하면 매번 같은 내용을 반복 입력할 필요가 없다.

---

### Phase 2 체크포인트

- [ ] Cursor Tab으로 멀티라인 자동완성을 사용해 봤는가?
- [ ] Cmd+K로 인라인 편집을 수행했는가?
- [ ] Cmd+L Chat에서 코드에 대해 질문했는가?
- [ ] Cmd+I Composer로 여러 파일을 동시에 편집했는가?
- [ ] @ 멘션으로 컨텍스트를 지정하여 더 정확한 결과를 얻었는가?

---

## Phase 3: Agent Mode (Composer Agent)

### 3-1. Agent Mode란

Agent Mode는 Composer의 확장 모드로, AI가 단순히 코드를 제안하는 것을 넘어 **자율적으로 계획을 수립하고, 파일을 편집하고, 터미널 명령을 실행하며, 결과를 검증**하는 모드이다.

**일반 Composer와의 차이:**

| 항목 | Composer (일반) | Agent Mode |
|------|----------------|------------|
| 파일 편집 | 제안 후 사용자 수락 | 자율적으로 편집 |
| 터미널 명령 | 수동 실행 | 자동 실행 (설치, 빌드, 테스트 등) |
| 컨텍스트 수집 | 사용자가 @ 멘션으로 지정 | 서브에이전트가 병렬로 코드베이스 탐색 |
| 오류 대응 | 사용자에게 보고 | 자동으로 오류 분석 후 수정 시도 |
| 작업 범위 | 한 번의 편집 | 복잡한 멀티스텝 작업 |

**활성화 방법:**

```
Cmd+I → 프롬프트 창 하단의 모드 선택 → "Agent" 선택
```

### 3-2. 자율적 파일 편집 및 터미널 실행

Agent Mode에서 작업을 요청하면:

```
사용자: "Next.js 프로젝트에 다크모드 토글 기능을 추가해줘"

Agent 수행 과정:
1. @codebase로 현재 테마 관련 코드 탐색
2. 테마 컨텍스트 프로바이더 생성 (src/context/ThemeContext.tsx)
3. 토글 컴포넌트 생성 (src/components/ThemeToggle.tsx)
4. 레이아웃 파일에 프로바이더 래핑 추가
5. CSS 변수 기반 다크모드 스타일 추가
6. `npm run build` 실행하여 빌드 검증
7. 오류 발생 시 자동 수정 후 재빌드
```

### 3-3. Auto-apply vs Review 모드

| 모드 | 동작 | 권장 상황 |
|------|------|----------|
| Auto-apply | 변경사항을 자동으로 적용 | 신뢰할 수 있는 반복 작업, 프로토타이핑 |
| Review (기본값) | 각 변경사항을 diff로 표시 후 수락/거절 | 프로덕션 코드, 처음 사용하는 경우 |

```
Settings → Cursor → Agent → Auto-apply 토글
```

> 💡 **개념 설명**: Claude Code에서 `--dangerously-skip-permissions`로 YOLO 모드를 활성화하는 것처럼, Cursor의 Auto-apply도 AI의 자율성을 높이는 설정이다. 프로덕션 코드에서는 Review 모드를 유지하고, 실험적 프로토타이핑에서만 Auto-apply를 사용하는 것을 권장한다.

### 3-4. Background Agents — 비동기 작업 실행

Background Agents는 Cursor의 강력한 차별 기능으로, **격리된 클라우드 VM에서 비동기적으로 작업을 수행**한다.

**핵심 특징:**
- 각 에이전트가 독립된 Ubuntu VM에서 실행
- 인터넷 접근 가능 (패키지 설치, API 호출 등)
- 별도 브랜치에서 작업 후 PR 생성
- 최대 8개 에이전트를 병렬 실행 가능
- Cursor, Slack, 웹/모바일에서 시작 가능

**사용 예시:**

```
Cursor 내에서:
  Cmd+I → "Background Agent로 실행" 선택
  → "전체 API 엔드포인트에 대한 통합 테스트를 작성해줘"

Slack에서:
  @cursor "users 테이블에 email 인덱스 추가하고 마이그레이션 PR 올려줘"
```

에이전트가 완료되면:
- 별도 브랜치에 변경사항 커밋
- PR(Pull Request) 자동 생성
- 스크린샷, 로그 등 아티팩트 첨부
- 알림으로 결과 보고

**Git Worktree 기반 격리:**

여러 에이전트가 동시에 같은 프로젝트에서 작업할 때, Git worktree를 사용하여 파일 충돌을 방지한다. 각 에이전트가 코드베이스의 독립된 복사본에서 작업한다.

### 3-5. Automations — 이벤트 기반 자동 에이전트

Automations는 특정 이벤트가 발생할 때 자동으로 에이전트를 실행하는 기능이다.

**지원하는 트리거:**
- **스케줄**: 매일/매주 정해진 시간에 실행
- **GitHub**: PR 생성, 이슈 등록 시
- **Slack**: 특정 채널 메시지
- **Linear**: 태스크 할당 시
- **PagerDuty**: 인시던트 발생 시
- **Webhook**: 커스텀 이벤트

```
예시: GitHub에 이슈가 등록되면 자동으로 에이전트가 분석 후 구현 PR 생성
```

### 3-6. Claude Code 권한 모드와의 비교

| Claude Code | Cursor Agent Mode |
|------------|-------------------|
| 기본 모드 (매번 승인) | Review 모드 (diff 확인 후 수락) |
| `--allowedTools` (특정 도구 자동 승인) | Auto-apply (변경사항 자동 적용) |
| YOLO 모드 (전체 자동 승인) | Agent + Auto-apply (전면 자율) |
| 서브에이전트 `Task()` | Background Agents (클라우드 VM) |

---

### Phase 3 체크포인트

- [ ] Agent 모드로 멀티스텝 작업을 수행해 봤는가?
- [ ] Auto-apply와 Review 모드의 차이를 이해했는가?
- [ ] Background Agent로 비동기 작업을 시작해 봤는가?
- [ ] 병렬 에이전트 실행의 Git Worktree 격리 개념을 이해했는가?

---

## Phase 4: 설정 파일 위치

### 4-1. .cursorrules (레거시 — 프로젝트 룰)

프로젝트 루트에 `.cursorrules` 파일을 생성하면 AI의 동작을 제어하는 규칙을 정의할 수 있다.

```markdown
# .cursorrules (레거시 형식)

## 코딩 규칙
- TypeScript strict 모드 사용
- any 타입 사용 금지
- 모든 함수에 JSDoc 주석 작성

## 테스트
- Jest를 사용하여 단위 테스트 작성
- 커버리지 80% 이상 유지

## 커밋
- Conventional Commits 형식 준수
- 커밋 메시지는 한국어로 작성
```

> 💡 **개념 설명**: `.cursorrules`는 Claude Code의 `CLAUDE.md`에 대응하는 개념이다. 프로젝트의 코딩 규칙, 스타일 가이드, 아키텍처 원칙을 정의하면 AI가 이를 따르도록 한다. 단, `.cursorrules`는 **레거시 형식**으로 향후 지원이 중단될 예정이며, 새 프로젝트는 `.cursor/rules/` 디렉토리 형식을 사용할 것을 권장한다.

### 4-2. .cursor/rules/ 디렉토리 (신규 형식)

`.cursor/rules/` 디렉토리에 `.mdc` 파일들을 생성하여 규칙을 카테고리별로 분리 관리한다.

```
<project-root>/
└── .cursor/
    └── rules/
        ├── coding-style.mdc
        ├── testing.mdc
        ├── git-conventions.mdc
        └── security.mdc
```

**MDC 파일 예시:**

```markdown
---
description: TypeScript 코딩 스타일 규칙
globs: "**/*.ts,**/*.tsx"
alwaysApply: false
---

# TypeScript 코딩 스타일

- strict 모드를 사용한다
- any 타입 대신 unknown 또는 구체적 타입을 사용한다
- 인터페이스 이름에 I 접두사를 사용하지 않는다
- 열거형(enum) 대신 const 객체와 타입 유니온을 사용한다
- 함수형 프로그래밍 패턴을 선호한다
```

**MDC 메타데이터 필드:**

| 필드 | 설명 |
|------|------|
| `description` | 규칙 설명 (에이전트가 관련성 판단에 사용) |
| `globs` | 규칙이 적용될 파일 패턴 |
| `alwaysApply` | `true`이면 항상 적용, `false`이면 관련 시에만 적용 |

> 💡 **개념 설명**: `.mdc` 형식의 장점은 규칙을 **파일 패턴별로 분리**할 수 있다는 것이다. TypeScript 파일에만 적용될 규칙, 테스트 파일에만 적용될 규칙을 별도로 관리할 수 있어 컨텍스트 오버헤드를 줄인다. Claude Code의 서브디렉토리별 `CLAUDE.md`와 유사한 효과를 낸다.

### 4-3. Cursor Settings UI

에디터 내에서 `Cmd+,`로 설정 UI에 접근한다.

주요 설정 항목:
- **Models**: AI 모델 선택 및 Auto 모드
- **Rules**: 전역 AI 규칙 (UI에서 직접 편집)
- **Privacy Mode**: 코드 저장 비활성화
- **Features**: Tab, Cmd+K, Agent 등 개별 기능 설정
- **MCP**: MCP 서버 관리

### 4-4. ~/.cursor/ — 전역 설정

사용자 전역 설정이 저장되는 디렉토리이다.

```
~/.cursor/
├── settings/          # 전역 사용자 설정
├── extensions/        # 설치된 확장
└── ...
```

### 4-5. 프로젝트 수준 .cursor/ 디렉토리

```
<project-root>/
└── .cursor/
    ├── rules/         # 프로젝트 규칙 (.mdc 파일)
    └── mcp.json       # 프로젝트별 MCP 서버 설정
```

### 4-6. 설정 파일 비교표 — 다른 도구와의 대응

| 범위 | Cursor | Claude Code | Gemini CLI |
|------|--------|------------|------------|
| 전역 규칙 | Settings UI / ~/.cursor/ | ~/.claude/CLAUDE.md | ~/.gemini/GEMINI.md |
| 프로젝트 규칙 | .cursor/rules/*.mdc | 프로젝트 루트 CLAUDE.md | .gemini/GEMINI.md |
| 레거시 프로젝트 규칙 | .cursorrules | — | — |
| 컴포넌트 규칙 | .mdc의 globs 패턴 | 서브디렉토리 CLAUDE.md | 서브디렉토리 GEMINI.md |
| 제외 파일 | .cursorignore | .claudeignore | .geminiignore |

---

### Phase 4 체크포인트

- [ ] `.cursor/rules/` 디렉토리에 프로젝트 규칙을 생성했는가?
- [ ] globs 패턴으로 파일 유형별 규칙을 분리했는가?
- [ ] 기존 `.cursorrules`가 있다면 새 형식으로 마이그레이션했는가?

---

## Phase 5: 핵심 기능 미리보기

이 섹션에서 다루는 각 주제는 별도의 심화 가이드에서 자세히 다룬다. 여기서는 Cursor에서 각 기능이 어떻게 동작하는지 개요만 소개한다.

### 5-1. Rules — 프로젝트 규칙 시스템 → 02_rule 가이드

Cursor의 규칙 시스템은 `.cursor/rules/*.mdc` 파일로 AI의 동작을 제어한다.

| 규칙 유형 | 설명 |
|----------|------|
| Always | `alwaysApply: true` — 모든 대화에 항상 포함 |
| Auto | `alwaysApply: false` + `globs` 패턴 — 해당 파일 작업 시 자동 포함 |
| Agent Requested | `description`만 설정 — 에이전트가 관련성 판단 후 선택적 로드 |
| Manual | 사용자가 `@rules`로 명시적 참조 시에만 포함 |

**Claude Code 대응**: `.cursor/rules/*.mdc` ↔ `CLAUDE.md` (프로젝트/서브디렉토리별)

자세한 내용은 [02_rule 가이드](../../02_rule/)를 참조한다.

### 5-2. MCP 서버 연동 → 03_mcp 가이드

Cursor는 MCP(Model Context Protocol)를 통해 외부 도구와 데이터 소스를 Agent에 연결한다.

**설정 위치:**
- 프로젝트 수준: `.cursor/mcp.json`
- 전역 수준: Cursor Settings → MCP

```json
// .cursor/mcp.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxxx"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/data"]
    }
  }
}
```

**특징:**
- 모든 플랜(무료 포함)에서 MCP 지원
- stdio 및 SSE 전송 방식 지원
- Agent 모드에서 MCP 도구를 자동으로 활용
- 최대 40개 도구까지 에이전트에 전달 가능
- 1,800개 이상의 커뮤니티 MCP 서버 활용 가능

**Claude Code 대응**: `.cursor/mcp.json` ↔ `.claude/mcp.json` (동일한 MCP 프로토콜)

자세한 내용은 [03_mcp 가이드](../../03_mcp/)를 참조한다.

### 5-3. Custom Instructions & Notepads → 04_skill 가이드

Cursor에서는 Notepads와 Rules를 조합하여 전문화된 AI 행동을 정의할 수 있다.

**Notepads 활용:**
- 복잡한 프롬프트를 영구 저장
- `@notepad:이름`으로 즉시 참조
- 팀원 간 프롬프트 패턴 공유

**Rules의 Agent Requested 유형:**
- description에 트리거 키워드를 설정
- 에이전트가 관련 작업 시 자동으로 해당 규칙을 로드
- Claude Code의 Skill/Subagent 패턴과 유사한 효과

**Claude Code 대응**: Notepads + Agent Requested Rules ↔ Custom Slash Commands + Task()

자세한 내용은 [04_skill 가이드](../../04_skill/)를 참조한다.

### 5-4. Agent Workflows → 05_workflow 가이드

Cursor Agent 모드에서의 워크플로우 자동화를 다룬다.

**Agent 모드 워크플로우 패턴:**
- Plan Mode: 실행 전 계획서 생성 및 검토
- 멀티스텝 작업: 파일 탐색 → 코드 수정 → 빌드 → 테스트를 순차 실행
- Automations: 이벤트 트리거 기반 자동 에이전트 실행
- Hooks: 특정 이벤트에서 커스텀 스크립트 실행

**Claude Code 대응**: Automations ↔ Claude Code의 Hooks + GitHub Actions 연동

자세한 내용은 [05_workflow 가이드](../../05_workflow/)를 참조한다.

### 5-5. Background Agents → 06_subagent 가이드

Cursor의 Background Agents는 클라우드 VM에서 독립적으로 작업하는 비동기 에이전트이다.

**주요 특징:**
- 격리된 Ubuntu VM에서 실행
- 별도 브랜치에서 작업, 완료 시 PR 생성
- 최대 8개 에이전트 병렬 실행
- Slack, 웹, 모바일에서도 에이전트 시작 가능
- 스크린샷, 영상, 로그 등 아티팩트 생성

**Claude Code 대응**: Background Agents ↔ 서브에이전트 Task() + 별도 프로세스

자세한 내용은 [06_subagent 가이드](../../06_subagent/)를 참조한다.

---

### Phase 5 체크포인트

- [ ] 각 심화 가이드의 Cursor 대응 기능을 파악했는가?
- [ ] MCP 서버 설정 방법을 이해했는가?
- [ ] Background Agents와 Automations의 차이를 이해했는가?

---

## Phase 6: Bug Finder — 자동 버그 탐지

Cursor의 Bug Finder는 코드베이스를 자동으로 분석하여 잠재적 버그를 탐지하는 기능이다.

**탐지 대상:**
- 논리적 오류
- 엣지 케이스 미처리
- 타입 불일치
- 잠재적 런타임 에러
- 보안 취약점

**사용법:**

```
Chat 패널에서: "이 파일에서 잠재적 버그를 찾아줘"
또는
Agent 모드에서: "전체 코드베이스에서 버그를 탐지하고 수정 PR 만들어줘"
```

> 💡 **개념 설명**: Bug Finder는 단순한 린터를 넘어 AI가 코드의 의미를 이해하고 논리적 오류를 찾는다. 에러 트레이스를 제공하면 해당 버그의 원인을 추적하고 수정까지 제안한다.

---

## 핵심 단축키 요약

| 동작 | macOS | Windows/Linux |
|------|-------|---------------|
| 인라인 편집 | `Cmd+K` | `Ctrl+K` |
| Chat 패널 열기 | `Cmd+L` | `Ctrl+L` |
| Composer 열기 | `Cmd+I` | `Ctrl+I` |
| Tab 자동완성 수락 | `Tab` | `Tab` |
| Tab 제안 거절 | `Esc` | `Esc` |
| 설정 열기 | `Cmd+,` | `Ctrl+,` |
| 명령 팔레트 | `Cmd+Shift+P` | `Ctrl+Shift+P` |

---

## Cursor만의 고유 강점 정리

1. **AI-First 설계**: AI가 부착된 것이 아니라 에디터 자체가 AI 중심으로 설계됨
2. **Cursor Tab**: 가장 진보된 자동완성 — 편집 의도를 예측하는 멀티라인 제안
3. **Composer**: 시각적 diff와 함께 여러 파일을 동시에 편집하는 강력한 인터페이스
4. **Background Agents**: 클라우드 VM에서 비동기 작업을 수행하고 PR로 결과 제출
5. **코드베이스 인덱싱**: `@codebase`로 프로젝트 전체를 시맨틱 검색
6. **모델 자유 선택**: Claude, GPT, Gemini, 커스텀 모델을 자유롭게 전환
7. **VS Code 완벽 호환**: 모든 확장, 설정, 키바인딩을 그대로 사용
8. **Privacy Mode**: 코드 미저장 옵션으로 기업 환경 대응
9. **Notepads**: 영구 컨텍스트 저장으로 프롬프트 재사용 효율화
10. **Automations**: 이벤트 기반 자동 에이전트로 워크플로우 자동화

---

## 마무리

Cursor는 AI를 에디터의 핵심에 배치한 도구로, Tab 자동완성부터 Background Agents까지 모든 기능이 AI와의 협업을 전제로 설계되었다. VS Code의 풍부한 생태계를 그대로 활용하면서 AI 기반 생산성을 극대화할 수 있다는 점이 최대 강점이다.

기존 VS Code 사용자라면 설치 후 즉시 익숙한 환경에서 AI 기능을 활용할 수 있으며, Phase별로 점진적으로 Cursor Tab → Cmd+K → Composer → Agent Mode → Background Agents 순서로 활용 범위를 넓혀가는 것을 권장한다.

---

## 참고 링크

- 공식 사이트: https://cursor.com
- 공식 문서: https://docs.cursor.com
- 가격 정보: https://cursor.com/pricing
- Changelog: https://cursor.com/changelog
- Rules 가이드: https://docs.cursor.com/context/rules
- MCP 설정: https://docs.cursor.com/context/model-context-protocol
- Cursor Directory (커뮤니티 규칙): https://cursor.directory
