---
kind: contract
contract_version: "1.1"
status: Active
related_plugins:
  - codeforge (wrapper, consumer of FIX routing data)
  - codeforge-test (lane plugin, producer + self-writer)
related_adrs:
  - ADR-008 (Inter-plugin Contract Versioning)
  - ADR-009 (Wrapper-only core + writer-distributed lane plugins)
authors:
  - CFP-38 ζ arc — Test lane extraction (2026-04-29)
  - CFP-47 — StatefulTestAgent + stateful_invariant_results (additive minor, 2026-04-30) [v1.1]
---

# test_verdict v1 — Inter-plugin Contract

`codeforge-test` plugin → `codeforge` core (Orchestrator) 단방향 schema. TestAgent 가 functional + performance subset 병렬 실행 후 self-write (phase comment + label transition) + Orchestrator 가 §10 FIX Ledger append 결정 (FAIL 시).

**상위 SSOT 위치**:
- `mclayer/plugin-codeforge-test/docs/inter-plugin-contracts/test-verdict-v1.md`: **canonical**
- `mclayer/plugin-codeforge/docs/inter-plugin-contracts/test-verdict-v1.md`: sibling reference

## 1. 흐름 개요

```
codeforge core (Orchestrator)
        │
        │ ① test_packet (subset enum, baseline path, scope globs, Story §8 Test Contract slice)
        ▼
codeforge-test plugin
  └─ TestAgent
        │
        │ ② subset 병렬 실행 (한 메시지에 dispatch):
        │   ├─ TestAgent(subset: functional) — 단위/통합/인프라
        │   └─ TestAgent(subset: performance) — baseline 비교
        │
        │ ③ Self-write (PASS 시):
        │    - mcp__github__add_issue_comment ([구현-테스트] prefix + 결과 표)
        │    - mcp__github__issue_write (phase:구현-테스트 → phase:보안-테스트 transition)
        ▼
        │ ④ test_verdict v1 typed output
        ▼
codeforge core (Orchestrator)
        │
        │ ⑤ Output 처리:
        │    - status=PASS → 보안 테스트 lane 진입
        │    - status=FAIL → §10 FIX Ledger append (Orchestrator 단독, fix-event v1 schema)
        │      → DeveloperPL/ArchitectPL 병렬 진단 (CFP-19 R4)
```

## 2. test_packet (Orchestrator → TestAgent)

```yaml
test_packet:
  contract_version: "1.0"
  story_key: <STORY_KEY>
  subsets:                          # 필수 — array, 최소 1개
    - functional                    # 단위/통합/인프라 (consumer overlay 가 러너 지정)
    - performance                   # baseline 비교 (consumer overlay 가 baseline 위치 지정)
  test_contract:                    # 필수 — Story §8 Test Contract markdown
    section: <markdown>
  consumer_overlay:                 # 필수 — runner/baseline 경로 (project.yaml 에서 도출)
    test_runner: <command>          # 예: "pytest -v"
    performance_baseline: <path>    # 예: ".perf-baseline.json"
    sequential_fallback: <bool>     # tests.performance.depends_on_functional
```

## 3. test_verdict (TestAgent → Orchestrator)

```yaml
test_verdict:
  contract_version: "1.0"
  story_key: <STORY_KEY>

  status: PASS | FAIL | ESCALATE_PACKET_INCOMPLETE

  results:                          # 필수
    functional:
      executed: <bool>
      pass_count: <int>
      fail_count: <int>
      failures:                     # array — FAIL 시 details
        - test_id: <string>
          file: <path>
          line: <int>
          message: <markdown>
    performance:
      executed: <bool>
      mean_delta_pct: <float>       # baseline 대비 % 차이 (음수 = 빨라짐)
      threshold_pct: 10             # 기본 — consumer overlay 가 변경 가능
      regression: <bool>            # mean_delta_pct > threshold_pct → true

  stateful_invariant_results:            # NEW v1.1 (optional — §8.5 N/A Story 에서는 부재, CFP-47 / ADR-015)
    long_running_passed: <int>
    long_running_invariant_drift_within_tolerance: <bool>
    restart_recovery_passed: <int>
    idempotency_replay_passed: <int>
    graceful_shutdown_passed: <int>
    time_window_drift_within_tolerance: <bool>
    duplicate_symptom_with_test_agent: <bool>  # ownership 매트릭스 — CFP-47 spec §3.3

  # Self-write 결과 audit
  writes_completed:
    phase_comment: <bool>           # [구현-테스트] prefix comment 게시
    phase_label_transitioned: <bool> # PASS 만 — phase:구현-테스트 → phase:보안-테스트

  # Orchestrator FIX 라우팅 input (FAIL 시)
  fix_routing_hint:                 # 선택 — null on PASS
    primary_failure: functional | performance
    suggested_cause:                # ArchitectPL 최종 판정 input
      - 설계                         # 성능 회귀가 baseline 자체 갱신 필요한 경우
      - 구현                         # 구현 결함이 명백한 경우
```

## 4. ESCALATE 처리

- `ESCALATE_PACKET_INCOMPLETE`: test_runner 명령어 부재, baseline 파일 부재 등 packet 불완전
- TestAgent 가 self-write 실패 (예: GitHub MCP timeout) 시 verdict 에 writes_completed=false + Orchestrator 가 fallback (DocsAgent 경유 — 단 ζ arc 후 DocsAgent 부재 시 ESCALATE)

## 5. v1 → v2 변경 가능성

- 새 subset 추가 (예: integration, e2e) — minor (v1.1, enum 추가)
- threshold_pct 정책 변경 — minor
- fix_routing_hint schema 확장 — minor

## v1.0 → v1.1 (additive minor — CFP-47, 2026-04-30)

본 v1.1 은 v1.0 대비 **additive minor** — `stateful_invariant_results` optional 필드 추가만. 기존 `test_results` 변경 없음. v1.0 consumer 가 v1.1 verdict 받아도 `stateful_invariant_results` 무시 — backward-compat.

### Schema 변경 enumeration (v1.0 → v1.1)

1. **`stateful_invariant_results` 추가** (optional) — 7 sub-field:
   - long_running_passed (int) — §8.5.1 long-running invariant test PASS count
   - long_running_invariant_drift_within_tolerance (bool) — §8.5.1 drift 허용 범위 내 여부
   - restart_recovery_passed (int) — §8.5.2 restart recovery test PASS count
   - idempotency_replay_passed (int) — §8.5.3 replay test PASS count (§11.6 active 시)
   - graceful_shutdown_passed (int) — §8.5.2 graceful shutdown test PASS count
   - time_window_drift_within_tolerance (bool) — §8.5.1 rolling window 정확성
   - duplicate_symptom_with_test_agent (bool) — TestAgent 도 같은 module fail 시 메타데이터 (CFP-47 spec §3.3 ownership matrix)

### Carrier ADR (v1.1)

- **[ADR-015 — Stateful test category](https://github.com/mclayer/plugin-codeforge/blob/main/docs/adr/ADR-015-stateful-test-category.md)** (CFP-47)

### Producer

- §8.5 N/A Story → TestAgent 만 spawn → `test_results` 만 보고 (`stateful_invariant_results` 부재 OK)
- §8.5 적용 Story → TestAgent + StatefulTestAgent 병렬 spawn → 양쪽 verdict 합쳐 `stateful_invariant_results` 채움 (Orchestrator aggregation)

## 6. 본 contract 시점 동결 ATTRIBUTION

- 동결 일시: 2026-04-29 (CFP-38)
- 협업: Claude (codification) · CFP-31 parent §5.8
- v1.1 동결: 2026-04-30 (CFP-47 — StatefulTestAgent + stateful_invariant_results, additive minor in-place)
