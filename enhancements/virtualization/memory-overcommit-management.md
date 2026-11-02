---
title: memory-overcommit-management
authors:
  - "@fdeutsch"
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - TBD # Virtualization team lead
  - TBD # Memory management SME
  - TBD # Performance team
approvers: # This should be a single approver. The role of the approver is to raise important questions, ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.
  - TBD # CNV/Virtualization approver
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None"
  - TBD # For VirtualMachine API changes
creation-date: 2026-11-02
last-updated: 2026-11-02
status: provisional
tracking-link: # link to the tracking ticket (for example: Jira Feature or Epic ticket) that corresponds to this enhancement
  - TBD
see-also:
  - "/enhancements/scheduling/load-aware-vm-rebalancing-through-descheduler.md"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Memory Overcommit Management for OpenShift Virtualization

## Summary

This enhancement proposes adding in-memory overcommit capabilities to OpenShift Virtualization, enabling clusters to run more virtual machines than available physical memory would traditionally allow. This includes support for memory ballooning, KSM (Kernel Samepage Merging), and configurable memory guarantees to optimize resource utilization while maintaining workload performance and stability.

Note: This focuses on in-memory overcommit techniques. Swap-based overcommit is supported by OpenShift (via Kubernetes 1.35+ swap support) but requires separate configuration and is out of scope for this enhancement.

## Motivation

Virtual machine workloads often have significant gaps between allocated memory and actively used memory. Currently, OpenShift Virtualization schedules VMs based on their full memory allocation, leading to underutilization of cluster memory resources. By enabling controlled memory overcommit, administrators can:

- Increase VM density on existing hardware
- Reduce infrastructure costs by running more VMs per node
- Better align memory allocation with actual usage patterns
- Provide flexible memory management strategies for different workload types

### User Stories

* As a cluster administrator, I want to enable memory overcommit on my virtualization cluster, so that I can run more VMs on the same hardware without compromising stability.

* As a platform engineer, I want to configure different memory overcommit ratios for different node pools, so that I can optimize resource utilization based on workload characteristics (e.g., higher overcommit for development environments, conservative ratios for production).

* As a VM operator, I want to specify memory guarantees for critical VMs, so that they receive dedicated physical memory while allowing non-critical VMs to share remaining resources.

* As a cluster administrator, I want to monitor actual memory usage versus allocated memory across all VMs, so that I can make informed decisions about overcommit ratios and capacity planning.

* As a VM operator, I want transparent memory ballooning in my VMs, so that the platform can reclaim unused memory without manual intervention or VM restarts.

* As a cluster administrator, I want KSM to automatically deduplicate identical memory pages across VMs, so that I can increase density for workloads running similar operating systems or applications.

### Goals

- Enable configurable memory overcommit ratios at the cluster and namespace level
- Implement memory ballooning support for guest VMs with balloon driver
- Integrate KSM (Kernel Samepage Merging) for transparent memory deduplication
- Provide memory guarantee mechanisms for critical workloads
- Add observability metrics for memory overcommit and utilization
- Ensure safe defaults that prevent node memory pressure
- Support smooth migration between overcommit configurations

### Non-Goals

- Swap-based overcommit (deferred to future enhancement - requires separate swap configuration proposal)
- Automatic live migration based on memory pressure (handled separately by descheduler)
- Cross-node memory sharing or distributed memory management
- Memory compression (may be addressed in future enhancement)
- Modifying Kubernetes pod memory management (focuses on VMs only)
- Single-Node OpenShift (SNO) support - deferred to future enhancement due to unique stability requirements

## Proposal

This enhancement introduces three complementary mechanisms for in-memory overcommit:

1. **Memory Ballooning**: Guest VMs run a balloon driver that can dynamically inflate/deflate based on host memory pressure
2. **KSM (Kernel Samepage Merging)**: Kernel-level deduplication of identical memory pages across VMs
3. **Memory Guarantees**: API to specify minimum guaranteed physical memory per VM

Note: This enhancement focuses on in-memory overcommit. Swap-based overcommit is supported by OpenShift (Kubernetes 1.35+) but requires separate configuration and is deferred to a future enhancement.

### Workflow Description

**cluster administrator** is responsible for configuring cluster-wide memory overcommit policies.

**VM operator** is responsible for creating and configuring individual VMs with appropriate memory settings.

**node** is a worker node running RHCOS with KubeVirt/virt-launcher pods.

#### Enabling Memory Overcommit (Admin Workflow)

1. The cluster administrator reviews current cluster memory utilization and workload patterns
2. The administrator creates a `MemoryOvercommitConfig` CR at cluster scope specifying:
   - Target overcommit ratio (e.g., 1.5x = 50% overcommit)
   - KSM enablement and scanning parameters
   - Safety thresholds for node memory pressure
3. The virt-controller observes the configuration and updates node-level settings:
   - Enables KSM on worker nodes via MachineConfig
   - Configures virt-launcher to support memory ballooning
   - Adjusts scheduler to account for overcommit ratio
4. The administrator monitors new metrics:
   - `kubevirt_memory_overcommit_ratio_current`
   - `kubevirt_memory_ksm_pages_shared`
   - `kubevirt_memory_balloon_actual_bytes`

#### Creating VMs with Memory Guarantees (VM Operator Workflow)

1. The VM operator defines a VirtualMachine CR with memory settings:
   ```yaml
   spec:
     template:
       spec:
         domain:
           memory:
             guest: 8Gi          # Allocated memory visible to guest
             guaranteed: 4Gi     # Minimum physical memory guarantee
   ```
2. When the VM is scheduled:
   - Scheduler considers `guaranteed` memory for node placement
   - virt-launcher requests `guaranteed` memory from kubelet
   - Guest OS sees full `guest` memory allocation
3. At runtime:
   - If host memory pressure occurs, balloon driver reclaims memory above guarantee
   - KSM may deduplicate pages across VMs
   - VM continues running as long as `guaranteed` memory is available

#### Memory Reclamation Under Pressure

1. Node memory utilization exceeds configured threshold (e.g., 85%)
2. virt-handler detects pressure via cgroup memory statistics
3. For VMs with ballooning enabled and usage above guarantee:
   - virt-handler sends balloon inflation request to guest driver
   - Guest balloon driver allocates memory internally, making it reclaimable
   - Host kernel reclaims ballooned memory for other workloads
4. If pressure continues to increase:
   - VMs with no guarantee may be evicted (existing pod eviction mechanism)
   - VMs with guarantees remain running with guaranteed allocation

#### Disabling Memory Overcommit

1. The administrator updates `MemoryOvercommitConfig` to disable overcommit
2. Controller updates scheduler to stop considering overcommit ratio
3. New VMs are scheduled with full memory allocation
4. Existing VMs continue running but may be migrated if node becomes overcommitted
5. KSM can be independently disabled via separate setting

### API Extensions

This enhancement modifies the `VirtualMachine` API and adds a new cluster-scoped configuration API.

#### New API: MemoryOvercommitConfig (Cluster-scoped)

```go
// MemoryOvercommitConfig configures cluster-wide memory overcommit behavior
// +genclient
// +genclient:nonNamespaced
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type MemoryOvercommitConfig struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   MemoryOvercommitConfigSpec   `json:"spec"`
    Status MemoryOvercommitConfigStatus `json:"status"`
}

type MemoryOvercommitConfigSpec struct {
    // OvercommitRatio is the target ratio of allocated to physical memory.
    // For example, 1.5 means 50% overcommit (can allocate 1.5x physical memory).
    // Valid range: 1.0 (no overcommit) to 4.0 (300% overcommit).
    // Default: 1.0 (no overcommit, subject to change in future releases)
    // +optional
    // +kubebuilder:validation:Minimum=1.0
    // +kubebuilder:validation:Maximum=4.0
    OvercommitRatio *float64 `json:"overcommitRatio,omitempty"`
    
    // KSM configures Kernel Samepage Merging for memory deduplication
    // +optional
    KSM *KSMConfig `json:"ksm,omitempty"`
    
    // NodeSelector applies this configuration only to matching nodes.
    // If empty, applies to all virtualization-capable multi-node clusters.
    // Single-Node OpenShift (SNO) nodes are automatically excluded.
    // +optional
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`
    
    // SafetyThresholds defines when to stop accepting new VMs
    // +optional
    SafetyThresholds *SafetyThresholds `json:"safetyThresholds,omitempty"`
}

type KSMConfig struct {
    // Enabled controls whether KSM is active on matching nodes
    // +optional
    Enabled bool `json:"enabled"`
    
    // ScanInterval is how often KSM scans for duplicate pages (milliseconds)
    // Default: 100ms
    // +optional
    ScanInterval *int32 `json:"scanInterval,omitempty"`
    
    // PagesToScan is number of pages scanned per interval
    // Default: 100
    // +optional
    PagesToScan *int32 `json:"pagesToScan,omitempty"`
}

type SafetyThresholds struct {
    // NodeMemoryPressureThreshold is the percentage of node memory usage
    // above which new VM scheduling is paused (but existing VMs continue)
    // Default: 85%
    // +optional
    // +kubebuilder:validation:Minimum=50
    // +kubebuilder:validation:Maximum=95
    NodeMemoryPressureThreshold *int32 `json:"nodeMemoryPressureThreshold,omitempty"`
}

type MemoryOvercommitConfigStatus struct {
    // Conditions track the state of the configuration
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    
    // ObservedGeneration is the last generation reconciled
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
    
    // NodesConfigured is the number of nodes with this config applied
    NodesConfigured int32 `json:"nodesConfigured"`
    
    // TotalPhysicalMemory is total physical memory across configured nodes
    TotalPhysicalMemory resource.Quantity `json:"totalPhysicalMemory"`
    
    // TotalAllocatableMemory considering overcommit ratio
    TotalAllocatableMemory resource.Quantity `json:"totalAllocatableMemory"`
}
```

#### Modified API: VirtualMachine Memory Spec

```go
// Existing VirtualMachine API extended with new fields
type Memory struct {
    // Guest is the memory size visible to the guest OS
    // This is the traditional memory allocation field
    // +optional
    Guest *resource.Quantity `json:"guest,omitempty"`
    
    // Guaranteed is the minimum physical memory guaranteed to this VM.
    // Must be less than or equal to Guest.
    // If unset, defaults to Guest value (no overcommit for this VM).
    // When set lower than Guest, the difference may be overcommitted.
    // +optional
    Guaranteed *resource.Quantity `json:"guaranteed,omitempty"`
    
    // Ballooning configures the memory balloon device
    // +optional
    Ballooning *BallooningConfig `json:"ballooning,omitempty"`
}

type BallooningConfig struct {
    // Enabled controls whether balloon device is attached to VM
    // Requires balloon driver in guest OS
    // Default: true (subject to change, will auto-enable for overcommit)
    // +optional
    Enabled *bool `json:"enabled,omitempty"`
    
    // StatsInterval is how often balloon stats are collected (seconds)
    // Default: 10
    // +optional
    StatsInterval *int32 `json:"statsInterval,omitempty"`
}
```

**API Behavior:**

- MemoryOvercommitConfig is singleton resource named `cluster`
- Scheduler multiplies node allocatable by overcommit ratio when ratio > 1.0
- `guaranteed` field sets virt-launcher pod memory request
- Finalizer ensures cleanup on VM deletion

### Topology Considerations

#### Hypershift / Hosted Control Planes

**Impact**: Moderate - virt components run in guest cluster, but configuration affects both clusters.

**Guest cluster**:
- VirtualMachine CRs and virt-launcher pods run here
- Memory overcommit applies to guest cluster worker nodes
- KSM runs on guest cluster worker nodes (not in HCP namespace)
- Metrics exposed in guest cluster Prometheus

**Management cluster**:
- No direct impact to management cluster memory
- HCP control plane pods have fixed memory (not affected by guest VM overcommit)
- Guest cluster node resources are independent of HCP resource accounting

**Version skew considerations**:
- Guest cluster CNV version determines feature availability
- Management cluster version does not gate this feature
- API changes follow standard CNV upgrade path in guest cluster

**Configuration**:
- `MemoryOvercommitConfig` CR lives in guest cluster (same as VirtualMachine CRs)
- MachineConfig for KSM targets guest cluster worker nodes

**Why this is HCP-specific**: In standalone clusters, enabling KSM affects control plane nodes if they're schedulable. In HCP, control plane runs as pods in management cluster and is unaffected by guest cluster KSM settings.

#### Standalone Clusters

**Impact**: Standard - applies to all worker nodes (and master nodes if schedulable).

This is the primary deployment model for the feature. Memory overcommit affects:
- Worker nodes running virt-launcher pods
- Master nodes if they're schedulable and running VMs (rare but possible)

**Configuration**: `MemoryOvercommitConfig` applies cluster-wide or via nodeSelector.

#### Single-node Deployments or MicroShift

**SNO**: Out of scope. SNO nodes are automatically excluded via `node.openshift.io/single-node-cluster` label. Memory overcommit risks control plane stability due to shared memory resources with no HA fallback.

**MicroShift**: Not applicable. OpenShift Virtualization is not included in MicroShift.

#### OpenShift Kubernetes Engine

Not applicable. OKE does not include OpenShift Virtualization.

### Implementation Details/Notes/Constraints

#### Memory Ballooning Implementation

virt-launcher pod requests `guaranteed` memory; VM sees `guest` memory. virtio-balloon device attached. virt-handler inflates balloon when memory pressure detected via cgroup stats. Requires guest driver (virtio_balloon for Linux, virtio-win for Windows). Balloon never deflates below `guaranteed` level.

#### KSM Integration

MachineConfig enables KSM via `/sys/kernel/mm/ksm/run = 1`. Scanning parameters configurable via spec. KSM CPU overhead is tunable via `pagesToScan` (higher scanning = more CPU, more deduplication).

#### Scheduler Changes

Multiplies node allocatable by overcommit ratio. Filters on `guaranteed` memory. Excludes SNO nodes via label. Respects safety thresholds.

#### Observability

New Prometheus metrics:

```
# Cluster-level
kubevirt_memory_overcommit_ratio{node="worker-1"} 1.5
kubevirt_memory_physical_bytes{node="worker-1"} 68719476736
kubevirt_memory_allocatable_bytes{node="worker-1"} 103079215104

# Per-VM
kubevirt_vm_memory_guest_bytes{vm="web-server",namespace="prod"} 8589934592
kubevirt_vm_memory_guaranteed_bytes{vm="web-server",namespace="prod"} 4294967296
kubevirt_vm_memory_balloon_actual_bytes{vm="web-server",namespace="prod"} 6442450944

# KSM
kubevirt_ksm_pages_sharing{node="worker-1"} 524288
kubevirt_ksm_cpu_seconds_total{node="worker-1"} 45.2
```

New alerts:

```yaml
- alert: MemoryOvercommitNodePressure
  expr: kubevirt_memory_overcommit_pressure_ratio > 0.85
  annotations:
    summary: "Node {{ $labels.node }} approaching memory pressure threshold"

- alert: VMMemoryBalloonInflationHigh
  expr: (kubevirt_vm_memory_balloon_actual_bytes / kubevirt_vm_memory_guest_bytes) < 0.5
  annotations:
    summary: "VM {{ $labels.vm }} balloon inflated >50%, possible memory pressure"
```

### Risks and Mitigations

**Risk: Node OOM killing VMs**  
*Mitigation*: Safety thresholds, memory guarantees, conservative defaults, OOMKiller score adjustment.

**Risk: Missing balloon driver**  
*Mitigation*: Detect at start, log warning, scheduler accounts for worst-case.

**Risk: KSM CPU overhead**  
*Mitigation*: Optional, disabled by default, configurable scan rates, metrics.

**Risk: Upgrade disruption**  
*Mitigation*: Changes only affect new VMs, live migration for rebalancing.

**Risk: Cascading evictions**  
*Mitigation*: Priority scores, PodDisruptionBudgets, gradual eviction.

### Drawbacks

Increased complexity in debugging. Performance unpredictability under memory pressure. Requires guest drivers. Additional operational concepts to understand. Incompatible with strict performance SLAs.

## Alternatives (Not Implemented)

### Alternative 1: Swap-based Overcommit

Deferred to separate enhancement. OpenShift supports swap for VM workloads (Kubernetes 1.35+ swap GA), but this enhancement focuses on in-memory overcommit only. Swap-based overcommit requires:
- Swap device provisioning strategy
- Performance tuning for VM workloads
- Interaction design with memory ballooning
- Per-VM swap policies

Could complement this enhancement in future with dedicated design.

### Alternative 2: Memory Compression (zswap/zram)

Not chosen. CPU overhead for compression/decompression. Unpredictable compression ratios. Could be future enhancement.

### Alternative 3: Automatic Memory Right-sizing

Not chosen. Requires VM restart. Should be separate day-2 operations enhancement.

### Alternative 4: Per-Namespace Overcommit Policies

Not chosen. Scheduler complexity. ResourceQuotas already provide namespace limits. Partially achievable via nodeSelector.

## Open Questions [optional]

1. Should VirtualMachine status expose `memoryPressure` condition when balloon is inflated?

2. How should `guaranteed` interact with ResourceQuotas - count guaranteed or full guest allocation?

3. Default behavior when `guaranteed` not specified: default to `guest` (safe) or `guest/ratio` (aggressive)?

4. Should KSM be separate CR or bundled in MemoryOvercommitConfig?

## Test Plan

### Unit Tests

- API validation (guaranteed <= guest, ratio ranges)
- Scheduler logic with various overcommit ratios
- Balloon inflation/deflation calculations
- KSM configuration generation

### Integration Tests (envtest)

- Controller reconciliation of MemoryOvercommitConfig
- Scheduler filtering based on guaranteed memory
- Metrics registration and updates
- Webhook validation of API constraints

### E2E Tests (Full Cluster)

**Test: Basic Memory Overcommit**
1. Enable overcommit ratio 1.5 on test cluster
2. Create VMs totaling 1.5x node physical memory
3. Verify all VMs scheduled and running
4. Verify memory metrics show overcommit

**Test: Memory Ballooning Under Pressure**
1. Create VM with guest=8Gi, guaranteed=4Gi
2. Simulate memory pressure on node (create memory-consuming pods)
3. Verify balloon inflates in VM (actual < guest)
4. Verify VM remains running and functional
5. Remove pressure, verify balloon deflates

**Test: Memory Guarantees Honored**
1. Create VM with guaranteed=4Gi
2. Create memory pressure exceeding available memory
3. Verify VM continues running with 4Gi available
4. Verify other pods may be evicted, but VM is not

**Test: KSM Deduplication**
1. Enable KSM with aggressive scan settings
2. Create 5 identical VMs (same OS image)
3. Wait for KSM to scan and merge pages
4. Verify `ksm_pages_sharing` metric increases
5. Verify total physical memory usage < sum of VM allocations

**Test: SNO Exclusion from Overcommit**
1. Deploy on SNO cluster
2. Attempt to enable overcommit configuration
3. Verify configuration is rejected OR has no effect on SNO node
4. Verify VMs continue to schedule with full memory allocation
5. Verify metrics show overcommit ratio = 1.0 for SNO nodes

**Test: Upgrade Path**
1. Deploy cluster with overcommit disabled
2. Create VMs without guaranteed field
3. Enable overcommit
4. Verify existing VMs continue running
5. Create new VMs, verify new overcommit behavior

**Test: Configuration Changes**
1. Enable overcommit, create VMs
2. Change overcommit ratio (e.g., 1.5 -> 1.3)
3. Verify existing VMs not disrupted
4. Verify new VMs use new ratio

**Test: Missing Balloon Driver**
1. Create VM without balloon driver in guest
2. Verify VM starts successfully
3. Verify warning logged about missing driver
4. Verify scheduler still accounts for guaranteed memory

### Performance Testing

- Measure balloon inflation/deflation latency
- Measure KSM CPU overhead at various scan rates
- Measure VM performance variance under different overcommit ratios
- Load testing: density improvements vs. performance impact

### Scalability Testing

- 100 VMs on single node with 2x overcommit
- Measure scheduler performance with overcommit calculations
- Measure virt-handler CPU usage monitoring balloons across many VMs
- Measure metrics system overhead with additional per-VM metrics

## Graduation Criteria

### Dev Preview -> Tech Preview

- Feature implemented behind feature gate
- Unit and integration tests passing
- E2E tests for core scenarios (basic overcommit, ballooning, KSM)
- Documentation for configuration and troubleshooting
- Metrics and alerts available
- Tested on SNO and standard HA clusters
- Feedback gathered from early adopters (QE, selected partners)

### Tech Preview -> GA

- At least 2 releases in Tech Preview
- No major bugs or regressions in Tech Preview
- Full E2E test coverage including edge cases
- Performance and scalability testing completed
- Upgrade testing (TP -> GA upgrade path validated)
- Documentation complete in openshift-docs
- SLI/SLO defined for memory overcommit stability
- Monitoring and alerting validated in production-like environments
- Field feedback incorporated (from support, partners, customers)
- Hypershift testing complete (if HCP supports CNV by GA)

**For GA, must have:**
- Default overcommit ratio determined (likely remains 1.0 = disabled)
- Runbook for troubleshooting memory pressure scenarios
- Support-approved escalation path for memory-related incidents

### Removing a deprecated feature

Not applicable - new feature.

## Upgrade / Downgrade Strategy

### Upgrade (without overcommit -> with overcommit)

New APIs added. Existing VMs continue working. Administrator opts in by creating `MemoryOvercommitConfig`. No automatic migration.

### Downgrade (with overcommit -> without overcommit)

Remove `MemoryOvercommitConfig`. VMs with `guaranteed` must be migrated or recreated. Webhook should block downgrade if incompatible VMs exist.

### Mid-Upgrade Version Skew

Safe: N virt-launcher + N-1 virt-controller (overcommit not enforced, but safe).
Risky: N virt-controller + N-1 virt-launcher (overcommit without ballooning).
Upgrade order: CRDs → virt-launcher → virt-controller → virt-api.

## Version Skew Strategy

### Component Version Skew

Guest driver version independent (feature detection at runtime). Uses standard pod memory requests (no kubelet changes). KSM always available in RHCOS.

### API Version Skew

New fields added to existing `v1` API (additive, backward compatible). No conversion webhook needed. Old clients ignore new fields.

## Operational Aspects of API Extensions

### CRD: MemoryOvercommitConfig

Minimal virt-controller overhead. No impact on VM start time.

### Webhook: VirtualMachine Memory Validation

Validates `guaranteed <= guest`. Adds <10ms per VM operation. Fails if virt-api unavailable (existing behavior).

### Finalizer: VirtualMachine Memory Cleanup

Ensures memory bookkeeping updated on deletion. Adds <1s to deletion time.

## Support Procedures

### Detecting Memory Overcommit Issues

**Performance degradation**: Check balloon inflation metrics and node memory pressure.

**Scheduling failures**: Check FailedScheduling events and safety thresholds.

**VM evictions**: Check VM events and node OOMKiller logs.

### Disabling Memory Overcommit

Set `overcommitRatio: 1.0` - affects new VMs only.

Disable ballooning per-VM - requires restart.

Disable KSM - stops scanning, merged pages un-share gradually.

### Graceful Failure Behavior

virt-handler failure: VMs continue, no new balloon operations.

Missing balloon driver: VM starts, warning logged, no overcommit for that VM.

KSM disabled: Pages un-share gradually.

## Infrastructure Needed [optional]

### CI/Cluster Requirements

**E2E test cluster**:
- Minimum 3 worker nodes with 64 GiB RAM each (multi-node cluster required)
- Virtualization enabled (nested virt or bare metal)
- Existing CNV CI clusters should suffice
- Additional SNO cluster for testing automatic exclusion behavior

**Performance test cluster**:
- 1 worker node with 128 GiB RAM (to test high VM density)
- Bare metal preferred (more realistic results)
- May need dedicated PSI environment allocation

### New Test Images

**VM images with balloon drivers**:
- RHEL 9 with virtio-balloon module
- Windows Server 2022 with virtio-win drivers
- Fedora CoreOS (minimal test image)

Should leverage existing [kubevirt/containerdisks](https://quay.io/organization/kubevirt) images where possible.

### Metrics Infrastructure

**Prometheus**:
- Existing CNV Prometheus should suffice
- New metrics added to existing exporters (virt-handler, virt-controller)
- No additional Prometheus instances needed

### Documentation Requirements

**Admin documentation**:
- Configuring memory overcommit (upstream + product docs)
- Tuning KSM parameters
- Troubleshooting memory pressure scenarios

**Developer documentation**:
- API reference for new fields
- Example VM manifests with guaranteed memory
- Best practices for memory sizing

**Must appear in**:
- [OpenShift Virtualization documentation](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [KubeVirt user guide](https://kubevirt.io/user-guide/) (upstream)
- Release notes for Tech Preview and GA
