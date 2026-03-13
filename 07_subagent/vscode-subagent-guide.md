# VS Code Copilot 서브에이전트 가이드

## 학습 목표

- VS Code Copilot의 서브에이전트 패턴(Explore, Copilot Coding Agent)을 이해한다
- GitHub 이슈 기반으로 Copilot Coding Agent를 활용하는 전체 흐름을 익힌다
- Explore 서브에이전트가 메인 에이전트의 계획 수립을 어떻게 지원하는지 파악한다
- GitHub Actions와의 연동으로 자동화 파이프라인을 구성하는 방법을 습득한다
- 서드파티 에이전트 플러그인으로 기능을 확장하는 방법을 이해한다

---

## 전체 흐름 한눈에 보기

```
VS Code Copilot 서브에이전트 아키텍처
─────────────────────────────────────────────────────────────────
[ 패턴 1: Explore 서브에이전트 (내부, 자동) ]
사용자 요청 → 메인 에이전트 (Plan 수립)
                    │
                    ├── Explore 서브에이전트 1 (경량)
                    │     코드베이스 병렬 탐색 → 관련 파일 수집
                    ├── Explore 서브에이전트 2 (경량)
                    │     문서/설정 탐색
                    └── 결과 통합 → 메인 에이전트 계획 보강

[ 패턴 2: Copilot Coding Agent (GitHub 이슈 기반) ]
GitHub 이슈 → @copilot 할당
  → 브랜치 자동 생성
  → 코드 구현 (클라우드)
  → 테스트 실행 (GitHub Actions)
  → PR 생성
  → 완료 알림

[ 패턴 3: Agent 플러그인 확장 ]
VS Code 확장 → 서드파티 에이전트 기능 추가
```

---

## Phase 1: 개념 이해

### VS Code Copilot의 서브에이전트 지원 수준

| 항목 | 내용 |
|------|------|
| Explore 서브에이전트 | 내장, 자동 동작 (사용자 직접 제어 불가) |
| Copilot Coding Agent | GitHub 이슈 기반, 완전 자율 |
| 병렬 실행 수 | 제한적 (Explore: 수 개 자동, Coding Agent: 이슈별) |
| GitHub 통합 | 완전 통합 (이슈, PR, Actions) |
| 로컬 환경 필요 | Coding Agent는 불필요 (GitHub Actions 기반) |
| 에디터 통합 | VS Code 완전 통합 |
| Claude 통합 | Claude Agent SDK (실험적) |

### Explore 서브에이전트: 내부 동작 원리

Explore 서브에이전트는 사용자가 직접 제어하지 않는 내부 메커니즘이다. Copilot이 복잡한 요청을 처리할 때 자동으로 동작한다.

```
[사용자 요청]
"인증 시스템을 OAuth 2.0으로 업그레이드해줘"

[내부 동작]
메인 에이전트: "먼저 현재 인증 구현을 파악해야겠다"
  │
  ├── Explore 서브에이전트 A (경량 모델):
  │     auth 관련 파일 검색: src/**/*auth*, src/**/*login*
  │     → 결과: 12개 파일 목록
  │
  ├── Explore 서브에이전트 B (경량 모델):
  │     기존 미들웨어 패턴 탐색: middleware/, interceptors/
  │     → 결과: 미들웨어 구조
  │
  ├── Explore 서브에이전트 C (경량 모델):
  │     현재 의존성 탐색: package.json, go.mod
  │     → 결과: 인증 관련 패키지 목록
  │
  └── 메인 에이전트: 세 탐색 결과 통합 → 구현 계획 수립
```

**사용자가 해야 할 일**: 없음. 복잡한 요청을 하면 Explore가 자동으로 작동한다.

### Copilot Coding Agent: GitHub 이슈 기반 자율 에이전트

Copilot Coding Agent는 GitHub 이슈를 입력으로 받아 완전히 자율적으로 작동하는 에이전트다.

```
전체 흐름:
  1. GitHub 이슈 생성 (또는 기존 이슈 사용)
  2. 이슈에 Copilot 할당 (Assignee → @copilot)
  3. Copilot이 자동으로:
     a. 이슈 내용 분석
     b. 저장소 코드베이스 탐색
     c. 별도 브랜치 생성 (copilot/issue-[번호])
     d. 코드 구현
     e. GitHub Actions 트리거 → 테스트 실행
     f. PR 생성 (이슈와 자동 연결)
     g. 완료 알림 (이슈 코멘트 + 이메일)
```

**특징**: 이 모든 과정이 GitHub 인프라에서 실행되므로 로컬 환경이 전혀 필요 없다.

### 체크포인트 1

- [ ] Explore 서브에이전트가 자동으로 동작한다는 것을 이해했다
- [ ] Copilot Coding Agent의 GitHub 이슈 기반 워크플로우를 파악했다
- [ ] 두 패턴의 차이(내부 자동 vs GitHub 이슈 기반)를 구별할 수 있다

---

## Phase 2: 서브에이전트 설정 및 실행

### Explore 서브에이전트 최대 활용

Explore는 자동이지만, 더 효과적으로 활용하는 요청 방식이 있다.

**효과적인 요청 패턴:**

```
# 탐색 범위를 명시하면 Explore가 더 정확하게 작동
"src/auth/ 디렉토리의 JWT 구현을 TypeScript strict 모드에 맞게 업데이트해줘"
→ Explore가 src/auth/ 집중 탐색

# 컨텍스트를 풍부하게 제공
"package.json의 의존성을 확인하고, 현재 사용 중인 인증 라이브러리를
 최신 버전으로 업그레이드하는 방법을 알려줘"
→ Explore가 package.json + 관련 사용 코드 탐색

# 다중 파일 변경이 필요함을 명시
"UserService와 이를 사용하는 모든 Controller에 새 메서드 addRole()을 추가해줘"
→ Explore가 UserService + 모든 Controller 탐색
```

**Copilot Chat에서 에이전트 모드 활성화:**

```
VS Code Copilot Chat → 드롭다운에서 "Agent" 선택
(기본값은 "Chat", "Agent"로 변경해야 Explore 서브에이전트 활성화)

설정 → copilot.chat.agent.enabled: true
```

### Copilot Coding Agent 설정

**사전 준비:**

```
1. GitHub Copilot Business 또는 Enterprise 플랜 필요
2. 저장소에 GitHub Actions 워크플로우 파일 존재 (테스트 실행용)
3. 저장소 설정 → Copilot → "Allow Copilot to create and approve pull requests" 활성화
```

**GitHub Actions 워크플로우 설정:**

```yaml
# .github/workflows/ci.yml
# Copilot Coding Agent가 PR 생성 후 자동 실행
name: CI

on:
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Run lint
        run: npm run lint

      - name: Build
        run: npm run build
```

### GitHub 이슈 작성 모범 사례

Copilot Coding Agent가 효과적으로 작업하려면 이슈가 잘 작성되어야 한다.

```markdown
## 이슈 제목: 사용자 프로필 이미지 업로드 기능 추가

## 설명
로그인한 사용자가 프로필 이미지를 업로드할 수 있어야 한다.

## 요구사항
- [ ] 이미지 파일(JPEG, PNG) 업로드 지원
- [ ] 최대 파일 크기: 5MB
- [ ] 이미지 리사이징: 200x200px 썸네일 자동 생성
- [ ] 기존 이미지 교체 기능
- [ ] 업로드 실패 시 명확한 에러 메시지

## 기술 제약
- 저장소: AWS S3 (설정은 .env에 있음)
- 백엔드: Node.js + Express
- 이미지 처리: sharp 라이브러리 사용
- 테스트: Jest + Supertest

## 완료 기준
- 모든 요구사항 구현
- 단위 테스트 및 통합 테스트 작성
- API 문서(JSDoc) 업데이트
```

**이슈에 Copilot 할당:**

```
이슈 페이지 → Assignees → copilot 검색 → @copilot 선택
또는
이슈 목록에서 → "Assign to Copilot" 버튼 클릭 (GitHub UI)
```

### VS Code에서 Coding Agent 시작

```
VS Code Copilot Chat:
> @github 이슈 #42를 구현해줘

또는 자연어로:
> 이슈 #42의 요구사항을 구현하는 PR을 만들어줘

Copilot이 GitHub 이슈 내용을 가져와 Coding Agent 시작
```

### 체크포인트 2

- [ ] GitHub Actions CI 워크플로우가 PR에서 자동 실행되도록 설정했다
- [ ] Copilot Coding Agent가 처리할 수 있는 상세한 GitHub 이슈를 작성했다
- [ ] 이슈에 @copilot을 할당하고 에이전트가 시작되는 것을 확인했다

---

## Phase 3: 병렬 처리 패턴

### 패턴 1: 여러 이슈 동시 처리

```
GitHub 이슈 목록에서 여러 이슈에 동시에 Copilot 할당:

이슈 #101: 검색 기능 최적화 → @copilot 할당
이슈 #102: 알림 이메일 템플릿 개선 → @copilot 할당
이슈 #103: 사용자 설정 페이지 구현 → @copilot 할당

각 이슈에 대해 독립적인 Copilot Coding Agent 실행:
  에이전트 1: copilot/issue-101 브랜치에서 검색 최적화
  에이전트 2: copilot/issue-102 브랜치에서 이메일 템플릿
  에이전트 3: copilot/issue-103 브랜치에서 설정 페이지

→ 3개 PR 자동 생성 → 팀 리뷰
```

### 패턴 2: Explore 활용 대규모 리팩토링

```
VS Code Copilot Chat (Agent 모드):

"프로젝트 전체에서 callback 패턴을 async/await로 변환해줘.
 Node.js 버전이 16+ 이상을 지원하므로 모든 비동기 코드를 현대화해줘."

내부적으로 Explore 서브에이전트가:
  ├── 전체 .js, .ts 파일에서 callback 패턴 탐색
  ├── Promise 체인 탐색
  ├── 에러 핸들링 패턴 탐색
  └── 결과 통합 → 변환 대상 목록 생성

메인 에이전트:
  → 파일별 변환 계획 수립
  → 의존성 순서대로 변환 실행
  → 테스트 실행 확인
```

### 패턴 3: GitHub Actions 병렬 파이프라인

Copilot Coding Agent가 생성한 PR에서 Actions가 병렬로 실행된다.

```yaml
# .github/workflows/parallel-checks.yml
name: Parallel Quality Checks

on:
  pull_request:

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run test:unit

  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run test:integration

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run lint

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/super-linter@v5
        with:
          VALIDATE_ALL_CODEBASE: false

  type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npx tsc --noEmit
```

```
Copilot Coding Agent가 PR 생성 → GitHub Actions 자동 트리거
  단위 테스트     (동시 실행)
  통합 테스트     (동시 실행)
  린트 검사       (동시 실행)
  보안 스캔       (동시 실행)
  타입 체크       (동시 실행)

→ 모든 체크 통과 시 merge 가능 표시
```

### 패턴 4: Claude Agent SDK 통합 (실험적)

```typescript
// .github/copilot-extensions/claude-subagent.ts
// Copilot Extension으로 Claude 서브에이전트 통합

import Anthropic from "@anthropic-ai/sdk";

export async function handleComplexTask(task: string) {
  const client = new Anthropic();

  // Claude 서브에이전트에 특정 작업 위임
  const response = await client.messages.create({
    model: "claude-opus-4-5",
    max_tokens: 8192,
    messages: [
      {
        role: "user",
        content: `다음 작업을 수행해줘: ${task}`,
      },
    ],
  });

  return response.content[0].text;
}
```

```
사용 시나리오:
VS Code Copilot: 전체 작업 조율 + 코드베이스 탐색
Claude Agent (via SDK): 복잡한 알고리즘 설계 + 심층 분석 작업
→ 두 모델의 강점 조합
```

### 체크포인트 3

- [ ] 여러 이슈에 동시에 @copilot을 할당해 병렬 처리를 경험했다
- [ ] GitHub Actions 병렬 파이프라인을 설정했다
- [ ] Agent 모드에서 Explore 서브에이전트가 자동 동작하는 것을 관찰했다

---

## Phase 4: 실전 시나리오

### 시나리오 1: 스프린트 계획과 Copilot Agent 연동

```
스프린트 계획 완료 후:

1. GitHub 이슈 일괄 생성 (스프린트 백로그에서)
   이슈 #201: [Story] 소셜 로그인 추가
   이슈 #202: [Bug] 결제 금액 반올림 오류
   이슈 #203: [Task] API 응답 캐싱 적용
   이슈 #204: [Story] 관리자 통계 대시보드

2. 독립적인 이슈에 @copilot 일괄 할당
   #201 → @copilot
   #202 → @copilot
   #203 → @copilot

   (복잡한 #204는 인간 개발자 직접 담당)

3. Copilot이 3개 이슈 병렬 처리
   → 3개 PR 자동 생성
   → 개발자는 #204 집중

4. 개발자가 Copilot PR 검토 + #204 구현 병행
   → 스프린트 효율 30~50% 향상
```

### 시나리오 2: 기술 부채 해소 캠페인

```
기술 부채 이슈 20개 생성 후 단계적 처리:

1주차: 이슈 5개 @copilot 할당 (중요도 높은 것)
  #101~105: ESLint 오류 수정

2주차: 이슈 5개 @copilot 할당
  #106~110: TypeScript strict 위반 수정

3주차: 이슈 5개 @copilot 할당
  #111~115: 테스트 커버리지 개선

4주차: 이슈 5개 @copilot 할당
  #116~120: 문서화 추가

→ 1개월 만에 기술 부채 20건 해소
→ 팀은 새 기능 개발에 집중 가능
```

### 시나리오 3: 오픈소스 이슈 대응

```
오픈소스 저장소에서:

GitHub Discussions/Issues에서 요청 들어옴:
  "feature request: dark mode support"
  "bug: crash on iOS 16"
  "question: how to use with Vue 3"

처리 방법:
  1. feature/bug 이슈에 @copilot 할당
     → Copilot이 구현 + PR 생성
     → 메인테이너가 검토 후 merge

  2. question 이슈는 @copilot에게:
     "이 질문에 대한 상세한 답변과 코드 예시를 이슈 코멘트로 달아줘"
     → Copilot이 자동으로 답변 코멘트 작성
```

---

## Agent 플러그인 확장

### 서드파티 에이전트 플러그인 설치

VS Code Marketplace에서 다양한 에이전트 기능을 추가할 수 있다.

```
VS Code Extensions → "copilot agent" 검색

주요 플러그인:
  - GitHub Copilot for Azure: Azure 리소스 관리
  - Copilot for Jira: Jira 이슈 기반 코딩
  - Copilot for Linear: Linear 태스크 기반 코딩
  - Docker Copilot: 컨테이너 최적화 에이전트
```

### 커스텀 에이전트 플러그인 개발

```typescript
// package.json (VS Code Extension)
{
  "contributes": {
    "chatParticipants": [
      {
        "id": "my-company.deploy-agent",
        "name": "deploy",
        "description": "배포 자동화 에이전트",
        "isSticky": true
      }
    ]
  }
}
```

```typescript
// extension.ts
import * as vscode from "vscode";

export function activate(context: vscode.ExtensionContext) {
  const handler: vscode.ChatRequestHandler = async (
    request: vscode.ChatRequest,
    context: vscode.ChatContext,
    stream: vscode.ChatResponseStream,
    token: vscode.CancellationToken
  ) => {
    stream.markdown("배포를 시작합니다...");

    // 배포 로직 실행
    const result = await deployToEnvironment(request.prompt);

    stream.markdown(`배포 완료: ${result.url}`);
  };

  const agent = vscode.chat.createChatParticipant(
    "my-company.deploy-agent",
    handler
  );
  context.subscriptions.push(agent);
}
```

```
사용법:
VS Code Copilot Chat:
@deploy "staging 환경에 현재 브랜치를 배포해줘"
→ 커스텀 에이전트가 배포 자동화 실행
```

---

## 다른 도구와의 비교표

| 항목 | Gemini CLI | Claude Code | Codex CLI | Antigravity | Cursor | VS Code Copilot |
|------|-----------|-------------|-----------|-------------|--------|-----------------|
| 서브에이전트 방식 | 없음 (수동) | Task() 내장 | 실험적 플래그 | Agent Manager | Background VM | Explore + Coding Agent |
| GitHub 이슈 기반 | 없음 | 없음 | 없음 | 없음 | 없음 | 완전 지원 |
| GitHub Actions 연동 | 없음 | 없음 | 없음 | 없음 | 부분 | 완전 자동 |
| PR 자동 생성 | 없음 | 없음 | 없음 | 없음 | 있음 | 있음 |
| Explore 서브에이전트 | 없음 | Task() 유사 | 없음 | 없음 | 없음 | 자동 내장 |
| 플러그인 확장 | 없음 | 없음 | 없음 | 없음 | 없음 | 있음 (Marketplace) |
| 로컬 환경 필요 | 있음 | 있음 | 있음 | 있음 | 없음 (VM) | 없음 (클라우드) |
| 에디터 통합 | 없음 | 연동 | 없음 | 완전 | 완전 | 완전 |
| 비용 | API 사용량 | API 사용량 | API 사용량 | 구독 | 구독 | GitHub 구독 포함 |

---

## 주의사항 및 모범 사례

### 이슈 품질이 에이전트 품질을 결정한다

```
나쁜 이슈 → 나쁜 결과:
  제목: "버그 수정"
  내용: "뭔가 안됩니다"
  → Copilot이 무엇을 해야 할지 알 수 없음

좋은 이슈 → 좋은 결과:
  제목: "로그인 페이지: 비밀번호 재설정 이메일 미발송 버그"
  내용:
    - 재현 단계: 1. 로그인 → 2. 비밀번호 찾기 클릭 → 3. 이메일 입력 → 4. 제출
    - 기대 결과: 재설정 이메일 수신
    - 실제 결과: 이메일 미수신, 에러 없음
    - 환경: 프로덕션, Node.js 18
  → Copilot이 정확히 원인을 찾아 수정 가능
```

### 보안 고려사항

```
Copilot Coding Agent에게 절대 접근 권한 부여 금지:
  - 프로덕션 데이터베이스 연결 정보
  - 프로덕션 AWS/GCP/Azure 자격증명
  - 사용자 개인정보가 담긴 데이터

안전한 접근 방법:
  - GitHub Secrets에 저장된 환경변수만 사용
  - 개발/테스트 환경 자격증명만 제공
  - 최소 권한 원칙 적용
```

### 코드 리뷰는 항상 인간이

```
Copilot이 생성한 PR 검토 체크리스트:
  □ 요구사항이 완전히 구현되었는가?
  □ 테스트가 충분한가? (엣지 케이스 포함?)
  □ 보안 취약점이 없는가?
  □ 성능 문제가 없는가?
  □ 기존 코드 스타일과 일치하는가?
  □ 불필요한 의존성이 추가되지 않았는가?

auto-merge는 절대 사용하지 않는다.
```

### GitHub Actions 비용 관리

```
Copilot Coding Agent가 많은 이슈를 처리하면
GitHub Actions 실행 시간이 크게 증가할 수 있다.

관리 방법:
  1. Actions 실행 조건 최적화 (paths-ignore 사용)
  2. 캐싱 적극 활용 (actions/cache)
  3. 중복 실행 방지 (concurrency 설정)
  4. 저비용 runner 사용 (ubuntu-latest는 상대적으로 저렴)
```

```yaml
# 비용 최적화 설정 예시
jobs:
  test:
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
      cancel-in-progress: true  # 이전 실행 자동 취소

    steps:
      - uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### 모범 사례 요약

1. **이슈 템플릿 사용**: `.github/ISSUE_TEMPLATE/`에 상세한 템플릿을 만들어 일관성 유지
2. **CI 필수 설정**: Copilot PR은 반드시 CI 통과 후 merge 가능하도록 Branch protection rule 설정
3. **작은 이슈 단위**: 하나의 이슈가 하나의 명확한 작업만 담당 (이슈 크기가 작을수록 품질 높음)
4. **레이블 활용**: `copilot-ready` 레이블이 붙은 이슈만 @copilot에 할당
5. **Explore 활용**: Agent 모드로 설정하면 탐색 품질이 크게 향상됨

---

## 참고 자료

- [GitHub Copilot Coding Agent 공식 문서](https://docs.github.com/copilot/using-github-copilot/coding-agent)
- [VS Code Copilot Chat 문서](https://code.visualstudio.com/docs/copilot/overview)
- [GitHub Actions 공식 문서](https://docs.github.com/actions)
- [VS Code Chat Extension API](https://code.visualstudio.com/api/extension-guides/chat)
- [Anthropic Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk)
- [GitHub Copilot for VS Code 릴리스 노트](https://github.com/microsoft/vscode-copilot-release)
