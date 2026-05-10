---
kind: contract
contract_version: "2.1"
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
supersedes: test-verdict-v1.md
carrier_story: CFP-367
date: 2026-05-10
---

# test-verdict-v2 — Integration Lane 결과 패킷 (Canonical)

**CANONICAL SSOT**: 본 파일이 원본. wrapper sibling: `mclayer/plugin-codeforge:docs/inter-plugin-contracts/test-verdict-v2.md`

## 상태

Active — CFP-367 / ADR-055 (2026-05-10)

test-verdict-v1 Archived. v1 → v2 이유: codeforge-test 통합테스트 전용 부활(ADR-048 Amendment 1)로 integration lane 전용 결과 패킷 스키마 신설.

v2 → v2.1 이유: CFP-371 / ADR-055 Amendment 2 — per-Story → Epic-level 실행 구조 전환. epic_key, stories_in_scope, responsible_stories, deployability_verified, suite_type, env_missing 필드 추가.

## 스키마

```yaml
test_verdict:
  version: "2.1"
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
      story_key: string | null  # suite_type=story → 해당 Story key 직접
                                # suite_type=baseline → blame 분석 결과 (분석 전 null 가능)
      failure_type: "regression" | "new_test" | "infra_setup" | "env_missing"
      error_summary: string     # 500자 이내

  responsible_stories: list     # FIX 대상 Story key 목록 e.g. ["CFP-NNN-S1"] (failures story_key 집계)

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

### Baseline 실패 시 story_key blame 절차

`suite_type: "baseline"` 이고 `story_key: null` 인 경우:

1. 실패 테스트의 관련 컴포넌트 경로 추출 (test_path 기반)
2. 이번 Epic의 각 Story PR merge commit에서 해당 컴포넌트 변경 이력 조회:
   ```bash
   git log --oneline --follow -- <컴포넌트 경로>
   ```
3. 가장 최근 변경 commit의 Story key → `story_key` 설정 → `responsible_stories` 추가
4. blame 불가 시 (변경 이력 없음) → ArchitectPL 에스컬레이션

### ESCALATE_PACKET_INCOMPLETE 처리

- `docker_compose_used: false` → InfraEngineerAgent에게 `docker-compose.test.yml` 작성 의뢰 후 재실행
- `stories_in_scope` 내 §8.6 없는 Story → TestContractArchitectAgent에게 §8.6 작성 의뢰 후 재실행
- `deployability_verified: false` (docker-compose 실행됐으나 health check 실패) → `failure_type: infra_setup` 으로 FIX 분기 (ESCALATE 아님)

## Wrapper sibling 동기화

wrapper sibling(`mclayer/plugin-codeforge:docs/inter-plugin-contracts/test-verdict-v2.md`) 이 ADR-010 wrapper-first 패턴에 따라 먼저 갱신됨. 본 파일은 canonical sync PR (CFP-371, ADR-010 §4 sibling sync policy 이행).
