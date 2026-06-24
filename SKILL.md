---
name: repo-validate
description: 현재 repo를 먼저 **타입 분류**(spec-data/library/service-app/cli/monorepo/docs)한 뒤, 그 타입에 맞는 검증 축으로 증거 기반 타당성 검증을 수행하고 Markdown 리포트를 생성한다. Use when 사용자가 "검증", "타당성 검증", "repo 검증", "validity check", "validate repo"를 요청할 때.
argument-hint: "[검증 대상 경로(기본 .)]"
allowed-tools: Read Glob Grep Bash
---

# Repo Validity Check (Type-Aware)

현재 repo가 (1) 기획의도대로 구현됐는지, (2) AI가 잘 읽을 수 있게 구조화됐는지, (3) goal 검증이 완료됐는지를 **증거 기반(evidence-based)** 으로 검증한다. 단, 검증 축은 고정이 아니라 **repo 타입에 따라 활성화되는 축이 달라진다** — 스키마 repo에 운영 준비성을 묻거나, 라이브러리에 데이터 drift를 묻는 식의 잘못된 기준 적용을 막는다. 분석은 read-only이며, 테스트/validator를 실제로 실행한다. repo 소스는 수정하지 않고 리포트 파일만 새로 만든다.

검증 대상 경로: `$ARGUMENTS` (없으면 현재 디렉토리 `.`).

타입 정의·축 활성화 매트릭스는 [references/repo-types.md](references/repo-types.md), 채점·게이팅은 [references/rubric.md](references/rubric.md), 리포트 형식은 [assets/report-template.md](assets/report-template.md)를 따른다.

## 원칙

- **증거 우선**: 모든 판정은 파일 경로(`path:line`), grep 결과, 또는 실제 명령 실행 로그로 뒷받침한다. 추측으로 PASS 판정하지 않는다.
- **타입에 맞는 축만 적용**: 비활성(N/A) 축은 채점·게이팅에서 제외한다. "해당 없음"을 미흡으로 처리하지 않는다.
- **테스트는 읽지 말고 실행한다**: 테스트/validator가 있으면 직접 돌려 stdout과 exit code를 증거로 캡처한다.
- **탐색 위임**: 3파일 이상을 훑어야 하는 탐색은 `Explore` 서브에이전트(haiku)에 위임한다.

## 절차

### Phase 0 — 입력 탐색 (기준선 확보)

1. **기획의도 문서**(intent baseline) 후보를 탐색: `docs/pr-faq*`, `docs/**`, `**/PRD*`, `**/prd*`, `**/spec*`, `AGENTS.md`, `README.md`, `CLAUDE.md`. 범위·목표·성공지표·제약을 담은 문서를 기준선으로 채택한다. (없으면 rubric.md의 fallback 모드.)
2. **검증 명령** 탐색: `package.json`의 scripts(`test`/`validate`/`lint`/`build`), `Makefile`, `.github/workflows/*`. 실행 가능한 검증 명령 목록을 만든다.
3. 채택한 기준 문서 경로와 명령 목록을 리포트 헤더에 기록한다.

### Phase 1 — Repo 타입 분류 + 활성 축 결정 (게이팅 전제)

1. [references/repo-types.md](references/repo-types.md)의 분류 신호로 **주 타입 / 부 타입**을 판정한다. 분류 근거(매칭 파일·패턴 경로)를 인용한다.
2. 타입을 축 활성화 매트릭스에 대입해 **이번 검증에서 활성화할 축 집합**과 각 축의 게이팅 등급(필수/조건부)을 확정한다. 조건부(⚠️) 축은 해당 신호(매니페스트·스키마 등)가 실제로 존재할 때만 활성화한다.
3. 활성/비활성 축 목록을 리포트 "검증 범위" 절에 기록한다. **이후 Phase는 활성 축에 대해서만 수행한다.**
   > Do not proceed: 타입과 활성 축을 확정하지 않은 채 채점하지 말 것.

### Phase 2 — 코어 축 검증 (A1 / A2 / A3, 모든 타입)

- **A1 기획의도 정합성** — 기준 문서에서 선언된 기능/범위 항목을 추출해 구현 증거에 매핑하고 `구현됨/부분구현/미구현`을 판정한다. "제외(out-of-scope)" 항목과 `AGENTS.md` Do-Not 규칙이 지켜지는지 grep으로 역검증한다(위반은 Critical, A1 FAIL).
- **A2 AI-readable 구조 + 문서 완전성** — `AGENTS.md`/`CLAUDE.md`/`README.md` 존재·품질, 디렉토리 일관성·예측 가능한 네이밍, 기계 판독 데이터, **문서-코드 정합성**(문서의 명령·경로가 실제 존재하는지; 불일치는 Major).
- **A3 goal 검증 + 테스트 충분성** — Phase 0의 테스트/validator를 **실제 실행**해 stdout·exit code를 캡처한다. pass/fail뿐 아니라 **충분성**(테스트가 핵심 기능·성공지표를 실제로 커버하는지, 빈/형식적 테스트는 아닌지)을 평가한다. 성공지표/DoD를 충족/미충족으로 매핑한다.

### Phase 3 — 조건부 모듈 검증 (Phase 1에서 활성화된 것만)

활성화된 모듈만 [references/repo-types.md](references/repo-types.md)·[rubric.md](references/rubric.md)의 항목으로 검증한다:

- **M-DATA** 데이터/스키마 무결성 — 스키마↔스냅샷 drift, 참조 무결성, 검증 스크립트 통과, 중복/고아 정의.
- **M-SEC** 보안/시크릿 위생 — 하드코딩 시크릿/키 grep, `.env` 커밋 여부.
- **M-DEP** 의존성 건전성 — lockfile 동기화, 선언↔실제 import 정합, (가능하면 `npm audit` 등 실행) 취약/미사용 의존성.
- **M-CHANGE** 변경 안전성/git 위생 — 미커밋 변경, 생성물↔소스 drift, `.gitignore` 누락.
- **M-API / M-OPS / M-PERF** — 타입상 활성일 때만(라이브러리·서비스 등).

### Phase 4 — 종합 판정 + 리포트

1. **활성 축에 대해서만** 점수(0–100)와 발견사항 severity(Critical/Major/Minor)를 산정한다(rubric.md 가중치). 비활성 축은 `N/A`로 표기한다.
2. **게이팅 규칙**: 활성 축 중 Critical(제약 위반·테스트 실패·필수 모듈 치명 결함) ≥1 → 전체 `FAIL`. Critical 0 + Major ≥2 → `CONCERNS`. 그 외 → `PASS`.
3. [assets/report-template.md](assets/report-template.md)를 채워 `artifacts/reports/repo-validation-<YYYY-MM-DD>.md`로 저장한다(폴더 없으면 repo 루트 또는 대화창 출력). 날짜는 `date +%F`.
4. 리포트 경로 + 판정한 타입 + 종합 판정 한 줄을 사용자에게 보고한다.
