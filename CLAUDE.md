# CLAUDE.md (codeforge-test)

> **[REVIVED — CFP-367 / ADR-055 (2026-05-10) · Epic-level 전환 — CFP-371 / ADR-055 Amendment 2]** 통합테스트 전용 lane으로 부활. IntegrationTestAgent(Sonnet) 신규 추가. TestAgent / StatefulTestAgent는 deprecated 유지(ADR-048 §결정 2).

codeforge 통합테스트 lane plugin. IntegrationTestAgent 전담 — Epic 하위 전체 Story CI gate PASS 이후 1회 실행. Deployability 검증(4-step) + Baseline Suite + Story Suite 자동 생성 + Baseline 자동 승격.

## Plugin position

본 plugin 은 codeforge wrapper 의 dependency. 단독 동작 불가 — codeforge core (>= 3.0.0).

**Lane 위치**: Epic 하위 **전체 Story** CI gate PASS 이후 (1회), 보안 테스트(SecurityTestPL) 이전. Orchestrator가 직접 spawn.

## Inter-plugin contracts

- `test_verdict v1` — [`docs/inter-plugin-contracts/test-verdict-v1.md`](docs/inter-plugin-contracts/test-verdict-v1.md) — Archived (CFP-317 / ADR-048)
- `test_verdict v2.1` — [`docs/inter-plugin-contracts/test-verdict-v2.md`](docs/inter-plugin-contracts/test-verdict-v2.md) (canonical SSOT) — Active (CFP-367 / ADR-055 · CFP-371 Epic-level)

## Self-write 책임

| Path | 책임 agent |
|---|---|
| `[통합-테스트]` prefix GitHub comment | IntegrationTestAgent |
| `phase:통합-테스트` → `phase:보안-테스트` transition | Orchestrator (verdict 수령 후) |
| `tests/integration/stories/<EPIC-KEY>/<STORY-KEY>/` 테스트 파일 write | IntegrationTestAgent |
| `tests/integration/baseline/<STORY-KEY>/` Baseline 자동 승격 write | IntegrationTestAgent |

> Story §9 통합테스트 섹션은 Orchestrator가 verdict 받아 처리 — agent 직접 write 안 함.

Story §10 FIX Ledger append는 **Orchestrator 단독** (CFP-32 monopoly).

## 통합 테스트 lane 동작

IntegrationTestAgent의 상세 동작 명세는 [`agents/IntegrationTestAgent.md`](agents/IntegrationTestAgent.md) 참조.

핵심 요약:
- **Spawn 조건**: Epic 하위 전체 Story CI gate(CodeReviewPL + GitHub CI) PASS 이후 (1회)
- **실행 순서**: Deployability 4-step 검증 → Baseline Suite → Story Suite → Baseline 자동 승격(PASS 시)
- **Story Suite 경로**: `tests/integration/stories/<EPIC-KEY>/<STORY-KEY>/` + `STORY_KEY`/`SUITE_TYPE` 메타데이터 주입
- **출력**: test-verdict-v2.1 패킷 → Orchestrator 수령 → 각 Story §9 append
- **FAIL 시**:
  - `regression`/`new_test` → DeveloperPL 1차 진단 → ArchitectPLAgent 최종 판정
  - `infra_setup`/`env_missing` → InfraEngineerAgent 직접 수정 (ArchitectPL 불필요)

## 구현 테스트 lane (TestAgent / StatefulTestAgent) — DEPRECATED 유지

TestAgent / StatefulTestAgent는 ADR-048 §결정 2에 의해 deprecated 유지.
역사적 참조용으로 `agents/TestAgent.md`, `agents/StatefulTestAgent.md` 보존.
신규 Story에서는 IntegrationTestAgent만 spawn.

## Failure ownership 매트릭스 (CFP-47 / ADR-015)

> **[DEPRECATED — ADR-048]** TestAgent / StatefulTestAgent 소유 영역. 신규 Story는 IntegrationTestAgent 사용.

| Failure 유형 | Owner verdict (해석 권한) | 다른 agent fail 시 처리 |
|---|---|---|
| Functional unit / integration | TestAgent | (일반적으로 stateful 영향 없음) |
| Infra (배포·config·smoke) | TestAgent | (일반적으로 stateful 영향 없음) |
| Performance baseline regression | TestAgent | (일반적으로 stateful 영향 없음) |
| **Long-running invariant** (cache drift / queue bound / time-window) | **StatefulTestAgent** | TestAgent 같은 module functional 도 fail 시 → StatefulTestAgent 가 `duplicate_symptom_with_test_agent: true` 메타데이터 첨부, Orchestrator 가 §10 FIX Ledger 1 entry 통합 |
| **Process restart recovery / idempotency replay / graceful shutdown** | **StatefulTestAgent** | 동일 |

**Orchestrator 룰**: 두 verdict 모두 보존 (FIX Ledger 기록), 같은 root cause 의심 시 우선순위 = StatefulTestAgent (stateful 영역 expert). DeveloperPL 1차 진단 시 두 verdict 모두 packet 첨부.

## 구현 테스트 lane 동작

> **[DEPRECATED — ADR-048 / CFP-317]** TestAgent / StatefulTestAgent 기반 동작 명세. 역사적 참조용으로 보존.

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
