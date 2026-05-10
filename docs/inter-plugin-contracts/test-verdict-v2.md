---
kind: contract
contract_version: "2.0"
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
supersedes: test-verdict-v1.md
carrier_story: CFP-367
date: 2026-05-10
---

# test-verdict-v2 (Integration Test Lane 결과 패킷)

## 상태

Active — CFP-367 / ADR-055 (2026-05-10)

test-verdict-v1 Archived. v1 → v2 이유: codeforge-test 통합테스트 전용 부활(ADR-048 Amendment 1)로 integration lane 전용 결과 패킷 스키마 신설.

## 스키마

```yaml
test_verdict:
  version: "2"
  story_key: string            # "CFP-367"
  lane: "integration"          # 고정값
  executed_at: ISO8601
  runner: "IntegrationTestAgent"

  suite_summary:
    total: int                 # tests/integration/ 전체 테스트 수
    passed: int
    failed: int
    skipped: int               # §8.6 면제 Story의 external-dependency-gate 포함
    regression_baseline: int   # 이번 Story 이전 suite 누적 수
    new_tests_added: int       # 이번 Story에서 추가된 테스트 수

  dynamic_test_compliance: boolean  # 내부 컴포넌트 정적 mock 미사용 여부
  docker_compose_used: boolean      # docker-compose.test.yml 실행 여부

  failures:                    # failed > 0 인 경우에만 존재
    - test_id: string
      test_path: string        # "tests/integration/CFP-XXX/test_order_flow.py::test_bithumb_order_create"
      failure_type: "regression" | "new_test" | "infra_setup"
      error_summary: string    # 500자 이내

  pl_recommendation: "PASS" | "FIX" | "ESCALATE_PACKET_INCOMPLETE"
  # PASS: 전체 suite green
  # FIX: 실패 존재 (regression 또는 new_test 실패)
  # ESCALATE_PACKET_INCOMPLETE: docker-compose 미실행 또는 §8.6 누락

  notes: string | null
```

## FIX 루프 연동

`pl_recommendation: FIX` 시:
- `failure_type: regression` → root-cause-decision: 구현 원인 1차 가정 (기존 코드 regression)
- `failure_type: new_test` → root-cause-decision: 구현 원인 1차 가정 (신규 시나리오 미구현)
- `failure_type: infra_setup` → root-cause-decision: InfraEngineerAgent `docker-compose.test.yml` 수정 필요
