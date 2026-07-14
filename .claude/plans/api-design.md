# HCO Integration Design for virt-cluster-validate

**Related Documents:**
- **Project Roadmap:** `.claude/plans/conquer-earth.md` - Overall project vision, features, and phases
- **OCPv Analysis:** `../.claude/analysis/ocpv-checkup-integration.md` - Analysis of existing ocp-virt-validation-checkup
- **HCO Patterns:** `../.claude/analysis/hco-integration-patterns.md` - HCO operand handler patterns

## Purpose of This Document

This document focuses **exclusively** on the HCO (hyperconverged-cluster-operator) integration implementation. For overall project features, roadmap, and long-term vision, see `conquer-earth.md`.

**Scope:**
- HCO operand handler implementation (handler code goes in HCO repo)
- Annotation-based API design
- ConfigMap writer implementation (in virt-cluster-validate repo)
- MVP implementation tasks and timeline

## Context

After analyzing HCO architecture and existing OCPv patterns (ocp-virt-validation-checkup), the optimal approach is to **integrate virt-cluster-validate into HCO** rather than building a standalone operator.

**Key findings:**
- OCPv already has `ocp-virt-validation-checkup` using simple Job-based execution
- HCO manages auxiliary components (AIE webhook, wasp-agent) via operand handlers
- HCO uses **annotations** (not CRD spec fields) for optional component configuration
- All auxiliary components ship as relatedImages in OCPv CSV

## Architecture Decision: HCO Integration

**Core approach:** Add operand handlers to HCO that create validation Jobs triggered by annotations.

### How Other Projects Integrate with HCO

**This is the standard pattern** - all auxiliary components patch HCO directly:

**AIE Webhook Example:**
- **Handler code:** `hyperconverged-cluster-operator/controllers/handlers/aie/*.go` (in HCO repo)
- **Service image:** `quay.io/kubevirt/kubevirt-aie-webhook` (separate repo builds image)
- **HCO creates:** Deployment running the image
- **Triggered by:** Annotation `hco.kubevirt.io/deployAIE: "true"`

**Wasp Agent Example:**
- **Handler code:** `hyperconverged-cluster-operator/controllers/handlers/wasp-agent/*.go` (in HCO repo)
- **Service image:** `quay.io/openshift-virtualization/wasp-agent` (separate repo builds image)
- **HCO creates:** DaemonSet running the image
- **Triggered by:** Annotation logic in handler

**Pattern Summary:**
1. Service project builds container image
2. Handler code lives IN HCO repository
3. HCO handlers create K8s resources (Deployment/DaemonSet/Job) running the image
4. Configuration via annotations on HyperConverged CR

**For virt-cluster-validate:**
- **Our repo:** Builds `quay.io/kubevirt/virt-cluster-validate` image
- **HCO repo:** Gets handlers in `controllers/handlers/cluster-validation/*.go`
- **HCO creates:** Job running our image
- **Triggered by:** Annotation `validation.kubevirt.io/run: "<timestamp>"`

### Why HCO Integration (Standard Pattern)

**Benefits:**
- ✅ Ships automatically with OpenShift Virtualization
- ✅ No separate operator to install/manage
- ✅ Follows proven pattern used by all auxiliary components
- ✅ Uses existing `kubevirt-hyperconverged` namespace
- ✅ Lifecycle managed by HCO
- ✅ Standard approach - not special-casing

**Alternative (NOT recommended):**
- ❌ Standalone operator would be unusual (no other components do this)
- ❌ Separate installation burden
- ❌ Additional CRDs to manage
- ❌ More complex for users

## API Design Principles

### Extensibility First

**Critical design principle:** The API must support future operations (cancel, pause, progress monitoring, multiple concurrent runs) even if not implemented in MVP. This prevents breaking changes when adding features.

**Key decisions for extensibility:**
1. **Structured Run IDs** - Enable tracking specific runs (for cancel, status, etc.)
2. **Multiple concurrent runs** - Handler supports multiple Jobs simultaneously
3. **Reserved annotations** - Document future operations upfront
4. **Versioned schema** - ConfigMap includes version for evolution
5. **Comprehensive labels** - Enable efficient querying and cleanup

### Annotation-Based Configuration

**NO CRD changes required** - uses annotations like AIE webhook and wasp-agent.

**Annotations (MVP - Implemented):**
```go
const (
    // Trigger validation job - value is structured run ID
    // Format: "val-<timestamp>-<shortid>" for uniqueness
    // Example: "val-1720623456-a3f2"
    // Multiple runs can coexist by using different run IDs
    ClusterValidationRunAnnotation = "validation.kubevirt.io/run"
    
    // Test selection patterns (comma-separated globs) - optional
    ClusterValidationIncludePatternsAnnotation = "validation.kubevirt.io/include-patterns"
    ClusterValidationExcludePatternsAnnotation = "validation.kubevirt.io/exclude-patterns"
)
```

**Annotations (Reserved for Post-MVP - Documented but Not Implemented):**
```go
const (
    // Cancel specific validation run by run ID
    // Example: validation.kubevirt.io/cancel: "val-1720623456-a3f2"
    ClusterValidationCancelAnnotation = "validation.kubevirt.io/cancel"
    
    // Pause specific validation run by run ID
    ClusterValidationPauseAnnotation = "validation.kubevirt.io/pause"
    
    // Resume specific validation run by run ID
    ClusterValidationResumeAnnotation = "validation.kubevirt.io/resume"
    
    // Structured parameters (JSON) - concurrency, timeout, etc.
    // Example: validation.kubevirt.io/params: '{"concurrency": 4, "timeout": 300}'
    ClusterValidationParamsAnnotation = "validation.kubevirt.io/params"
    
    // Cron schedule for periodic validation (creates CronJob)
    ClusterValidationScheduleAnnotation = "validation.kubevirt.io/schedule"
    
    // Trigger validation on HCO upgrades (adds to handleUpgrade())
    ClusterValidationRunOnUpgradeAnnotation = "validation.kubevirt.io/run-on-upgrade"
)
```

### Usage Examples

**Trigger validation (MVP):**
```bash
# Generate structured run ID
RUN_ID="val-$(date +%s)-$(uuidgen | cut -d- -f1)"

# Trigger validation
oc annotate hyperconverged kubevirt-hyperconverged \
  -n kubevirt-hyperconverged \
  validation.kubevirt.io/run="$RUN_ID" --overwrite

# Returns: val-1720623456-a3f2
```

**With test selection:**
```bash
oc annotate hyperconverged kubevirt-hyperconverged \
  -n kubevirt-hyperconverged \
  validation.kubevirt.io/run="val-$(date +%s)-$(uuidgen | cut -d- -f1)" \
  validation.kubevirt.io/exclude-patterns="checks.d/*/80-performance.d/*" \
  --overwrite
```

**Multiple concurrent runs:**
```bash
# Start run 1
oc annotate hyperconverged kubevirt-hyperconverged \
  validation.kubevirt.io/run="val-001" --overwrite

# Start run 2 (concurrent with run 1)
oc annotate hyperconverged kubevirt-hyperconverged \
  validation.kubevirt.io/run="val-002" --overwrite

# Both Jobs run concurrently, each with unique run ID
```

**Check job status:**
```bash
# List all validation jobs
oc get jobs -n kubevirt-hyperconverged -l validation.kubevirt.io/component=cluster-validation

# Check specific run
RUN_ID="val-1720623456-a3f2"
oc get job -n kubevirt-hyperconverged virt-cluster-validate-$RUN_ID

# Watch logs
oc logs -n kubevirt-hyperconverged job/virt-cluster-validate-$RUN_ID -f
```

**Check results:**
```bash
# List all result ConfigMaps
oc get configmap -n kubevirt-hyperconverged \
  -l validation.kubevirt.io/component=cluster-validation \
  --sort-by=.metadata.creationTimestamp

# Get specific run results
oc get configmap -n kubevirt-hyperconverged \
  virt-cluster-validate-results-$RUN_ID -o yaml

# Query by run ID label
oc get configmap -n kubevirt-hyperconverged \
  -l validation.kubevirt.io/run-id=$RUN_ID -o yaml
```

**Future (Post-MVP) - Cancel run:**
```bash
# Not implemented in MVP, but API reserved
oc annotate hyperconverged kubevirt-hyperconverged \
  validation.kubevirt.io/cancel="val-1720623456-a3f2" --overwrite
```

### Result Storage (Versioned Schema)

Jobs write results to ConfigMap in `kubevirt-hyperconverged` namespace with **versioned schema** to enable future evolution.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: virt-cluster-validate-results-val-1720623456-a3f2
  namespace: kubevirt-hyperconverged
  labels:
    # Enables efficient querying
    validation.kubevirt.io/run-id: "val-1720623456-a3f2"
    validation.kubevirt.io/component: "cluster-validation"
    validation.kubevirt.io/schema-version: "v1alpha1"
    validation.kubevirt.io/status: "completed"  # completed, failed, cancelled
  ownerReferences:
    - apiVersion: batch/v1
      kind: Job
      name: virt-cluster-validate-val-1720623456-a3f2
      uid: ...
data:
  # Schema version for evolution (REQUIRED in MVP)
  schema_version: "v1alpha1"
  
  # Run identification
  run_id: "val-1720623456-a3f2"
  
  # OCPv-compatible schema (matches ocp-virt-validation-checkup)
  tests_run: "18"
  tests_passed: "15"
  tests_failures: "2"
  tests_warned: "1"
  tests_skipped: "0"
  tests_duration: "2m45s"
  
  # Test lists
  failed_tests: |
    checks.d/50-openshift-virtualization.d/50-basic.d/test.sh
  warned_tests: |
    checks.d/50-openshift-virtualization.d/60-storage.d/test.sh
  
  # Full JSON for programmatic access
  results_json: |
    {"summary": "Passed: 15, Failed: 2, Total: 18", "results": [...]}
```

**Schema Evolution Strategy:**
- **v1alpha1 (MVP):** Current schema above
- **v1alpha2 (Future):** Could add: `test_categories`, `execution_environment`, `cluster_version`
- **v1beta1 (Future):** Stable schema after field testing

**Consumers:** Check `schema_version` field and handle accordingly.

## Implementation Plan

This follows the standard HCO integration pattern: handler code in HCO repo, service image built in our repo.

### Phase 1: ConfigMap Integration (virt-cluster-validate repo) [3-4 days]

**Repository:** `virt-cluster-validate/`

**Goal:** Make CLI write OCPv-compatible ConfigMap results when running as Job.

**Why first:** This makes the image HCO-ready - handlers can be added later without image changes.

**Tasks:**
1. Implement ConfigMap writer module with versioned schema
   - Parse existing JSON output from runner
   - Convert to OCPv schema (`tests_run`, `tests_passed`, etc.)
   - **Add schema_version field: "v1alpha1"** (enables evolution)
   - **Add run_id field** (from environment variable)
   - Write ConfigMap using Kubernetes Python client
2. Add environment variables support
   - `CONFIGMAP_NAMESPACE` - Where to create ConfigMap (default: current namespace)
   - `CONFIGMAP_NAME` - ConfigMap name (passed from handler with run ID)
   - `VALIDATION_RUN_ID` - Run ID for tracking (format: val-<timestamp>-<shortid>)
3. Add comprehensive labels to ConfigMap
   - `validation.kubevirt.io/run-id: <run-id>` (enables querying specific run)
   - `validation.kubevirt.io/component: cluster-validation` (standard label)
   - `validation.kubevirt.io/schema-version: v1alpha1` (version tracking)
   - `validation.kubevirt.io/status: completed|failed` (final status)
4. Update CLI entrypoint
   - Detect in-cluster execution (check KUBERNETES_SERVICE_HOST)
   - Extract run ID from environment or generate if missing
   - Call ConfigMap writer at end of run
5. Add `bin/oc` wrapper for resource label injection (test resource tracking)
6. Unit tests for schema conversion and versioning

**Files to create/modify:**
- `virt_cluster_validate/configmap_writer.py` - NEW
- `virt-cluster-validate` - Modify to call ConfigMap writer when in-cluster
- `bin/oc` - NEW wrapper for label injection
- `tests/unit/test_configmap_writer.py` - NEW
- `requirements.txt` - Add `kubernetes` library

**Deliverable:** 
- Image can run as Job and write ConfigMap results
- Test: `oc create -f job.yaml` → ConfigMap created with results

### Phase 2: HCO Handler Implementation (HCO repo) [5-7 days]

**Repository:** `../hyperconverged-cluster-operator/`

**Goal:** Add operand handlers that create validation Jobs when HyperConverged CR annotated.

**Why second:** Handlers depend on Phase 1 image being able to write ConfigMaps.

**Tasks:**
1. Study existing handler patterns
   - Read AIE webhook handler (`controllers/handlers/aie/`)
   - Read wasp-agent handler (`controllers/handlers/wasp-agent/`)
   - Understand `operands.ConditionalHandler` pattern
2. Implement handlers following exact pattern with extensibility
   - `common.go` - Annotation constants (MVP + reserved), utility functions
   - `validation_sa.go` - ServiceAccount handler (unconditional, always created)
   - `validation_rbac.go` - ClusterRoleBinding handler (unconditional, cluster-admin)
   - `validation_job.go` - Job handler with **concurrent run support**:
     - **Check if Job already exists for run ID** (idempotent)
     - **Allow multiple Jobs with different run IDs** (concurrent runs)
     - **Pass run ID to container via env var** (for ConfigMap naming)
     - **Add comprehensive labels** (run-id, component, schema-version)
3. Register handlers in operandHandler
   - Add to handler list in registration order
   - Follow exact pattern from AIE/wasp registration
4. Add image environment variable
   - `VIRT_CLUSTER_VALIDATE_IMAGE` in `deploy/operator.yaml`
   - Add to CSV relatedImages
5. Write functional tests
   - Test annotation → Job creation (single run)
   - Test multiple concurrent runs (different run IDs)
   - Test idempotency (same run ID twice doesn't create duplicate Jobs)
   - Test ConfigMap result presence and schema version
   - Test RBAC creation
   - Test label-based querying
6. Local testing with CRC or similar

**Files to create/modify in HCO repo:**
- `controllers/handlers/cluster-validation/common.go` - NEW (constants, helpers)
- `controllers/handlers/cluster-validation/validation_sa.go` - NEW
- `controllers/handlers/cluster-validation/validation_rbac.go` - NEW  
- `controllers/handlers/cluster-validation/validation_job.go` - NEW
- `controllers/operandhandler/operandHandler.go` - MODIFY (register handlers)
- `deploy/operator.yaml` - MODIFY (add env var)
- `bundle/manifests/kubevirt-hyperconverged-operator.clusterserviceversion.yaml` - MODIFY (add relatedImage)
- `tests/func/cluster_validation_test.go` - NEW
- `pkg/util/hco.go` - MODIFY if needed (add constant for image env var name)

### Phase 3: Documentation & Upstream Submission [2-3 days]

**Goal:** Document usage, prepare upstream PRs.

**Tasks:**
1. Update virt-cluster-validate README
   - Add "Running in OpenShift Virtualization" section
   - Document annotation-based triggering
   - Document ConfigMap schema
2. Write HCO integration docs
   - `docs/hco-integration.md` in virt-cluster-validate repo
   - `docs/cluster-validation.md` in HCO repo (user guide)
3. Create upstream PRs
   - HCO PR: Handlers + CSV changes
   - virt-cluster-validate PR: ConfigMap writer
4. Address review comments

**Files to create/modify:**
- `README.md` - MODIFY (add OCPv integration section)
- `docs/hco-integration.md` - NEW (in virt-cluster-validate repo)
- `docs/configmap-schema.md` - NEW (in virt-cluster-validate repo)
- `docs/cluster-validation.md` - NEW (in HCO repo)

### Total Timeline: 10-14 days

**Critical Path:**
- Phase 1 must complete before Phase 2 (handlers need working image)
- Phase 2 and Phase 3 can partially overlap (docs while waiting for local testing)

**Parallelization:**
- Phase 1: One developer focused on ConfigMap writer
- Phase 2: Could be same developer or different (HCO expertise helpful)
- Phase 3: Can start docs while Phase 2 testing in progress

**Handler implementation pattern (with extensibility):**

```go
// common.go
const (
    // MVP annotations
    ClusterValidationRunAnnotation = "validation.kubevirt.io/run"
    ClusterValidationIncludePatternsAnnotation = "validation.kubevirt.io/include-patterns"
    ClusterValidationExcludePatternsAnnotation = "validation.kubevirt.io/exclude-patterns"
    
    // Reserved for post-MVP (document but don't implement)
    ClusterValidationCancelAnnotation = "validation.kubevirt.io/cancel"
    ClusterValidationPauseAnnotation  = "validation.kubevirt.io/pause"
    ClusterValidationResumeAnnotation = "validation.kubevirt.io/resume"
    ClusterValidationParamsAnnotation = "validation.kubevirt.io/params"
    
    AppComponentValidation = "cluster-validation"
    SchemaVersion          = "v1alpha1"
)

func shouldCreateValidationJob(hc *hcov1.HyperConverged) bool {
    return hc.Annotations[ClusterValidationRunAnnotation] != ""
}

// validation_job.go
func NewValidationJobHandler(client client.Client, scheme *runtime.Scheme) operands.Operand {
    return operands.NewConditionalHandler(
        operands.NewJobHandler(client, scheme, newValidationJob),
        shouldCreateValidationJob,
        func(hc *hcov1.HyperConverged) client.Object {
            runID := hc.Annotations[ClusterValidationRunAnnotation]
            return &batchv1.Job{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      fmt.Sprintf("virt-cluster-validate-%s", runID),
                    Namespace: hcoutil.GetOperatorNamespaceFromEnv(),
                },
            }
        },
    )
}

func newValidationJob(hc *hcov1.HyperConverged) *batchv1.Job {
    image := os.Getenv(hcoutil.VirtClusterValidateImageEnvV)
    runID := hc.Annotations[ClusterValidationRunAnnotation]
    namespace := hcoutil.GetOperatorNamespaceFromEnv()
    
    // Build comprehensive labels for querying and cleanup
    labels := operands.GetLabels(AppComponentValidation)
    labels["validation.kubevirt.io/run-id"] = runID
    labels["validation.kubevirt.io/component"] = AppComponentValidation
    labels["validation.kubevirt.io/schema-version"] = SchemaVersion
    
    return &batchv1.Job{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("virt-cluster-validate-%s", runID),
            Namespace: namespace,
            Labels:    labels,
        },
        Spec: batchv1.JobSpec{
            TTLSecondsAfterFinished: ptr.To(int32(3600)), // 1 hour auto-cleanup
            Template: corev1.PodTemplateSpec{
                Spec: corev1.PodSpec{
                    ServiceAccountName: "virt-cluster-validate",
                    RestartPolicy:      corev1.RestartPolicyNever,
                    Containers: []corev1.Container{
                        {
                            Name:  "validate",
                            Image: image,
                            Env: []corev1.EnvVar{
                                {Name: "CONFIGMAP_NAMESPACE", Value: namespace},
                                {Name: "CONFIGMAP_NAME", Value: fmt.Sprintf("virt-cluster-validate-results-%s", runID)},
                                {Name: "VALIDATION_RUN_ID", Value: runID}, // For schema population
                            },
                        },
                    },
                },
            },
        },
    }
}
```

**Key extensibility features in this handler:**
1. **Idempotent:** Job name includes run ID - recreating same annotation doesn't duplicate Jobs
2. **Concurrent runs:** Different run IDs create different Jobs that run in parallel
3. **Comprehensive labels:** Enable querying by run ID, component, schema version
4. **Reserved annotations:** Documented in common.go for future implementation
5. **Versioned:** Schema version passed to Job for ConfigMap labeling


## MVP Success Criteria

**Phase 1 (ConfigMap Writer) Complete When:**
- ✅ CLI detects in-cluster execution
- ✅ CLI writes ConfigMap with OCPv-compatible schema
- ✅ Environment variables control ConfigMap name/namespace/labels
- ✅ Unit tests pass for schema conversion
- ✅ Manual test passes: Run as Job → ConfigMap created

**Phase 2 (HCO Handlers) Complete When:**
- ✅ RBAC handlers create ServiceAccount + ClusterRoleBinding on HyperConverged deployment
- ✅ Job handler watches for annotation changes
- ✅ Annotating HyperConverged CR creates validation Job with correct image
- ✅ Job writes results to ConfigMap
- ✅ Image ships as relatedImage in OCPv CSV
- ✅ Functional tests pass in HCO test suite
- ✅ Local testing confirms end-to-end flow works

**Phase 3 (Documentation) Complete When:**
- ✅ virt-cluster-validate README updated
- ✅ HCO integration guide written
- ✅ ConfigMap schema documented
- ✅ PRs submitted to both repos (virt-cluster-validate + HCO)
- ✅ Review comments addressed

## Post-MVP Features

See `conquer-earth.md` for detailed feature roadmap. Key items deferred from HCO integration:
- Upgrade-triggered validation (add to HCO's `handleUpgrade()`)
- Scheduled CronJob validation (add CronJob handler)
- Result aggregation in HyperConverged status (add Condition)
- Configuration change detection (watch ConfigMaps/CRs)

All require additional handlers in HCO following the same pattern as MVP.

## Open Questions

1. **ConfigMap retention:** How long to keep old results?
   - Option A: Keep last 10
   - Option B: TTL-based (7 days)
   - Option C: Both (whichever hits first)
   - **Recommendation:** Option A (last 10) for MVP

2. **Job naming:** Should Job name include timestamp or random ID?
   - **Recommendation:** Use annotation value as suffix (user controls uniqueness)

3. **RBAC:** Start with cluster-admin or scope down?
   - **Recommendation:** cluster-admin for MVP (matches ocp-virt-validation-checkup), scope later

4. **HCO PR strategy:** Single PR or split into multiple?
   - **Recommendation:** Single PR for MVP (small enough), can split post-MVP features

5. **Standalone mode priority:** Include in MVP or defer?
   - **Recommendation:** Include Phase 2 (standalone generator) - provides fallback if HCO integration delayed
