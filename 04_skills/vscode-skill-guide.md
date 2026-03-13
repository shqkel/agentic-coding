# VS Code Copilot Skills 작성 가이드 — Prompt Files와 Instructions

## 학습 목표

VS Code Copilot에서 "Skills"에 해당하는 Prompt Files(`.github/prompts/*.prompt.md`)와 경로 기반 Instructions(`.github/instructions/*.instructions.md`)의 차이를 이해하고, Chat Participants와 함께 활용하여 재사용 가능한 전문 지식 패키지를 구성할 수 있다.

## 사전 준비

- VS Code 최신 버전 설치
- GitHub Copilot 확장 설치 및 구독 활성화
- VS Code Settings에서 `chat.promptFilesLocations` 활성화 (선택)
- 작업할 프로젝트 디렉토리

---

## 전체 흐름 한눈에 보기

VS Code Copilot의 Instructions 파일(`.github/copilot-instructions.md`)은 항상 적용되는 Rule과 같다. Skills에 해당하는 기능은 세 가지 방식으로 구현한다:

- **Prompt Files**: 재사용 가능한 프롬프트 템플릿 (Chat에서 `/파일명`으로 호출)
- **경로 기반 Instructions**: 특정 디렉토리 파일 작업 시 자동 적용되는 전문 지침
- **Chat Participants**: `@workspace`, `@terminal` 등 도메인 특화 컨텍스트 제공자

1. **개념 이해** — VS Code Copilot의 Skills 구현 방식
2. **Prompt Files 작성** — `.github/prompts/` 구조와 변수 활용
3. **경로 기반 Instructions** — `applyTo` 필드 활용
4. **실전 예시** — 실무에서 바로 쓸 수 있는 파일 작성

---

## Phase 1: VS Code Copilot Skills 개념 이해

### 목표

VS Code Copilot의 다양한 컨텍스트 주입 방식을 이해하고 Skills에 해당하는 부분을 정확히 파악한다.

### 단계별 구현

#### Step 1.1 — VS Code Copilot의 컨텍스트 계층

> **💡 개념 설명: VS Code Copilot의 지침 계층 구조**
>
> VS Code Copilot은 여러 계층의 지침 파일을 조합한다. 가장 넓은 범위는 `.github/copilot-instructions.md`(항상 적용)이고, 더 좁은 범위는 경로 기반 Instructions이다. Prompt Files는 사용자가 명시적으로 호출해야 한다.
>
> 중요한 점은 VS Code 1.96 이후 `.github/instructions/*.instructions.md` 형식이 추가되어, Cursor의 Rule 시스템과 유사한 세밀한 제어가 가능해졌다.
>
> **핵심 한 줄:** Instructions = 자동 적용 규칙 / Prompt Files = 명시적 호출 Skills

| 파일 유형 | 경로 | 적용 시점 | Skills 유사도 |
|----------|------|----------|--------------|
| Global Instructions | `.github/copilot-instructions.md` | 항상 | ❌ (Rule) |
| Path Instructions | `.github/instructions/*.instructions.md` | `applyTo` 경로 매칭 시 | 부분적 |
| Prompt Files | `.github/prompts/*.prompt.md` | 사용자 호출 시 | ✅ |
| Chat Participants | `@workspace`, `@terminal` 등 | 사용자 `@` 입력 시 | 부분적 |

#### Step 1.2 — Instructions vs Prompt Files 비교

| 구분 | Instructions | Prompt Files |
|------|--------------|--------------|
| 파일 확장자 | `.instructions.md` | `.prompt.md` |
| 저장 위치 | `.github/instructions/` | `.github/prompts/` |
| 호출 방식 | 자동 (applyTo 매칭) | 수동 (`/파일명`) |
| 인자 전달 | 없음 | `${input:변수}` |
| 파일 참조 | 없음 | `#file:경로` |
| 팀 공유 | git 공유 | git 공유 |

### 체크포인트

Prompt Files와 Instructions의 핵심 차이(수동 호출 vs 자동 적용)를 설명할 수 있는가?

---

## Phase 2: Prompt Files — 재사용 가능한 프롬프트 템플릿

### 목표

`.github/prompts/` 디렉토리에 Prompt Files를 작성하고 Chat에서 `/파일명`으로 호출하는 방법을 익힌다.

### 단계별 구현

#### Step 2.1 — Prompt Files 디렉토리 생성 및 기본 구조

```bash
# 디렉토리 생성
mkdir -p .github/prompts
```

> **💡 개념 설명: Prompt Files 활성화**
>
> VS Code에서 Prompt Files 기능을 사용하려면 설정에서 활성화해야 할 수 있다:
> 1. `Cmd+Shift+P` → "Preferences: Open Settings (JSON)"
> 2. `"chat.promptFiles": true` 추가
>
> 또는 Settings UI에서 "chat.promptFiles" 검색 후 활성화.
>
> **핵심 한 줄:** Prompt Files = Chat에서 `/이름`으로 호출하는 재사용 프롬프트

```markdown
<!-- .github/prompts/review-pr.prompt.md -->

---
mode: ask
description: PR 코드 리뷰 — 코드 품질, 보안, 성능을 종합 분석
---

# PR 코드 리뷰

다음 변경사항을 종합적으로 검토해줘.

## 검토 항목

### 1. 코드 품질
- 변수/함수명의 명확성
- 함수 단일 책임 원칙
- 중복 코드 여부
- 주석의 적절성 (코드 자체로 이해 안 되는 부분만)

### 2. 보안
- 입력 검증 누락
- 인증/인가 처리
- SQL 인젝션, XSS 가능성
- 민감 정보 노출 여부

### 3. 성능
- 알고리즘 복잡도
- DB 쿼리 최적화 가능성
- 불필요한 연산

### 4. 테스트
- 핵심 로직에 테스트 존재 여부
- 엣지 케이스 커버리지

## 결과 형식

### ✅ 잘된 점
(구체적 항목)

### 🔧 개선 제안
[파일:라인] 문제 + 제안 + 이유

### 🚨 즉시 수정 필요
(보안/버그 등 긴급 항목)

### 전체 평점: [1-5] / 5
```

#### Step 2.2 — 변수가 있는 Prompt Files

`${input:변수명}` 형식으로 실행 시 값을 입력받을 수 있다:

```markdown
<!-- .github/prompts/generate-test.prompt.md -->

---
mode: agent
description: 지정한 파일의 테스트 코드 자동 생성
---

# 테스트 코드 생성

다음 파일의 테스트를 작성해줘:

대상 파일: ${input:테스트할 파일 경로 (예: src/utils/auth.ts)}
테스트 프레임워크: ${input:테스트 프레임워크 (jest/pytest/vitest, 기본값: 프로젝트 설정 자동 감지)}

## 테스트 작성 기준

1. **정상 케이스** (Happy Path)
   - 예상되는 주요 입력으로 올바른 출력 확인

2. **경계값 케이스**
   - null, undefined, 빈 배열, 최솟값, 최댓값

3. **에러 케이스**
   - 잘못된 입력, 예외 발생, 비동기 실패

4. **Mock 처리**
   - 외부 의존성(DB, API, 파일시스템)은 Mock으로 처리

## 출력 형식

테스트 파일을 생성하고 커버리지 개요를 요약해줘.
테스트는 Given-When-Then 패턴으로 설명을 작성한다.
```

#### Step 2.3 — 파일 참조가 있는 Prompt Files

`#file:경로` 형식으로 프로젝트 파일을 프롬프트에 포함할 수 있다:

```markdown
<!-- .github/prompts/api-consistency-check.prompt.md -->

---
mode: ask
description: API 설계 일관성 검사 — 팀 가이드라인 기준
---

# API 일관성 검사

현재 파일의 API 설계를 팀 가이드라인과 비교해줘.

팀 API 가이드라인: #file:.github/docs/api-guidelines.md

## 확인 항목

- URL 네이밍 컨벤션 준수
- HTTP 메서드 올바른 사용
- 응답 형식 표준화 여부
- 에러 코드 일관성
- 인증 처리 방식

어긴 항목을 구체적으로 찾고, 수정 방법을 제안해줘.
```

#### Step 2.4 — Prompt Files 호출 방법

VS Code Copilot Chat에서:

```
/review-pr
```

변수가 있는 경우 실행 시 입력 창이 표시된다.

에이전트 모드에서 파일과 함께 사용:

```
/generate-test
```

(실행 후 파일 경로 입력 프롬프트가 나타남)

### 체크포인트

`.github/prompts/review-pr.prompt.md`를 작성하고 Chat에서 `/review-pr`로 호출했을 때 해당 지침이 적용되면 성공이다.

---

## Phase 3: 경로 기반 Instructions — 파일별 자동 지침

### 목표

`.github/instructions/` 디렉토리에 `applyTo` 필드를 활용한 Instructions 파일을 작성하여 특정 경로의 파일 작업 시 자동으로 전문 지침이 적용되도록 한다.

### 단계별 구현

#### Step 3.1 — Instructions 파일 기본 구조

> **💡 개념 설명: applyTo 필드**
>
> `applyTo` 필드는 glob 패턴으로 파일 경로를 지정한다. Copilot이 해당 경로의 파일을 열거나 편집할 때 자동으로 이 Instructions가 컨텍스트에 포함된다. Cursor의 Auto Attached Rule과 동일한 개념이다.
>
> `applyTo: "**"` 는 모든 파일에 적용된다 (전역 적용, `.github/copilot-instructions.md`와 유사).
>
> **핵심 한 줄:** applyTo = "이 경로의 파일을 작업할 때 자동으로 나를 컨텍스트에 포함해라"

```markdown
<!-- .github/instructions/api.instructions.md -->

---
applyTo: "src/api/**"
---

# API 모듈 지침

`src/api/` 하위 모든 파일에 자동으로 적용된다.

## 이 모듈의 규칙

- 모든 엔드포인트는 `/api/v1/` 접두사를 사용한다
- 응답 형식은 반드시 `{ data, meta, error }` 구조를 따른다
- 인증이 필요한 엔드포인트는 `authMiddleware`를 반드시 적용한다
- 입력 검증은 Zod schema를 사용한다

## 에러 처리

```typescript
// 표준 에러 응답 형식
res.status(400).json({
  data: null,
  error: {
    code: 'VALIDATION_ERROR',
    message: '입력값이 올바르지 않습니다',
    details: errors
  }
});
```

## 금지 사항

- `any` 타입 사용 금지
- 직접적인 SQL 쿼리 금지 (반드시 ORM 사용)
- 인증 없이 민감한 데이터 반환 금지
```

#### Step 3.2 — 복수 경로 지정

```markdown
<!-- .github/instructions/test-files.instructions.md -->

---
applyTo: "**/*.test.ts,**/*.test.tsx,**/*.spec.ts,**/__tests__/**"
---

# 테스트 파일 지침

모든 테스트 파일에 자동으로 적용된다.

## 테스트 작성 원칙

- **AAA 패턴** 사용: Arrange → Act → Assert
- 테스트 설명은 "should + 동사" 형식 또는 한국어 사용
- 각 `it/test` 블록은 단 하나의 동작만 검증
- Mock은 가능한 최소한으로 사용

## 필수 케이스

모든 함수는 다음 케이스를 반드시 테스트한다:
1. 정상 동작 케이스
2. null/undefined 입력 케이스
3. 비즈니스 규칙 위반 케이스 (유효성 검사 실패 등)

## 예시 구조

```typescript
describe('UserService', () => {
  describe('createUser', () => {
    it('유효한 데이터로 사용자를 생성해야 한다', async () => {
      // Arrange
      const userData = { name: '홍길동', email: 'hong@example.com' };

      // Act
      const result = await userService.createUser(userData);

      // Assert
      expect(result.id).toBeDefined();
      expect(result.email).toBe(userData.email);
    });
  });
});
```
```

#### Step 3.3 — 전체 Instructions 파일 구조

```bash
.github/
├── copilot-instructions.md           # 전역 지침 (항상 적용)
├── instructions/
│   ├── api.instructions.md           # src/api/** 자동 적용
│   ├── test-files.instructions.md    # **/*.test.* 자동 적용
│   ├── react-components.instructions.md  # src/components/** 자동 적용
│   └── database.instructions.md     # src/models/**, src/db/** 자동 적용
└── prompts/
    ├── review-pr.prompt.md           # /review-pr 수동 호출
    ├── generate-test.prompt.md       # /generate-test 수동 호출
    ├── api-consistency-check.prompt.md
    └── security-audit.prompt.md
```

### 체크포인트

`src/api/` 경로의 파일을 열었을 때 api.instructions.md의 지침이 Copilot 응답에 반영되면 성공이다.

---

## Phase 4: 실전 예시

### 목표

실무에서 즉시 활용할 수 있는 Prompt File 2개와 Instructions 1개를 작성한다.

### 예시 1: 보안 감사 Prompt File

```markdown
<!-- .github/prompts/security-audit.prompt.md -->

---
mode: ask
description: 보안 취약점 종합 감사 — OWASP Top 10 기준
---

# 보안 감사

현재 코드를 OWASP Top 10 기준으로 보안 감사해줘.

## 검사 항목

### 인젝션 공격 (A03)
- SQL 인젝션 가능 지점
- Command 인젝션 가능 지점
- ORM을 우회하는 Raw 쿼리

### 인증 및 세션 (A07)
- 비밀번호 해시 강도 (bcrypt/argon2 여부)
- JWT 검증 누락
- 세션 고정 공격 가능성

### 민감 데이터 노출 (A02)
- 로그에 민감 정보 출력 여부
- API 응답에 불필요한 필드 포함 여부
- 암호화되지 않은 민감 데이터 저장

### 접근 제어 (A01)
- 수평적 권한 상승 가능성
- 역할 기반 접근 제어(RBAC) 구현 여부

## 결과 보고 형식

| 취약점 | 위치 | 심각도 | 설명 | 수정 방법 |
|--------|------|--------|------|----------|
| SQL Injection | users.ts:42 | Critical | ... | ... |

심각도 기준:
- Critical: 즉시 수정, 서비스 중단 고려
- High: 다음 배포 전 수정 필수
- Medium: 계획된 스프린트 내 수정
- Low: 기술 부채로 추적
```

### 예시 2: 리팩토링 가이드 Prompt File

```markdown
<!-- .github/prompts/refactor.prompt.md -->

---
mode: agent
description: 코드 리팩토링 — 가독성과 유지보수성 향상
---

# 코드 리팩토링

선택한 코드를 리팩토링해줘. 기능 변경 없이 코드 품질만 개선한다.

## 리팩토링 원칙

1. **작은 단계로 진행**: 한 번에 하나의 리팩토링만 적용
2. **테스트 먼저**: 리팩토링 전 테스트가 없다면 먼저 작성
3. **이름 개선**: 의도를 드러내는 이름으로 변경
4. **함수 추출**: 50줄 이상의 함수는 분리
5. **중복 제거**: DRY 원칙 적용

## 리팩토링 카탈로그 (우선 적용)

- Extract Function — 긴 함수에서 논리적 단위 추출
- Rename Variable — 의도를 드러내는 이름으로 변경
- Replace Conditional with Polymorphism — 복잡한 조건문 단순화
- Extract Class — 클래스가 너무 많은 책임을 가질 때
- Introduce Parameter Object — 파라미터가 3개 이상일 때

## 출력 형식

1. 식별된 리팩토링 항목 목록
2. 우선순위 및 적용 순서 제안
3. 리팩토링된 코드
4. 변경 전후 비교 및 이유 설명

기능이 동일함을 확인하는 테스트도 함께 제공해줘.
```

### 예시 3: React 컴포넌트 Instructions

```markdown
<!-- .github/instructions/react-components.instructions.md -->

---
applyTo: "src/components/**,src/pages/**,src/app/**"
---

# React 컴포넌트 지침

## 컴포넌트 구조 표준

```tsx
// 1. import 순서: React → 외부 라이브러리 → 내부 모듈 → 스타일
import React, { useState, useCallback } from 'react';
import { motion } from 'framer-motion';
import { Button } from '@/components/ui';
import styles from './Component.module.css';

// 2. 타입 정의
interface Props {
  title: string;
  onSubmit: (value: string) => void;
  disabled?: boolean;
}

// 3. 컴포넌트 (화살표 함수)
const ComponentName: React.FC<Props> = ({ title, onSubmit, disabled = false }) => {
  // 4. State hooks
  // 5. 계산된 값 (useMemo)
  // 6. 이벤트 핸들러 (useCallback)
  // 7. Effects (useEffect) — 가능한 마지막에
  // 8. 조기 반환 (loading, error 등)
  // 9. JSX
};

export default ComponentName;
```

## 성능 규칙

- 리스트 렌더링에는 반드시 key prop 사용 (index 사용 금지)
- 무거운 계산은 `useMemo` 적용
- 이벤트 핸들러는 `useCallback` 적용 (자식에 props로 전달할 때)
- 이미지는 `next/image` 사용 (Next.js 프로젝트)

## 금지 사항

- 인라인 스타일 사용 금지 (Tailwind 또는 CSS Module 사용)
- `any` 타입 금지
- 컴포넌트 내부에서 직접 API 호출 금지 (훅으로 분리)
```

### 체크포인트

3개의 예시 파일을 작성하고 각각 올바르게 동작하는지 확인했는가?

---

## Chat Participants 활용

### `@workspace` — 프로젝트 전체 컨텍스트

```
@workspace 이 프로젝트에서 사용자 인증을 처리하는 코드는 어디에 있어?
@workspace 프로젝트 전체에서 deprecated된 API를 사용하는 곳을 찾아줘
```

### `@terminal` — 터미널 컨텍스트

```
@terminal 방금 나온 에러를 분석하고 수정 방법을 알려줘
@terminal npm run build 결과를 보고 빌드 실패 원인을 찾아줘
```

### `@github` — GitHub 통합

```
@github PR #123의 변경사항을 요약해줘
@github 이 이슈와 관련된 코드를 찾아줘
```

### MCP를 통한 커스텀 Chat Participant

VS Code 1.96 이후 MCP(Model Context Protocol) 서버를 통해 커스텀 Chat Participant를 등록할 수 있다:

```json
// .vscode/mcp.json
{
  "servers": {
    "my-db-tool": {
      "type": "stdio",
      "command": "node",
      "args": ["./mcp-servers/db-tool.js"]
    }
  }
}
```

---

## 다른 도구와의 비교

| 기능 | VS Code Copilot | Claude Code | Antigravity | Cursor |
|------|-----------------|-------------|-------------|--------|
| Skills 방식 | Prompt Files + Instructions | `.claude/commands/*.md` | SKILL.md 디렉토리 | Notepads + Agent Rules |
| 명시적 호출 | `/파일명` | `/명령어` | 없음 (자동) | `@notepad:이름` |
| 자동 적용 | applyTo 경로 기반 | 없음 | description 매칭 | globs 패턴 |
| 인자 전달 | `${input:변수}` | `$ARGUMENTS` | 없음 | 없음 |
| 파일 참조 | `#file:경로` | `@파일경로` | 없음 | `@파일명` |
| 버전 관리 | 가능 | 가능 | 가능 | 가능 (Rules) |
| 전역 설정 | User Instructions | `~/.claude/commands/` | `~/.gemini/antigravity/skills/` | 없음 |
| 에이전트 확장 | MCP + Chat Participants | 없음 | MCP | 없음 |
| 항상 적용 Rule | `copilot-instructions.md` | `CLAUDE.md` | `GEMINI.md` | `alwaysApply: true` |

---

## 마무리

### 이 가이드에서 배운 것

- VS Code Copilot의 Skills = Prompt Files(명시적) + Instructions(자동)
- `applyTo` 필드로 파일 경로 기반 자동 지침 적용
- `${input:변수}`로 동적 입력 받는 Prompt Files 작성
- `#file:경로`로 프로젝트 파일을 프롬프트에 포함
- `@workspace`, `@terminal` 등 Chat Participants로 도메인 컨텍스트 활용

### 효과적인 Prompt File 작성 팁

```markdown
# ❌ 나쁜 예 (너무 단순함)
코드를 검토해줘.

# ✅ 좋은 예 (구체적 항목 + 출력 형식 명시)
---
mode: ask
description: 코드 품질 종합 리뷰
---

다음 기준으로 코드를 검토하고, 지정된 형식으로 결과를 제공해줘:

항목: [가독성, 보안, 성능, 테스트]
형식: 항목별 점수(1-5) + 구체적 개선 제안 + 우선순위
```

### 다음 단계

- `05_workflow` — Prompt Files를 조합한 멀티스텝 에이전트 워크플로우
- VS Code Extensions Marketplace에서 도메인 특화 Copilot 확장 탐색
- MCP 서버를 통한 커스텀 Chat Participant 개발
