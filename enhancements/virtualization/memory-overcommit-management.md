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

This enhancement implemented memory overcommit capabilities for OpenShift Virtualization, enabling clusters to run more virtual machines than available physical memory through statistical multiplexing. It extends OpenShift Virtualization's memory architecture from native DRAM (Tier 0) to include swap-backed secondary storage (Tier 2), part of a broader three-tier memory model vision that will eventually include CXL memory expansion (Tier 1).

The implementation uses defense-in-depth mechanisms to AVOID swap traffic while keeping swap available as a safety net.

Primary mechanisms work proactively to prevent swapping:
- **Free Page Reporting (FPR)**: Asynchronous reclamation of unused pages
- **Descheduler with PSI**: Live-migrates VMs before memory pressure causes swapping
- **MGLRU**: Modern swap with thrashing prevention (when swap unavoidable)
- **KSM Zero Pages**: Deduplicates zero-filled pages across VMs

The virt-platform-autopilot component automates configuration to minimize swap usage while maintaining the safety net for worst-case scenarios.

## Motivation

### Three-Tier Memory Architecture Vision

OpenShift Virtualization is architecting toward a three-tier memory model to provide flexible memory hierarchy options:

- **Tier 0 (DRAM)**: Traditional memory, natively supported in OpenShift Virtualization (~100ns latency, baseline performance)
- **Tier 1 (CXL)**: Byte-addressable memory expansion via Compute Express Link, work in progress (~200ns latency, capacity expansion with near-DRAM performance)
- **Tier 2 (Swap)**: Disk-backed secondary storage, **this proposal** (~ms latency, safety net for overcommit)

Each tier addresses different use cases with distinct constraints, requirements, and implications:
- **DRAM (Tier 0)**: Performance-critical workloads, no configuration needed, limited by physical capacity
- **CXL (Tier 1)**: Memory capacity expansion, requires CXL-capable hardware, transparent to workloads
- **Swap (Tier 2)**: Overcommit safety net, works with existing hardware, performance impact if heavily used

This enhancement focuses on **Tier 2 (Swap)** to enable memory overcommit. CXL (Tier 1) support is planned as future work with different optimization strategies focused on capacity expansion rather than overcommit.

### Memory Overcommit Need

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

- **Load-aware VM rebalancing and live migration**: Critical for safe overcommit but covered by separate [descheduler enhancement][descheduler-enhancement]. The descheduler with KubeVirtRelieveAndMigrate strategy is REQUIRED for production memory overcommit - it monitors PSI (Pressure Stall Information) metrics and live-migrates VMs from overloaded nodes BEFORE swapping occurs. This enhancement provides the mechanisms (FPR, MGLRU, KSM); descheduler provides the intelligence to prevent overcommit from causing memory pressure.
- Cross-node memory sharing or distributed memory management
- Memory compression (zswap/zram)
- Modifying Kubernetes pod memory management for non-VM workloads
- Single-Node OpenShift (SNO) support - deferred due to control plane stability requirements
- Full KSM (arbitrary page merging) - security and performance concerns

[descheduler-enhancement]: /enhancements/scheduling/load-aware-vm-rebalancing-through-descheduler.md

## Proposal

This enhancement implements layered defense to enable overcommit while AVOIDING swap traffic:

**Proactive Mechanisms** (prevent swapping):
1. **Free Page Reporting (FPR)**: Guest-cooperative asynchronous reporting of unused pages to hypervisor (requires virtio driver in guest, primary reclamation, enabled by default)
2. **Descheduler with PSI**: Live-migrates VMs when pressure detected, BEFORE swapping starts
3. **KSM Zero Pages**: Guest-agnostic deduplication of zero-filled pages across VMs (no guest driver required, reduces total memory footprint)
4. **Memory Overcommit Percentage**: Reduces pod memory requests to enable statistical multiplexing

**Safety Net** (graceful degradation when unavoidable):

5. **Swap (Secondary Memory Tier)**: Extends physical RAM with secondary storage (SSD/NVMe recommended). When physical memory exhausted, kernel moves pages to swap device. Swap is Tier 2 memory (~ms latency vs ~100ns for DRAM Tier 0).

6. **MGLRU (Node-level Memory Page Eviction Heuristic)**: Multi-Generational LRU algorithm running in the kernel at the node level, intelligently selecting WHICH memory pages to evict to swap. Targets cold pages (rarely accessed) for eviction while protecting hot pages (working set). Uses generational tracking and Bloom filters to prevent thrashing.

**Goal**: Swap exists but should rarely be used. FPR + descheduler + KSM work together to keep memory available without swapping. When memory pressure cannot be avoided:
- **Swap** provides the storage tier for evicted pages (extends physical memory)
- **MGLRU** provides the intelligence for selecting which pages to evict (targets cold pages)
- Together: Graceful degradation (not OOM kills) through targeted eviction to secondary storage

**Critical Dependency** - Load-Aware Rebalancing (see [descheduler enhancement][descheduler-enhancement]):

The descheduler with KubeVirtRelieveAndMigrate strategy is REQUIRED for safe production memory overcommit. While this enhancement provides the mechanisms (FPR, MGLRU, KSM, autopilot), the descheduler provides the intelligence:

- Monitors PSI (Pressure Stall Information) metrics at >80% utilization
- Live-migrates VMs from saturated nodes BEFORE swapping starts
- Soft-taints overloaded nodes (PreferNoSchedule)
- Enables proactive rebalancing instead of reactive swapping

**Without descheduler**: Memory overcommit works but relies on MGLRU swap as primary defense (slower, performance impact).

**With descheduler**: Memory overcommit avoids swapping through proactive live migration (swap becomes true safety net).

**Related Components**:
- virt-platform-autopilot: Tunes all mechanisms to minimize swap traffic
- Descheduler (KubeVirtRelieveAndMigrate): Provides load-aware rebalancing intelligence (separate enhancement)

### Workflow Description

#### Feature Enablement Evolution

| Feature | 4.22 | 5.0 | Future |
|---------|------|-----|--------|
| **Free Page Reporting (FPR)** | Enabled by default | Enabled by default | Enabled by default |
| **KSM Zero Pages** | Manual (annotation) | Autopilot | Autopilot |
| **Memory Overcommit %** | Manual (spec field) | Autopilot calculates | Autopilot |
| **MGLRU Parameters** | Autopilot | Autopilot | Autopilot |
| **Descheduler Thresholds** | Manual (separate config) | Autopilot | Autopilot |
| **Swap Configuration** | Manual (node-level) | Manual | Autopilot |
| **virt-platform-autopilot** | Manual enable (annotation) | Enabled by default | Enabled by default |

**Trend**: Moving from manual configuration toward autopilot-managed optimization based on cluster characteristics. Autopilot enabled by default starting in 5.0.

#### Enabling Memory Overcommit (Admin Workflow)

**OpenShift 5.0+** (autopilot enabled by default):

1. Autopilot automatically calculates and applies:
   - MGLRU parameters (based on node memory, workload patterns)
   - Memory overcommit percentage (default 20% at node level)
   - Descheduler thresholds for pressure-based rebalancing
   - Control group throttling limits

2. Administrator optionally configures custom overcommit percentage:
   ```bash
   oc patch hyperconverged kubevirt-hyperconverged -n openshift-cnv \
     --type='merge' -p='{"spec":{"virtualization":{"memoryOvercommitPercentage":150}}}'
   ```

3. VMs automatically benefit from FPR (enabled by default):
   - Guest OS reports free pages via virtio driver
   - Hypervisor reclaims pages asynchronously
   - No forced balloon inflation (unlike traditional ballooning)

**OpenShift 4.22** (autopilot requires manual enablement):

1. Administrator enables autopilot on HyperConverged resource:
   ```bash
   oc annotate hyperconverged kubevirt-hyperconverged \
     -n openshift-cnv platform.kubevirt.io/autopilot="true" --overwrite
   ```

2. Follow steps from 5.0+ workflow above (autopilot calculates parameters, optional custom percentage, FPR enabled by default)

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

| Feature | Normal (<70%) | Moderate (70-80%) | High (>80%) | Worst-case |
|---------|---------------|-------------------|-------------|------------|
| **FPR** | Continuously reports unused pages → hypervisor reclaims[^reclaim] | Continues reclaiming[^reclaim] | Continues reclaiming[^reclaim] | Continues reclaiming[^reclaim] |
| **KSM Zero Pages** | Merges zero-filled pages across VMs → reduces footprint | Continues merging | Continues merging | Continues merging |
| **Descheduler** | Monitors utilization metrics | Live-migrates[^migration] VMs from nodes trending upward | Switches to PSI metrics, aggressive migration[^migration], soft-taints saturated nodes (PreferNoSchedule) | Cannot keep up with pressure |
| **MGLRU** | Inactive (no eviction[^eviction] needed) | Inactive (reclaim[^reclaim] sufficient) | Inactive (migration[^migration] sufficient) | Selects cold pages for eviction[^eviction] using generational tracking + Bloom filters |
| **Swap** | Unused | Unused | Unused or minimal | Active - evicted[^eviction] pages stored in Tier 2 (graceful degradation, no OOM kills) |

[^reclaim]: **Reclamation**: Guest-cooperative process where VM reports unused memory pages to hypervisor via FPR. Pages returned to host without eviction or migration. See [virtio-balloon docs][virtio-balloon].

[^eviction]: **Eviction**: Kernel moving memory pages from DRAM (Tier 0) to swap device (Tier 2 secondary storage). MGLRU algorithm selects WHICH pages to evict based on access patterns. Pages moved to slower storage tier but VM continues running. See [MGLRU kernel docs][kernel-mglru].

[^migration]: **Migration**: Live-migration of entire VM from one physical node to another node in the cluster. Descheduler orchestrates VM migration to balance memory load across cluster. VM memory contents transferred over network, not paged to disk. See [descheduler enhancement][descheduler-enhancement].

**Goal**: Normal + Moderate + High pressure bands (first 3 columns) handle 99% of cases through reclamation[^reclaim] and rebalancing. Swap is safety net for worst-case column.

**Result by pressure band**:
- **Normal/Moderate**: Overcommit achieved WITHOUT swap traffic
- **High**: Swap minimally used or avoided entirely via aggressive migration
- **Worst-case**: Swap traffic occurs but VMs stay running; alerts trigger for operator intervention

**Important - Swap Persistence**: Once pages are evicted to swap during a pressure event, they remain in swap even after pressure subsides. No component actively migrates swap pages back to DRAM. Cold pages only return to DRAM on demand (page fault when accessed). This means:
- Swap utilization persists after pressure events until pages are accessed
- Node reboot clears swap and returns all memory to DRAM
- Monitoring swap usage over time reveals historical pressure patterns, not just current pressure

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
    # Enable autopilot for automated configuration (4.22 only - enabled by default in 5.0+)
    platform.kubevirt.io/autopilot: "true"
    # Enable KSM zero-page merging
    platform.kubevirt.io/enable-ksm-zero-only: "true"
spec:
  virtualization:
    # Memory overcommit percentage (optional - autopilot calculates if not specified in 5.0+)
    # Formula: pod requests = VM guest * 100 / memoryOvercommitPercentage
    # Example: 150 means 12Gi VM requests 8Gi (12 * 100 / 150)
    # Default: 100 (no overcommit) in 4.22; autopilot-calculated in 5.0+
    memoryOvercommitPercentage: 150
    
    # KSM configuration (alternative to annotation)
    # Not used when autopilot enabled with KSM zero-only annotation
    ksmConfiguration:
      nodeLabelSelector:
        matchLabels:
          kubevirt.io/ksm-enabled: "true"
```

**Behavior**:

- **autopilot annotation**: 
  - 4.22: Must be set to `"true"` to enable virt-platform-autopilot
  - 5.0+: Enabled by default; annotation only needed to disable (`"false"`)
  - Configures: MGLRU parameters, descheduler thresholds, memory limits
- **memoryOvercommitPercentage**: 
  - Reduces pod requests; scheduler can pack more VMs per node
  - 4.22: Manual configuration required
  - 5.0+: Autopilot calculates optimal value if not explicitly set
- **FPR**: Enabled by default for all VMs (no configuration needed)
- **KSM zero-only**: 
  - 4.22: Manual annotation required
  - 5.0+: Managed by autopilot
  - Merges zero-filled pages; guest-agnostic, secure

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

#### Swap - Secondary Memory Tier

**Basic Mechanism**: Extends physical RAM with secondary storage.

**Kubernetes Integration**: Enabled via [KEP-2400: Node Memory Swap Support][kep-2400], which provides native swap support in Kubernetes (alpha in 1.22, GA in 1.34). OpenShift 4.18+ includes this capability.

**How Swap Works**:
- Kernel moves pages from DRAM (Tier 0) to disk/SSD (Tier 2)
- Frees physical RAM for other uses
- Pages can be swapped back in when accessed (swap-in)

**Storage Tier Classification**:
- **Tier 0 (DRAM)**: ~100ns latency, primary memory
- **Tier 1 (CXL)**: ~200ns latency, byte-addressable expansion (future)
- **Tier 2 (Swap)**: ~ms latency, disk-backed secondary storage

**Swap Device Recommendations**:
- SSD/NVMe strongly recommended (not spinning disks)
- Lower latency = less performance impact when swapping occurs
- Future: CXL-attached memory may become Tier 1 alternative

**Configuration**:
- Swap devices provisioned per node (not per VM)
- Shared across all VMs on that node
- Size typically 1-2x node physical RAM

**Swap Persistence Behavior**:
- Pages evicted to swap during pressure events remain in swap after pressure subsides
- No active swap-to-DRAM migration mechanism exists
- Pages return to DRAM only on-demand when accessed (page fault triggers swapin)
- This means swap utilization reflects historical peak pressure, not current pressure
- Node reboot clears all swap and returns memory to DRAM
- Monitoring: Persistent swap usage indicates past pressure events; use PSI metrics for current pressure

#### MGLRU - Node-level Memory Page Eviction Heuristic

**Default since OpenShift 4.18**. Multi-Generational LRU (MGLRU) is a kernel-level algorithm running on each node, providing intelligent selection of WHICH memory pages to evict to swap.

**MGLRU vs Swap - Distinct Roles**:
- **Swap** = WHERE evicted pages go (secondary storage tier)
- **MGLRU** = WHICH memory pages to evict at the node level (kernel selection algorithm)
- Together: Intelligent swap that minimizes performance impact

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

**Enablement**:
- 5.0+: Enabled by default; no action required
- 4.22: Manual activation via annotation:
  ```bash
  oc annotate hyperconverged kubevirt-hyperconverged \
    -n openshift-cnv platform.kubevirt.io/autopilot="true"
  ```

**Deactivation** (5.0+ only, does not disable already-configured features):
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

Deprecated in favor of Kubernetes native swap support. WASP (Workload-Aware Swap Provisioning) was an alternative Kubernetes integration mechanism for swap. Replaced by upstream Kubernetes swap support via [KEP-2400: Node Memory Swap Support][kep-2400] (alpha in Kubernetes 1.22, GA in Kubernetes 1.34, available in OpenShift 4.18+).

Kubernetes native swap (KEP-2400) provides:
- Standardized upstream integration with Kubernetes cgroups v2
- Better compatibility with Kubernetes memory management
- Direct support in kubelet without additional components
- LimitedSwap mode: Burstable QoS pods get proportional swap access; Guaranteed/BestEffort pods get zero swap
- Aligned with upstream Kubernetes roadmap

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

Tested across OpenShift 4.17 → 4.18 (MGLRU) → 4.22 (Autopilot, KSM zero) → 5.0 (Autopilot by default).

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

**Recording Rules** (see [memory monitoring rules][memory-rules]):

```promql
# Node Memory Metrics
openshift:node:memory:bytes{scope, tier, utilized, temperature, usage}
openshift:node:memory:utilization:ratio
openshift:node:memory:pressure:ratio
openshift:node:memory:overcommit:ratio
openshift:node:memory:requests:bytes

# VM Memory Metrics  
openshift:vm:memory:bytes{tier, temperature}
openshift:vm:memory:utilization:ratio
openshift:vm:memory:overcommit:ratio
openshift:vm:virtual:memory:bytes
openshift:vm:memory:requests:bytes

# Cluster-wide Metrics
openshift:cluster:memory:utilization:ratio
openshift:cluster:memory:overcommit:ratio
openshift:cluster:memory:imbalance:distance{quantile="0.80"}
openshift:cluster:memory:requests:bytes

# Memory Tier Information
openshift:node:memory:tier:info{tier, technology, latency, addressable}
```

**Key Metrics by Tier** (see [memory monitoring reference][memory-rules-md]):

- **Tier 0 (DRAM)**: ~100ns latency, primary memory
- **Tier 1 (CXL)**: ~200ns latency, byte-addressable expansion (future)
- **Tier 2 (Swap)**: ~ms latency, disk-backed safety net

**Temperature Classification**:

- `hot`: Working set, recently accessed (high OOM risk if unavailable)
- `warm`: Inactive file cache (can be freed without OOM, but disk I/O impact)

**Utilization Targets** (see [memory monitoring reference][memory-rules-md]):

| Metric | Target Range | Interpretation |
|--------|--------------|----------------|
| `utilization` | 0.7 - 0.8 | Approaching 1.0 signals need for more capacity |
| `pressure` | < 0.1 | Detects contention before OOM |
| `overcommit` | < 1.5 | >1.0 is normal, high values increase OOM risk |

**Imbalance Severity** (cluster-wide):

| Distance (p80) | Status |
|----------------|--------|
| < 0.11 | Healthy balanced |
| 0.11-0.22 | Unhealthy balance |
| 0.22-0.33 | Severely unbalanced |
| > 0.33 | Critically unbalanced |

**Alert Conditions** (examples based on recording rules):

```yaml
# High VM Utilization
- alert: VMMemoryUtilizationHigh
  expr: openshift:vm:memory:utilization:ratio > 0.8
  for: 5m
  annotations:
    summary: "VM {{ $labels.name }} memory utilization >80%"

# High VM Overcommit
- alert: VMMemoryOvercommitHigh
  expr: openshift:vm:memory:overcommit:ratio > 1.5
  for: 10m
  annotations:
    summary: "VM {{ $labels.name }} overcommit >1.5x, elevated OOM risk"

# Node Pressure
- alert: NodeMemoryPressureHigh
  expr: openshift:node:memory:pressure:ratio > 0.1
  for: 5m
  annotations:
    summary: "Node {{ $labels.node }} memory pressure >10%, descheduler may rebalance"

# System Memory Overflow
- alert: SystemMemoryExceedsReservation
  expr: openshift:node:memory:bytes{scope="workloads", utilized="true", usage="system-overflow"} > 0
  for: 5m
  annotations:
    summary: "Node {{ $labels.node }} system memory borrowing from workloads"

# Cluster Imbalance
- alert: ClusterMemoryImbalanceSevere
  expr: openshift:cluster:memory:imbalance:distance{quantile="0.80"} > 0.22
  for: 15m
  annotations:
    summary: "Cluster memory severely unbalanced, descheduler should rebalance"

# Swap Usage (Tier 2)
- alert: NodeSwapUtilizationHigh
  expr: |
    openshift:node:memory:bytes{tier="2", utilized="true"} 
    / (openshift:node:memory:bytes{tier="2", utilized="true"} 
       + openshift:node:memory:bytes{tier="2", utilized="false"}) > 0.5
  for: 10m
  annotations:
    summary: "Node {{ $labels.node }} swap usage >50%, memory pressure safety net active"
```

## Infrastructure Needed [optional]

No additional infrastructure. Uses existing:
- HyperConverged CR from openshift-cnv operator
- Descheduler operator
- Prometheus for metrics and recording rules

## References

### Implementation Documentation

- [Memory Management in OpenShift Virtualization][rh-blog-2025] - Overview of FPR, MGLRU, and memory management mechanisms (Red Hat Developer Blog, January 2025)
- [Red Hat OpenShift Virtualization Memory Management Overview][rh-article-2026] - Detailed guide on memory overcommit, MGLRU, autopilot, and PSI metrics (Red Hat Developer Articles, October 2026)

### Monitoring and Observability

- [Memory Monitoring Rules Reference][memory-rules-md] - Detailed explanation of memory metrics, thresholds, and monitoring strategies
- [Memory Monitoring Rules (YAML)][memory-rules] - Prometheus recording rules for memory tier tracking, VM metrics, and cluster-wide monitoring

### Upstream Documentation

- [KEP-2400: Node Memory Swap Support][kep-2400] - Kubernetes Enhancement Proposal for native swap support (alpha in 1.22, GA in 1.34)
- [MGLRU (Multi-Generational LRU)][kernel-mglru] - Kernel documentation on MGLRU memory page eviction algorithm
- [virtio-balloon Free Page Reporting][virtio-balloon] - Kernel documentation on free page reporting mechanism

### Related Enhancements

- [Load-aware VM Rebalancing through Descheduler](/enhancements/scheduling/load-aware-vm-rebalancing-through-descheduler.md) - Descheduler integration with PSI metrics

[rh-blog-2025]: https://developers.redhat.com/blog/2025/01/31/memory-management-openshift-virtualization
[rh-article-2026]: https://developers.redhat.com/articles/2026/10/15/red-hat-openshift-virtualization-memory-management-overview
[memory-rules-md]: https://github.com/fabiand/perses-dashboards/blob/main/rules/memory.md
[memory-rules]: https://github.com/fabiand/perses-dashboards/blob/main/rules/memory.yaml
[kep-2400]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2400-node-swap
[kernel-mglru]: https://docs.kernel.org/mm/multigen_lru.html
[virtio-balloon]: https://docs.kernel.org/mm/free_page_reporting.html
