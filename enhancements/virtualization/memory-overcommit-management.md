---
title: memory-overcommit-management
authors:
  - "@fdeutsch"
reviewers:
  - TBD # Virtualization team lead
  - TBD # Memory management SME
  - TBD # Performance team
approvers:
  - TBD # CNV/Virtualization approver
api-approvers:
  - None # Uses existing HyperConverged CR
creation-date: 2026-11-02
last-updated: 2026-11-02
status: implemented
tracking-link:
  - TBD
see-also:
  - "/enhancements/scheduling/load-aware-vm-rebalancing-through-descheduler.md"
replaces:
  - "WASP (deprecated swap mechanism)"
  - "Legacy LRU page reclaim"
  - "Traditional balloon inflation"
superseded-by:
  - N/A
---

# Memory Overcommit Management for OpenShift Virtualization

## Summary

This enhancement implemented memory overcommit capabilities for OpenShift Virtualization, enabling clusters to run more virtual machines than available physical memory through statistical multiplexing. The implementation uses defense-in-depth mechanisms to AVOID swap traffic while keeping swap available as a safety net.

Primary mechanisms work proactively to prevent swapping:
- **Free Page Reporting (FPR)**: Asynchronous reclamation of unused pages
- **Descheduler with PSI**: Live-migrates VMs before memory pressure causes swapping
- **MGLRU**: Modern swap with thrashing prevention (when swap unavoidable)
- **KSM Zero Pages**: Deduplicates zero-filled pages across VMs

The virt-platform-autopilot component automates configuration to minimize swap usage while maintaining the safety net for worst-case scenarios.

## Motivation

Operational data showed VMs typically utilize only ~20% of allocated RAM, leading to significant memory waste. Traditional 1:1 memory allocation prevented maximizing cluster density and made it impossible to meet common memory utilization target requirements of 70-80%.

Without overcommit, achieving 70-80% memory utilization requires perfectly sizing VMs to match actual usage - impractical due to workload variability. With statistical multiplexing, clusters can hit these utilization targets by running more VMs than physical memory would traditionally allow.

This enhancement enables statistical multiplexing - running more VMs by betting they won't all peak simultaneously. The key insight: **swap exists as a safety net, but all mechanisms work to AVOID actually swapping**. FPR proactively reclaims unused memory, descheduler rebalances before pressure builds, and MGLRU prevents thrashing if swapping becomes unavoidable. The goal is overcommit WITHOUT swap traffic under normal operation.

### User Stories

* As a cluster administrator, I want to enable memory overcommit to increase VM density on existing hardware without purchasing additional nodes.

* As a platform engineer, I want memory overcommit configuration automated so that MGLRU parameters, descheduler thresholds, and safety limits are calculated based on my cluster's characteristics rather than requiring manual tuning.

* As a VM operator, I want transparent memory management so VMs can report unused pages to the hypervisor without experiencing performance degradation from forced balloon inflation.

* As a cluster administrator, I want safe KSM that only merges zero-filled pages so I get memory deduplication benefits without the security and performance concerns of full KSM.

* As a cluster administrator, I want the descheduler to rebalance VMs based on actual memory pressure (PSI metrics) so live migration prevents node saturation before emergency evictions occur.

* As an SRE, I want observability into memory overcommit ratios, swap usage, and pressure stall metrics so I can make informed capacity planning decisions.

### Goals

- Enable configurable memory overcommit ratios via HyperConverged CR
- Implement Free Page Reporting (FPR) as default asynchronous memory reclamation
- Integrate MGLRU (Multi-Generational LRU) for modern swap with per-VM tracking
- Provide KSM zero-page merging for safe memory deduplication
- Add virt-platform-autopilot for automated configuration
- Integrate descheduler with PSI metrics for pressure-based rebalancing
- Deprecate WASP, legacy LRU, and traditional balloon inflation
- Ensure safe defaults (0% overcommit unless explicitly enabled)

### Non-Goals

- Automatic live migration based on memory pressure (handled by descheduler, not this enhancement)
- Cross-node memory sharing or distributed memory management
- Memory compression (zswap/zram)
- Modifying Kubernetes pod memory management for non-VM workloads
- Single-Node OpenShift (SNO) support - deferred due to control plane stability requirements
- Full KSM (arbitrary page merging) - security and performance concerns

## Proposal

This enhancement implements layered defense to enable overcommit while AVOIDING swap traffic:

**Proactive Mechanisms** (prevent swapping):
1. **Free Page Reporting (FPR)**: Guest-cooperative asynchronous reporting of unused pages to hypervisor (requires virtio driver in guest, primary reclamation, enabled by default)
2. **Descheduler with PSI**: Live-migrates VMs when pressure detected, BEFORE swapping starts
3. **KSM Zero Pages**: Guest-agnostic deduplication of zero-filled pages across VMs (no guest driver required, reduces total memory footprint)
4. **Memory Overcommit Percentage**: Reduces pod memory requests to enable statistical multiplexing

**Swap as Safety Net** (graceful degradation when unavoidable):
5. **MGLRU-based Swap**: Modern multi-generational LRU provides intelligent swap when proactive mechanisms cannot prevent memory pressure. Targets cold pages for eviction while protecting hot pages, with Bloom filter thrashing prevention and per-VM tracking.

**Goal**: Swap exists but should rarely be used. FPR + descheduler + KSM work together to keep memory available without swapping. When memory pressure cannot be avoided, MGLRU-based swap ensures graceful degradation (not OOM kills) through targeted eviction of truly cold pages.

**Related Components**:
- virt-platform-autopilot: Tunes all mechanisms to minimize swap traffic
- Descheduler (KubeVirtRelieveAndMigrate): Uses PSI metrics to detect pressure early and rebalance

### Workflow Description

**cluster administrator** configures memory overcommit settings.

**virt-platform-autopilot** calculates optimal parameters.

**descheduler** monitors pressure and rebalances VMs.

#### Enabling Memory Overcommit (Admin Workflow)

1. Administrator enables autopilot on HyperConverged resource:
   ```bash
   oc annotate hyperconverged kubevirt-hyperconverged \
     -n openshift-cnv platform.kubevirt.io/autopilot="true" --overwrite
   ```

2. Autopilot calculates and applies:
   - MGLRU parameters (based on node memory, workload patterns)
   - Memory overcommit percentage (default 20% at node level)
   - Descheduler thresholds for pressure-based rebalancing
   - Control group throttling limits

3. Administrator optionally configures custom overcommit percentage:
   ```bash
   oc patch hyperconverged kubevirt-hyperconverged -n openshift-cnv \
     --type='merge' -p='{"spec":{"virtualization":{"memoryOvercommitPercentage":150}}}'
   ```

4. VMs automatically benefit from FPR (enabled by default):
   - Guest OS reports free pages via virtio driver
   - Hypervisor reclaims pages asynchronously
   - No forced balloon inflation (unlike traditional ballooning)

#### Enabling KSM Zero Pages (Optional)

1. Administrator enables KSM zero-page merging:
   ```bash
   oc annotate hyperconverged kubevirt-hyperconverged \
     -n openshift-cnv platform.kubevirt.io/enable-ksm-zero-only="true" --overwrite
   ```

2. Host kernel merges zero-filled pages across VMs:
   - Guest-agnostic (no guest driver required)
   - Secure (only zero-filled pages, not arbitrary application data)
   - Common in Windows workloads

#### Memory Pressure Response (Avoids Swapping)

**Normal operation** (no pressure):
1. FPR continuously reports unused pages → hypervisor reclaims
2. KSM merges zero pages across VMs → reduces footprint
3. Result: Overcommit achieved WITHOUT swap traffic

**Moderate pressure** (70-80% utilization):
1. Descheduler monitors utilization metrics
2. If trending upward: Live-migrate VMs to less-utilized nodes
3. Rebalancing prevents pressure from building
4. Result: Swap still NOT used

**High pressure** (>80% utilization):
1. Descheduler switches to PSI (Pressure Stall Information) metrics
2. PSI detects when workloads START waiting for memory operations
3. Aggressive live-migration from saturated nodes
4. Soft-taint saturated nodes (PreferNoSchedule)
5. Result: Swap minimally used or avoided entirely

**Worst-case** (rebalancing cannot keep up):
1. MGLRU tiers cold pages to swap (graceful degradation)
2. Bloom filters prevent thrashing
3. Swap traffic occurs but VM stays running (not OOM killed)
4. Alerting triggers for operator intervention

**Goal**: Steps 1-3 should handle 99% of cases. Swap is safety net for step 4.

### API Extensions

No new CRDs. Uses existing HyperConverged CR from openshift-cnv operator.

#### Modified API: HyperConverged

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
  annotations:
    # Enable autopilot for automated configuration
    platform.kubevirt.io/autopilot: "true"
    # Enable KSM zero-page merging
    platform.kubevirt.io/enable-ksm-zero-only: "true"
spec:
  virtualization:
    # Memory overcommit percentage (optional)
    # Formula: pod requests = VM guest * 100 / memoryOvercommitPercentage
    # Example: 150 means 12Gi VM requests 8Gi (12 * 100 / 150)
    # Default: 100 (no overcommit unless explicitly set)
    memoryOvercommitPercentage: 150
    
    # KSM configuration (alternative to annotation)
    # Not used when autopilot enabled with KSM zero-only annotation
    ksmConfiguration:
      nodeLabelSelector:
        matchLabels:
          kubevirt.io/ksm-enabled: "true"
```

**Behavior**:

- **autopilot annotation**: Enables virt-platform-autopilot to configure MGLRU, descheduler, limits
- **memoryOvercommitPercentage**: Reduces pod requests; scheduler can pack more VMs per node
- **FPR**: Enabled by default for all VMs (no configuration needed)
- **KSM zero-only**: Merges zero-filled pages; guest-agnostic, secure

### Topology Considerations

#### Hypershift / Hosted Control Planes

Memory overcommit applies to guest cluster worker nodes only:

- VirtualMachine CRs run in guest cluster
- HyperConverged CR configured in guest cluster
- FPR, MGLRU, KSM operate on guest worker nodes
- Management cluster HCP pods unaffected

Autopilot runs in guest cluster and configures guest worker nodes. No cross-cluster dependencies.

#### Standalone Clusters

Primary deployment model. All mechanisms supported:
- FPR on all worker nodes
- MGLRU for swap management
- KSM zero pages across workers
- Descheduler rebalances VMs across cluster

#### Single-node Deployments or MicroShift

**SNO**: Out of scope. Control plane stability risk. Nodes automatically excluded.

**MicroShift**: Not applicable. OpenShift Virtualization not included.

#### OpenShift Kubernetes Engine

Not applicable. OKE does not include OpenShift Virtualization.

### Implementation Details/Notes/Constraints

#### Free Page Reporting (FPR) Implementation

**Guest-Cooperative Mechanism** - requires guest driver cooperation.

**How it Works**:
- Enabled by default (no configuration needed)
- **Guest** virtio driver scans free memory lists
- **Guest** reports free pages to hypervisor via QMP channel
- **Hypervisor** returns pages to host allocation pool
- **Asynchronous**: No forced allocation (unlike traditional ballooning)

**Advantages over Traditional Ballooning**:
- Traditional ballooning inflates balloon device inside guest, forcing allocation
- Can cause performance degradation during sudden spikes
- Conflicts with guest-level memory compression
- FPR observes already-freed pages without forcing allocation

**Guest Driver Requirements** (cooperative mechanism):
- **Linux**: virtio_balloon module (included in most modern distributions)
- **Windows**: virtio-win guest tools (must be installed)
- **Without driver**: VM runs but FPR unavailable (no automatic reclamation)

**Disabled When**:
- VM uses hugepages (memory pre-allocated and pinned)
- Guest does not have virtio driver installed

**Implication**: FPR effectiveness depends on guest cooperation. VMs without drivers get no automatic memory reclamation.

#### Targeted Eviction and Thrashing Prevention (MGLRU)

**Default since OpenShift 4.18**. Multi-Generational LRU (MGLRU) replaces legacy active/inactive LRU and WASP, providing intelligent page replacement that minimizes swap traffic through targeted eviction.

**Why MGLRU is Better**:

Legacy LRU used simple active/inactive lists, leading to:
- Poor targeting: Hot pages could be evicted if they aged out
- Thrashing: Evicted pages re-swapped in immediately
- High CPU overhead: Constant rescanning

MGLRU solves these through generational tracking and Bloom filters.

**Generational Page Tracking**:

Pages grouped by access recency into generations:
- **Generation 0**: Hot pages (frequently accessed, protected from eviction)
- **Generation 1**: Aging pages (transitioning)
- **Generation 2/3**: Cold pages (rarely accessed, eviction candidates)

Eviction targets Gen 2/3 pages first, protecting actually-used memory.

**Thrashing Prevention with Bloom Filters**:

Bloom filters track recently evicted page addresses:
- When evicting page: Add to Bloom filter
- When accessing page: Check Bloom filter
- If in filter: Page was recently evicted, protect from re-eviction
- Result: Prevents re-swapping same pages repeatedly

Lower CPU overhead than legacy LRU's constant rescanning.

**Per-VM Memory Tracking**:

Linux cgroups track memory per virt-launcher pod:
- Accurate per-VM swap attribution
- Metrics show which VM is swapping
- Enables targeted troubleshooting

**Configuration**:

virt-platform-autopilot calculates optimal MGLRU parameters:
- Generation advancement thresholds
- Bloom filter sizes
- Eviction scan rates
- Based on: node memory, number of VMs, workload patterns

Manual configuration not recommended due to complex parameter interactions.

**Secondary Memory Tiers**:

MGLRU orchestrates tiering to high-performance devices:
- Recommend NVMe/SSD for swap devices (not spinning disks)
- Future: CXL (Compute Express Link) byte-addressable memory expansion
- Hardware-accelerated tiering for sub-millisecond latency

#### KSM Zero Pages Implementation

**Guest-Agnostic Mechanism** - no guest driver or cooperation required.

**Since OpenShift 4.22**. Replaces full KSM.

**How it Works**:
- **Host** kernel scans memory pages from hypervisor perspective
- Identifies zero-filled pages (all zeros) across all VMs
- Merges identical zero pages, reducing total memory footprint
- **Guest-agnostic**: Works regardless of guest OS, no driver needed

**Why Zero-Only** (not full KSM):
- Full KSM security concerns: arbitrary page content exposure risk
- Full KSM performance concerns: slows memory operations
- Zero pages are:
  - Common in Windows workloads (uninitialized memory)
  - Safe to merge (no information disclosure)
  - Fast to scan (simple comparison)
- Lower CPU overhead than full KSM

**Advantages of Guest-Agnostic Approach**:
- Works on any guest OS (Linux, Windows, others)
- No guest driver installation required
- No guest configuration needed
- No compatibility concerns with guest kernel versions

**Configuration**:
```bash
oc annotate hyperconverged kubevirt-hyperconverged \
  -n openshift-cnv platform.kubevirt.io/enable-ksm-zero-only="true"
```

**Implication**: Unlike FPR (guest-cooperative), KSM zero pages work universally across all VMs without guest involvement.

#### Virtualization Platform Autopilot

**Purpose**: Automates complex configuration calculations.

**Calculates and Configures**:
- MGLRU parameters (scan rates, generation thresholds)
- Memory overcommit hard limits (safety bounds)
- Descheduler thresholds (when to trigger rebalancing)
- Control group throttling limits

**Activation**:
```bash
oc annotate hyperconverged kubevirt-hyperconverged \
  -n openshift-cnv platform.kubevirt.io/autopilot="true"
```

**Deactivation** (does not disable already-configured features):
```bash
oc annotate hyperconverged kubevirt-hyperconverged \
  -n openshift-cnv platform.kubevirt.io/autopilot="false"
```

#### Descheduler Integration

**Strategy**: KubeVirtRelieveAndMigrate

**Normal Operation** (<80% saturation):
- Monitors real-time memory and CPU utilization metrics
- Rebalances based on utilization thresholds

**High Saturation** (>80%):
- Switches to PSI (Pressure Stall Information) metrics
- PSI measures "time workloads spend waiting for memory operations"
- Live-migrates VMs from saturated to underutilized nodes
- Applies soft taint (PreferNoSchedule) to saturated nodes

**Asymmetric Deviation Thresholds**:
- Prevents migration churn
- Requires significant pressure delta before triggering migration

### Risks and Mitigations

**Risk: Node OOM despite overcommit**  
*Mitigation*: Defense-in-depth (FPR, MGLRU, descheduler, node-pressure eviction). Autopilot sets conservative hard limits.

**Risk: Swap thrashing degrading VM performance**  
*Mitigation*: MGLRU Bloom filters prevent re-swapping. Descheduler migrates VMs before thrashing. PSI metrics detect thrashing early. Recommend SSD/NVMe for swap.

**Risk: KSM CPU overhead**  
*Mitigation*: KSM zero-only is faster and less CPU-intensive than full KSM. Only scans for zero pages.

**Risk: Over-aggressive autopilot settings**  
*Mitigation*: Autopilot uses conservative defaults. Manual override available via memoryOvercommitPercentage.

**Risk: Descheduler migration churn**  
*Mitigation*: Asymmetric deviation thresholds. Soft taint prevents immediate re-scheduling. PSI metrics reduce false positives.

### Drawbacks

Increased complexity. Swap I/O slower than RAM. Autopilot calculations opaque. Performance unpredictability under high overcommit. Incompatible with strict performance SLAs.

## Alternatives (Not Implemented)

### Alternative 1: WASP

Deprecated in favor of MGLRU. WASP was a dedicated swap component. MGLRU provides:
- Better integration with Kubernetes cgroups
- Per-VM tracking and accounting
- Bloom filter thrashing prevention
- Lower CPU overhead

### Alternative 2: Traditional Balloon Inflation

Replaced by FPR. Traditional ballooning forces allocation inside guest, causing:
- Performance degradation during spikes
- Latency spikes
- Conflicts with guest memory compression
FPR observes already-freed pages asynchronously.

### Alternative 3: Full KSM

Replaced by KSM zero-only. Full KSM security/performance concerns:
- Arbitrary page content exposure risk
- Slows memory operations
- High CPU overhead
KSM zero-only is guest-agnostic, secure, faster.

### Alternative 4: Manual Configuration

Rejected in favor of autopilot. MGLRU parameters complex and interdependent. Autopilot calculates based on cluster characteristics.

## Open Questions [optional]

None. Implementation completed in OpenShift 4.18-4.22.

## Test Plan

Tested across OpenShift 4.17 → 4.18 (MGLRU) → 4.22 (Autopilot, KSM zero).

### Unit Tests

- MGLRU parameter calculation
- Overcommit percentage application to pod requests
- Autopilot configuration logic

### Integration Tests

- FPR reporting guest free pages
- MGLRU generation transitions
- KSM zero-page merging
- Descheduler PSI threshold switching

### E2E Tests

**Memory Overcommit Density**:
1. Enable 150% overcommit
2. Create VMs totaling 1.5x node physical memory
3. Verify all scheduled and running
4. Verify pod requests reduced per formula

**FPR Reclamation**:
1. Create VM, allocate memory in guest
2. Free memory in guest
3. Verify FPR reports pages
4. Verify hypervisor reclaims

**MGLRU Swap**:
1. Create memory pressure
2. Verify MGLRU tiers cold pages to swap
3. Verify Bloom filters prevent re-swapping
4. Monitor swap usage metrics

**KSM Zero Pages**:
1. Enable KSM zero-only
2. Create VMs with zero-filled pages
3. Verify pages merged
4. Verify ksm_pages_sharing metric

**Autopilot Configuration**:
1. Enable autopilot annotation
2. Verify MGLRU parameters configured
3. Verify descheduler thresholds set
4. Verify hard limits applied

**Descheduler PSI Rebalancing**:
1. Create >80% memory saturation on node
2. Verify descheduler switches to PSI metrics
3. Verify VM live-migrated
4. Verify saturated node soft-tainted

### Performance Testing

- Measure swap I/O impact on VM latency
- Measure MGLRU CPU overhead
- Measure KSM zero-page CPU overhead
- Density improvements vs. performance degradation
- PSI metric accuracy vs. traditional utilization

### Scalability Testing

- 100 VMs with 2x overcommit
- Autopilot calculation time at scale
- Descheduler rebalancing overhead
- Metrics system overhead

## Graduation Criteria

### Dev Preview -> Tech Preview (OpenShift 4.17-4.18)

- MGLRU implementation replacing legacy LRU
- FPR enabled by default
- Basic memory overcommit percentage via HyperConverged CR
- Documentation for manual configuration

### Tech Preview -> GA (OpenShift 4.22)

- Virtualization platform autopilot
- KSM zero-page merging
- Descheduler PSI integration
- Full E2E test coverage
- Performance and scalability validation
- Runbooks for troubleshooting
- Field testing with partners

## Upgrade / Downgrade Strategy

### Upgrade (4.17 → 4.18 → 4.22)

**4.17 → 4.18**:
- MGLRU replaces legacy LRU automatically
- FPR enabled by default (requires virtio driver in guest)
- Existing VMs continue running
- No configuration migration needed

**4.18 → 4.22**:
- Autopilot available as opt-in annotation
- KSM zero-only available as annotation
- Existing overcommit percentage settings preserved
- No automatic changes unless autopilot enabled

### Downgrade (4.22 → 4.18)

- Remove autopilot annotation before downgrade
- Autopilot-configured MGLRU parameters persist (safe)
- Remove KSM zero-only annotation
- Manual memoryOvercommitPercentage settings preserved

## Version Skew Strategy

No version skew concerns. FPR requires guest driver (virtio_balloon), version-independent. MGLRU is host kernel feature, no guest dependencies. KSM zero-only is host-only.

## Operational Aspects of API Extensions

Uses existing HyperConverged CR. No new CRDs. Annotations are metadata changes (no webhook or admission control needed).

**SLIs**:
- Existing virt-controller and virt-handler metrics
- New metrics for swap, KSM, PSI (see Observability section)

**Impact**: Minimal overhead. FPR asynchronous. MGLRU CPU overhead lower than legacy LRU. KSM zero-only faster than full KSM.

## Support Procedures

### Detecting Memory Overcommit Issues

**Symptom: High swap usage**

Check metrics:
```bash
oc get --raw /metrics | grep kubevirt_swap
```

Check node-level swap:
```bash
oc debug node/<node> -- chroot /host free -h
```

**Symptom: PSI metrics high**

Check descheduler events:
```bash
oc get events -n openshift-kube-descheduler-operator | grep Migrat
```

**Symptom: VMs experiencing slow performance**

Check if VM swapping:
```bash
oc exec -it virt-launcher-<vm> -- cat /sys/fs/cgroup/memory.swap.current
```

### Disabling Memory Overcommit

Set overcommit percentage to 100 (1:1 allocation):
```bash
oc patch hyperconverged kubevirt-hyperconverged -n openshift-cnv \
  --type='merge' -p='{"spec":{"virtualization":{"memoryOvercommitPercentage":100}}}'
```

Disable autopilot:
```bash
oc annotate hyperconverged kubevirt-hyperconverged \
  -n openshift-cnv platform.kubevirt.io/autopilot="false"
```

Disable KSM zero-only:
```bash
oc annotate hyperconverged kubevirt-hyperconverged \
  -n openshift-cnv platform.kubevirt.io/enable-ksm-zero-only="false"
```

### Observability

**Metrics** (available in Prometheus):

```
# Memory overcommit
kubevirt_memory_overcommit_percentage
kubevirt_vm_memory_guest_bytes{vm="...",namespace="..."}
kubevirt_vm_memory_request_bytes{vm="...",namespace="..."}

# Swap
kubevirt_swap_total_bytes{node="..."}
kubevirt_swap_used_bytes{node="..."}
kubevirt_vm_swap_used_bytes{vm="...",namespace="..."}

# KSM
kubevirt_ksm_pages_sharing{node="..."}
kubevirt_ksm_pages_shared{node="..."}

# PSI (Pressure Stall Information)
kubevirt_psi_memory_some_seconds_total{node="..."}
kubevirt_psi_memory_full_seconds_total{node="..."}

# Descheduler
descheduler_vm_migrations_total{reason="MemoryPressure"}
```

**Alerts**:

```yaml
- alert: MemoryOvercommitSwapHigh
  expr: kubevirt_swap_used_bytes / kubevirt_swap_total_bytes > 0.8
  annotations:
    summary: "Node swap >80%, consider rebalancing or adding capacity"

- alert: VMSwapThrashing
  expr: rate(kubevirt_vm_swap_in_bytes[5m]) > 10485760  # 10MB/s
  annotations:
    summary: "VM swapping heavily, performance degraded"

- alert: PSIMemoryPressureHigh
  expr: rate(kubevirt_psi_memory_full_seconds_total[5m]) > 0.1
  annotations:
    summary: "Node experiencing memory pressure, descheduler may migrate VMs"
```

## Infrastructure Needed [optional]

No additional infrastructure. Uses existing:
- HyperConverged CR from openshift-cnv operator
- Descheduler operator
- Prometheus for metrics
