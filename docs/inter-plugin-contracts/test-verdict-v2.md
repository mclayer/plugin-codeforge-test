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

# test-verdict-v2 — Integration Lane 결과 패킷 (Canonical)

**CANONICAL SSOT**: 본 파일이 원본. wrapper sibling: `mclayer/plugin-codeforge:docs/inter-plugin-contracts/test-verdict-v2.md`

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
    total: int                 # 전체 실행 테스트 수
    passed: int
    failed: int
    regression_baseline: int   # 이번 Story 이전 존재하던 테스트 수 (회귀 검증 대상)
    new_tests_added: int       # 이번 Story에서 추가된 신규 테스트 수
  dynamic_test_compliance: boolean  # docker-compose.test.yml 환경 사용 여부 (PASS 조건 — true 필수)
  docker_compose_used: boolean      # dynamic_test_compliance 전제 조건 — 컨테이너 실제 구동 여부
  failures:
    - test_id: string          # "{test_file}::{test_name}"
      failure_type: "regression" | "new_test" | "infra_setup"
      error_summary: string    # 한 줄 요약
  pl_recommendation: "PASS" | "FIX" | "ESCALATE_PACKET_INCOMPLETE"
```

## pl_recommendation 결정 기준

| 조건 | pl_recommendation |
|---|---|
| 모든 테스트 PASS + dynamic_test_compliance: true | PASS |
| failures 존재 (regression / new_test) | FIX |
| infra_setup 실패 (docker-compose.test.yml 구동 불가) | FIX |
| §8.6 스키마 파싱 불가 / story_key 주입 누락 | ESCALATE_PACKET_INCOMPLETE |
| §8.6 면제 Story (N/A) | PASS (suite_summary 전 필드 0, dynamic_test_compliance: false) |

## FIX 루프 라우팅 (failure_type별)

| failure_type | 1차 진단 | 최종 판정 |
|---|---|---|
| `regression` | DeveloperPL (기존 기능 파손 가능성) | ArchitectPLAgent (설계 vs 구현 분기) |
| `new_test` | DeveloperPL (신규 구현 미완성) | ArchitectPLAgent |
| `infra_setup` | InfraEngineerAgent (docker-compose.test.yml 수정) | DeveloperPL 확인 |

## Wrapper sibling 동기화

wrapper sibling(`mclayer/plugin-codeforge:docs/inter-plugin-contracts/test-verdict-v2.md`) 변경 시 본 canonical 동기화 PR 의무 (ADR-010 §4 sibling sync policy).
