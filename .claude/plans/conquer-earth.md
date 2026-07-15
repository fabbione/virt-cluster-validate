# Plan: Conquer Earth

## Status: Active

## Overview
Transform virt-cluster-validate into a production-ready validation service shipped with OpenShift Virtualization, providing API-driven cluster validation accessible from CLI, internal APIs, and web UI.

**Note:** Detailed Kubernetes API design (CRDs, operator architecture, implementation phases) is documented in `.claude/plans/api-design.md`.

## Documentation Requirement
**Every phase must include focused, accurate documentation in docs/ directory.**
- Keep docs few and focused - no verbose AI-generated content
- Update docs as features evolve
- Document design decisions, not just usage

## End Goal
Ship as part of OCPv (OpenShift Container Platform Virtualization) to provide:
- **API-driven validation**: Start/stop/cancel tests via API
- **Multiple interfaces**:
  - External CLI (wrapper to oc commands)
  - Internal OCP API (for virt migration pre-flight checks)
  - OpenShift web UI (Virtualization → Checks)
- **Non-disruptive tests**: Minimal cluster impact
- **Scoped RBAC**: Read access cluster-wide, write/create limited to dedicated namespace and specific resources (network, CSI storage)

## TODO

### MVP: Production-Ready CLI (2-3 weeks)
**Goal:** Ship a production-ready CLI tool that can be used immediately for cluster validation, either standalone or as part of CNV.

**Scope:** Enhanced CLI with proper licensing, documentation, test validation, and basic metadata support.

- [ ] Add LICENSE file (Apache 2.0) (Priority: high)
- [x] Create CONTRIBUTING.md guide (Priority: high) - **MERGED #9**
- [x] Audit all existing tests in checks.d/ for safety (Priority: high) - **MERGED via commit eede443 - 36 new checks added**
- [x] Verify none of the current tests are destructive (Priority: high) - **MERGED via commit eede443 - all checks are read-only**
- [ ] Add basic metadata to tests (scope: cluster-wide vs node-specific, description) (Priority: high)
- [ ] Add --help or description to each test (what/why/how) (Priority: high)
- [x] Set up basic CI pipeline (linting, unit tests) (Priority: high) - **MERGED via commit 77ed3da**
- [ ] Update README.md with comprehensive usage examples (Priority: high)
- [ ] Ensure container build works for disconnected environments (Priority: high)
- [ ] Add basic test suites/groups (e.g., basic, extended) (Priority: medium)
- [ ] Validate RBAC requirements and document per test (Priority: medium)
- [ ] Create initial docs/ with architecture overview (Priority: medium)
- [ ] Test on real OCPv clusters and fix any issues (Priority: high)

**Deliverable:** A production-quality CLI tool that can be:
- Run from a laptop against any cluster (existing capability)
- Run as a container in disconnected environments (existing capability)
- Shipped as part of CNV with proper licensing and documentation
- Used as foundation for future CRD/operator work

---

## Implementation Strategy Update

**Based on OCPv infrastructure analysis** (see `../.claude/analysis/ocpv-checkup-integration.md`):

OpenShift Virtualization already ships `ocp-virt-validation-checkup` with a simple Job-based architecture (no operator, no CRDs). To maximize compatibility and adoption, virt-cluster-validate will support **dual modes**:

1. **Simple Mode (Phase 1a):** Job-based manifest generation - works without operator, compatible with OCPv patterns
2. **API Mode (Phase 1b+):** CRD + Operator - for automation, scheduling, event-driven execution

**Detailed implementation plan:** `.claude/plans/api-design.md`

**Key alignment points:**
- ConfigMap result schema compatible with ocp-virt-validation-checkup
- Ship as relatedImage in OCPv CSV (like existing checkup)
- Start with cluster-admin RBAC (same as existing), scope down later
- Support both one-off execution and API-driven automation

---

### Phase 0: Test Audit & Metadata
- [ ] Add LICENSE file (Apache 2.0) (Priority: high)
- [x] Create initial CONTRIBUTING.md guide (Priority: high) - **MERGED #9**
- [x] Add OWNERS files (Priority: high) - **MERGED #9**
- [x] Must-gather integration complete (Priority: high) - **MERGED #4, #6, #8**
  - gather script with auto-detect mode (must-gather vs normal)
  - Runtime download of oc/virtctl from cluster (no RBAC required)
  - DEBUG_TOOL support for testing
  - Human-readable results summary
  - RBAC fixes and namespace scoping
- [x] GitHub Actions lint workflow (Priority: high) - **MERGED commit 77ed3da**
  - Shellcheck validation for all bash scripts
  - Automated CI on pull requests
- [x] 36 new validation checks added (Priority: high) - **MERGED commit eede443**
  - **OpenShift platform checks (16 new):** cluster version, operators, limits, deprecated APIs, node health, cgroup version, machine config, MCP health, machineset replicas, IP stack, SR-IOV, DNS, monitoring, storage provisioners, PV status, pod restarts, PDB blocking, operator state
  - **Virtualization checks (20 new):** node memory, CPU/memory overhead, machine health checks, VM run strategy, quota resources, Windows VM config, disk expansion, HPP, storage checkup, SR-IOV operator, migration (storage/policies/network/CPU/headroom), upgrade readiness, backup
  - **Total checks now: 54** (26 OpenShift platform + 28 virtualization)
- [x] oc_cached helper (Priority: medium) - **MERGED commit eede443**
  - Caches oc command results to reduce API calls
  - Improves performance for must-gather execution
- [ ] Define test development workflow and validation process (Priority: high)
- [ ] Create docs/ directory with initial architecture overview (Priority: high)
- [x] Set up CI pipeline (linting, unit tests, integration tests, test validation) (Priority: high) - **MERGED via commit 77ed3da**
- [x] Audit all existing tests in checks.d/ (Priority: high) - **MERGED via commit eede443**
- [x] Verify none of the current tests are destructive (Priority: high) - **MERGED via commit eede443**
- [ ] Add metadata to tests (cluster-wide vs node-specific) (Priority: high)
- [ ] Define test metadata schema (scope, RBAC requirements, disruption level, description, version, class, dependencies, timeout, resource limits, parameters, suites, OCP version compatibility, topology requirements, artifacts, node selector/affinity, warmup/cooldown, external service dependencies, cacheable, architecture requirements, config dependencies) (Priority: high)
- [ ] Define test classes (basic [default], performance, load-testing, scalability) (Priority: high)
- [ ] Define resource constraints per test class (default limits by class) (Priority: medium)
- [ ] Define test suites/groups (e.g., pre-upgrade-checks, post-migration-validation) (Priority: medium)
- [ ] Define test parametrization mechanism (VM sizes, storage classes, etc.) (Priority: medium)
- [ ] Define parallel vs sequential execution controls (Priority: medium)
- [ ] Add --help or descriptive metadata to each test (what/why/how) (Priority: high)
- [ ] Define test versioning strategy (Priority: medium)
- [ ] Define test blueprint/template mechanism (Priority: low)
- [ ] Define test dependencies mechanism (execution order) (Priority: medium)
- [ ] Define multi-language test script support (Python, Go, etc. beyond bash) (Priority: low)
- [ ] Define cluster topology and zone/region awareness (SNO vs multi-node, multi-zone) (Priority: medium)
- [ ] Define OpenShift version compatibility requirements per test (Priority: medium)
- [ ] Define multi-architecture support strategy (x86_64, ARM64, s390x) (Priority: medium)
- [ ] Categorize tests by resource requirements (Priority: medium)
- [ ] Define test coverage tracking mechanism (Priority: low)
- [ ] Define test mutation testing strategy (inject failures to verify detection) (Priority: low)
- [ ] Identify tests requiring elevated permissions (Priority: medium)
- [ ] Document test metadata schema in docs/ (Priority: medium)
- [ ] Document test development workflow in docs/ (Priority: medium)

### Phase 1: API Foundation
- [ ] Design API schema for test lifecycle (start/stop/cancel/pause/resume/status/progress) (Priority: high)
- [ ] Design test selection/exclusion mechanism (by name, tag, metadata, class, suite, annotations) (Priority: high)
- [ ] Design test prioritization mechanism (queue priority for multiple runs) (Priority: medium)
- [ ] Design test result caching/memoization for identical runs (selective, per-test cacheable flag) (Priority: low)
- [ ] Design incremental testing (run only tests affected by changes) (Priority: low)
- [ ] Design test recommendation engine (suggest tests based on cluster config) (Priority: low)
- [ ] Design pre-flight checks (verify cluster meets minimum requirements) (Priority: medium)
- [ ] Design dry-run mode for test validation without execution (Priority: medium)
- [ ] Design scheduled/periodic test execution (CronJob-style) (Priority: medium)
- [ ] Design event-driven test execution (node reboot, cluster events) (Priority: medium)
- [ ] Design configuration change monitoring and test recommendations (Priority: medium)
- [ ] Design known issues/skip list mechanism with documentation (Priority: medium)
- [ ] Design export/import for test configurations and results (Priority: low)
- [ ] Design cluster labeling/tagging for result grouping (Priority: low)
- [ ] Design test run annotations (arbitrary key-value metadata) (Priority: medium)
- [ ] Design API pagination for large result sets (Priority: medium)
- [ ] Design test result search/filtering with full-text search (Priority: low)
- [ ] Design test result permalinks (stable, shareable URLs) (Priority: low)
- [ ] Design conditional test execution (beyond dependencies) (Priority: low)
- [ ] Design operator configuration validation (beyond webhooks) (Priority: medium)
- [ ] Design operator self-diagnostics and health checks (Priority: medium)
- [ ] Design test dependency graph visualization (Priority: low)
- [ ] Design canary test releases (new test versions on subset before rollout) (Priority: low)
- [ ] Design test execution budgets (time/resource limits) (Priority: low)
- [ ] Design test result compression for storage efficiency (Priority: low)
- [ ] Design operator telemetry (opt-in anonymous usage statistics) (Priority: low)
- [ ] Design test result sharing/publishing (secure external sharing) (Priority: low)
- [ ] Design chaos engineering hooks (inject failures during testing) (Priority: low)
- [ ] Design test execution priority by user role (VIP prioritization) (Priority: low)
- [ ] Design test execution warm pools (pre-warmed environments) (Priority: low)
- [ ] Design test result provenance/chain of custody (audit trail) (Priority: low)
- [ ] Design test result federation (aggregate across clusters) (Priority: low)
- [ ] Define CustomResourceDefinition (CRD) for validation runs (Priority: high)
- [ ] Define CRD for operator configuration (Priority: medium)
- [ ] Design operator configuration hot-reload (no restart required) (Priority: medium)
- [ ] Define API versioning strategy for CRD evolution (Priority: high)
- [ ] Design webhook validations for CRD (request validation) (Priority: medium)
- [ ] Design multi-tenancy model (concurrent runs, result isolation) (Priority: high)
- [ ] Design quota management (per-user/team test run limits) (Priority: medium)
- [ ] Design API rate limiting per user (Priority: medium)
- [ ] Design audit trail (who ran what tests when) (Priority: medium)
- [ ] Define namespace strategy (operator namespace, test execution namespace) (Priority: high)
- [ ] Define operator lifecycle (install/uninstall, result cleanup) (Priority: medium)
- [ ] Implement Kubernetes operator/controller (Priority: high)
- [ ] Design resource tracking mechanism (pods, services, volumes created by tests) (Priority: high)
- [ ] Design rate limiting and resource quotas for test runs (Priority: medium)
- [ ] Design per-node test execution concurrency limits (Priority: medium)
- [ ] Design maintenance mode (prevent new test runs cluster-wide) (Priority: medium)
- [ ] Design operator high availability with leader election (Priority: medium)
- [ ] Design operator upgrade/rollback strategy for in-flight runs (Priority: high)
- [ ] Design disaster recovery (operator crash, test run resumption) (Priority: high)
- [ ] Define RBAC model (cluster-wide read, namespace-scoped write) (Priority: high)
- [ ] Define test timeout strategy in operator context (Priority: medium)
- [ ] Design test timeout escalation (progressive vs hard kills) (Priority: low)
- [ ] Design test execution SLOs (service level objectives) (Priority: low)
- [ ] Document API design and CRD spec in docs/ (Priority: high)

### Phase 2: Service Layer
- [ ] Convert Python runner to Kubernetes-native service (Priority: high)
- [ ] Implement test lifecycle management (start/stop/cancel/pause/resume) (Priority: high)
- [ ] Implement resource tracking per test (pods, services, volumes) (Priority: high)
- [ ] Implement test result caching/memoization (selective, respects cacheable flag) (Priority: low)
- [ ] Implement incremental testing engine (Priority: low)
- [ ] Implement test isolation to prevent concurrent test interference (Priority: high)
- [ ] Design test sandboxing strategy (gVisor, Kata, namespace isolation) (Priority: low)
- [ ] Implement test dependency resolution and execution ordering (Priority: medium)
- [ ] Implement test prioritization in execution queue (Priority: medium)
- [ ] Implement test execution order optimization (run fast tests first) (Priority: low)
- [ ] Implement per-node concurrency limits (Priority: medium)
- [ ] Implement node selector/affinity for test placement (Priority: medium)
- [ ] Implement test execution zones/regions awareness (Priority: low)
- [ ] Implement parallel vs sequential execution controls (Priority: medium)
- [ ] Implement test parametrization (pass config to tests) (Priority: medium)
- [ ] Implement test blueprint/template instantiation (Priority: low)
- [ ] Implement test warmup/cooldown periods (Priority: low)
- [ ] Implement external service dependency checking (DNS, NTP, registries) (Priority: medium)
- [ ] Implement resource limits enforcement per test (CPU/memory) (Priority: medium)
- [ ] Implement resource constraints per test class (Priority: medium)
- [ ] Implement graceful degradation under resource constraints (Priority: medium)
- [ ] Implement cleanup on test failure/pod crash/user deletion (Priority: high)
- [ ] Design error handling and retry logic for transient failures (Priority: high)
- [ ] Implement timeout handling for long-running tests (Priority: high)
- [ ] Implement timeout escalation (progressive warnings vs hard kills) (Priority: low)
- [ ] Implement pre-flight checks before test execution (Priority: medium)
- [ ] Implement node event watcher (node reboot detection) (Priority: medium)
- [ ] Implement automatic per-node test execution on node reboot (Priority: medium)
- [ ] Implement configuration change watchers (ConfigMaps, CRs, etc.) (Priority: medium)
- [ ] Implement test recommendations on config changes (Priority: medium)
- [ ] Design result format (human-readable + machine-parsable JSON) (Priority: high)
- [ ] Design unique identifiers (cluster ID, test run ID, node ID) (Priority: high)
- [ ] Design version metadata capture (OCP, operators, kernel, etc.) per test result (Priority: high)
- [ ] Design cluster configuration capture (topology, network, storage) (Priority: high)
- [ ] Design hardware setup capture (CPU, memory, NICs, storage devices) (Priority: medium)
- [ ] Design AI-consumable data format for known issue tracking (Priority: medium)
- [ ] Design test output artifacts storage and access (logs, dumps, screenshots) (Priority: medium)
- [ ] Design test output size limits and overflow handling (Priority: medium)
- [ ] Design test result signing/verification for compliance (Priority: low)
- [ ] Add result persistence (CRD status, events) (Priority: medium)
- [ ] Define result archival and retention policy (Priority: medium)
- [ ] Define result auto-expiration strategy (Priority: medium)
- [ ] Define result retention based on outcome (keep failures longer than successes) (Priority: low)
- [ ] Define test result deduplication strategy (repeated failures) (Priority: low)
- [ ] Define data retention compliance requirements (GDPR, privacy) (Priority: medium)
- [ ] Design backup/restore strategy for test results (Priority: low)
- [ ] Implement progress reporting via API (Priority: high)
- [ ] Implement test result streaming (real-time results as tests execute) (Priority: medium)
- [ ] Implement cluster labeling/tagging system (Priority: low)
- [ ] Implement maintenance mode (Priority: medium)
- [ ] Implement operator high availability with leader election (Priority: medium)
- [ ] Implement test run annotations system (Priority: medium)
- [ ] Implement test result deduplication for repeated failures (Priority: low)
- [ ] Implement outcome-based retention (failures kept longer) (Priority: low)
- [ ] Ensure results available in both human and JSON formats (Priority: high)
- [ ] Implement result comparison/diff between runs (Priority: low)
- [ ] Implement test result aggregation (roll-up views across multiple runs) (Priority: low)
- [ ] Implement test result streaming to external analytics platforms (Priority: low)
- [ ] Implement generic webhook notifications for test events (Priority: low)
- [ ] Add observability (metrics, logging, tracing) (Priority: high)
- [ ] Define operator-specific Prometheus metrics (queue depth, test duration percentiles, etc.) (Priority: medium)
- [ ] Add health checks (liveness/readiness probes, health endpoints) (Priority: high)
- [ ] Implement operator configuration hot-reload (Priority: medium)
- [ ] Implement operator configuration validation (Priority: medium)
- [ ] Implement operator self-diagnostics (Priority: medium)
- [ ] Implement API pagination for large result sets (Priority: medium)
- [ ] Implement multi-architecture detection and compatibility checking (Priority: medium)
- [ ] Track test execution history per cluster (Priority: medium)
- [ ] Track test execution history per node (Priority: medium)
- [ ] Implement historical trending and degradation analysis with version correlation (Priority: medium)
- [ ] Implement version comparison across test runs (OCP, operators, kernel) (Priority: medium)
- [ ] Implement test stability tracking (flaky test detection) (Priority: medium)
- [ ] Implement test flakiness scoring (0-100 based on history) (Priority: low)
- [ ] Implement test execution time estimates (based on historical data) (Priority: low)
- [ ] Implement test coverage tracking visualization (Priority: low)
- [ ] Implement test result search/filtering with full-text search (Priority: low)
- [ ] Implement test result permalinks (Priority: low)
- [ ] Implement post-execution annotations (add comments/notes after run) (Priority: low)
- [ ] Implement conditional test execution (beyond dependencies) (Priority: low)
- [ ] Implement test result expiry warnings (before auto-deletion) (Priority: low)
- [ ] Implement test execution replay (re-run with identical parameters) (Priority: low)
- [ ] Implement cluster resource recommendations for test execution (Priority: low)
- [ ] Implement test result visualization dashboards (built-in charts/graphs) (Priority: low)
- [ ] Implement test dependency graph visualization (Priority: low)
- [ ] Implement canary test releases (Priority: low)
- [ ] Implement test mutation testing (Priority: low)
- [ ] Implement test result compression (Priority: low)
- [ ] Implement test execution budgets (time/resource limits) (Priority: low)
- [ ] Implement test result visual diffing (Priority: low)
- [ ] Implement operator telemetry (opt-in) (Priority: low)
- [ ] Implement test result sharing/publishing (Priority: low)
- [ ] Implement test execution tracing (distributed traces) (Priority: low)
- [ ] Implement test result correlation analysis (find patterns across failures) (Priority: low)
- [ ] Implement chaos engineering hooks (Priority: low)
- [ ] Implement test result forecasting/prediction (ML-based) (Priority: low)
- [ ] Implement test execution priority by user role (Priority: low)
- [ ] Implement test result anomaly detection (Priority: low)
- [ ] Implement test execution warm pools (Priority: low)
- [ ] Implement test result provenance/chain of custody (Priority: low)
- [ ] Implement test result change detection (behavior changes) (Priority: low)
- [ ] Implement multi-language test script support (Python, Go, etc.) (Priority: low)
- [ ] Implement test result federation (aggregate across clusters) (Priority: low)
- [ ] Implement alerts for new tests that haven't been executed (Priority: medium)
- [ ] Implement reporting/notification system for test failures (Priority: medium)
- [ ] Implement cluster topology detection (SNO vs multi-node, zones/regions) (Priority: medium)
- [ ] Implement OpenShift version detection and compatibility checking (Priority: medium)
- [ ] Implement certification/compliance reporting (Priority: low)
- [ ] Define image management strategy (versioning, pull policies) (Priority: medium)
- [ ] Define network policies for operator and test pods (Priority: medium)
- [ ] Performance and scalability testing (concurrent run limits) (Priority: medium)
- [ ] Create pre-built Grafana dashboards for operator metrics (Priority: low)
- [ ] Document service architecture and result format in docs/ (Priority: medium)

### Phase 3: Interfaces
- [ ] Create oc plugin wrapper for CLI access with test selection/exclusion (Priority: medium)
- [ ] CLI support for test class selection (basic [default], performance, load-testing, scalability) (Priority: medium)
- [ ] CLI support for test suite selection (Priority: medium)
- [ ] CLI support for test parametrization (Priority: medium)
- [ ] CLI support for test pause/resume (Priority: low)
- [ ] CLI support for dry-run mode (Priority: medium)
- [ ] CLI support for viewing test recommendations (Priority: low)
- [ ] CLI support for export/import of configurations and results (Priority: low)
- [ ] CLI support for cluster tagging and test run annotations (Priority: low)
- [ ] CLI support for result aggregation views (Priority: low)
- [ ] CLI support for enabling/disabling maintenance mode (Priority: low)
- [ ] CLI support for test result search/filtering (Priority: low)
- [ ] CLI support for viewing test execution time estimates (Priority: low)
- [ ] CLI support for test dependency graph visualization (Priority: low)
- [ ] CLI support for test result visual diffing (Priority: low)
- [ ] CLI auto-detection: check if cluster has CRD installed, offer local vs remote execution (Priority: medium)
- [ ] Design cleanup mechanism for CLI mode (laptop crash/interrupt recovery) (Priority: high)
- [ ] Design internal API for virt migration integration (Priority: medium)
- [ ] Design pre/post migration hooks for virt migration (Priority: medium)
- [ ] Design notification/alerting integration (Prometheus, PagerDuty, etc.) (Priority: low)
- [ ] Design localization/i18n strategy for error messages and reports (Priority: low)
- [ ] Define OpenShift Console plugin API contract with test filtering (Priority: low)
- [ ] Implement Console UI integration (Virtualization → Checks) (Priority: low)
- [ ] Document CLI usage and API integration in docs/ (Priority: medium)

### Phase 4: RBAC & Security
- [ ] Define minimal ClusterRole for read operations (Priority: high)
- [ ] Define namespace-scoped Role for test execution (Priority: high)
- [ ] Define RBAC for operator itself (Priority: high)
- [ ] Implement admission webhooks for CRD validation (Priority: medium)
- [ ] Implement test result signing/verification (Priority: low)
- [ ] Implement data retention compliance (GDPR, privacy requirements) (Priority: medium)
- [ ] Document required permissions per check type (Priority: medium)
- [ ] Security review and threat modeling (Priority: high)
- [ ] Implement rate limiting enforcement (cluster-wide) (Priority: medium)
- [ ] Implement per-user API rate limiting (Priority: medium)
- [ ] Implement quota enforcement (per-user/team test run limits) (Priority: medium)
- [ ] Air-gapped/disconnected environment considerations for operator (Priority: medium)
- [ ] Document RBAC model and security considerations in docs/ (Priority: high)

### Phase 5: Production Readiness
- [ ] Container registry and distribution strategy (Priority: medium)
- [ ] Integration with OCPv release process (Priority: medium)
- [ ] Operator lifecycle testing (install/uninstall/upgrade/rollback) (Priority: high)
- [ ] Operator high availability testing (leader election, failover) (Priority: medium)
- [ ] Operator upgrade/rollback testing with in-flight runs (Priority: high)
- [ ] Operator configuration hot-reload testing (Priority: medium)
- [ ] Maintenance mode testing (Priority: low)
- [ ] Per-node concurrency limits testing (Priority: medium)
- [ ] Test operator uninstall with result cleanup verification (Priority: medium)
- [ ] Maintain local laptop container for external cluster testing (CLI only, no API/WebUI) (Priority: medium)
- [ ] Air-gapped deployment testing and documentation (Priority: medium)
- [ ] Investigate kcli with nested virtualization for local dev/testing (Priority: low)
- [ ] Load testing and scalability validation (Priority: high)
- [ ] Test execution SLO validation and monitoring (Priority: low)
- [ ] Test execution order optimization testing (Priority: low)
- [ ] Test result caching/memoization validation (Priority: low)
- [ ] Test result deduplication testing (Priority: low)
- [ ] Outcome-based retention policy testing (Priority: low)
- [ ] Data retention compliance testing (Priority: medium)
- [ ] Zone/region awareness testing (Priority: low)
- [ ] Incremental testing validation (Priority: low)
- [ ] Test pause/resume functionality testing (Priority: low)
- [ ] Disaster recovery testing (operator crash scenarios) (Priority: high)
- [ ] Graceful degradation testing under resource pressure (Priority: medium)
- [ ] Test sandboxing verification (if implemented) (Priority: low)
- [ ] Test blueprint/template system testing (Priority: low)
- [ ] Custom test runner framework testing (Priority: low)
- [ ] External service dependency checking testing (Priority: medium)
- [ ] Result auto-expiration testing (Priority: low)
- [ ] Result expiry warning testing (Priority: low)
- [ ] Result aggregation testing (Priority: low)
- [ ] Multi-architecture support testing (Priority: medium)
- [ ] API pagination testing (Priority: medium)
- [ ] Test result search/filtering testing (Priority: low)
- [ ] Test flakiness scoring testing (Priority: low)
- [ ] Test execution time estimates testing (Priority: low)
- [ ] Test coverage tracking testing (Priority: low)
- [ ] Post-execution annotations testing (Priority: low)
- [ ] Conditional test execution testing (Priority: low)
- [ ] Test execution replay testing (Priority: low)
- [ ] Cluster resource recommendations testing (Priority: low)
- [ ] Operator self-diagnostics testing (Priority: medium)
- [ ] Operator configuration validation testing (Priority: medium)
- [ ] Grafana dashboard validation (Priority: low)
- [ ] Test result visualization dashboards testing (Priority: low)
- [ ] Test dependency graph visualization testing (Priority: low)
- [ ] Canary test releases testing (Priority: low)
- [ ] Test mutation testing validation (Priority: low)
- [ ] Test result compression testing (Priority: low)
- [ ] Test execution budgets testing (Priority: low)
- [ ] Test result visual diffing testing (Priority: low)
- [ ] Operator telemetry testing (Priority: low)
- [ ] Test result sharing/publishing testing (Priority: low)
- [ ] Test execution tracing testing (Priority: low)
- [ ] Test result correlation analysis testing (Priority: low)
- [ ] Chaos engineering hooks testing (Priority: low)
- [ ] Test result forecasting/prediction testing (Priority: low)
- [ ] Test execution priority by role testing (Priority: low)
- [ ] Test result anomaly detection testing (Priority: low)
- [ ] Test execution warm pools testing (Priority: low)
- [ ] Test result provenance testing (Priority: low)
- [ ] Test result change detection testing (Priority: low)
- [ ] Multi-language test script testing (Priority: low)
- [ ] Test result federation testing (Priority: low)
- [ ] Configuration change watchers testing (Priority: medium)
- [ ] Test recommendations on config changes testing (Priority: medium)
- [ ] Webhook notification testing (Priority: low)
- [ ] Result streaming testing (real-time and to external platforms) (Priority: low)
- [ ] Export/import functionality testing (Priority: low)
- [ ] Localization/i18n testing (if implemented) (Priority: low)
- [ ] Documentation for cluster admins in docs/ (Priority: medium)
- [ ] End-to-end testing in real clusters (Priority: high)
- [ ] Review and finalize all docs/ content for accuracy (Priority: high)

## In Progress
<!-- Tasks currently being worked on -->

## Done
<!-- Completed tasks -->

## Evaluated but Not Implementing
<!-- Features that were considered but decided against or deferred indefinitely -->
<!-- Move items here with reasoning when they're rejected during design/implementation -->
<!-- Format: - Feature name - Reason for rejection -->

## Notes
- Current implementation: CLI tool with Python runner and bash checks (already supports container via Containerfile, has --select flag)
- Target: Kubernetes-native service with CRD-based API
- Scope: Tool is local to a given cluster (no multi-cluster, no cost tracking)
- Design phase: Many features still need design decisions
- Key constraint: Non-disruptive testing in production clusters
- Integration point: Virt migration needs pre-flight validation before VM conversion
- Result formats: Human-readable (CLI/WebUI) + JSON (internal API sharing)
- CI/CD required for quality gates before merging
- Critical: Resource tracking required for cleanup on failure/crash/deletion
- API must support progress polling for long-running test suites
- Important: Maintain local container support for clusters that cannot update OCPv (CLI only, no API/WebUI)
- All interfaces (CLI, API, WebUI) must support test selection/exclusion
- CLI/oc wrapper must handle cleanup on laptop crash/interrupt (design TBD)
- Multi-tenancy: Support concurrent test runs from multiple users/teams with isolated results
- Observability: Metrics, logging, and tracing required for production service
- Backward compatibility: Not a concern - new project, CLI breaking changes acceptable during development
- CLI can auto-detect if cluster has CRD installed and offer local vs remote execution choice
- Alert mechanism: Notify when new version introduces tests that haven't been executed on the cluster
- Test versioning: Track test versions to handle changes over time
- Error handling: Retry logic for transient cluster issues
- Result retention: Define archival and cleanup policy for old test results
- Test classes: basic (default), performance, load-testing, scalability - user selectable
- Test suites: Named groups (e.g., pre-upgrade-checks, post-migration-validation)
- Test parametrization: Tests can accept configuration (VM sizes, storage classes, etc.)
- Test dependencies: Some tests may need specific execution order
- Parallel vs sequential: Explicit control over execution concurrency
- Resource limits: Individual tests should have CPU/memory constraints
- Resource constraints per class: Different defaults for different test classes
- Timeout handling: Tests need configurable timeouts
- Operator resilience: Handle upgrades/crashes without losing in-flight runs
- Configuration: Operator config via CRD, namespace strategy defined
- Health checks: Operator and service need proper probes
- Image management: Versioning and pull policies for test containers
- Network policies: Define for both operator and test pods
- Scalability: Performance test concurrent run limits
- Dry-run mode: Validate test selection without execution
- Notification integration: Failed tests should trigger alerts (Prometheus, etc.)
- Air-gapped support: Both CLI container and operator must work disconnected
- Test development: Clear workflow for adding/validating new tests
- Scheduled tests: CronJob-style periodic execution
- Event-driven tests: Auto-run per-node tests on node reboot to detect regressions
- Configuration change monitoring: Watch ConfigMaps/CRs and recommend re-running affected tests when configs change
- Historical trending: Analysis of results over time to identify degradation, correlated with versions
- Version tracking: Capture OCP, operator, kernel versions with each test result for trend analysis
- Unique identifiers: Cluster ID, test run ID, node ID for tracking and correlation
- Comprehensive data capture: Cluster config, hardware setup for AI-driven known issue analysis
- AI-consumable format: Machine-parsable output designed for feeding to AI analysis systems
- Test artifacts: Storage and access for test-generated outputs (logs, dumps)
- Result comparison: Ability to diff results between runs
- Known issues: Skip list for known-failing tests with documentation
- Flaky test tracking: Detect and track unstable tests
- API versioning: Strategy for CRD evolution
- Audit trail: Track who ran what tests when (compliance)
- Backup/restore: Test results in cluster backup procedures
- Migration hooks: Pre/post hooks for virt migration integration
- Topology awareness: Tests aware of cluster topology (SNO vs multi-node, zones)
- OCP version compatibility: Tests specify required OpenShift versions
- Test recommendations: Suggest tests based on cluster configuration
- Certification/compliance: Reporting for cluster certification requirements
- Test prioritization: Queue priority mechanism for multiple concurrent runs
- Quota management: Per-user/team limits on test runs
- API rate limiting per user: Individual rate limits beyond cluster-wide
- Graceful degradation: Tests adapt or fail gracefully under resource constraints
- Test sandboxing: Advanced isolation (gVisor, Kata) beyond namespace isolation
- Operator lifecycle: Installation, uninstallation, cleanup strategy
- Test output size limits: Limits and overflow handling for large artifacts
- Result auto-expiration: Automatic deletion of old results based on policy
- Test prioritization in queue: Order of execution when multiple runs queued
- Pre-flight checks: Verify cluster requirements before test execution
- Test timeout escalation: Progressive warnings before hard termination
- Test execution SLOs: Service level objectives for test run times
- Node selector/affinity: Tests specify which nodes they can run on
- Export/import: Configuration and result portability
- Localization/i18n: Multi-language support for messages and reports
- Result signing/verification: Cryptographic signing for compliance scenarios
- Test result caching: Selective memoization for cacheable tests (not all tests can be cached - e.g., external network/storage tests)
- Test pause/resume: Ability to pause and resume long-running tests
- Incremental testing: Run only tests affected by recent changes
- Test blueprints/templates: Predefined test configurations for instantiation
- Cluster labeling/tagging: Tag clusters for grouping results (dev, staging, prod)
- Test run annotations: Arbitrary key-value metadata on test runs
- Result streaming: Real-time test results as execution progresses
- Result aggregation: Roll-up views across multiple test runs
- Webhook notifications: Generic webhooks for test events (beyond specific integrations)
- Custom test runners: Support for execution frameworks beyond bash
- Test warmup/cooldown: Pre/post execution periods for specific tests
- External service dependencies: Check and validate DNS, NTP, registry availability
- Operator metrics: Prometheus metrics for queue depth, test duration percentiles, operator health
- Operator config hot-reload: Update operator configuration without restart
- Result streaming to external platforms: Push results to external analytics systems
- Per-node concurrency limits: Limit test execution per node to avoid overwhelming individual nodes
- Maintenance mode: Cluster-wide mode to prevent new test runs (for maintenance windows)
- Operator high availability: Leader election and failover for operator resilience
- Test execution order optimization: Run fast tests first to provide quick feedback
- Test result deduplication: Avoid storing duplicate results for repeatedly failing tests
- Outcome-based retention: Keep failure results longer than success results
- Data retention compliance: GDPR and privacy requirements for test result storage
- Zone/region awareness: Tests aware of and can target specific zones in multi-zone clusters
- Multi-architecture support: Tests aware of cluster architecture (x86_64, ARM64, s390x)
- API pagination: Handle thousands of test runs/results efficiently
- Test result search: Full-text search across test results and logs
- Test execution time estimates: Predict duration based on historical data
- Test flakiness scoring: Numeric score (0-100) based on test stability history
- Test coverage tracking: Visualize which cluster components have been validated
- Test result permalinks: Stable, shareable URLs for specific test results
- Post-execution annotations: Add comments/notes to results after run completes
- Conditional test execution: Advanced logic beyond basic dependencies
- Test result expiry warnings: Notify before auto-deletion
- Operator self-diagnostics: Built-in operator health checks and self-tests
- Test execution replay: Re-run exact test with identical parameters for debugging
- Cluster resource recommendations: Advise on resources needed for test execution
- Pre-built Grafana dashboards: Ready-to-import dashboards for operator metrics
- Operator configuration validation: Validate config before applying (beyond webhooks)
- Test result visualization dashboards: Built-in charts/graphs of results over time
- Test dependency graph visualization: Visual DAG of test dependencies
- Canary test releases: Roll out new test versions gradually to subset of runs
- Test mutation testing: Inject failures to verify tests detect problems correctly
- Test result compression: Compress stored results for storage efficiency
- Test execution budgets: Stop execution after time/resource limits
- Test result visual diffing: Show exact differences between test outputs
- Operator telemetry: Anonymous opt-in usage statistics
- Test result sharing/publishing: Securely share results with external parties
- Test execution tracing: Distributed traces for debugging slow tests
- Test result correlation analysis: Find patterns across multiple failing tests
- Chaos engineering hooks: Deliberately inject failures during test execution
- Test result forecasting/prediction: ML-based prediction of likely failures
- Test execution priority by user role: VIP users get higher priority
- Test result anomaly detection: Alert when results deviate from normal patterns
- Test execution warm pools: Pre-warmed test environments for instant starts
- Test result provenance/chain of custody: Complete audit trail for compliance
- Test result change detection: Detect when test behavior changes unexpectedly
- Multi-language test scripts: Support Python, Go, etc. beyond bash
- Test result federation: Aggregate results across multiple clusters

---

**Note:** This plan has reached truly ridiculous levels of comprehensiveness! It covers everything from basic test execution to ML-based predictions, chaos engineering, multi-cluster federation, and compliance chain-of-custody tracking. This plan could probably guide development for the next 5 years! 🚀🎉

If we've missed anything at this point, it probably doesn't exist yet. This plan is ready to conquer not just Earth, but probably a few neighboring planets too!
