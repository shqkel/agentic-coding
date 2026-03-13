# Cursor 서브에이전트 가이드

## 학습 목표

- Cursor Background Agent의 격리된 VM 기반 아키텍처를 이해한다
- 최대 8개 에이전트를 병렬로 실행하는 방법과 관리 전략을 익힌다
- Cursor, Slack, 웹/모바일 등 다양한 진입점에서 에이전트를 시작하는 방법을 습득한다
- Git Worktree 기반 파일 충돌 방지 메커니즘을 이해한다
- 에이전트 완료 후 생성되는 PR과 아티팩트를 효과적으로 활용한다

---

## 전체 흐름 한눈에 보기

```
Cursor Background Agent 아키텍처
─────────────────────────────────────────────────────────────────
[ 다양한 진입점 ]
  Cursor IDE / Slack / 웹 / 모바일
        │
        ▼
[ Background Agent 스폰 ]
        │
  ┌─────┼─────┐
  │     │     │
  ▼     ▼     ▼   (최대 8개 병렬)
[VM1] [VM2] [VM3]  ... 격리된 Ubuntu VM
  │     │     │
  ▼     ▼     ▼
[Git Worktree 1] [Worktree 2] [Worktree 3]
  │                 │              │
  ▼                 ▼              ▼
[별도 브랜치 커밋] [PR 생성] [아티팩트 생성]
        │
        ▼
[ 완료 알림 + 결과 보고 ]
```

---

## Phase 1: 개념 이해

### Cursor Background Agent의 서브에이전트 지원 수준

| 항목 | 내용 |
|------|------|
| 에이전트 방식 | 격리된 Ubuntu VM에서 실행 |
| 최대 병렬 수 | 8개 |
| 인터넷 접근 | 가능 (패키지 설치, API 호출) |
| 파일 충돌 방지 | Git Worktree 기반 자동 격리 |
| 결과 전달 | 별도 브랜치 + PR 자동 생성 |
| 진입점 | Cursor IDE, Slack, 웹, 모바일 |
| 아티팩트 | 스크린샷, 로그, diff 자동 생성 |
| 모니터링 | 실시간 로그 스트리밍 |

### 격리된 VM 아키텍처의 의미

Background Agent는 로컬 머신이 아닌 클라우드의 Ubuntu VM에서 실행된다. 이 구조가 주는 이점:

**1. 환경 독립성**
```
로컬 머신 상태와 무관하게 실행:
  - 로컬에서 다른 작업 중이어도 에이전트 영향 없음
  - 노트북을 닫아도 에이전트 계속 실행
  - 로컬 Node.js/Python 버전 충돌 없음
```

**2. 실제 환경 실행**
```
VM에서 가능한 작업:
  - npm install, pip install (패키지 설치)
  - 테스트 실행 및 결과 수집
  - 빌드 프로세스 실행
  - 외부 API 호출
  - 스크린샷 캡처 (Playwright, Puppeteer)
```

**3. 완전한 프로세스 격리**
```
에이전트 A가 npm install 실행 중이어도
에이전트 B의 테스트 실행에 영향 없음
```

### Git Worktree 기반 파일 충돌 방지

여러 에이전트가 같은 저장소에서 동시에 작업해도 충돌이 발생하지 않는 메커니즘이다.

```
저장소 구조 (Background Agent 실행 시):

main 브랜치
  └── agent-feature-auth (Worktree 1) ← 에이전트 1 전용
  └── agent-feature-payment (Worktree 2) ← 에이전트 2 전용
  └── agent-fix-login (Worktree 3) ← 에이전트 3 전용

각 Worktree는 독립적인 작업 공간
→ 같은 파일을 동시에 수정해도 브랜치가 달라 충돌 없음
→ 에이전트 완료 후 PR로 검토 → merge
```

### 체크포인트 1

- [ ] Background Agent가 로컬 VM이 아닌 클라우드 Ubuntu VM에서 실행된다는 것을 이해했다
- [ ] Git Worktree 기반 충돌 방지 메커니즘을 설명할 수 있다
- [ ] 최대 8개 병렬 실행의 의미를 파악했다

---

## Phase 2: 서브에이전트 설정 및 실행

### Cursor IDE에서 Background Agent 시작

**방법 1: Composer에서 시작**

```
Cursor → Composer (Cmd/Ctrl + I)
→ "Background Agent로 실행" 옵션 선택
→ 작업 지시 입력
→ [Submit] 클릭

예시:
"전체 API 엔드포인트에 대한 통합 테스트를 작성해줘.
 Jest + Supertest를 사용하고, 인증이 필요한 엔드포인트는
 모킹 없이 실제 테스트 DB에 연결해서 테스트해줘."
```

**방법 2: 커맨드 팔레트에서 시작**

```
Cmd/Ctrl + Shift + P → "Cursor: Start Background Agent"
→ 에이전트 이름 지정 (선택)
→ 작업 지시 입력
```

**방법 3: 현재 대화에서 백그라운드로 전환**

```
Composer 대화 중:
> 이 작업을 백그라운드에서 처리해줘. 완료되면 알려줘.

Cursor가 자동으로 Background Agent로 전환
→ 에이전트가 별도 VM에서 실행 시작
→ Composer 탭은 새 대화를 시작할 수 있게 해제
```

### Slack에서 Background Agent 시작

```
Slack에서 @cursor 멘션으로 에이전트 시작:

@cursor "users 테이블에 email 인덱스를 추가하고 마이그레이션 파일을 생성해줘.
         PR을 올려줘."

@cursor "최근 배포 이후 API 응답 시간이 느려졌어.
         원인을 분석하고 최적화 PR 올려줘."

@cursor "다음 이슈들을 모두 수정해줘: #123, #124, #125
         각각 별도 PR로 만들어줘."
```

Slack 에이전트 기능 사용 조건:
- Cursor Slack 앱 설치 필요
- 저장소 연결 설정 필요
- Team/Business 플랜에서 사용 가능

### 웹/모바일에서 에이전트 시작

```
Cursor 웹 인터페이스 (cursor.com):
  → 저장소 선택
  → "New Agent" 클릭
  → 작업 지시 입력
  → 실행 확인

모바일 앱 (iOS/Android):
  → 저장소 선택
  → 자연어로 작업 지시
  → 에이전트 시작
  → 완료 알림 수신
```

### 에이전트 설정 옵션

```
Background Agent 고급 설정:

Branch name: "agent/integrate-stripe-payment"
Base branch: "main"
Max execution time: 60분 (기본값)
Auto-create PR: 켜기
PR reviewer: @team-lead
Run tests: 켜기
Allow internet: 켜기 (패키지 설치 필요 시)
```

### 체크포인트 2

- [ ] Cursor Composer에서 Background Agent를 시작했다
- [ ] Slack에서 @cursor 멘션으로 에이전트를 지시해봤다
- [ ] 에이전트의 고급 설정(브랜치명, PR 설정)을 구성했다

---

## Phase 3: 병렬 처리 패턴

### 패턴 1: 기능별 병렬 개발

```
스프린트 목표: 3개 기능 동시 개발

에이전트 1 "feature/user-profile":
  지시: "사용자 프로필 페이지 구현.
         프로필 사진 업로드, 닉네임 변경, 비밀번호 변경 포함.
         React + TypeScript, API까지 풀스택으로 구현해줘."
  Worktree: feature/user-profile

에이전트 2 "feature/notifications":
  지시: "실시간 알림 시스템 구현.
         WebSocket 기반, 읽음 표시, 알림 종류별 필터링 포함."
  Worktree: feature/notifications

에이전트 3 "feature/dashboard":
  지시: "관리자 대시보드 구현.
         사용자 통계, 매출 차트, 최근 활동 로그 포함.
         Chart.js 사용."
  Worktree: feature/dashboard

→ 3개 에이전트 동시 실행 (각각 별도 Worktree)
→ 각 완료 시 PR 자동 생성
→ 팀 리뷰 후 순차 merge
```

### 패턴 2: 대규모 이슈 배치 처리

```
Slack에서:
@cursor "다음 GitHub 이슈를 모두 처리해줘, 각각 별도 PR:
  #201 - 로그인 페이지 로딩 느림
  #202 - 다크모드 토글 오류
  #203 - 검색 결과 정렬 버그
  #204 - 첨부파일 다운로드 실패
  #205 - 이메일 인증 만료 처리 누락"

Cursor가 자동으로:
  ├── 에이전트 1: 이슈 #201 (성능 최적화)
  ├── 에이전트 2: 이슈 #202 (CSS 버그)
  ├── 에이전트 3: 이슈 #203 (정렬 로직)
  ├── 에이전트 4: 이슈 #204 (파일 처리)
  └── 에이전트 5: 이슈 #205 (인증 흐름)

5개 에이전트 동시 실행 → 5개 PR 자동 생성
```

### 패턴 3: 플랫폼 별 병렬 구현

```
목표: iOS, Android, 웹 세 플랫폼에 동일 기능 동시 추가

에이전트 1 "web-dark-mode":
  지시: "웹 앱(React)에 다크모드 구현.
         시스템 설정 감지, 수동 토글, localStorage 저장 포함."
  Worktree: feature/web-dark-mode

에이전트 2 "ios-dark-mode":
  지시: "iOS 앱(Swift)에 다크모드 지원 추가.
         iOS 13+ API 활용, 다이나믹 컬러 적용."
  Worktree: feature/ios-dark-mode

에이전트 3 "android-dark-mode":
  지시: "Android 앱(Kotlin)에 다크모드 지원 추가.
         Material3 테마 적용, 앱 재시작 없이 전환 가능."
  Worktree: feature/android-dark-mode

→ 3개 플랫폼 동시 구현
→ 각 플랫폼 팀에서 PR 검토
```

### 패턴 4: 테스트 피라미드 병렬 구성

```
에이전트 1 "unit-tests":
  지시: "모든 서비스 클래스에 대한 단위 테스트를 작성해줘.
         각 public 메서드, 엣지 케이스, 오류 경로 포함."

에이전트 2 "integration-tests":
  지시: "모든 API 엔드포인트에 대한 통합 테스트를 작성해줘.
         실제 테스트 DB 연결, 인증 흐름 포함."

에이전트 3 "e2e-tests":
  지시: "핵심 사용자 플로우(회원가입, 로그인, 핵심 기능)에 대한
         Playwright E2E 테스트를 작성해줘.
         스크린샷 캡처 포함."

→ 3개 에이전트가 동시에 테스트 피라미드 구성
→ CI/CD 파이프라인에 자동 통합
```

### 체크포인트 3

- [ ] 기능별 병렬 개발 패턴으로 3개 이상의 에이전트를 동시 실행했다
- [ ] Git Worktree가 각 에이전트에 자동으로 할당되는 것을 확인했다
- [ ] 에이전트 완료 후 PR이 자동 생성되는 것을 확인했다

---

## Phase 4: 실전 시나리오

### 시나리오 1: 주말 대규모 리팩토링

```
금요일 퇴근 전 에이전트 8개 동시 시작:

에이전트 1: "src/models/ 전체를 Prisma ORM으로 마이그레이션"
에이전트 2: "CommonJS를 ES Modules로 변환"
에이전트 3: "JavaScript를 TypeScript로 변환 (백엔드)"
에이전트 4: "JavaScript를 TypeScript로 변환 (프론트엔드)"
에이전트 5: "Jest를 Vitest로 마이그레이션"
에이전트 6: "CSS를 Tailwind로 마이그레이션"
에이전트 7: "API 에러 핸들링 표준화"
에이전트 8: "전체 JSDoc 작성"

주말 동안 8개 에이전트 병렬 실행
월요일 출근 시 8개 PR 검토
```

### 시나리오 2: 보안 취약점 긴급 패치

```
보안 감사에서 7개 취약점 발견:

각 취약점을 별도 에이전트로 처리 (7개 병렬):

에이전트 1: "SQL 인젝션 취약점 패치 (7개 파일)"
에이전트 2: "XSS 취약점 입력 검증 추가"
에이전트 3: "CORS 설정 강화"
에이전트 4: "JWT 만료 처리 개선"
에이전트 5: "Rate limiting 추가"
에이전트 6: "민감 정보 로그 출력 제거"
에이전트 7: "HTTPS 강제 리다이렉션 추가"

→ 7개 에이전트 동시 실행
→ 보안팀 리뷰 후 긴급 merge
→ 전체 작업 시간: 7개 × 30분 → 병렬로 30분
```

### 시나리오 3: 모바일 앱에서 에이전트 시작 (이동 중)

```
점심 식사 중 모바일 앱에서:

Cursor 모바일 → 저장소 선택
→ "결제 모듈에서 발생한 프로덕션 버그를 수정하고 PR 올려줘.
   에러 로그는 Sentry에서 확인해봐.
   #SENTRY-ERROR-2024-789 참고."

→ 에이전트가 자동으로:
  1. Sentry API로 에러 상세 확인
  2. 관련 코드 분석
  3. 버그 수정
  4. 재현 테스트 작성
  5. PR 생성
  6. 슬랙으로 완료 알림

→ 식사 후 슬랙 알림 확인 → PR 리뷰
```

---

## 에이전트 완료 후 워크플로우

### 완료 알림

에이전트가 작업을 완료하면:
- Cursor IDE에 알림 표시
- Slack DM 또는 채널 알림 (설정 시)
- 이메일 알림 (설정 시)
- 모바일 푸시 알림

### PR 검토 프로세스

```
알림 수신 → PR 링크 클릭

GitHub PR 페이지에서:
  - Changed Files: 에이전트가 수정한 파일 목록
  - Diff: 세부 변경 내용
  - Checks: 자동 테스트 결과
  - Screenshots: E2E 테스트 스크린샷 (있는 경우)
  - Logs: 실행 로그 (에이전트 작업 과정)

리뷰 후:
  - Approve + Merge: 작업 승인
  - Request Changes: 추가 수정 요청 → 에이전트에게 피드백
  - Close: 작업 거부 (Worktree 자동 정리)
```

### 에이전트에게 피드백 전달

```
PR 코멘트에서:
"@cursor 인증 미들웨어가 public 엔드포인트에도 적용되고 있어.
 /health와 /status 엔드포인트는 인증 제외해줘."

→ 에이전트가 피드백을 읽고 같은 브랜치에서 추가 수정
→ PR 자동 업데이트
```

---

## 다른 도구와의 비교표

| 항목 | Gemini CLI | Claude Code | Codex CLI | Antigravity | Cursor | VS Code Copilot |
|------|-----------|-------------|-----------|-------------|--------|-----------------|
| 서브에이전트 방식 | 없음 (수동) | Task() 내장 | 실험적 플래그 | Agent Manager | Background VM | Explore + Coding Agent |
| 실행 환경 | 로컬 | 로컬 | 로컬/클라우드 | 로컬 | 격리된 Ubuntu VM | 클라우드 |
| 최대 병렬 수 | 수동 | 7개 | 제한적 | 메모리 제한 | 8개 | 제한적 |
| 인터넷 접근 | 없음 | 없음 | 없음 | 없음 | 있음 (VM) | 있음 |
| 파일 충돌 방지 | 없음 | 수동 | 없음 | 수동 | Git Worktree 자동 | 없음 |
| PR 자동 생성 | 없음 | 없음 | 없음 | 없음 | 있음 | 있음 |
| 모바일 시작 | 없음 | 없음 | 없음 | 없음 | 있음 | 없음 |
| Slack 통합 | 없음 | 없음 | 없음 | 없음 | 있음 | 없음 |
| 비용 | API 사용량 | API 사용량 | API 사용량 | 구독 | 구독 + 컴퓨팅 | GitHub 구독 |

---

## 주의사항 및 모범 사례

### 비용 관리

```
Background Agent 비용 구조:
  - VM 실행 시간 기반 과금
  - 에이전트 수 × 실행 시간 = 총 비용
  - 인터넷 접근 시 외부 API 비용 추가 가능

비용 절감 방법:
  1. Max execution time을 작업에 맞게 조정
  2. 불필요한 에이전트는 중단 (Cursor → Agent 목록 → Cancel)
  3. 작은 단위로 작업을 나눠 중간 검토 후 진행
```

### 보안 고려사항

```
VM에서 실행되는 코드 주의사항:
  1. 환경변수 설정: .env 파일의 실제 값이 VM에 전달되지 않도록 주의
  2. API 키: 에이전트에게 프로덕션 API 키 접근 권한 부여 최소화
  3. 데이터베이스: 에이전트용 별도 읽기 전용 DB 연결 사용
  4. 네트워크: 에이전트의 인터넷 접근 범위를 필요한 도메인으로 제한

Cursor 설정:
  Settings → Background Agents → Allowed Domains
  예: npm.registry.com, pypi.org, api.stripe.com (필요한 것만)
```

### 에이전트 수 제한 전략

```
최대 8개이지만, 실제로는 품질을 위해 제한하는 것이 좋다:

3개 이하: 각 에이전트 결과를 꼼꼼히 리뷰할 수 있는 수준
4~6개: 어느 정도 자동화 신뢰 시
7~8개: 패턴이 잘 정의된 반복 작업만 (예: 이슈 배치 처리)
```

### Git Worktree 관리

```bash
# 에이전트가 생성한 Worktree 목록 확인
git worktree list

# 사용하지 않는 Worktree 정리 (에이전트 완료 후 PR merge된 것)
git worktree prune

# 특정 Worktree 제거
git worktree remove agent/feature-name
```

### 모범 사례 요약

1. **명확한 완료 기준 제시**: "PR을 생성하면 완료"처럼 에이전트가 언제 멈춰야 하는지 명시
2. **브랜치 명명 규칙**: `agent/기능명` 형식으로 에이전트 생성 브랜치를 구분
3. **테스트 실행 포함**: 지시사항에 "테스트도 실행하고 통과 여부 확인해줘" 항상 포함
4. **범위 제한**: 전체 저장소보다 특정 디렉토리/모듈로 범위 제한
5. **PR 리뷰 필수**: `auto-merge`는 사용하지 않고 반드시 인간이 검토 후 merge

---

## 참고 자료

- [Cursor Background Agents 공식 문서](https://docs.cursor.com/background-agents)
- [Cursor Slack 통합 가이드](https://docs.cursor.com/integrations/slack)
- [Git Worktree 공식 문서](https://git-scm.com/docs/git-worktree)
- [Cursor 변경 내역](https://cursor.com/changelog)
