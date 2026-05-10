---
kind: contract
contract_version: "2.2"
status: Active
related_plugins:
  - codeforge (wrapper, consumer + IntegrationTestAgent spawn 주체)
  - codeforge-test (lane plugin, producer + self-writer)
related_adrs:
  - ADR-008  # Inter-plugin Contract Versioning
  - ADR-010  # Inter-plugin Contract Sibling Sync
  - ADR-055  # Integration Test Lane Policy (본 v2 carrier)
  - ADR-048  # Amendment 1 — codeforge-test 통합테스트 전용 부활
authors:
  - CFP-367 (2026-05-10) — test-verdict v1 → v2 (integration lane 전용 패킷, ADR-055)
  - CFP-371 (2026-05-10) — v2 → v2.1: Epic-level 필드, story_attribution, env_missing, deployability_verified
  - CFP-373 (2026-05-10) — v2.1 → v2.2: story_key→story_keys[] 복수 attribution + attribution_confidence
supersedes: test-verdict-v1.md
carrier_story: CFP-367
date: 2026-05-10
---

# test-verdict-v2 — Integration Lane 결과 패킷 (Canonical)

**CANONICAL SSOT**: 본 파일이 원본. wrapper sibling: `mclayer/plugin-codeforge:docs/inter-plugin-contracts/test-verdict-v2.md`

## 상태

Active — CFP-373 (2026-05-10)

test-verdict-v1 Archived. v1 → v2 이유: codeforge-test 통합테스트 전용 부활(ADR-048 Amendment 1)로 integration lane 전용 결과 패킷 스키마 신설.

v2 → v2.1 이유: CFP-371 / ADR-055 Amendment 2 — per-Story → Epic-level 실행 구조 전환. epic_key, stories_in_scope, responsible_stories, deployability_verified, suite_type, env_missing 필드 추가.

v2.1 → v2.2 이유: CFP-373 — `failures[].story_key: string|null` → `story_keys: list[string]` + `attribution_confidence` 추가. 단일 baseline failure가 복수 Story 변경에 기인할 수 있는 현실 반영. ADR-008 SemVer MINOR bump.

## 스키마

```yaml
test_verdict:
  version: "2.2"
  epic_key: string              # "CFP-NNN" — Epic 단위 실행
  stories_in_scope: list        # ["CFP-NNN-S1", "CFP-NNN-S2"] — 이번 Epic 포함 Story key 목록
  lane: "integration"           # 고정값
  executed_at: ISO8601
  runner: "IntegrationTestAgent"
  trigger: "epic_complete"      # 고정값 — Epic 하위 전체 Story CI gate PASS 후 1회

  suite_summary:
    baseline_total: int         # Baseline Suite 전체 테스트 수 (이전 Epic까지 누적)
    baseline_passed: int
    baseline_failed: int
    story_total: int            # Story Suite 전체 테스트 수 (이번 Epic §8.6 기반 자동생성)
    story_passed: int
    story_failed: int
    skipped: int                # docker-compose 환경 미구성 or §8.6 면제 Story

  dynamic_test_compliance: boolean     # 내부 컴포넌트 정적 mock 미사용 여부
  docker_compose_used: boolean         # docker-compose.test.yml 실행 여부
  deployability_verified: boolean      # .env 키 + container 기동 + DB 연결 + health check 통과

  failures:                     # failed > 0 인 경우에만 존재
    - test_id: string
      test_path: string         # "tests/integration/stories/CFP-NNN/CFP-NNN-S1/test_order_flow.py::test_bithumb_order_create"
      suite_type: "baseline" | "story"
      story_keys: list[string]  # suite_type=story → [해당 Story key] (단일 항목)
                                # suite_type=baseline → blame 분석 결과 목록 (분석 전 [] 가능)
                                # 단일 baseline failure가 복수 Story 변경에 기인할 수 있으므로 list
      attribution_confidence: "definite" | "inferred" | "unknown"
                                # definite: STORY_KEY 메타데이터 or §8.6 related_components 직접 매핑
                                # inferred: static import 분석으로 추론
                                # unknown: blame 불가 (story_keys=[])
      failure_type: "regression" | "new_test" | "infra_setup" | "env_missing"
      error_summary: string     # 500자 이내

  responsible_stories: list     # FIX 대상 Story key 목록 e.g. ["CFP-NNN-S1"] (failures story_keys 합집합)

  pl_recommendation: "PASS" | "FIX" | "ESCALATE_PACKET_INCOMPLETE"
  # PASS: 전체 suite green + deployability_verified true. responsible_stories: []
  # FIX: 실패 존재 → responsible_stories 목록 Story FIX loop
  # ESCALATE_PACKET_INCOMPLETE: docker-compose 미실행 or §8.6 누락

  notes: string | null
```

## FIX 루프 연동

`pl_recommendation: FIX` 시 `responsible_stories` 의 각 Story에 대해 FIX loop 진입:

| failure_type | 1차 가정 | FIX 담당 |
|---|---|---|
| `regression` | 구현 원인 (기존 코드 regression) | DeveloperPL → ArchitectPL 판정 |
| `new_test` | 구현 원인 (신규 시나리오 미구현) | DeveloperPL → ArchitectPL 판정 |
| `infra_setup` | 인프라 원인 (docker-compose 누락/오류) | InfraEngineerAgent 직접 수정 |
| `env_missing` | 환경 설정 누락 (.env 키 / 컨테이너 설정) | InfraEngineerAgent or 사용자 action |

### Baseline 실패 시 story_keys blame 절차

`suite_type: "baseline"` 이고 `story_keys: []` (빈 목록) 인 경우 3-tier 순서로 컴포넌트 경로 추출:

**Tier 1 (§8.6 related_components)**:
- 실패 테스트 `test_path` → 해당 scenario → `coverage_targets[].related_components[]` 조회
- related_components 존재 시 해당 경로 목록을 blame 대상으로 사용

**Tier 2 (static import 분석)**:
- related_components 미제공 시 실패 테스트 파일의 import 구문 정적 분석
- `from src.XXX import ...` / `import src.XXX` 패턴에서 실제 파일 경로 추출

**Tier 3 (ESCALATE)**:
- Tier 1·2 모두 실패 시 `story_keys: []`, `attribution_confidence: "unknown"` + ArchitectPL 에스컬레이션 메모

컴포넌트 경로 확정 후:
```bash
git log --oneline --follow -- <컴포넌트 경로>
```
가장 최근 변경 commit의 Story key들 → `story_keys` 목록에 추가 → `responsible_stories` union 갱신

### ESCALATE_PACKET_INCOMPLETE 처리

- `docker_compose_used: false` → InfraEngineerAgent에게 `docker-compose.test.yml` 작성 의뢰 후 재실행
- `stories_in_scope` 내 §8.6 없는 Story → TestContractArchitectAgent에게 §8.6 작성 의뢰 후 재실행
- `deployability_verified: false` (docker-compose 실행됐으나 health check 실패) → `failure_type: infra_setup` 으로 FIX 분기 (ESCALATE 아님)

## Wrapper sibling 동기화

wrapper sibling(`mclayer/plugin-codeforge:docs/inter-plugin-contracts/test-verdict-v2.md`) 이 ADR-010 wrapper-first 패턴에 따라 먼저 갱신됨. 본 파일은 canonical sync PR (CFP-373, ADR-010 §4 sibling sync policy 이행).

이전 sync: CFP-371 (v2.0 → v2.1).
