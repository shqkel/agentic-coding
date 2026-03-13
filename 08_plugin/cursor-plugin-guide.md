# Cursor Plugin 학습 가이드

## 개요

Cursor는 **두 가지 확장 체계**를 가지고 있다. 하나는 VS Code에서 물려받은 기존 확장(Extension) 시스템이고, 다른 하나는 Cursor 2.5(2026년 2월)부터 도입된 네이티브 Plugin 시스템이다. 전자는 에디터 기능을 확장하고, 후자는 AI 에이전트의 능력을 확장한다. 이 가이드는 두 체계의 차이를 이해하고, 특히 새로운 Plugin 시스템을 실무에서 활용하는 방법을 다룬다.

---

## 1. Cursor의 두 가지 확장 체계

### 한 줄 정리

> VS Code 확장 = 에디터를 확장한다. Cursor Plugin = AI 에이전트를 확장한다.

Cursor는 VS Code를 직접 포크한 에디터이다. 따라서 VS Code의 확장 생태계를 그대로 사용할 수 있다. 여기까지는 "VS Code와 다를 게 없다"고 생각할 수 있다.

하지만 Cursor 2.5부터 도입된 Plugin 시스템은 완전히 다른 개념이다. VS Code 확장이 에디터의 UI, 언어 지원, 디버거 같은 "편집 환경"을 확장한다면, Cursor Plugin은 AI 에이전트가 사용할 수 있는 "스킬, 지식, 도구, 규칙"을 확장한다. 스마트폰에 비유하면, VS Code 확장은 홈 화면 위젯이고 Cursor Plugin은 AI 비서에게 새로운 업무 능력을 가르치는 것이다.

### 왜 두 개가 공존하는가

Cursor 팀의 설계 철학은 "기존 에디터 경험을 깨지 않으면서 AI 네이티브 확장을 추가한다"이다. 이미 VS Code 확장 생태계에는 수만 개의 확장이 존재한다. 이를 버리지 않고 그대로 호환하면서, AI 에이전트 전용 확장 레이어를 새로 올렸다. 결과적으로 사용자는 에디터 기능은 VS Code 확장으로, AI 에이전트 기능은 Cursor Plugin으로 각각 확장할 수 있다.

---

## 2. VS Code 확장 호환

### 어떻게 동작하는가

Cursor는 VS Code를 직접 포크했으므로 VS Code 확장을 **완전히 호환**한다. 기존에 VS Code를 사용하던 개발자라면 설정, 확장, 키바인딩, 테마를 원클릭으로 임포트할 수 있다.

```
Cursor 설정 → VS Code Import → 원클릭 마이그레이션
```

VS Code Marketplace에서 확장을 검색하고 설치하는 방식도 동일하다. 기존에 사용하던 Prettier, ESLint, GitLens 같은 확장이 Cursor에서도 그대로 동작한다.

### 제약사항

모든 VS Code 확장이 Cursor에서 동작하는 것은 아니다. Microsoft가 직접 개발한 일부 확장은 **이용약관상 제한**이 있다.

**제한되는 대표적 확장**

| 확장 | 이유 |
|------|------|
| C/C++ (ms-vscode.cpptools) | Microsoft 전용 라이선스 |
| Remote - SSH (ms-vscode-remote) | "in-scope products"만 허용 |
| Live Share | Microsoft 계정 필수 |

Microsoft의 VS Code Marketplace 이용약관은 확장 사용을 "in-scope products"로 제한한다. Cursor는 VS Code의 포크이지 공식 배포판이 아니므로, Microsoft가 직접 제공하는 일부 확장이 마켓플레이스에서 설치되지 않는다.

**우회 방법**: `.vsix` 파일을 직접 다운로드하여 수동 설치할 수 있다. GitHub Releases나 Open VSX Registry에서 대체 확장을 찾는 것도 방법이다.

```bash
# .vsix 파일 수동 설치
# 1. https://marketplace.visualstudio.com 에서 .vsix 다운로드
# 2. Cursor에서 설치
Cursor → Extensions → ... → Install from VSIX → 파일 선택
```

---

## 3. Cursor Plugin 시스템 (2.5+)

### Plugin이란 무엇인가

> Plugin = Skills + Subagents + MCP Servers + Hooks + Rules가 **하나의 패키지로 묶인 AI 에이전트 확장 단위**이다.

Cursor Plugin은 Claude Code의 Plugin과 가장 유사한 개념이다. 에디터의 UI를 바꾸는 것이 아니라, AI 에이전트가 "무엇을 알고 있는지", "어떤 도구를 쓸 수 있는지", "어떤 규칙을 따르는지"를 확장한다.

예를 들어 Stripe Plugin을 설치하면, Cursor의 AI 에이전트는 Stripe API 문서를 이해하고, 결제 관련 코드를 작성할 때 Stripe의 모범 사례를 자동으로 적용하며, Stripe 대시보드에서 데이터를 직접 가져올 수 있게 된다. 이 모든 것이 하나의 Plugin 안에 묶여 있다.

### Plugin 아키텍처 - 5가지 통합 구성 요소

하나의 Cursor Plugin은 다음 5가지 구성 요소를 포함할 수 있다. 모두 필수는 아니며, 필요한 것만 선택적으로 포함하면 된다.

```
Cursor Plugin (확장 단위)
│
├── Skills (스킬)              → 도메인별 프롬프트/코드, 에이전트가 자동 발견·실행
│   └── 예: "Stripe 결제 통합 시 이 패턴을 따르라"
│
├── Subagents (서브에이전트)     → 병렬 작업 전문 에이전트
│   └── 예: 테스트 생성, 문서화, 보안 감사를 동시에 수행
│
├── MCP Servers (외부 연결)     → 외부 서비스 실시간 연결
│   └── 예: Linear 이슈 읽기, Figma 디자인 가져오기
│
├── Hooks (이벤트 핸들러)        → 에이전트 행동 관찰/제어
│   └── 예: 코드 수정 후 자동 린팅, 민감 파일 접근 시 경고
│
└── Rules (코딩 규칙)           → 코딩 표준 강제
    └── 예: "이 프로젝트에서는 반드시 TypeScript strict 모드를 사용하라"
```

각 구성 요소를 상세히 살펴본다.

---

### 3-1. Skills (스킬)

**한 줄 설명**: 도메인별 전문 지식을 담은 프롬프트와 코드이다. 에이전트가 맥락에 따라 **자동으로 발견하고 실행**한다.

**어떻게 동작하는가**

스킬은 "이런 상황에서는 이렇게 하라"는 지식 블록이다. 사용자가 명시적으로 호출하지 않아도, 에이전트가 현재 작업의 맥락을 파악하여 관련 스킬을 자동으로 활성화한다.

예를 들어 AWS Plugin의 스킬은 "Lambda 함수를 작성할 때는 콜드 스타트를 최소화하는 패턴을 사용하라", "DynamoDB 테이블 설계 시 단일 테이블 디자인을 우선 고려하라" 같은 전문 지식을 담고 있다. 사용자가 AWS 관련 코드를 작성하면 에이전트가 이 스킬들을 자동으로 참조한다.

**Claude Code의 Skills와의 차이**: Claude Code에서 스킬은 `SKILL.md` 파일로 정의되며, 대화 맥락에 직접 로드된다. Cursor의 스킬도 동일한 개념이지만 Plugin 매니페스트를 통해 관리되고, Cursor의 에이전트 런타임이 활성화를 결정한다.

**언제 쓰는가**: 특정 도메인(클라우드 서비스, 프레임워크, 결제 시스템 등)의 모범 사례를 에이전트에 내재화하고 싶을 때.

---

### 3-2. Subagents (서브에이전트)

**한 줄 설명**: **독립된 컨텍스트에서 병렬로 작업하는 전문 에이전트**이다. 메인 에이전트가 서브태스크를 위임하는 대상이다.

**어떻게 동작하는가**

복잡한 작업을 하나의 에이전트가 순차적으로 처리하면 시간이 오래 걸리고 컨텍스트가 오염된다. Subagent는 서브태스크를 격리된 환경에서 병렬로 처리한 뒤 결과만 메인 에이전트에게 돌려준다.

```
메인 에이전트: "이 PR을 리뷰해줘"
├── Subagent A: 코드 품질 분석 (병렬)
├── Subagent B: 보안 취약점 스캔 (병렬)
└── Subagent C: 테스트 커버리지 확인 (병렬)
→ 결과 통합 후 종합 리뷰 제공
```

**2026년 3월 업데이트 - 트리 구조**: 최근 업데이트에서 Subagent가 자체적으로 하위 Subagent를 생성할 수 있게 되었다. 이로써 작업을 트리 구조로 분해하여 더 복잡한 태스크를 효율적으로 처리할 수 있다.

```
메인 에이전트
├── Subagent: 프론트엔드 리팩토링
│   ├── Sub-subagent: 컴포넌트 분리
│   └── Sub-subagent: 스타일 정리
└── Subagent: 백엔드 API 수정
    ├── Sub-subagent: 엔드포인트 구현
    └── Sub-subagent: 테스트 작성
```

**Claude Code의 Agents와의 차이**: Claude Code의 서브에이전트는 `.md` 파일로 정의되며, `tools` 필드로 사용 가능한 도구를 제한한다. Cursor의 Subagent도 비슷한 개념이지만, 트리 구조 생성이 가능하다는 점이 차별화된다.

**언제 쓰는가**: 병렬로 처리할 수 있는 독립적인 서브태스크가 여러 개일 때. 코드 리뷰, 대규모 리팩토링, 멀티 파일 생성 같은 작업에 적합하다.

---

### 3-3. MCP Servers (외부 연결)

**한 줄 설명**: 에이전트가 **외부 서비스(API, DB, SaaS)와 실시간으로 통신하는 통로**이다.

**어떻게 동작하는가**

MCP(Model Context Protocol) 서버는 외부 서비스의 기능을 에이전트가 호출할 수 있는 "도구(tool)"로 노출한다. Cursor Plugin에 MCP 서버를 포함시키면, 설치만으로 에이전트가 해당 서비스에 접근할 수 있게 된다.

예를 들어 Linear Plugin은 MCP 서버를 통해 에이전트가 Linear 이슈를 읽고, 상태를 변경하고, 새 이슈를 생성할 수 있게 한다. Figma Plugin은 디자인 토큰을 가져와 코드에 반영할 수 있게 한다.

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@linear/mcp-server"],
      "env": {
        "LINEAR_API_KEY": "${LINEAR_API_KEY}"
      }
    },
    "figma": {
      "command": "npx",
      "args": ["-y", "@figma/mcp-server"],
      "env": {
        "FIGMA_TOKEN": "${FIGMA_TOKEN}"
      }
    }
  }
}
```

**언제 쓰는가**: 에이전트가 외부 서비스의 데이터를 읽거나 조작해야 할 때. 프로젝트 관리 도구, 디자인 도구, 모니터링 서비스 등과의 연동에 사용한다.

---

### 3-4. Hooks (이벤트 핸들러)

**한 줄 설명**: 에이전트의 **특정 행동을 관찰하고 제어하는 이벤트 핸들러**이다.

**어떻게 동작하는가**

Hooks는 에이전트가 파일을 수정하거나 명령을 실행하는 등의 행동을 할 때 자동으로 반응한다. "감시자" 역할로, 에이전트의 행동을 모니터링하고 필요하면 개입할 수 있다.

| 사용 시나리오 | Hook 동작 |
|-------------|-----------|
| 코드 파일 수정 후 | 자동으로 포매터(Prettier, Black) 실행 |
| `.env` 파일 접근 시도 | 경고 표시 및 차단 |
| 터미널 명령 실행 전 | 위험한 명령 필터링 |
| 작업 완료 시 | 결과 로깅, 알림 전송 |

**Claude Code의 Hooks와의 차이**: Claude Code는 `PreToolUse`, `PostToolUse`, `Notification`, `Stop` 네 가지 훅 유형을 명시적으로 정의한다. Cursor의 Hooks도 동일한 패턴(행동 전/후 개입)을 따르며, Plugin 매니페스트를 통해 설정한다.

**언제 쓰는가**: 에이전트의 행동에 자동으로 반응하는 워크플로우가 필요할 때. 품질 보증, 보안 정책 적용, 자동화된 후처리 등에 활용한다.

---

### 3-5. Rules (코딩 규칙)

**한 줄 설명**: 에이전트가 코드를 작성할 때 **반드시 따라야 하는 코딩 표준**이다.

**어떻게 동작하는가**

Rules는 프로젝트의 코딩 컨벤션, 아키텍처 패턴, 금지 사항 등을 에이전트에게 강제한다. Plugin에 Rules를 번들링하면, 해당 Plugin을 설치한 모든 팀원의 에이전트가 동일한 규칙을 따르게 된다.

```
# Plugin에 번들된 Rules 예시

## TypeScript 규칙
- `any` 타입 사용 금지
- 모든 함수에 리턴 타입 명시
- enum 대신 const assertion 사용

## React 규칙
- 클래스 컴포넌트 사용 금지
- 상태 관리는 Zustand만 사용
- 컴포넌트 파일당 하나의 export만 허용
```

**Claude Code와의 차이**: Claude Code에는 Plugin 내에 Rules를 번들링하는 기능이 없다. 대신 `CLAUDE.md` 파일이나 `.claude/settings.json`을 통해 프로젝트별 규칙을 설정한다. Cursor Plugin의 Rules 번들링은 팀 전체에 동일한 코딩 표준을 배포할 때 매우 편리하다.

**언제 쓰는가**: 팀의 코딩 표준을 일관되게 강제하고 싶을 때. 특히 대규모 팀에서 새 팀원이 합류했을 때 별도의 온보딩 없이 규칙이 적용되도록 할 때 유용하다.

---

## 4. Plugin vs VS Code 확장 비교

두 시스템은 대상이 다르다. 하나는 에디터를, 다른 하나는 AI 에이전트를 확장한다. 정확한 차이를 이해해야 올바른 선택을 할 수 있다.

| 구분 | VS Code 확장 | Cursor Plugin |
|------|-------------|---------------|
| **확장 대상** | 에디터 기능 (UI, 언어 지원, 디버거) | AI 에이전트 (스킬, 도구, 규칙) |
| **구성 요소** | 단일 확장 패키지 | Skills + Subagents + MCP + Hooks + Rules |
| **설치 경로** | VS Code Marketplace | Cursor Marketplace / `/add-plugin` |
| **에이전트 통합** | 없음 | 완전 통합 |
| **병렬 실행** | 미지원 | Subagent로 지원 |
| **공유 방법** | Marketplace 공개만 | Marketplace + 프라이빗 팀 배포 |
| **런타임** | Node.js Extension Host | Cursor Agent Runtime |

**실무 판단 기준**

- "에디터에 새로운 언어 하이라이팅을 추가하고 싶다" → VS Code 확장
- "AI 에이전트가 우리 회사의 API 규약을 이해하게 하고 싶다" → Cursor Plugin
- "코드 스니펫을 자동완성하고 싶다" → VS Code 확장
- "에이전트가 Jira 티켓을 읽고 구현하게 하고 싶다" → Cursor Plugin

---

## 5. Claude Code Plugin과의 비교

Claude Code Plugin과 Cursor Plugin은 모두 "AI 에이전트를 확장한다"는 동일한 목표를 가지고 있다. 하지만 실행 환경(CLI vs 에디터)과 구성 요소에서 차이가 있다.

| 구분 | Claude Code | Cursor |
|------|-------------|--------|
| **실행 환경** | CLI (터미널) | 에디터 (GUI) |
| **매니페스트** | `plugin.json` | Plugin 매니페스트 |
| **마켓플레이스** | Plugin Marketplace (Git 기반) | Cursor Marketplace (cursor.com) |
| **구성 요소** | Commands + Skills + Agents + MCP + Hooks + LSP | Skills + Subagents + MCP + Hooks + Rules |
| **에디터 확장** | 없음 (순수 CLI) | VS Code 호환 확장 별도 지원 |
| **프라이빗 공유** | Git 저장소 기반 마켓플레이스 | Team/Enterprise 프라이빗 마켓플레이스 |
| **고유 기능** | LSP 서버, 슬래시 Commands | Rules 번들링, 인터랙티브 UI, Subagent 트리 구조 |
| **설치 명령** | `/plugin install` | `/add-plugin` |

### 구성 요소 대응 관계

```
Claude Code                    Cursor
─────────────────────────────────────────────────
Commands (슬래시 명령어)    →  해당 없음 (에디터 UI로 대체)
Skills (자동 활성화 지식)   →  Skills (동일 개념)
Agents (서브에이전트)       →  Subagents (트리 구조 추가)
MCP Servers (외부 연결)     →  MCP Servers (동일 개념)
Hooks (이벤트 핸들러)       →  Hooks (동일 개념)
LSP Servers (코드 분석)     →  해당 없음 (VS Code 확장으로 제공)
해당 없음                   →  Rules (코딩 규칙 번들링)
```

### 핵심 차이 해설

**Commands vs 에디터 UI**: Claude Code는 CLI이므로 `/deploy` 같은 슬래시 명령어가 워크플로우의 핵심이다. Cursor는 GUI 에디터이므로 에이전트 채팅 창에서 자연어로 요청하거나 에디터 UI를 통해 동일한 기능을 수행한다.

**LSP vs 내장 언어 지원**: Claude Code는 별도의 LSP Plugin이 필요하지만, Cursor는 VS Code 포크이므로 VS Code 확장을 통해 이미 LSP 기능을 가지고 있다. 별도의 Plugin 구성 요소로 제공할 필요가 없다.

**Rules 번들링**: Claude Code에는 없는 Cursor만의 기능이다. 팀 전체의 코딩 규칙을 Plugin에 포함시켜 설치만으로 일괄 적용할 수 있다. Claude Code에서는 `CLAUDE.md`를 Git으로 공유하는 방식으로 유사한 효과를 얻는다.

---

## 6. Cursor Marketplace

### 개요

Cursor Marketplace는 **cursor.com/marketplace**에서 접근할 수 있는 Plugin 카탈로그이다. Claude Code의 Git 기반 마켓플레이스와 달리 웹 기반 전용 마켓플레이스를 운영한다.

### 공식 파트너 Plugin

Cursor는 주요 SaaS 벤더와 파트너십을 맺고 공식 Plugin을 제공한다.

| 파트너 | Plugin 기능 |
|--------|-----------|
| Amplitude | 분석 이벤트 코드 자동 생성, 대시보드 데이터 참조 |
| AWS | 클라우드 서비스 설정, IaC 코드 생성, 모범 사례 적용 |
| Figma | 디자인 토큰 추출, UI 컴포넌트 코드 생성 |
| Linear | 이슈 읽기/쓰기, 작업 상태 관리, PR 연동 |
| Stripe | 결제 API 통합, 웹훅 설정, 테스트 데이터 생성 |

### Team/Enterprise 프라이빗 마켓플레이스

팀이나 기업은 **프라이빗 마켓플레이스**를 운영할 수 있다. 회사 내부의 Plugin을 외부에 공개하지 않고 팀원들에게만 배포할 수 있다. Enterprise 플랜에서 지원되며, 관리자가 승인한 Plugin만 팀원이 설치할 수 있도록 제어할 수 있다.

---

## 7. 빠른 시작

### Plugin 설치

```bash
# 1. 에이전트 채팅에서 /add-plugin 명령 사용
/add-plugin stripe

# 2. 또는 Marketplace에서 직접 검색
#    cursor.com/marketplace → 원하는 Plugin 검색 → Install 버튼

# 3. 설치된 Plugin 확인
#    Settings → Cursor → Plugins 에서 확인 가능
```

### VS Code 확장 임포트

기존 VS Code 사용자는 한 번에 모든 설정을 가져올 수 있다.

```
1. Cursor 실행
2. Command Palette (Ctrl+Shift+P / Cmd+Shift+P)
3. "Import VS Code Settings" 검색
4. 설정, 확장, 키바인딩, 테마 일괄 임포트
```

### Plugin 사용 예시

Plugin이 설치되면 에이전트 채팅에서 자연어로 요청하기만 하면 된다.

```
# Linear Plugin 설치 후
사용자: "LINEAR-1234 이슈를 읽고 구현해줘"
에이전트: Linear에서 이슈를 읽고, 요구사항을 분석하고, 코드를 작성한다.

# Figma Plugin 설치 후
사용자: "Figma의 로그인 화면 디자인을 React 컴포넌트로 만들어줘"
에이전트: Figma에서 디자인 토큰과 레이아웃을 가져와 컴포넌트를 생성한다.

# Stripe Plugin 설치 후
사용자: "구독 결제 기능을 추가해줘"
에이전트: Stripe 모범 사례에 따라 결제 로직을 구현한다.
```

### 인터랙티브 UI (2026년 3월 업데이트)

최근 업데이트에서 에이전트 채팅 내에 **인터랙티브 UI**가 지원된다. Plugin이 텍스트 응답 대신 버튼, 폼, 차트 같은 UI 요소를 에이전트 채팅 안에 직접 렌더링할 수 있다.

```
사용자: "AWS 비용 분석해줘"
에이전트: [인터랙티브 차트가 채팅 안에 렌더링됨]
         └── 서비스별 비용 막대 그래프
         └── 기간 선택 드롭다운
         └── "상세 보기" 버튼
```

---

## 8. Plugin 만들기

### 기본 구조

Cursor Plugin을 직접 만들려면 다음 구조를 따른다.

```
my-cursor-plugin/
├── manifest.json           ← Plugin 메타데이터 (필수)
├── skills/                 ← 도메인별 스킬 정의
│   └── code-review/
│       └── skill.md
├── subagents/              ← 서브에이전트 정의
│   └── security-scanner.md
├── mcp/                    ← MCP 서버 설정
│   └── mcp-config.json
├── hooks/                  ← 이벤트 핸들러
│   └── hooks.json
└── rules/                  ← 코딩 규칙
    └── typescript-rules.md
```

### 매니페스트 작성

```json
{
  "name": "my-awesome-plugin",
  "version": "1.0.0",
  "description": "우리 팀의 개발 워크플로우를 자동화하는 플러그인",
  "author": "Team Name",
  "skills": "./skills/",
  "subagents": "./subagents/",
  "mcpServers": "./mcp/mcp-config.json",
  "hooks": "./hooks/hooks.json",
  "rules": "./rules/"
}
```

### 스킬 작성 예시

```markdown
<!-- skills/api-design/skill.md -->
---
name: api-design
description: |
  REST API를 설계하거나 수정할 때 자동 활성화된다.
  "API 만들어줘", "엔드포인트 추가", "REST 설계" 등의 표현에 반응한다.
---

# API 설계 전문가

이 스킬이 활성화되면 다음 규칙에 따라 API를 설계한다:

- RESTful 명명 규칙을 준수한다 (복수형 리소스명, 소문자 케밥 케이스)
- 에러 응답은 RFC 7807 Problem Details 형식을 따른다
- 페이지네이션은 커서 기반을 기본으로 사용한다
- 버전은 URL 경로에 포함한다 (예: /v1/users)
```

### Cursor Marketplace 등록

1. Plugin 저장소를 GitHub에 공개한다
2. cursor.com/marketplace/submit 에서 등록 신청한다
3. Cursor 팀의 리뷰를 거쳐 Marketplace에 게시된다

프라이빗 팀 배포의 경우 Marketplace 등록 없이 Team/Enterprise 설정에서 직접 추가할 수 있다.

---

## 9. 보안 고려사항

### Plugin 설치 시 주의할 점

**MCP 서버 권한**: Plugin에 포함된 MCP 서버는 외부 서비스에 접근하는 권한을 가진다. 신뢰할 수 있는 출처의 Plugin만 설치해야 한다. 환경 변수로 전달되는 API 키가 안전하게 관리되는지 확인한다.

**Hooks의 명령 실행**: Hooks는 시스템 명령을 자동으로 실행할 수 있다. 악의적인 Plugin이 Hooks를 통해 시스템에 손상을 줄 수 있으므로, 설치 전에 Hooks 설정을 검토해야 한다.

**Rules의 영향 범위**: Plugin의 Rules는 에이전트의 코드 생성 패턴을 변경한다. 프로젝트의 기존 규칙과 충돌하지 않는지 확인한다.

### 팀 환경에서의 보안

| 권장 사항 | 설명 |
|-----------|------|
| 프라이빗 마켓플레이스 사용 | 승인된 Plugin만 팀원이 설치할 수 있도록 제한 |
| 환경 변수 관리 | API 키를 코드에 하드코딩하지 않고 환경 변수로 관리 |
| Plugin 버전 고정 | 자동 업데이트 대신 검증된 버전을 명시적으로 사용 |
| 정기 감사 | 설치된 Plugin 목록과 권한을 정기적으로 검토 |

### VS Code 확장 보안

VS Code 확장은 Node.js Extension Host에서 실행되므로 파일 시스템, 네트워크, 프로세스에 접근할 수 있다. Cursor Plugin보다 오히려 더 넓은 시스템 권한을 가질 수 있으므로, 신뢰할 수 없는 확장 설치에 주의해야 한다.

---

## 10. 자주 하는 실수

**실수 1**: VS Code 확장과 Cursor Plugin을 혼동한다

```
❌ "AI 에이전트가 Jira를 읽게 하려면 VS Code 확장을 설치하면 되겠지?"
✅ "AI 에이전트의 기능을 확장하려면 Cursor Plugin을 설치해야 한다."
```

VS Code 확장은 에디터의 UI와 편집 기능을 확장한다. AI 에이전트의 능력을 확장하는 것은 Cursor Plugin이다.

**실수 2**: Microsoft 전용 확장이 설치되지 않아 당황한다

```
❌ VS Code Marketplace에서 C/C++ 확장 설치 시도 → 실패
✅ .vsix 파일 직접 다운로드하여 수동 설치, 또는 Open VSX에서 대체 확장 검색
```

**실수 3**: Plugin 설치 후 환경 변수를 설정하지 않는다

```
❌ Linear Plugin 설치 → "이슈를 읽어줘" → 인증 오류
✅ LINEAR_API_KEY 환경 변수를 먼저 설정한 후 사용
```

MCP 서버가 포함된 Plugin은 대부분 외부 서비스의 API 키가 필요하다. Plugin 설치 후 필요한 환경 변수를 반드시 설정해야 한다.

**실수 4**: Rules가 기존 프로젝트 설정과 충돌한다

```
❌ Plugin Rules: "세미콜론 사용 금지" + 프로젝트 ESLint: "세미콜론 필수" → 혼란
✅ Plugin 설치 전에 Rules 내용을 검토하고, 프로젝트 규칙과 충돌 여부를 확인
```

**실수 5**: Plugin과 VS Code 확장에서 동일 기능이 중복 실행된다

```
❌ VS Code ESLint 확장 + Plugin의 Hooks에서 ESLint 실행 → 이중 린팅
✅ 기능이 겹치는 경우 한쪽만 활성화
```

---

## 참고 자료

- Cursor 공식 사이트: https://cursor.com
- Cursor Marketplace: https://cursor.com/marketplace
- Cursor 변경 로그: https://cursor.com/changelog
- Cursor Plugin 개발 문서: https://docs.cursor.com/plugins
