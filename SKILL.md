---
name: repo-validate
description: 현재 repo를 먼저 **타입 분류**(spec-data/library/service-app/cli/monorepo/docs)한 뒤, 그 타입에 맞는 검증 축으로 증거 기반 타당성 검증을 수행하고 Markdown 리포트를 생성한다. `--interview` 시 위험 명령 실행 여부와 A1 의도 모호점을 1회 배치 질의로 확정한다. Use when 사용자가 "검증", "타당성 검증", "repo 검증", "validity check", "validate repo"를 요청할 때.
argument-hint: "[검증 대상 경로(기본 .)] [--interview]"
allowed-tools: Read Glob Grep Bash Write AskUserQuestion
---

# Repo Validity Check (Type-Aware)

현재 repo가 (1) 기획의도대로 구현됐는지, (2) AI가 잘 읽을 수 있게 구조화됐는지, (3) goal 검증이 완료됐는지를 **증거 기반(evidence-based)** 으로 검증한다. 단, 검증 축은 고정이 아니라 **repo 타입에 따라 활성화되는 축이 달라진다** — 스키마 repo에 운영 준비성을 묻거나, 라이브러리에 데이터 drift를 묻는 식의 잘못된 기준 적용을 막는다. 분석은 read-only이며, 테스트/validator를 실제로 실행한다. repo 소스는 수정하지 않고 **리포트는 대상 repo 바깥 또는 화면 출력**으로 만든다(P3 — 자기 산출물이 대상 repo의 git 위생 검사를 오염시키지 않도록).

검증 대상 경로: `$ARGUMENTS`에서 `--interview` 플래그를 뺀 첫 인자 (없으면 현재 디렉토리 `.`). `--interview`가 있으면 **위험 명령 실행 여부**(Phase 0)와 **A1 의도 모호점**(Phase 2)에 대해 각각 1회 배치 질의를 수행한다. 한 모델로 통합: "혼자 안전하게 못 정하는 건 사용자에게 묻는다".

타입 정의·축 활성화 매트릭스는 [references/repo-types.md](references/repo-types.md), 채점·게이팅은 [references/rubric.md](references/rubric.md), 리포트 형식은 [assets/report-template.md](assets/report-template.md)를 따른다.

## 원칙

- **증거 우선**: 모든 판정은 파일 경로(`path:line`), grep 결과, 또는 실제 명령 실행 로그로 뒷받침한다. 추측으로 PASS 판정하지 않는다.
- **타입에 맞는 축만 적용**: 비활성(N/A) 축은 채점·게이팅에서 제외한다. "해당 없음"을 미흡으로 처리하지 않는다.
- **테스트는 읽지 말고 실행한다**: 테스트/validator가 있으면 직접 돌려 stdout과 exit code를 증거로 캡처한다.
- **탐색 위임**: 3파일 이상을 훑어야 하는 탐색은 `Explore` 서브에이전트(haiku)에 위임한다.

## 절차

### Phase 0 — 입력 탐색 (기준선 확보)

0. **git 상태를 먼저 캡처**: 어떤 리포트 산출물을 만들기 전에 `git status`를 캡처해 둔다(M-CHANGE 증거로 재사용). 이후 생성하는 리포트 파일은 이 캡처에 포함되지 않으므로 자기 산출물이 미커밋 변경으로 잡히지 않는다.
1. **기획의도 문서**(intent baseline) 후보를 탐색: `docs/pr-faq*`, `docs/**`, `**/PRD*`, `**/prd*`, `**/spec*`, `AGENTS.md`, `README.md`, `CLAUDE.md`. 범위·목표·성공지표·제약을 담은 문서를 기준선으로 채택한다. (없으면 rubric.md의 fallback 모드.)
2. **검증 명령** 탐색 + **위험도 분류**: `package.json`의 scripts(`test`/`validate`/`lint`/`build`), `Makefile`, `.github/workflows/*`에서 실행 가능한 명령을 모은다. 각 명령을 **안전**(test/lint/validate/build 등 read-only류)과 **위험**(`deploy`/`migrate`/`publish`/`push`/`seed` 등 외부 부수효과·과금·DB 쓰기 가능)으로 분류한다.
   - **위험 명령**: 기본은 **미실행**(리포트에 "위험 명령 — 미실행, opt-in 필요"로 기록). `--interview` 시 위험 명령을 **1회 배치 질의**(AskUserQuestion)로 묶어 각각 [실행 / 건너뛰기 / 수정] 선택받고, 수정하면 사용자가 고친 명령으로 실행한다.
   - **안전 명령**: **타임아웃을 걸고** 실행한다(무한 대기·행 방지).
3. 채택한 기준 문서 경로와 명령 목록(위험도 분류 포함)을 리포트 헤더에 기록한다.

### Phase 1 — Repo 타입 분류 + 활성 축 결정 (게이팅 전제)

1. [references/repo-types.md](references/repo-types.md)의 분류 신호로 **주 타입 / 부 타입**을 판정한다. 분류 근거(매칭 파일·패턴 경로)를 인용한다. **어느 타입도 신호가 약하면 목록에 없는 타입을 새로 지어내지 말고 generic 폴백**(repo-types.md)으로 처리한다.
2. 타입을 축 활성화 매트릭스에 대입해 **이번 검증에서 활성화할 축 집합**과 각 축의 게이팅 등급(필수/조건부)을 확정한다. 조건부(⚠️) 축은 해당 신호(매니페스트·스키마 등)가 실제로 존재할 때만 활성화한다.
3. 활성/비활성 축 목록을 리포트 "검증 범위" 절에 기록한다. **이후 Phase는 활성 축에 대해서만 수행한다.**
   > Do not proceed: 타입과 활성 축을 확정하지 않은 채 채점하지 말 것.

### Phase 2 — 코어 축 검증 (A1 / A2 / A3, 모든 타입)

- **A1 기획의도 정합성** — 기준 문서에서 선언된 기능/범위 항목을 추출해 구현 증거에 매핑하고 `구현됨/부분구현/미구현/확인필요`를 판정한다. "제외(out-of-scope)" 항목과 `AGENTS.md` Do-Not 규칙이 지켜지는지 grep으로 역검증한다(위반은 Critical, A1 FAIL).
  - **`확인필요`(needs-confirmation)** — 증거(문서·코드)가 침묵하거나 상충해 의도 일치 여부를 판독할 수 없는 항목. 추측으로 구현됨/미구현을 매기지 말고 이 상태로 둔다. **점수 보류**(범위 추적성 분모에서 제외, pass/fail 아님). 채택 기준은 좁게: ① 증거가 양방향으로 존재, 또는 ② '의도된 결정 vs 버그' 구분이 **판정을 바꾸는** 경우만 — 증거로 도출 가능한 트리비얼은 `확인필요`로 두지 않는다. 진행 중 누적한 모호점을 **의도 모호점 리스트**로 모은다.
  - **기본(플래그 없음)**: 질문 0회. 모호점은 모두 `확인필요`로 리포트에 명시한다. 핵심 범위 항목 중 `확인필요`가 1건 이상이면 **A1 상태 상한은 concern**(미검증을 pass로 위장하지 않는다). 리포트에 "`--interview`로 재실행하면 확정 가능"을 안내한다.
  - **`--interview` 시**: 증거 패스를 끝낸 뒤, 가장 판정-영향이 큰 모호점 **최대 4개**를 `AskUserQuestion` **1회 호출**로 묶어 묻는다. 답변으로 각 `확인필요`를 구현됨/부분구현/미구현/제약위반으로 확정하고, 증거 칸에 "사용자 확인: <답변 요약>"을 기록한다. **후속 라운드는 없다(1회 한정).** 모호점이 4개를 넘으면 상위 4개만 묻고 나머지는 `확인필요`로 남긴 뒤 그 사실을 명시한다.
- **A2 AI-readable 구조 + 문서 완전성** — `AGENTS.md`/`CLAUDE.md`/`README.md` 존재·품질, 디렉토리 일관성·예측 가능한 네이밍, 기계 판독 데이터, **문서-코드 정합성**(문서의 명령·경로가 실제 존재하는지; 불일치는 Major).
- **A3 goal 검증 + 테스트 충분성** — Phase 0의 테스트/validator를 **실제 실행**(타임아웃)해 stdout·exit code를 캡처한다. **exit≠0이면 환경 탓과 진짜 실패를 구분**한다 — `command not found`·`ENOENT(모듈 없음)`·`ECONNREFUSED`·필수 env 미설정 등 **환경 시그니처면 `검증불가(blocked)`** 로 기록(축③ 실패로 치지 않음), 그 외는 real failure로 친다. pass/fail뿐 아니라 **충분성**(테스트가 핵심 기능·성공지표를 실제로 커버하는지, 빈/형식적 테스트는 아닌지)을 평가한다. 성공지표/DoD를 충족/미충족으로 매핑한다.

### Phase 3 — 조건부 모듈 검증 (Phase 1에서 활성화된 것만)

활성화된 모듈만 [references/repo-types.md](references/repo-types.md)·[rubric.md](references/rubric.md)의 항목으로 검증한다:

- **M-DATA** 데이터/스키마 무결성 — 스키마↔스냅샷 drift, 참조 무결성, 검증 스크립트 통과, 중복/고아 정의.
- **M-SEC** 보안/시크릿 위생 — 하드코딩 시크릿/키 grep, `.env` 커밋 여부. **탐지 결과를 리포트에 적을 때는 값을 마스킹**(앞 2~4자 + `***`)하고 위치(`path:line`)만 기록한다 — 리포트는 Slack·Jira로 이동하므로 평문 값을 옮겨 적으면 노출 범위가 넓어진다(P4). **오탐 줄이기(보수)**: 스킬이 자기 자신을 스캔할 때는 자기 `references/*.md`를 제외하고, 문서·주석 안의 단어 매칭(실제 값 대입이 아닌 경우)은 Critical 대신 "review 필요"로 강등한다. 단 **test/example 파일은 제외하지 않는다**(진짜 키가 박힐 수 있어 거짓음성 위험, P1).
- **M-DEP** 의존성 건전성 — lockfile 동기화, 선언↔실제 import 정합, (가능하면 `npm audit` 등 실행) 취약/미사용 의존성.
- **M-CHANGE** 변경 안전성/git 위생 — 미커밋 변경, 생성물↔소스 drift, `.gitignore` 누락.
- **M-API / M-OPS / M-PERF** — 타입상 활성일 때만(라이브러리·서비스 등).

### Phase 4 — 종합 판정 + 리포트

1. **활성 축에 대해서만** 점수(0–100)와 발견사항 severity(Critical/Major/Minor)를 산정한다(rubric.md 가중치). 비활성 축은 `N/A`로 표기한다.
2. **게이팅 규칙**: 활성 축 중 Critical(제약 위반·테스트 실패·필수 모듈 치명 결함) ≥1 → 전체 `FAIL`. Critical 0 + Major ≥2 → `CONCERNS`. 그 외 → `PASS`.
3. [assets/report-template.md](assets/report-template.md)를 채워 `repo-validation-<YYYY-MM-DD>.md`로 저장한다(날짜는 `date +%F`). **저장 위치는 대상 repo 바깥이 기본**: 호출 cwd가 repo 밖이면 거기, 아니면 스크래치 디렉토리 또는 화면 출력. 부득이 repo 안에 써야 하면 `.gitignore` 제외를 전제로 하고 그 사실을 리포트에 명시한다. **이 리포트 산출물 자체는 M-CHANGE(git 위생) 검사 대상에서 제외**한다.
4. 리포트 경로 + 판정한 타입 + 종합 판정 한 줄을 사용자에게 보고한다.
