# Agentic Coding 한국어 실습 가이드

![version](https://img.shields.io/badge/version-v2026.03.13-blue)

AI 코딩 에이전트 도구 6종의 설치부터 고급 활용까지를 체계적으로 다루는 **한국어 실습 가이드** 프로젝트입니다.

## 다루는 도구

| 유형 | 도구 | 특징 |
|------|------|------|
| CLI 기반 | **Claude Code** | 확장 사고, Hooks, 메모리, MCP 연동 |
| CLI 기반 | **Gemini CLI** | Google 통합 환경, Hook, 빠른 응답 |
| CLI 기반 | **Codex** | OpenAI 자율 주행 CLI |
| 에디터 네이티브 | **Cursor** | AI 코어 내장, Composer, 백그라운드 Agent |
| 에디터 네이티브 | **Antigravity** | GUI 기반 Agent Manager, 무제한 병렬 |
| 에디터 네이티브 | **VS Code Copilot** | VS Code 네이티브, GitHub 통합 |

## 가이드 구성

```
01_agentic_coding_tool/   도구별 설치 및 기본 사용법
02_rules/                 Rule 파일 작성 (CLAUDE.md, GEMINI.md 등)
03_mcp/                   MCP 서버 연동 (GitHub, Notion, Slack 등)
04_skills/                커스텀 Skills/Commands 작성
05_workflow/              Workflow 및 Slash Command 구현
06_hooks/                 Hooks 이벤트 자동화
07_subagent/              서브에이전트/멀티에이전트 구현
08_plugin/                Plugin/Extension 패키지 개발
```

각 카테고리 안에 6개 도구별 가이드가 있으며, 도구 간 기능 비교표와 실행 가능한 코드 예제를 포함합니다.

## 학습 경로

**입문** → `01_agentic_coding_tool` → `02_rules`
**중급** → `03_mcp` → `04_skills` → `05_workflow` → `06_hooks`
**고급** → `07_subagent` → `08_plugin`

## 문서 공통 구조

모든 가이드는 다음 흐름으로 구성됩니다:

1. 개요 및 학습 목표
2. 설치/설정
3. 핵심 기능 (Phase별 단계 실습)
4. 도구 비교표 (Claude Code 기준 대응 관계)
5. 고급 활용 및 모범 사례
6. 체크포인트 (이해도 확인)

## 라이센스

[MIT](LICENSE)
