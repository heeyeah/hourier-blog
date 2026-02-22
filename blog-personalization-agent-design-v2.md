# 에이전트 설계서: 블로그 개성화 — 실제 코드 수정까지 실행

> Claude Code 구현 참조 계획서
> 목표: tailwind-nextjs-starter-blog를 진단부터 실제 파일 수정까지 자동 실행
> Allowed 범위 준수 / PR 단위 순차 적용 / 빌드 검증 포함
> **v2: 인터뷰 기반 설계 결정사항 반영 (2026-02-21)**

---

## 0. 인터뷰 기반 설계 결정사항 요약

| 항목 | 결정 | 비고 |
|------|------|------|
| PAUSE 메커니즘 | 파일 플래그 기반 비동기 | AWAITING_APPROVAL.md 생성 후 claude 종료 → 사용자 확인 → `APPROVED: true` 설정 후 claude 재실행 |
| 에이전트 재호출 | 사용자가 수동으로 `claude` 재실행 | CLAUDE.md가 시작 시 AWAITING_APPROVAL.md 체크 → 중단 지점에서 재개 |
| 시각적 맹점 | 소스 코드 분석만으로 충분 | 스크린샷 도구 불필요. diff 확인은 사람이 담당 |
| PR 선택 | brand-proposal 승인 시 pr-plan.md 체크박스 동반 제공 | 운영자가 체크박스 편집 후 승인. 체크된 PR만 실행 |
| PAUSE 횟수 | 2회 (brand-proposal 승인 + 각 PR diff 확인) | PR diff PAUSE는 각 PR 완료 후 1회씩 |
| 빌드 재시도 | 에러 로그 누적 후 재시도 (최대 3회) | 시도마다 이전 에러 누적 → 패턴 인식. 1차: 에러 라인 수정, 2차: 해당 PR 전체 롤백 후 재적용, 3차: 에스컬레이션 |
| 스크립트 방식 | 실제 Python/bash 파일 생성 | scan_repo.py: 표준 라이브러리 기반. requirements.txt 허용 |
| Git 전략 | draft 브랜치에 직접 커밋 (PR마다 커밋 1개) | 롤백: `git revert`로 해당 커밋만 복원 (이전 PR 유지) |
| 이미지 처리 | 운영자 수동 제공 → 에이전트가 경로만 업데이트 | PR-03: 운영자가 `public/static/`에 파일 배치 후 AWAITING_APPROVAL로 안내 |
| Pretendard 연동 | CDN 임포트 (`app/layout.tsx`) | jsDelivr CDN. 영문 보조 폰트(JetBrains Mono 등)는 next/font/google |
| 브랜드 힌트 형식 | `brand-hint.md` (자연어 텍스트) | 소셜/URL 정보 포함. 루트에 위치 |
| siteMetadata 실제값 | brand-hint.md에서 참조 | PR-05에서 에이전트가 brand-hint.md 읽어 채움 |
| Locale | `ko-KR` | language: 'ko', locale: 'ko-KR' |
| 다크모드 기본값 | `system` 유지 | 운영체제 설정 따름 |
| 네비게이션 맵 | 현행 유지 | Home/Blog/Tags/Projects/About 5개 |

---

## 1. 작업 컨텍스트

### 목적

에이전트가 블로그 레포를 직접 읽고, 브랜드 방향을 설계한 뒤, 실제 파일을 수정한다. 브랜드 제안 승인 후 선택된 PR 단위로 파일 수정·빌드 확인을 진행하며, 각 PR 완료 시 사람이 diff를 확인·승인한 후 커밋된다.

### 입력

| 파일 | 위치 | 설명 |
|------|------|------|
| 블로그 레포 | 로컬 경로 | 수정 대상 |
| `brand-hint.md` | 레포 루트 | 운영자 브랜드 방향 (자연어). 색상 참고, 폰트 취향, 소셜 URL 포함 |

### 출력

| 산출물 | 설명 |
|--------|------|
| `/output/style-audit.md` | 현재 스타일 진단 |
| `/output/brand-proposal.md` | 브랜드 방향 제안 (사람 승인 후 구현) |
| `/output/pr-plan.md` | PR 단위 작업 계획 + 체크박스 (운영자가 편집) |
| `/output/AWAITING_APPROVAL.md` | PAUSE 시 생성. 사람이 확인 후 `APPROVED: true` 설정 |
| **실제 파일 수정** | 레포 내 Allowed 파일 직접 수정 |
| `/output/compliance-report.md` | 변경 파일 목록 + Allowed 여부 + 이유 |

### Allowed / Forbidden

**Allowed**
```
data/siteMetadata.js, data/authors/default.mdx, data/headerNavLinks.ts
public/static/  (로고, 파비콘, 프로필 이미지)
css/tailwind.css, tailwind.config.js
components/MDXComponents.js
layouts/*
app/*
next.config.js  (CSP 수정 최소 범위만)
```

**Forbidden — 수정 시 예외 문서 작성 후 승인 요청**
```
contentlayer.config.ts
package.json (단, Pretendard CDN은 app/layout.tsx로 처리하므로 해당 없음)
라우팅·앱 구조 대개편
```

### DoD (PR 공통)

- [ ] `npm run build` 통과
- [ ] 모바일/데스크탑 레이아웃 깨짐 없음
- [ ] 다크/라이트 모드 양쪽 정상 렌더링
- [ ] heading 구조, 키보드 탭, 이미지 alt 접근성 기본 준수
- [ ] Compliance Report 업데이트
- [ ] draft 브랜치에 커밋 완료

---

## 2. 워크플로우 정의

### 전체 흐름

```
[START] brand-hint.md 존재 확인
  │     없으면: 에스컬레이션 (brand-hint.md 작성 요청)
  │
  ▼
[0단계] AWAITING_APPROVAL.md 체크 (재호출 시)
  │     파일 존재 + APPROVED: true → 중단된 단계부터 재개
  │     없거나 APPROVED: false → 1단계부터 시작
  │
  ▼
[1단계] 레포 스캔 (scan_repo.py)
  │  Allowed 파일 내용 수집 → repo-snapshot.json
  │  실패 시: 에스컬레이션 (경로 재확인 요청)
  ▼
[2단계] 스타일 진단 + 브랜드 제안 + PR 계획 (Designer 서브에이전트)
  │  brand-hint.md + repo-snapshot.json 참조
  │  → style-audit.md + brand-proposal.md + pr-plan.md (체크박스 포함)
  │  → AWAITING_APPROVAL.md 생성 (APPROVED: false)
  │  → [PAUSE] 에이전트 종료
  │
  │  ── 사용자: brand-proposal.md 확인 + pr-plan.md 체크박스 편집
  │             AWAITING_APPROVAL.md에 APPROVED: true 설정
  │             `claude` 재실행
  │
  │  수정 요청 → 2단계 재실행 (AWAITING_APPROVAL.md에 피드백 작성)
  ▼
[3단계] PR별 순차 구현 루프 (Implementer 서브에이전트)
  │
  │  pr-plan.md의 체크된 PR만 순서대로:
  │    ├─ compliance_check.py — 대상 파일 Allowed 여부 확인
  │    │     위반 → 예외 문서 생성 → [PAUSE: 승인] → 미승인 시 해당 PR 스킵
  │    ├─ 파일 수정 실행 (Implementer 직접 편집)
  │    ├─ run_build.sh (npm run build) 실행
  │    │     실패 → 에러 로그 누적 → 재시도 전략:
  │    │       1회: 에러 라인만 타겟 수정
  │    │       2회: 해당 PR 변경 전체 롤백 후 최소 변경으로 재적용
  │    │       3회: 에스컬레이션 (누적 에러 로그 + 시도 내역 출력)
  │    ├─ git commit (draft 브랜치, PR당 커밋 1개)
  │    ├─ git diff HEAD~1 출력
  │    ├─ AWAITING_APPROVAL.md 생성 (PR 제목 + 파일 목록 + diff 요약 + 다음 단계 안내)
  │    │  → [PAUSE] 에이전트 종료
  │    │
  │    │  ── 사용자: diff 확인 후 APPROVED: true 또는 ROLLBACK: true 설정
  │    │             `claude` 재실행
  │    │
  │    │     승인 → 다음 PR 진행
  │    │     롤백 → git revert (해당 커밋만, 이전 PR 유지) → 재설계
  │    │
  │    └─ Compliance Report 업데이트
  │
  ▼
[END] compliance-report.md 최종 저장 + AWAITING_APPROVAL.md 삭제
```

### LLM 판단 vs 스크립트 처리

| 에이전트 판단 | 스크립트 처리 |
|--------------|-------------|
| 현재 스타일의 문제점 식별 (Designer) | 파일 트리 스캔, 파일 내용 읽기 (scan_repo.py) |
| 브랜드 방향 제안 — 색상, 폰트, 레이아웃 (Designer) | Allowed vs 제안 파일 대조 (compliance_check.py) |
| 구체적 코드 수정 내용 결정 (Implementer) | `npm run build` 실행 및 에러 로그 파싱 (run_build.sh) |
| 빌드 에러 원인 분석 및 수정 전략 결정 (Implementer) | git commit, git diff, git revert (git_ops.sh) |
| Compliance 위반 여부 최종 판단 | — |
| AWAITING_APPROVAL.md APPROVED 여부 판단 | — |

### 단계별 상세

#### 0단계: 재호출 감지 (매 실행 시 첫 번째 체크)

CLAUDE.md 오케스트레이터가 시작 시 `/output/AWAITING_APPROVAL.md` 존재를 확인한다.

```
APPROVED: false → 에이전트 종료 (사용자가 아직 미승인)
APPROVED: true  → CURRENT_STAGE 필드 읽어 해당 단계부터 재개
파일 없음       → 1단계부터 시작
```

#### 1단계: 레포 스캔

- **처리**: `scan_repo.py` (Python 3, 표준 라이브러리)
- **수집**: Allowed 파일 전체 내용 + 파일 트리 + `brand-hint.md` 내용
- **출력**: `/output/repo-snapshot.json`
- **성공 기준**: 필수 파일(tailwind.config.js, css/tailwind.css, layouts/, app/) 존재 확인
- **실패 시**: 에스컬레이션 (경로 재확인 요청)

#### 2단계: 스타일 진단 + 브랜드 제안

- **처리**: Designer 서브에이전트 (LLM) — UX/UI 디자이너 역할
- **입력**: `repo-snapshot.json` 경로 + `brand-hint.md` 내용
- **수행 내용**:
  1. **진단**: primary 색상 토큰, 폰트 패밀리/스케일, 레이아웃 요소, 다크모드 변수 분석
  2. **제안**: 컬러 팔레트(hex), 타이포그래피(CDN/Google Fonts 기준), 레이아웃 개성화 포인트, 항목별 수정 대상 파일 + Allowed 여부
  3. **PR 계획**: pr-plan.md 생성 (체크박스 형식, 운영자 편집 가능)
- **출력**: `style-audit.md` + `brand-proposal.md` + `pr-plan.md`
- **[PAUSE]**: AWAITING_APPROVAL.md 생성 후 에이전트 종료
- **성공 기준**: 색상·폰트·레이아웃 3영역 진단 완료, 모든 제안 파일 Allowed 범위 내
- **실패 시**: Allowed 위반 항목은 예외 문서 초안 자동 생성

#### 3단계: PR별 순차 구현

- **처리**: Implementer 서브에이전트 (LLM) + 스크립트 — 엔지니어 역할
- **PR 구성** (brand-proposal + pr-plan.md 체크박스 기반):

  | PR | 내용 | 핵심 파일 | 비고 |
  |----|------|-----------|------|
  | PR-01 | 컬러 토큰 교체 | `tailwind.config.js`, `css/tailwind.css` | 그라데이션 accent 포함 |
  | PR-02 | 폰트 교체 | `tailwind.config.js`, `app/layout.tsx` | Pretendard CDN + 영문 Google Font |
  | PR-03 | 로고·파비콘·프로필 이미지 경로 업데이트 | `data/siteMetadata.js` | **운영자가 먼저 이미지 파일 배치 필요** — AWAITING_APPROVAL로 안내 |
  | PR-04 | 레이아웃 개성화 | `layouts/*`, `components/MDXComponents.js` | 장문 에세이·코드 스니펫 최적화 |
  | PR-05 | siteMetadata 브랜딩 | `data/siteMetadata.js` | brand-hint.md의 소셜/URL 값 반영, locale: 'ko-KR' |

- **각 PR 실행 절차**:
  1. `compliance_check.py` — 대상 파일 Allowed 여부 확인
  2. Implementer가 파일 직접 편집
  3. `run_build.sh` (`npm run build`) 실행
  4. 빌드 실패 시: 에러 로그 누적 후 재시도 (최대 3회, 전략 단계화)
  5. 3회 초과 시: 에스컬레이션 (누적 에러 로그 출력 후 사람 개입 요청, 에이전트 종료)
  6. 빌드 성공 시: `git commit -m "feat: PR-0N 설명"` (draft 브랜치)
  7. `git diff HEAD~1` 출력 → AWAITING_APPROVAL.md 생성 → [PAUSE] 에이전트 종료
  8. 재호출 시: APPROVED: true → 다음 PR / ROLLBACK: true → `git revert HEAD` (해당 커밋만)

- **성공 기준**: 빌드 통과 + 사람 승인 + 커밋 완료
- **실패 시**: 3회 재시도 후 에스컬레이션, AWAITING_APPROVAL.md에 에러 상세 포함

---

## 3. 구현 스펙

### 폴더 구조

```
/blog-project-root
  ├── CLAUDE.md                            # 오케스트레이터 지침
  ├── brand-hint.md                        # 운영자 브랜드 힌트 (자연어)
  ├── /.claude
  │   ├── /skills
  │   │   ├── /repo-scanner
  │   │   │   ├── SKILL.md
  │   │   │   └── /scripts
  │   │   │       ├── scan_repo.py          # 파일 트리 + Allowed 파일 내용 수집 (표준 라이브러리)
  │   │   │       └── compliance_check.py   # 제안 파일 vs Allowed 대조
  │   │   └── /build-runner
  │   │       ├── SKILL.md
  │   │       └── /scripts
  │   │           ├── run_build.sh          # npm run build 실행, 에러 로그 반환
  │   │           └── git_ops.sh            # git commit, git diff, git revert
  │   └── /agents
  │       ├── /designer
  │       │   └── AGENT.md                  # 스타일 진단 + 브랜드 설계 전담 (UX/UI 디자이너)
  │       └── /implementer
  │           └── AGENT.md                  # PR별 파일 수정 + 빌드 검증 전담 (엔지니어)
  ├── /output
  │   ├── repo-snapshot.json
  │   ├── style-audit.md
  │   ├── brand-proposal.md
  │   ├── pr-plan.md                        # 체크박스 형식 — 운영자가 편집
  │   ├── AWAITING_APPROVAL.md              # PAUSE 시 생성, 승인 후 삭제
  │   └── compliance-report.md
  └── /docs
      └── allowed-files.md
```

### CLAUDE.md 핵심 섹션

1. 역할 (오케스트레이터 — Designer·Implementer 조율)
2. Allowed / Forbidden 파일 목록
3. 시작 시 AWAITING_APPROVAL.md 체크 로직 (0단계)
4. 워크플로우 3단계 및 서브에이전트 호출 시점
5. PAUSE 조건 및 AWAITING_APPROVAL.md 생성 형식
6. 빌드 실패 재시도 정책 (에러 로그 누적, 단계화 전략, 최대 3회 → 에스컬레이션)
7. Git 전략 (draft 브랜치 직접 커밋, PR당 커밋 1개, 롤백: git revert 해당 커밋만)
8. DoD 공통 기준
9. 데이터 전달 규칙 (파일 경로만 전달)

### 에이전트 구조

```
CLAUDE.md (오케스트레이터)
  │
  ├── Designer           ← 2단계: 스타일 진단 + 브랜드 제안 (UX/UI 디자이너)
  │   참조 스킬: repo-scanner
  │   입력: repo-snapshot.json + brand-hint.md 내용
  │
  └── Implementer        ← 3단계: 파일 수정 + 빌드 검증 (엔지니어)
      참조 스킬: build-runner, repo-scanner (compliance_check)
      입력: brand-proposal.md + pr-plan.md + PR 번호
```

**서브에이전트 2개로 분리한 이유**: 디자이너는 "무엇을 어떻게 바꿀지" 판단하고, 엔지니어는 "실제로 어떻게 코드로 구현할지"를 담당 — 역할과 필요 컨텍스트가 명확히 다름. 또한 PAUSE(사람 승인) 사이에 자연스럽게 에이전트가 교체되어 컨텍스트 리셋 효과도 있음.

### 서브에이전트 상세

| 에이전트 | 역할 | 트리거 | 입력 | 출력 |
|----------|------|--------|------|------|
| `designer` | 현재 스타일 진단 → 브랜드 방향 설계까지 일관되게 수행 | 1단계 완료 후 | `repo-snapshot.json` 경로 + `brand-hint.md` | `style-audit.md` + `brand-proposal.md` + `pr-plan.md` |
| `implementer` | PR별 파일 수정, 빌드 실행, git commit, diff 출력 | 브랜드 제안 승인 후, 체크된 PR마다 | `brand-proposal.md` + `pr-plan.md` + PR 번호 | 실제 파일 수정 + 커밋 + `compliance-report.md` 업데이트 |

### 스킬 상세

| 스킬 | 스크립트 | 역할 | 트리거 조건 |
|------|----------|------|------------|
| `repo-scanner` | `scan_repo.py` | 파일 트리 + Allowed 파일 내용 수집 → JSON | 1단계 시작 시 |
| `repo-scanner` | `compliance_check.py` | 제안 파일 목록 vs Allowed 대조, 위반 목록 반환 | 3단계 각 PR 시작 전 |
| `build-runner` | `run_build.sh` | `npm run build` 실행, 성공/실패 + 에러 로그 반환 | 3단계 각 PR 파일 수정 후 |
| `build-runner` | `git_ops.sh` | `git commit`, `git diff HEAD~1`, `git revert HEAD` | PR 커밋 시, diff 출력 시, 롤백 요청 시 |

### AWAITING_APPROVAL.md 형식

```markdown
# 승인 대기 — [단계명] (예: 브랜드 제안 검토 / PR-02 diff 확인)

APPROVED: false
ROLLBACK: false
CURRENT_STAGE: brand_proposal  # or pr_01, pr_02, pr_03, pr_04, pr_05

## 무엇을 확인해야 하나요?

[사람이 읽어야 할 내용 요약]

## 변경/제안 내용

[brand-proposal 요약 또는 PR 제목 + 변경 파일 목록]

## Git Diff 요약

```diff
[git diff HEAD~1 출력 (PR 단계 시)]
```

## DoD 체크리스트

- [x] npm run build 통과
- [ ] 시각적 확인 (모바일/데스크탑)
- [ ] 다크/라이트 모드 확인

## 다음 단계 안내

**승인**: 이 파일에서 `APPROVED: false` → `APPROVED: true` 로 변경 후 `claude`를 다시 실행하세요.
**롤백**: `ROLLBACK: true` 로 변경 후 `claude`를 다시 실행하면 이 PR의 커밋만 되돌립니다.
**수정 요청**: 이 파일 아래에 피드백을 작성하고 `claude`를 다시 실행하세요.
```

### pr-plan.md 형식 (운영자 체크박스 편집용)

```markdown
# PR 실행 계획

brand-proposal.md를 검토한 후 아래 체크박스를 편집하세요.
체크된(x) PR만 실행됩니다.

- [x] PR-01: 컬러 토큰 교체 (tailwind.config.js, css/tailwind.css)
- [x] PR-02: 폰트 교체 — Pretendard CDN + JetBrains Mono (app/layout.tsx, tailwind.config.js)
- [x] PR-03: 이미지 경로 업데이트 (data/siteMetadata.js) ⚠️ 먼저 public/static/에 파일 직접 배치 필요
- [x] PR-04: 레이아웃 개성화 (layouts/*, components/MDXComponents.js)
- [x] PR-05: siteMetadata 브랜딩 (data/siteMetadata.js) — locale ko-KR, 소셜 URL
```

### 폰트 구현 상세 (PR-02)

| 폰트 | 역할 | 연동 방식 |
|------|------|-----------|
| Pretendard | 본문 (한/영 혼합) | `app/layout.tsx`에 jsDelivr CDN link 태그 삽입 |
| 영문 보조 (ex. Inter) | 영문 전용 구간 | `next/font/google` |
| 코드 블록 (ex. JetBrains Mono) | 코드·스니펫 | `next/font/google` 또는 CDN |

Pretendard CDN 예시:
```html
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/variable/pretendardvariable-dynamic-subset.min.css" />
```

### 빌드 재시도 전략 (단계화)

| 시도 | 전략 | 근거 |
|------|------|------|
| 1회 | 에러 로그의 에러 라인만 타겟 수정 | 오타, 단순 문법 오류 대응 |
| 2회 | 해당 PR 변경 전체 `git revert HEAD` 후 최소 변경만 재적용 | 복합 오류, 의존성 충돌 대응 |
| 3회 | 에스컬레이션 — 누적 에러 로그 + 3회 시도 내역을 AWAITING_APPROVAL.md에 포함 후 종료 | 사람 판단 필요 |

### 데이터 전달 방식

```
오케스트레이터 → 서브에이전트: 파일 경로 + 간단한 지시 (인라인 최소화)
서브에이전트 → 오케스트레이터: "완료, {파일경로}" 또는 에러 내용
중간 산출물: 모두 /output/ 에 파일로 저장
```

### 주요 산출물 형식

| 파일 | 형식 | 핵심 구조 |
|------|------|-----------|
| `repo-snapshot.json` | JSON | `{ "allowed_files": [], "style_files": { "tailwind_config": "...", "tailwind_css": "...", "layouts": {} }, "brand_hint": "..." }` |
| `style-audit.md` | Markdown | 색상 / 폰트 / 레이아웃 3섹션, 현황 + 변경 포인트 |
| `brand-proposal.md` | Markdown | 컬러 팔레트(hex) / 타이포 / 레이아웃 개선안, 항목별 대상 파일 + Allowed 여부 |
| `pr-plan.md` | Markdown | PR별 체크박스 + 파일 목록 (운영자 편집용) |
| `AWAITING_APPROVAL.md` | Markdown | APPROVED/ROLLBACK 플래그 + 확인 내용 + 다음 단계 안내 |
| `compliance-report.md` | Markdown | PR별 변경 파일, Allowed 여부, 변경 이유 (PR 완료마다 누적 업데이트) |

---

## 4. siteMetadata 수정 계획 (PR-05)

현재 `data/siteMetadata.js` 상태 기준, PR-05에서 변경할 항목:

| 필드 | 현재값 | 변경 방향 |
|------|--------|-----------|
| `language` | `'en-us'` | `'ko'` |
| `locale` | `'en-US'` | `'ko-KR'` |
| `description` | 템플릿 기본값 | brand-hint.md의 블로그 소개 반영 |
| `siteUrl` | 스타터 블로그 URL | brand-hint.md의 실제 URL |
| `siteRepo` | timlrx 레포 | 실제 레포 URL |
| `email` | 플레이스홀더 | brand-hint.md 값 |
| `github` | 기본값 | brand-hint.md 값 |
| `linkedin` | 기본값 | brand-hint.md 값 (불필요 시 제거) |
| `youtube`, `instagram` | 기본값 | brand-hint.md 기준 필요 없는 항목 주석 처리 |
| `theme` | `'system'` | 유지 |
| `stickyNav` | `false` | 유지 |

네비게이션 맵 (`data/headerNavLinks.ts`): **현행 유지** (Home/Blog/Tags/Projects/About)

---

## 5. 브랜드 힌트 (운영자 제공 내용)

> 이 섹션은 `brand-hint.md`에 작성될 실제 내용의 요약이다.
> Designer 에이전트는 `brand-hint.md`를 읽어 컬러·폰트·레이아웃 제안에 반영한다.

### 컬러 참고

- **참고 사이트**: https://heeyeah.github.io/
- **참고 포인트**: 헤더의 그라데이션 색상 팔레트
  - 기존 블로그 헤더: `linear-gradient(72deg, #fcc304, #c90035)` (황금→크림슨)
  - 이 그라데이션 기조를 새 블로그의 accent 컬러로 이어받을 것
  - 다크모드: 그라데이션이 과하지 않도록 채도 조정 고려
  - 라이트모드: 배경은 오프화이트 또는 화이트, 본문 텍스트는 짙은 차콜

### 폰트

- **본문**: Pretendard (CDN, Variable weight) — 한/영 혼합 콘텐츠에 적합
- **영문 보조**: Inter 또는 유사한 sans-serif Google Font
- **코드 블록**: JetBrains Mono (Google Fonts) — 코드 스니펫 중심 콘텐츠

### 콘텐츠 성격

- 수필/에세이 혼합 (장문 기술 에세이 + 단상)
- 코드 블록·스니펫 자주 등장
- 수식(KaTeX) 필요 시 있음
- 한국어 콘텐츠 중심, 영문 혼용

### 레이아웃 방향

- 읽기에 집중된 넓은 본문 여백
- 코드 블록: 가독성 높은 syntax highlight (다크테마 기반)
- 카드 리스트: 너무 빽빽하지 않게 — 넓은 간격
- 네비/푸터: 심플하고 과하지 않게

### 소셜/URL (brand-hint.md에 기록, PR-05에서 반영)

- 참고 기존 블로그: https://heeyeah.github.io/
- 기타 소셜 링크는 brand-hint.md에 운영자가 직접 작성

---

## 6. 미결 사항 및 운영자 액션 아이템

| 항목 | 담당 | 시점 |
|------|------|------|
| `brand-hint.md` 실제 작성 (GitHub URL, LinkedIn, 블로그 설명 등) | 운영자 | 에이전트 실행 전 |
| PR-03 이미지 파일 준비 (로고, favicon, profile 이미지) | 운영자 | PR-03 PAUSE 시 |
| Giscus 댓글 설정 (NEXT_PUBLIC_GISCUS_* 환경변수) | 운영자 | 별도 (에이전트 범위 외) |
| Umami Analytics ID | 운영자 | 별도 (에이전트 범위 외) |
| draft 브랜치 → main 머지 | 운영자 | 전체 완료 후 |
