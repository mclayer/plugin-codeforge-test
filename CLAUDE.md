# CLAUDE.md (codeforge-test)

> **[DEPRECATED — CFP-317 / ADR-048, 2026-05-09]** TestAgent / StatefulTestAgent spawn 폐지. GitHub CI가 구현 테스트 실행을 담당. QADeveloperAgent (codeforge-develop)가 `.github/workflows/test.yml` 작성. 본 plugin은 역사적 참조용으로 보존 — 신규 consumer는 사용하지 않음.

codeforge ζ arc Test lane plugin. TestAgent 단독 + owner doc 부재 (가장 단순한 lane).

## Plugin position

본 plugin 은 codeforge wrapper 의 dependency. 단독 동작 불가 — codeforge core (>= 3.0.0).

## Inter-plugin contracts

- `test_verdict v1` — [`docs/inter-plugin-contracts/test-verdict-v1.md`](docs/inter-plugin-contracts/test-verdict-v1.md) (canonical SSOT)

## Self-write 책임

| Path | 책임 agent |
|---|---|
| `[구현-테스트]` prefix GitHub comment (functional 영역) | TestAgent |
| `[구현-테스트]` prefix GitHub comment (stateful 영역) | StatefulTestAgent |
| `phase:구현-테스트` → `phase:보안-테스트` transition | (Orchestrator — 두 verdict 통합 후) |

> Story §9.3 (테스트 결과) 는 Orchestrator 가 verdict 받아 처리 — agent 직접 write 안 함.

Story §10 FIX Ledger append 는 **Orchestrator 단독** (codeforge core CFP-32 monopoly). TestAgent / StatefulTestAgent 는 verdict 에 `fix_routing_hint` 첨부만.

## Failure ownership 매트릭스 (CFP-47 / ADR-015)

| Failure 유형 | Owner verdict (해석 권한) | 다른 agent fail 시 처리 |
|---|---|---|
| Functional unit / integration | TestAgent | (일반적으로 stateful 영향 없음) |
| Infra (배포·config·smoke) | TestAgent | (일반적으로 stateful 영향 없음) |
| Performance baseline regression | TestAgent | (일반적으로 stateful 영향 없음) |
| **Long-running invariant** (cache drift / queue bound / time-window) | **StatefulTestAgent** | TestAgent 같은 module functional 도 fail 시 → StatefulTestAgent 가 `duplicate_symptom_with_test_agent: true` 메타데이터 첨부, Orchestrator 가 §10 FIX Ledger 1 entry 통합 |
| **Process restart recovery / idempotency replay / graceful shutdown** | **StatefulTestAgent** | 동일 |

**Orchestrator 룰**: 두 verdict 모두 보존 (FIX Ledger 기록), 같은 root cause 의심 시 우선순위 = StatefulTestAgent (stateful 영역 expert). DeveloperPL 1차 진단 시 두 verdict 모두 packet 첨부.

## 구현 테스트 lane 동작

Orchestrator 가 TestAgent 를 **subset 병렬** 로 spawn (R9 — [CFP-19 spec](https://github.com/mclayer/codeforge-internal-docs/blob/main/wrapper/specs/2026-04-27-cfp-19-orchestration-parallelization.md)):

- `TestAgent(subset: functional)` ∥ `TestAgent(subset: performance)` — 한 메시지에 dispatch
- 두 subset 모두 PASS → 보안 lane 진입

### Subset 1: functional

단위 / 통합 / 인프라 테스트. consumer overlay 가 러너·경로 지정.

### Subset 2: performance

baseline 대비 mean 10% 이상 악화 시 FAIL. consumer overlay 가 baseline 위치 지정.

### Sequential fallback

consumer overlay `tests.performance.depends_on_functional: true` 시 sequential 실행 (functional → performance). 기본은 parallel.

### Consumer overlay 위임

- 러너 (pytest / npm test / cargo test 등) · 테스트 경로 · baseline 파일 위치 · performance 의존성 모두 consumer overlay (`.claude/_overlay/project.yaml` `tests.*` slice) 지정
- TestAgent 는 overlay 명시값 follow — hardcoded path/runner 없음

### FAIL → 진단

FAIL 시 Orchestrator 경유 DeveloperPL 1차 진단 → ArchitectPLAgent 최종 판정. 본 lane plugin 은 `verdict.status=FAIL` 반환만 — 진단 logic 미보유.

## Dogfood policy (CFP-45)

본 plugin repo 는 runtime SSOT 만 보유. dogfood artifacts (specs/plans/retros/stories/change-plans) 는 [`mclayer/codeforge-internal-docs`](https://github.com/mclayer/codeforge-internal-docs) 단일 monorepo SSOT. 본 plugin 폴더는 `codeforge-internal-docs/test/`. 상세 정책 + Story workflow 흐름은 wrapper [CLAUDE.md](https://github.com/mclayer/plugin-codeforge/blob/main/CLAUDE.md) canonical SSOT 참조 + [ADR-013](https://github.com/mclayer/plugin-codeforge/blob/main/docs/adr/ADR-013-codeforge-family-dogfood-out-policy.md) (PR-I 머지 후 Adopted).

Plugin repo 측 GitHub Issue 와 internal-docs 측 Story file 의 binding:
- Issue body frontmatter: `story_uri: <internal-docs URL>`
- Story file frontmatter: `story_issues: [{repo: "mclayer/plugin-codeforge-test", number: <N>}]`
- `.github/workflows/phase-gate-mergeable.yml` (본 repo) 가 cross-repo Story fetch via GitHub App
