# repo-validate

Type-aware, evidence-based **repo validity-check** skill for [Claude Code](https://claude.com/claude-code).

이 스킬은 현재 repo가 (1) 기획 의도대로 구현됐는지, (2) AI가 잘 읽을 수 있게 구조화됐는지, (3) goal 검증이 완료됐는지를 **증거 기반(evidence-based)** 으로 검증한다. 검증 축은 고정이 아니라 **repo 타입**(`spec-data` / `library` / `service-app` / `cli` / `monorepo` / `docs`)에 따라 활성화되는 축이 달라진다 — 스키마 repo에 운영 준비성을 묻거나 라이브러리에 데이터 drift를 묻는 식의 잘못된 기준 적용을 막는다.

분석은 read-only이며 테스트/validator를 실제로 실행한다. repo 소스는 수정하지 않고 Markdown 리포트 파일만 새로 만든다.

## 설치

### A) git clone (권장)

```bash
git clone https://github.com/hl-kim-mz/claude-skill-repo-validate.git \
  ~/.claude/skills/repo-validate
```

### B) symlink (소스를 따로 두고 싶을 때)

```bash
git clone https://github.com/hl-kim-mz/claude-skill-repo-validate.git ~/dev/repo-validate
ln -s ~/dev/repo-validate ~/.claude/skills/repo-validate
```

`~/.claude/skills/` 아래에 두면 모든 프로젝트에서 전역으로 인식된다. 설치 후 Claude Code를 재시작하면 스킬 목록에 잡힌다.

## 사용

```
/repo-validate [검증 대상 경로]
```

경로를 생략하면 현재 디렉토리(`.`)를 검증한다. 또는 "검증", "타당성 검증", "validate repo" 등으로 자연어 호출할 수 있다.

## 동작 개요

| Phase | 내용 |
|-------|------|
| **0** | 입력 탐색 — 기획 의도 문서(PRD/spec/README/AGENTS.md)와 실행 가능한 검증 명령 수집 |
| **1** | repo 타입 분류 + 활성 축 결정 (게이팅 전제) |
| **2** | 코어 축 검증 — A1 기획의도 정합성 / A2 AI-readable 구조·문서 완전성 / A3 goal 검증·테스트 충분성 |
| **3** | 조건부 모듈 검증 — 데이터/스키마, 보안/시크릿, 의존성, 변경 안전성, API/OPS/PERF (타입상 활성일 때만) |
| **4** | 종합 판정(PASS / CONCERNS / FAIL) + Markdown 리포트 생성 |

타입 정의·축 활성화 매트릭스는 [`references/repo-types.md`](references/repo-types.md), 채점·게이팅 규칙은 [`references/rubric.md`](references/rubric.md), 리포트 형식은 [`assets/report-template.md`](assets/report-template.md)를 따른다.

## 구조

```
SKILL.md                      스킬 본문 (frontmatter + 절차)
references/repo-types.md      타입 분류 신호 + 축 활성화 매트릭스
references/rubric.md          채점 가중치 + 게이팅 규칙
assets/report-template.md     리포트 출력 템플릿
```

## License

[MIT](LICENSE)
