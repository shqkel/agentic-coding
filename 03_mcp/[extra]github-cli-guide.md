# GitHub CLI (gh) 학습 노트

## 개요

GitHub CLI는 GitHub의 기능을 터미널에서 직접 사용할 수 있도록 GitHub이 공식으로 제공하는 커맨드라인 도구이다.
브라우저를 열지 않고도 이슈 생성, PR 리뷰, 저장소 관리 등 대부분의 GitHub 작업을 수행할 수 있다.

**왜 GitHub CLI를 사용하는가?**

- 브라우저와 터미널 사이의 컨텍스트 전환을 줄여 개발 흐름을 유지할 수 있다
- 반복 작업을 스크립트로 자동화할 수 있다
- CI/CD 파이프라인에서 GitHub 작업을 자동화할 수 있다

---

## 핵심 내용

### 1. 설치 및 인증

#### 설치

```bash
# macOS (Homebrew)
brew install gh

# Ubuntu/Debian
sudo apt install gh

# Windows (winget)
winget install --id GitHub.cli
```

#### 인증 (Authentication)

```bash
# 인증 시작 (브라우저 또는 토큰 방식 선택 가능)
gh auth login

# 인증 상태 확인 (현재 부여된 스코프 확인 가능)
gh auth status
# 출력 예시:
# ✓ Token scopes: gist, read:org, repo, workflow

# 로그아웃
gh auth logout
```

인증 방식은 두 가지이다.

| 방식 | 설명 |
|------|------|
| 브라우저 인증 | 브라우저에서 GitHub 로그인 → 코드 입력 |
| 토큰 인증 | GitHub에서 발급한 Personal Access Token 입력 |

#### 스코프(Scope) 관리

스코프(Scope)는 토큰에 부여된 권한 범위이다. 기본 로그인 시 `repo`, `read:org` 등 기본 스코프만 부여되며, GitHub Projects 관리 등 추가 기능을 사용하려면 필요한 스코프를 명시적으로 요청해야 한다.

```bash
# 처음 로그인 시 스코프 지정
gh auth login --scopes "project"

# 이미 로그인된 상태에서 스코프 추가 (재로그인 불필요)
gh auth refresh --scopes "project"
```

주요 스코프 종류는 다음과 같다.

| 스코프 | 설명 |
|--------|------|
| `repo` | 저장소 읽기/쓰기 (기본 포함) |
| `read:org` | 조직 정보 읽기 (기본 포함) |
| `workflow` | GitHub Actions 워크플로우 관리 |
| `project` | GitHub Projects(v2) 읽기/쓰기 |
| `gist` | Gist 관리 |

> `project` 스코프가 없으면 `gh project` 명령어 실행 시 GraphQL 권한 오류가 발생한다. 프로젝트 관련 작업 전에 반드시 추가해야 한다.

---

### 2. 저장소(Repository) 관리

```bash
# 저장소 생성 (현재 디렉토리 기준)
gh repo create my-project

# 공개/비공개 설정과 함께 생성
gh repo create my-project --public
gh repo create my-project --private

# 저장소 클론
gh repo clone owner/repository

# 저장소 정보 조회
gh repo view

# 브라우저에서 저장소 열기
gh repo view --web

# 저장소 포크(Fork)
gh repo fork owner/repository

# 저장소 목록 조회
gh repo list
gh repo list owner --limit 20
```

---

### 3. 이슈(Issue) 관리

```bash
# 이슈 목록 조회
gh issue list

# 특정 상태 이슈 조회
gh issue list --state open
gh issue list --state closed

# 이슈 상세 조회
gh issue view 42

# 이슈 생성 (인터랙티브)
gh issue create

# 이슈 생성 (옵션 지정)
gh issue create \
  --title "버그: 로그인 실패" \
  --body "로그인 시 500 에러 발생" \
  --label "bug" \
  --assignee "@me"

# 이슈 닫기
gh issue close 42

# 이슈 다시 열기
gh issue reopen 42

# 이슈에 댓글 작성
gh issue comment 42 --body "확인 중입니다."

# 브라우저에서 이슈 열기
gh issue view 42 --web
```

#### GitHub Projects 이슈 코멘트

GitHub Projects의 이슈는 저장소에 속한 일반 이슈이다. 따라서 코멘트 작성은 `gh project`가 아닌 `gh issue comment`로 동일하게 수행한다.

```bash
# 이슈 번호로 코멘트 작성
gh issue comment 42 --body "다음 스프린트에 반영하겠습니다."

# 에디터로 작성 (--body 생략 시 에디터 열림)
gh issue comment 42

# 다른 저장소의 이슈에 코멘트 작성
gh issue comment 42 --repo owner/repository --body "코멘트 내용"
```

---

### 4. Pull Request 관리

PR(Pull Request)은 GitHub CLI의 핵심 활용 영역이다.

```bash
# PR 목록 조회
gh pr list

# 현재 브랜치의 PR 조회
gh pr view

# PR 생성 (인터랙티브)
gh pr create

# PR 생성 (옵션 지정)
gh pr create \
  --title "feat: 로그인 기능 추가" \
  --body "로그인 폼 및 JWT 인증 구현" \
  --base main \
  --head feature/login

# PR 체크아웃 (리뷰 목적으로 로컬에 받기)
gh pr checkout 123

# PR 머지
gh pr merge 123
gh pr merge 123 --squash      # 스쿼시 머지
gh pr merge 123 --rebase      # 리베이스 머지

# PR 리뷰 제출
gh pr review 123 --approve
gh pr review 123 --request-changes --body "수정 필요합니다"
gh pr review 123 --comment --body "좋은 코드입니다"

# PR 닫기
gh pr close 123

# PR 상태 확인 (CI 상태 포함)
gh pr checks 123
```

#### Fork 저장소 → 원본(upstream) 저장소로 PR 생성

오픈소스 기여 등 Fork한 저장소에서 원본 저장소로 PR을 보내는 것도 가능하다.
`--repo` 옵션으로 PR이 생성될 대상 저장소(upstream)를 명시한다.

```bash
# 1. 원본 저장소를 Fork & 클론
gh repo fork owner/original-repo --clone

# 2. 로컬에서 브랜치 생성 후 작업
git checkout -b fix/typo-in-readme

# ... 수정 작업 ...

git add .
git commit -m "fix: README 오타 수정"

# 3. 내 Fork 저장소로 푸시
git push origin fix/typo-in-readme

# 4. 원본 저장소(upstream)를 대상으로 PR 생성
#    --repo: PR이 열릴 대상 저장소 (원본)
#    --head: 내 Fork의 브랜치 (my-username:브랜치명 형식)
#    --base: 원본 저장소의 타겟 브랜치
gh pr create \
  --repo owner/original-repo \
  --head my-username:fix/typo-in-readme \
  --base main \
  --title "fix: README 오타 수정" \
  --body "오타를 수정했습니다."
```

**`--head` 옵션의 형식**

Fork PR에서 `--head`는 반드시 `내계정명:브랜치명` 형식으로 지정한다.
GitHub은 이 정보를 통해 어느 계정의 Fork에서 온 브랜치인지 구분한다.

```
내 계정: my-username
내 브랜치: fix/typo-in-readme

→ --head my-username:fix/typo-in-readme
```

**Fork PR 생성 흐름 요약**

```
[내 Fork 저장소]              [원본(upstream) 저장소]
my-username/original-repo  →  owner/original-repo

   fix/typo-in-readme    →PR→  main
```

---

### 5. 워크플로우(Actions) 관리

GitHub Actions는 GitHub CLI로 트리거하고 모니터링할 수 있다.

```bash
# 워크플로우 목록 조회
gh workflow list

# 워크플로우 실행
gh workflow run deploy.yml

# 특정 브랜치에서 워크플로우 실행
gh workflow run deploy.yml --ref feature/login

# 실행 이력 조회
gh run list

# 실행 상세 조회 및 로그 확인
gh run view 1234567890
gh run view 1234567890 --log

# 실행 중인 워크플로우 취소
gh run cancel 1234567890

# 실패한 워크플로우 재실행
gh run rerun 1234567890
```

---

### 6. 릴리즈(Release) 관리

```bash
# 릴리즈 목록 조회
gh release list

# 릴리즈 상세 조회
gh release view v1.0.0

# 릴리즈 생성
gh release create v1.0.0 \
  --title "v1.0.0 정식 출시" \
  --notes "최초 정식 버전 릴리즈"

# 파일 첨부와 함께 릴리즈 생성
gh release create v1.0.0 ./dist/*.zip \
  --title "v1.0.0" \
  --generate-notes    # 커밋 기반 자동 릴리즈 노트 생성

# 릴리즈 삭제
gh release delete v1.0.0
```

---

### 7. GitHub Gist 관리

```bash
# Gist 생성
gh gist create script.py

# 공개 Gist 생성
gh gist create script.py --public

# 설명과 함께 생성
gh gist create script.py --desc "유용한 파이썬 스크립트"

# Gist 목록 조회
gh gist list

# Gist 수정
gh gist edit gist_id
```

---

### 8. 별칭(Alias) 설정

자주 사용하는 명령어를 짧게 줄여 별칭으로 등록할 수 있다.

```bash
# 별칭 설정
gh alias set prl 'pr list'
gh alias set prc 'pr create'
gh alias set il 'issue list'

# 별칭 목록 조회
gh alias list

# 별칭 삭제
gh alias delete prl
```

별칭 활용 예시:

```bash
# 등록 전
gh pr list --state open --author "@me"

# 별칭 등록
gh alias set mypr 'pr list --state open --author "@me"'

# 등록 후
gh mypr
```

---

### 9. 필터링 및 출력 형식

```bash
# JSON 형식으로 출력
gh issue list --json number,title,state

# 특정 필드만 출력 (jq 조합)
gh issue list --json number,title | jq '.[].title'

# 라벨로 필터링
gh issue list --label "bug"

# 담당자로 필터링
gh issue list --assignee "@me"

# 검색어로 필터링
gh issue list --search "로그인"
gh pr list --search "feat"
```

---

## 코드 예제

### 실전 예제 1: 기능 개발 전체 흐름

```bash
# 1. 이슈 생성
gh issue create \
  --title "feat: 사용자 프로필 페이지 구현" \
  --body "사용자 정보 조회 및 수정 기능" \
  --label "enhancement"

# 2. 로컬에서 브랜치 생성 및 개발
git checkout -b feature/user-profile

# ... 개발 작업 ...

git add .
git commit -m "feat: 사용자 프로필 페이지 구현"
git push origin feature/user-profile

# 3. PR 생성
gh pr create \
  --title "feat: 사용자 프로필 페이지 구현" \
  --body "Closes #이슈번호" \
  --base main

# 4. CI 상태 확인
gh pr checks

# 5. 머지
gh pr merge --squash
```

### 실전 예제 2: 릴리즈 자동화 스크립트

```bash
#!/bin/bash
# release.sh - 릴리즈 자동화 스크립트

VERSION=$1  # 예: ./release.sh v1.2.0

if [ -z "$VERSION" ]; then
  echo "버전을 입력하세요. 예: ./release.sh v1.2.0"
  exit 1
fi

# 태그 생성
git tag $VERSION
git push origin $VERSION

# 릴리즈 생성 (자동 릴리즈 노트 포함)
gh release create $VERSION \
  --title "$VERSION" \
  --generate-notes

echo "릴리즈 완료: $VERSION"
```

### 실전 예제 3: 내가 담당한 이슈 & PR 한눈에 보기

```bash
# 내가 담당한 열린 이슈 목록
gh issue list --assignee "@me" --state open

# 내가 올린 리뷰 대기 중인 PR 목록
gh pr list --author "@me" --state open

# 내가 리뷰해야 할 PR 목록
gh pr list --reviewer "@me" --state open
```

---

## 자주 하는 실수

**실수 1: 인증 없이 명령어 실행**

```bash
# 오류 메시지
# To get started with GitHub CLI, please run:  gh auth login

# 해결
gh auth login
```

**실수 2: 저장소 디렉토리 밖에서 실행**

```bash
# 오류 메시지
# could not determine current repository

# 해결: 저장소 디렉토리 안으로 이동
cd my-project
gh pr list

# 또는 명시적으로 저장소 지정
gh pr list --repo owner/repository
```

**실수 3: PR 생성 시 base 브랜치 미지정**

```bash
# 기본 브랜치가 main이 아닌 경우 명시적으로 지정한다
gh pr create --base develop --head feature/login
```

**실수 4: JSON 출력 시 필드명 오타**

```bash
# 사용 가능한 JSON 필드 확인
gh issue list --json  # 에러 메시지에서 사용 가능한 필드 목록 출력
```

---

## 핵심 정리

1. `gh auth login`으로 인증 후 모든 명령어를 사용할 수 있다.
2. `gh repo`, `gh issue`, `gh pr`, `gh workflow`, `gh release`가 5대 핵심 명령어 그룹이다.
3. `--web` 옵션을 붙이면 해당 페이지를 브라우저에서 바로 열 수 있다.
4. `--json` 옵션과 `jq`를 조합하면 출력 결과를 자유롭게 가공할 수 있다.
5. `gh alias set`으로 자주 쓰는 명령어를 단축어로 등록하면 생산성이 높아진다.

---

## 명령어 빠른 참조표

| 카테고리 | 명령어 | 설명 |
|----------|--------|------|
| 인증 | `gh auth login` | GitHub 인증 |
| 저장소 | `gh repo create` | 저장소 생성 |
| 저장소 | `gh repo clone` | 저장소 클론 |
| 저장소 | `gh repo view --web` | 브라우저로 열기 |
| 이슈 | `gh issue list` | 이슈 목록 |
| 이슈 | `gh issue create` | 이슈 생성 |
| 이슈 | `gh issue view [번호]` | 이슈 상세 |
| PR | `gh pr list` | PR 목록 |
| PR | `gh pr create` | PR 생성 |
| PR | `gh pr merge` | PR 머지 |
| PR | `gh pr checkout [번호]` | PR 체크아웃 |
| PR | `gh pr review` | PR 리뷰 |
| Actions | `gh workflow run` | 워크플로우 실행 |
| Actions | `gh run list` | 실행 이력 조회 |
| 릴리즈 | `gh release create` | 릴리즈 생성 |
| 설정 | `gh alias set` | 별칭 등록 |

---

## 관련 개념

- **Git**: GitHub CLI는 Git을 대체하지 않는다. Git은 버전 관리를, GitHub CLI는 GitHub 플랫폼 기능을 담당한다.
- **GitHub Actions**: `gh workflow`, `gh run` 명령어로 Actions 파이프라인을 CLI에서 제어할 수 있다.
- **GitHub API**: `gh api` 명령어로 GitHub REST/GraphQL API를 직접 호출할 수 있다.
- **GitHub Codespaces**: `gh codespace` 명령어로 클라우드 개발환경을 관리할 수 있다.

---

## 참고

- 공식 문서: https://cli.github.com/manual/
- 명령어 도움말: `gh <command> --help` (예: `gh pr --help`)
