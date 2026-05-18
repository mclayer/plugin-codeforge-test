---
title: codeforge-test lane 구조 (통합테스트 — Epic-level 통합 검증)
last_captured: 2026-05-18
kind: architecture_doc
family_ref: ../../../plugin-codeforge/docs/architecture/codeforge-family.md#모듈
---

> **목표 invariant (ADR-078 §결정 1 verbatim)**: 코드 직접 read 없이 architecture_doc 1개 read 로 전체 구조 (모듈 + 경계 + 인터페이스 + 데이터 흐름) 파악.

<!-- 본 file = codeforge-test lane plugin self-owned seed (CFP-972 / Sub-Epic CFP-949 Wave 2 / Epic CFP-756 / ADR-078).
     누적 현재 상태 SSOT. Story key 독립 (고정 경로). 델타 = Change Plan SSOT (disjoint).
     family 전체 구조 = wrapper repo codeforge-family.md SSOT (frontmatter family_ref 참조).
     본 doc = lane internal 구조만 채운다. -->

## 모듈

codeforge-test lane = **Epic-level 통합테스트 — Epic 하위 전체 Story CI gate PASS 이후 1회 통합 검증** 책임. **active 1 agent (IntegrationTestAgent) + deprecated 2 agent (TestAgent / StatefulTestAgent — ADR-048 §결정 2 historical preservation only, 신규 Story 미spawn)**. `[verified: lane plugin agents/*.md tree direct enumerate @ d4506d2d — 3 file 보존: IntegrationTestAgent.md / StatefulTestAgent.md / TestAgent.md]` + `[verified: lane plugin CLAUDE.md @ d4506d2d "REVIVED — CFP-367 / ADR-055 (2026-05-10) · Epic-level 전환 — CFP-371 / ADR-055 Amendment 2" + "구현 테스트 lane (TestAgent / StatefulTestAgent) — DEPRECATED 유지" 단락]`:

**Active (1 agent)** — 모든 Epic-level 통합테스트 진입 시 Orchestrator 가 spawn:

| 모듈 (agent) | tier | 책임 1줄 |
|---|---|---|
| **IntegrationTestAgent** | Sonnet (rate-limit fallback → Opus, ADR-057) | Epic 하위 `stories_in_scope` 전체 CI gate PASS 후 1회 spawn — Deployability 검증 (4-step) + Baseline Suite + Story Suite (§8.6 자동 생성) + Baseline 자동 승격 (self-commit) + story_keys blame 3-tier — `test_verdict v2.2` packet 반환 |

**Deprecated (2 agent, ADR-048 §결정 2 historical 보존)** — agents/ tree 에 file 존재하나 신규 Story 미spawn (역사적 참조용):

| 모듈 (agent) | 상태 | 비고 |
|---|---|---|
| **TestAgent** | deprecated | CFP-317 / ADR-048 Amendment 1 부활 이전 구현 테스트 lane (functional / performance subset 병렬) 담당 |
| **StatefulTestAgent** | deprecated | long-running invariant / process restart recovery 담당 (위와 동일 deprecation 시점) |

> wrapper CLAUDE.md SSOT (L65 "1 (IntegrationTestAgent)") = **active count 정확** — deprecated 2 file 은 historical 보존이라 active roster 외. wrapper 진술과 lane plugin actual = **합치** (`[verified: wrapper CLAUDE.md @ 9e11011c L65 + lane plugin CLAUDE.md @ d4506d2d "구현 테스트 lane ... DEPRECATED 유지"]`). 본 doc = active + deprecated 명시 분리 보강 (architecture_doc layer 의 ground truth carrier).

## 경계

**Lane self-write boundary** `[verified: lane plugin CLAUDE.md @ d4506d2d "Self-write 책임" 표]`:

| Path | 책임 agent |
|---|---|
| `[통합-테스트]` prefix GitHub comment | IntegrationTestAgent |
| `tests/integration/stories/<EPIC-KEY>/<STORY-KEY>/` 테스트 파일 write | IntegrationTestAgent (Story Suite 자동 생성, §8.6 기반) |
| `tests/integration/baseline/<STORY-KEY>/` Baseline 자동 승격 write + self-commit | IntegrationTestAgent (PASS 시만 — FAIL 상태 승격 invariant 차단) |
| `docs/architecture/codeforge-test.md` (본 doc 영역) | IntegrationTestAgent or 설계 lane ArchitectAgent (lane gate — ADR-078 §결정 1 4 H2 영역 갱신 의무, 매 Change Plan merge 시) |
| `phase:통합-테스트` → `phase:보안-테스트` transition | Orchestrator (verdict 수령 후, lane plugin agent write 영역 외) |

> Story §9 통합테스트 섹션은 IntegrationTestAgent 가 **직접 write 안 함** — test_verdict packet 을 Orchestrator 에 반환 → Orchestrator 가 Epic 내 각 관련 Story §9 append.

**Epic-level only invariant** `[verified: lane plugin CLAUDE.md @ d4506d2d "Lane 위치" + agents/IntegrationTestAgent.md @ d4506d2d "호출 시점"]`:

- **per-Story 통합테스트 부재** — Story-level 통합 검증 trigger 0건. Orchestrator 는 Epic 하위 `stories_in_scope` 전체 CI gate PASS 확인 후에만 IntegrationTestAgent spawn (1회).
- **선행 gate 의무** — Epic 하위 1개 Story 라도 CI gate 미통과 상태에서 본 lane 진입 금지.
- **후행 gate** — PASS 시 보안 테스트 레인 (SecurityTestPL, opt-in) 또는 Epic 완료. FAIL 시 4-way 라우팅 (아래 FIX 루프 데이터 흐름 참조).

**Deployability 검증 4-step boundary** (선행 필수, 미통과 시 즉시 verdict 반환 + 테스트 실행 차단) `[verified: agents/IntegrationTestAgent.md @ d4506d2d §1]`:

| step | 검증 대상 | 실패 시 failure_type |
|---|---|---|
| (a) `.env` 키 확인 | `required_env_keys` 목록 grep | `env_missing` |
| (b) 컨테이너 기동 | `docker-compose -f docker-compose.test.yml up -d --wait` | `infra_setup` |
| (c) DB 연결 확인 | `project.yaml integration_test.db_probes[]` (consumer overlay) | `infra_setup` |
| (d) health check endpoint | `project.yaml integration_test.health_checks[]` (default `:8000/health` 200) | `infra_setup` |

4-step 모두 PASS → `deployability_verified: true` → Baseline Suite + Story Suite 실행 진입. 어느 하나라도 FAIL → 검증 중단 + verdict 반환.

**Baseline 자동 승격 boundary** (FAIL 차단 invariant) `[verified: agents/IntegrationTestAgent.md @ d4506d2d §7]`:

- Story Suite 전체 PASS 시만 `tests/integration/stories/<EPIC-KEY>/<STORY-KEY>/` → `tests/integration/baseline/<STORY-KEY>/` 복사 + 메타데이터 `SUITE_TYPE: "story" → "baseline"` 갱신 + IntegrationTestAgent self-commit.
- **FAIL 상태 승격 절대 금지** — 깨진 테스트가 Baseline 유입 = 영구 regression risk (unconditional guard, intent: 무조건 PASS 시점 검사).

**FIX 루프 cross-lane boundary** `[verified: lane plugin CLAUDE.md @ d4506d2d "통합 테스트 lane 동작" + agents/IntegrationTestAgent.md @ d4506d2d "FAIL 시 회귀 경로"]`:

| failure_type | 1차 가정 | FIX 담당 |
|---|---|---|
| `regression` | 구현 원인 (기존 코드 regression) | DeveloperPL → ArchitectPLAgent 최종 판정 |
| `new_test` | 구현 원인 (신규 시나리오 미구현) | DeveloperPL → ArchitectPLAgent 최종 판정 |
| `infra_setup` | 인프라 원인 (docker-compose 누락/오류) | InfraEngineerAgent 직접 수정 (ArchitectPL 불필요) |
| `env_missing` | 환경 설정 누락 (.env 키) | InfraEngineerAgent or 사용자 action |

**Scope partition** (dogfood-out, ADR-013):

- 본 plugin repo = runtime SSOT 만 (agents/* + `docs/inter-plugin-contracts/test-verdict-v*.md` + `CLAUDE.md` + 본 architecture_doc).
- dogfood artifacts (specs/plans/retros/stories/change-plans) = `mclayer/codeforge-internal-docs/codeforge-test/` monorepo SSOT.

**Disjoint scope** (ADR-078 §결정 3):

- 본 doc (architecture_doc) = lane internal 누적 현재 상태, Story key 독립, 영속.
- Change Plan = Story별 변경 델타, Story key 종속, 1회 작성.
- ADR = 단일 결정 단위 (불변).
- 본 doc ↔ Change Plan = 상보 disjoint (구조 vs 델타).

## 인터페이스 계약

lane 외부 surface — kind:contract producer / consumer overlay slice / governance ADR anchor.

**Producer (1 contract)** `[verified: docs/inter-plugin-contracts/test-verdict-v2.md @ d4506d2d]`:

| contract | 위치 | 용도 |
|---|---|---|
| `test_verdict` | `docs/inter-plugin-contracts/test-verdict-v2.md` (canonical, lane plugin repo) + wrapper sibling sync mirror (ADR-010) | Epic-level 통합테스트 결과 패킷. IntegrationTestAgent → Orchestrator 핸드오프. `lane: "integration"` + `trigger: "epic_complete"` 고정. `stories_in_scope` / `suite_summary` / `failures[].story_keys[]` + `attribution_confidence` enum / `pl_recommendation` enum |

> contract schema field-level 상세 + version 값 = canonical contract file SSOT + wrapper `MANIFEST.yaml`. version literal 미박제 (drift 회피, ADR-008).

**Consumer overlay slice** (consumer project `.claude/_overlay/project.yaml`):

| key | 용도 |
|---|---|
| `integration_test.required_env_keys[]` | Deployability step (a) 검증 대상 — `.env` 필수 키 목록 |
| `integration_test.docker_compose_test_path` | Deployability step (b) — `docker-compose.test.yml` 경로 |
| `integration_test.db_probes[]` | Deployability step (c) — DB 연결 ping (per-dialect `connection_env` + `ping_command`) |
| `integration_test.health_checks[]` | Deployability step (d) — HTTP GET endpoint + `expected_status` + `timeout_seconds` |

> consumer overlay = 확장만 허용 (wrapper / lane plugin 정책 축소 불가, ADR-027).

**story_keys blame 3-tier** (Baseline 실패 시 `failures[].story_keys` 산출 mechanism) `[verified: agents/IntegrationTestAgent.md @ d4506d2d §4 + test-verdict-v2.md @ d4506d2d "Baseline 실패 시 story_keys blame 절차"]`:

| tier | 출처 | attribution_confidence |
|---|---|---|
| Tier 1 | §8.6 `coverage_targets[].related_components[]` 직접 매핑 | `"definite"` |
| Tier 2 | 테스트 파일 import 구문 정적 분석 (`from src.XXX import ...`) | `"inferred"` |
| Tier 3 | Tier 1·2 모두 실패 → `story_keys: []` + ArchitectPL ESCALATE | `"unknown"` |

→ 컴포넌트 경로 확정 후 `git log --oneline --follow -- <path>` 의 최근 commit Story key 추출 → `story_keys` 추가 + `responsible_stories` union 갱신.

**Governance ADR anchor**:

- **ADR-048 Amendment 1** — codeforge-test lane 통합테스트 전용 부활 (CFP-367 / 2026-05-10). TestAgent / StatefulTestAgent deprecated. 본 lane 의 존재 근거.
- **ADR-055** — Integration Test Lane Policy SSOT. Amendment 2 (CFP-371) = per-Story → Epic-level 실행 구조 전환 (1회 spawn / `epic_complete` trigger).
- **ADR-067** — Max FIX 3/3 + cross-lane RESET. 본 lane 의 FIX verdict 가 4 lane (CodeReview / SecurityTest / 본 lane) 누적 FIX 카운터에 합산.
- **ADR-72** — Production cutover gate + ProductionEvidenceDeputy. Epic-level 통합테스트 PASS → 보안 lane → production cutover Story 시 ProductionEvidenceDeputy 동반.
- **ADR-057** — Sonnet → Opus rate-limit fallback (IntegrationTestAgent Sonnet tier 적용 대상).

**Skill anchor** (Orchestrator lane 진입 시 호출):

- `codeforge:review-responsibility` — 본 lane 진입 시 (보안 테스트 lane preflight 와 동일 호출 점). 4 lane 체크 항목 분담 SSOT.
- `codeforge:fix-ledger-schema` + `codeforge:root-cause-decision` — FIX 루프 진입 시.

> 본 섹션 = surface enumeration (계약 이름 + SSOT pointer). 계약 schema field-level 상세 = 해당 contract file SSOT.

## 데이터 흐름

**Epic-level lane spawn 흐름** (입력 → 변환 → 출력, anti-scope guard 준수 — 함수 호출 trace / 변수 전달 라인 0건):

```
[upstream gate] Epic 하위 stories_in_scope 전체 CI gate PASS (CodeReviewPL + GitHub CI)
  │
  ▼
Orchestrator → IntegrationTestAgent spawn (single-shot, 1회 per Epic — ADR-044 §결정 5)
  │ packet 주입: { epic_key, stories_in_scope, story_8_6_contracts, baseline_suite_path,
  │               required_env_keys, docker_compose_test_path }
  ▼
[step 1] Deployability 4-step 검증
  ├─ (a) .env 키 grep              → fail → failure_type: env_missing → verdict 반환 (테스트 차단)
  ├─ (b) docker-compose up -d --wait → fail → failure_type: infra_setup → verdict 반환
  ├─ (c) db_probes ping             → fail → failure_type: infra_setup → verdict 반환
  └─ (d) health_checks GET          → fail → failure_type: infra_setup → verdict 반환
  │ (4-step PASS → deployability_verified: true)
  ▼
[step 2] §8.6 Integration Test Contract 수집
  ├─ §8.6 없는 Story → ESCALATE_PACKET_INCOMPLETE (TestContractArchitectAgent 의뢰)
  ├─ §8.6 = N/A     → skipped 증분, Story Suite 생성 생략
  └─ dynamic_test_required: true → Story Suite 생성 대상
  ▼
[step 3] Story Suite 자동 생성
  ├─ 경로: tests/integration/stories/<EPIC-KEY>/<STORY-KEY>/test_<scenario_snake>.py
  ├─ 파일 상단 메타데이터 주입: STORY_KEY = "<STORY-KEY>" + SUITE_TYPE = "story"
  │     (failure 발생 시 story_keys=[STORY_KEY] + attribution_confidence="definite" 자동 매핑)
  └─ 내부 컴포넌트 mock 금지 / 외부 의존성 WireMock 허용 (동적 테스트 원칙)
  ▼
[step 4] Baseline Suite 실행 (이전 Epic까지 누적된 baseline)
  └─ FAIL 시 story_keys blame 3-tier:
       ├─ Tier 1: §8.6 related_components → confidence="definite"
       ├─ Tier 2: import 정적 분석          → confidence="inferred"
       └─ Tier 3: 모두 실패 → story_keys=[] + confidence="unknown" + ESCALATE
       → git log --follow → 최근 commit Story key 추출
  ▼
[step 5] Story Suite 실행
  └─ FAIL 시 STORY_KEY 메타데이터 매핑 (definite)
  ▼
[step 6] docker-compose down (PASS/FAIL 관계없이 의무 — 환경 파괴 금지 invariant)
  ▼
[step 7] Baseline 자동 승격 (전체 suite PASS 시만 — FAIL 차단 invariant)
  ├─ cp tests/integration/stories/<EPIC>/<STORY>/test_*.py → tests/integration/baseline/<STORY>/
  ├─ SUITE_TYPE: "story" → "baseline" 갱신
  └─ git commit (IntegrationTestAgent self-commit, "test(baseline): <EPIC-KEY> Story Suite 자동승격")
  ▼
test_verdict v2.2 packet emit → Orchestrator 반환
  ▼
Orchestrator 분기:
  ├─ pl_recommendation: PASS                  → 보안 테스트 lane or Epic 완료 (epic_close_ready)
  ├─ pl_recommendation: FIX                   → responsible_stories 별 FIX 루프 (4-way 라우팅)
  └─ pl_recommendation: ESCALATE_PACKET_INCOMPLETE → docker-compose 미실행 or §8.6 누락
```

**FIX 4-way routing** (test_verdict.failure_type → 담당 agent):

```
test_verdict.failures[].failure_type
  │
  ├─ regression  ──→ DeveloperPL 1차 진단 → ArchitectPL 최종 판정 → FIX → CI gate 재통과 → 본 lane 재진입
  ├─ new_test    ──→ DeveloperPL 1차 진단 → ArchitectPL 최종 판정 → FIX → CI gate 재통과 → 본 lane 재진입
  ├─ infra_setup ──→ InfraEngineerAgent 직접 수정 (ArchitectPL 불필요) → 본 lane 재진입
  └─ env_missing ──→ InfraEngineerAgent or 사용자 action → 본 lane 재진입
```

**artifact propagation**:

- **Story file** (`internal-docs/codeforge-test/stories/<KEY>.md` §9 통합테스트 섹션) — Orchestrator 가 verdict 수령 후 Epic 내 각 관련 Story §9 append (lane plugin agent write 영역 외).
- **`tests/integration/stories/<EPIC-KEY>/<STORY-KEY>/`** — IntegrationTestAgent self-write (Story Suite 자동 생성, §8.6 기반).
- **`tests/integration/baseline/<STORY-KEY>/`** — IntegrationTestAgent self-commit (전체 suite PASS 시만 자동 승격).
- **test_verdict v2.2 packet** — Orchestrator 핸드오프 carrier (Story §9 append + §10 FIX Ledger row append carrier).
- **Epic close gate** — `epic_close_ready` 신호 (test_verdict PASS + 보안 lane PASS / N/A 후 ProductionEvidenceDeputy 분기 시 ADR-72).

**ADR-067 cross-lane RESET 연동**:

- 본 lane FIX verdict 1건 → §10 FIX Ledger 1 row append (Orchestrator monopoly, fix-event-v1).
- max FIX 3/3 도달 시 ArchitectPL implementability reassessment 진입.

> 본 흐름 = lane spawn / event / artifact propagation 수준 (라인 단위 함수 trace / 변수 전달 0건, anti-scope guard 준수).

---

### ADR-076 declarative reconciliation 3-layer cross-ref

본 lane 의 architecture_doc 운용은 [ADR-076](../../../plugin-codeforge/docs/adr/ADR-076-declarative-reconciliation-upgrade.md) declarative reconciliation 3-layer 패턴을 도메인 disjoint 로 답습 (ADR-078 §결정 2 정합, codeforge-{requirements,design,develop} Wave 1 precedent 답습):

- **desired state** = 본 doc 의 4 H2 closed-enum (모듈 + 경계 + 인터페이스 계약 + 데이터 흐름) 누적 현재 상태 SSOT.
- **current state** = lane plugin agent file (`agents/IntegrationTestAgent.md` + deprecated `agents/TestAgent.md` / `agents/StatefulTestAgent.md`) + `CLAUDE.md` + contract file (`docs/inter-plugin-contracts/test-verdict-v2.md`) 의 실제 정의 상태.
- **converge** = ArchitectAgent self-write 확장 + design lane verdict gate (drift lint CFP-923 detection class d, architecture-drift lint 후속 carrier).

> 본 cross-ref = 패턴 답습 (pattern). 도메인 (upgrade flow ↔ 통합테스트 lane) 은 disjoint. wording SSOT = ADR-076 본문 + ADR-078 §결정 2.

---

### anti-scope guard (ADR-078 §결정 1 verbatim — 작성자 필독)

본 doc 은 **구조 수준 only**. closed-enum 4 영역 외 다음 4종 패턴은 **금지** (라인 수준 허용 시 갱신 즉시 stale + "코드에 한 단계 더한 것" 전락 — Epic §위험신호 §1):

1. **클래스 / 함수 / 변수 라인 단위 열거** — 클래스 list, 변수 enumeration 금지.
2. **의존성 import graph 라인-level** — import 관계 라인 단위 그래프 금지.
3. **함수 signature / parameter list / return type** — API 의 line-level 시그니처 금지.
4. **코드 mirror** — `agents/` 또는 `tests/integration/` 구조를 1:1 복사한 디렉터리 트리 dump 금지.

→ 위 4종이 필요하면 그것은 코드 / Change Plan / ADR 영역. architecture_doc 은 "코드 read 없이 구조 파악" 목표만 만족하면 된다.
