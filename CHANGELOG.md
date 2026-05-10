# Changelog

`codeforge-test` plugin 릴리스 이력.

버전 체계: [Semantic Versioning 2.0.0](https://semver.org/lang/ko/). v1.0 이전은 minor bump도 breaking 가능.

## [1.0.0] - 2026-05-10

### CFP-367 / ADR-055 — IntegrationTestAgent 도입 (통합테스트 lane 전용 부활)

ADR-048로 deprecated된 codeforge-test plugin을 통합테스트 lane 전용으로 부활. MAJOR bump = 기존 TestAgent/StatefulTestAgent deprecated + IntegrationTestAgent 신규 도입.

### Added

- `agents/IntegrationTestAgent.md` — Sonnet tier, §8.6 Integration Test Contract 이행, docker-compose.test.yml 동적 실행, 전체 suite regression 검증, test-verdict-v2 생성
- `docs/inter-plugin-contracts/test-verdict-v2.md` — canonical contract (lane: integration, suite_summary, dynamic_test_compliance, §8.6 N/A 면제 패킷 포함)

### Changed

- `CLAUDE.md` — DEPRECATED → REVIVED (ADR-055 / ADR-048 Amendment 1); 통합테스트 lane 동작 문서화; 기존 TestAgent/StatefulTestAgent 섹션에 deprecated 배너 추가

### Deprecated

- `agents/TestAgent.md`, `agents/StatefulTestAgent.md` — CFP-317 / ADR-048로 deprecated 유지 (파일 보존, spawn 불가)
- `docs/inter-plugin-contracts/test-verdict-v1.md` — Archived (superseded by test-verdict-v2)

## [DEPRECATED] - 2026-05-09

### CFP-317 — CI-native 테스트 전환으로 인한 deprecated 선언

TestAgent 및 StatefulTestAgent spawn 폐지. GitHub Actions CI가 구현 테스트 실행을 담당하게 되어 별도 test lane plugin이 불필요해짐.

- QADeveloperAgent (codeforge-develop plugin)가 `.github/workflows/test.yml` 작성 의무 추가
- Orchestrator가 `gh pr checks` polling으로 CI 결과 직접 처리
- `test_verdict v1` contract Archived
- 본 plugin은 역사적 참조용으로 보존 (ADR-023 lifecycle — 삭제 아님)

관련: [ADR-048](https://github.com/mclayer/plugin-codeforge/blob/main/docs/adr/ADR-048-ci-native-test-execution.md)

## [0.1.0] - 2026-04-29

### CFP-38 (codeforge ζ arc) — Initial extraction (NEW)

codeforge ζ arc 네 번째 lane plugin (parent spec mclayer/plugin-codeforge CFP-31 §5.8). 가장 단순한 lane (TestAgent 1개 + owner doc 부재).

### Added

- `agents/TestAgent.md` — codeforge wrapper 에서 이전. self-write 권한 추가 (mcp__github__add_issue_comment, mcp__github__issue_write — phase comment + phase 전환)
- `docs/inter-plugin-contracts/test-verdict-v1.md` — canonical contract
- `overlay/hooks/{regen-agents,session-start-deps-check}.sh`
- README + CLAUDE.md

### Why

CFP-31 §5.8: TestAgent 1개 + owner doc 부재로 가장 단순한 lane 추출. Codex round 2 권고 sequencing (Sequence #4) 따름. 이전 3 plugin (review v2 + pmo + requirements) 검증 후 진입.

### Compatibility

- **Wire**: codeforge >= 3.0.0
- **Migration**: Story §10 FIX Ledger append 는 그대로 Orchestrator 단독. lane plugin 은 fix_routing_hint 만 verdict 에 첨부 (FAIL 시)
