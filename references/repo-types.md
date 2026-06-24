# Repo 타입 분류 & 축 활성화 매트릭스

검증은 **하나의 고정된 축 집합**이 아니라, repo 타입에 따라 **활성화되는 축이 달라진다**. Phase 1에서 아래 신호로 타입을 판정하고, 매트릭스로 검증 축을 결정한다.

## 분류 절차 (증거 기반)

1. 루트와 1~2단계 하위에서 아래 신호를 grep/glob로 수집한다.
2. 신호 점수가 가장 높은 타입을 **주 타입(primary)** 으로, 그다음을 **부 타입(secondary)** 으로 채택한다.
3. 분류 근거(매칭된 파일/패턴 경로)를 리포트에 반드시 인용한다. 애매하면 주 타입을 택하고 부 타입을 명시한다.
4. **분류 불가 시(어느 타입도 신호가 약하거나 동률) 목록에 없는 타입을 새로 지어내지 말고 generic 모드로 폴백한다** (아래 "Generic 폴백" 참조). 분류 근거에 "신호 미약 → generic"을 기록한다.

## 타입 정의 & 분류 신호

| 타입 | 판정 신호 (있으면 +) |
|---|---|
| **spec-data** (스키마·ERD·스펙·데이터 모델링 워크벤치) | `erd/`, `schemas/`, `*.snapshot*`, 다수의 `*.json`/`*.sql`/`*.yaml` 데이터, `scripts/validate-*` 가 있고 런타임 앱 진입점은 없음 |
| **library** (배포용 라이브러리/패키지) | `package.json`에 `main`/`module`/`exports`/`bin`, 퍼블리시 설정(`publishConfig`, `files`), `src/index.*`, 서버 프레임워크 미사용 |
| **service-app** (배포 가능한 서버/앱) | `Dockerfile`, `Procfile`, `k8s/`/`helm/`, 서버 프레임워크 deps(express/fastify/nest/spring/django/flask 등), `start` 스크립트, `.env*` |
| **cli-tool** | `package.json` `bin`, argparse/commander/clap/cobra 등 CLI 프레임워크, `man/` |
| **monorepo** | `workspaces`/`pnpm-workspace.yaml`/`turbo.json`/`nx.json`/`lerna.json`, 여러 하위 패키지 — 각 패키지를 위 타입으로 재분류 |
| **docs-only** | 대부분 `*.md`/`docs/`, 빌드·테스트·소스 코드 거의 없음 |

> 복합 신호는 흔하다. 예) ERD 스키마 + node 검증 스크립트 = **primary: spec-data, secondary: cli-tool**.

## 축 활성화 매트릭스

축 상태는 `pass | concern | fail | n/a`. **N/A 축은 종합 판정에서 제외**된다(게이팅 트리거 아님).

| 축 / 모듈 | spec-data | library | service-app | cli-tool | docs-only |
|---|:--:|:--:|:--:|:--:|:--:|
| **A1 기획의도 정합성** (코어) | ✅ | ✅ | ✅ | ✅ | ✅ |
| **A2 AI-readable 구조 + 문서 완전성** (코어) | ✅ | ✅ | ✅ | ✅ | ✅ |
| **A3 goal 검증 + 테스트 충분성** (코어) | ✅ | ✅ | ✅ | ✅ | ⚠️도구있을때 |
| **M-DATA 데이터/스키마 무결성** | ✅필수 | ⚠️스키마있을때 | ⚠️ | ⚠️ | n/a |
| **M-SEC 보안/시크릿 위생** | ✅ | ✅ | ✅필수 | ✅ | ⚠️ |
| **M-DEP 의존성 건전성** | ⚠️매니페스트있을때 | ✅필수 | ✅필수 | ✅ | n/a |
| **M-CHANGE 변경 안전성/git 위생** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **M-API API/계약 안정성** | n/a | ✅ | ✅ | ⚠️ | n/a |
| **M-OPS 운영 준비성** (12-factor 부분집합) | n/a | n/a | ✅필수 | ⚠️ | n/a |
| **M-PERF 성능 특성** | n/a | ⚠️ | ⚠️선택 | n/a | n/a |

- ✅ = 항상 활성, ✅필수 = 활성 + 핵심 게이팅 축, ⚠️ = 조건부(신호 있을 때만 활성), n/a = 비활성.
- monorepo는 패키지별로 위 매트릭스를 적용하고 결과를 합산한다.

### Generic 폴백 (분류 불가 시)

어느 타입에도 명확히 해당하지 않으면 **임의 타입을 만들지 말고** 아래 최소 축만 활성화한다. 조건부 모듈은 해당 신호가 실제로 있으면 일반 규칙대로 활성화해도 된다.

| 활성 축 | 비활성 |
|---|---|
| A1, A2, A3 (코어) + M-SEC + M-CHANGE | M-DATA / M-DEP / M-API / M-OPS / M-PERF (신호 있을 때만 예외적 활성) |

## 조건부 모듈 검증 항목 요약

활성화된 모듈만 검증한다. 상세 채점은 [rubric.md](rubric.md) 참조.

- **M-DATA**: 스키마 ↔ 스냅샷 drift(동일 엔티티가 한쪽만 수정됐는지), 참조 무결성(FK·관계 대상 존재), 스키마 파일이 검증 스크립트를 통과하는지, 중복/고아 정의.
- **M-SEC**: 하드코딩된 시크릿/토큰/키 grep(`api[_-]?key`, `secret`, `password`, `BEGIN PRIVATE KEY`, AWS 키 패턴), `.env`가 커밋됐는지, `SECURITY.md` 필요 여부.
- **M-DEP**: lockfile 존재·동기화, 선언된 deps와 실제 import 정합성, 알려진 취약 버전(가능하면 `npm audit`/`pip-audit` 실행), 미사용/유령 의존성.
- **M-CHANGE**: `git status` 미커밋 변경, 생성물이 소스와 drift(예: 스냅샷만 또는 원본만 수정), `.gitignore` 누락으로 생성물이 추적되는지.
- **M-API**: 공개 API/타입/스키마의 하위호환, 버전 표기(semver), CHANGELOG.
- **M-OPS**: config 외부화(env, 하드코딩 금지), 로그 전략, 빌드/실행 분리, 헬스체크, graceful shutdown.
- **M-PERF**: 명시된 성능 목표 대비 측정 근거(있을 때만).
