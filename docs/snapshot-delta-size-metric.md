# Snapshot Delta Size Metric — Research & Proposal

## Summary

This document captures research into exposing VMware snapshot delta disk sizes as a Prometheus metric via the vmware_exporter. Snapshot delta growth is a key indicator of both performance degradation and datastore capacity risk.

## Why Monitor Snapshot Delta Size?

### Performance Impact

VMware snapshots use a copy-on-write mechanism with delta (redo) files. Every write to a snapshotted VM goes to the newest delta file, and every read must traverse the delta chain from newest to oldest until the requested block is found. This has a direct and measurable performance cost:

- **Up to 85% lower IOPS** and significantly higher latencies observed in VMware's own benchmarks (FIO, HammerDB) for VMs with snapshots [1].
- **The first snapshot has the greatest impact**, particularly on VMFS-backed storage [2].
- **Each additional snapshot in the chain compounds the penalty** — VMware recommends a maximum of 2–3 snapshots per chain [3].
- Delta files grow in 16 MB increments and can reach the size of the original base disk, increasing fragmentation and I/O overhead [4].

### Operational Risk

- **Stale snapshots** silently accumulate writes, growing delta files that consume datastore space.
- **Datastore capacity exhaustion** can occur when delta files grow unchecked, potentially affecting all VMs on the same datastore.
- **Snapshot consolidation** of very large deltas is itself an expensive operation that can cause additional performance disruption.

### VMware's Official Guidance

- Snapshots should be temporary — delete within 24–72 hours [3].
- Keep snapshot chains to 2–3 levels maximum [3].
- Consolidate early and often [3].
- Never use snapshots as a long-term backup strategy [3].

## How the vSphere API Exposes Delta Size

The `VirtualMachineSnapshotTree` object (currently used by the exporter) does **not** include size information. Delta sizes must be obtained through a separate API path:

### Required API Properties

| Property | Purpose | Available Since |
|---|---|---|
| `vm.layoutEx` (`VirtualMachineFileLayoutEx`) | Extended file layout with sizes | vSphere API 4.0 |
| `layoutEx.file[]` | List of all VM files with `size` (bytes), `type`, and `name` | vSphere API 4.0 |
| `layoutEx.snapshot[]` | Maps snapshot keys to their associated file keys | vSphere API 4.0 |

### Relevant File Types (`vim.vm.FileLayoutEx.FileType`)

| Type | Description | Available Since |
|---|---|---|
| `diskExtent` | Includes `-delta.vmdk` files (the actual delta disks) | vSphere API 4.0 |
| `snapshotData` | `.vmsn` snapshot state files | vSphere API 4.0 |
| `snapshotMemory` | `.vmem` snapshot memory dump files | vSphere API 6.0 |

### Correlation Logic

To compute per-snapshot delta size:

1. Fetch `layoutEx` property alongside the existing `snapshot` property.
2. Use `layoutEx.snapshot` to map each snapshot's `key` to its list of file keys (`dataKey` / `memoryKey`).
3. Look up each file key in `layoutEx.file` to get the `size` in bytes.
4. Alternatively, for total snapshot disk usage, filter `layoutEx.file` entries where `type == 'snapshotData'` or the filename matches the delta disk pattern (`0000\d\d`).

```python
# Simplified example
import re

def get_snapshot_disk_usage(vm):
    size = 0
    for file_info in vm.layoutEx.file:
        if file_info.type == 'snapshotData':
            size += file_info.size
        elif re.search(r'0000\d\d', file_info.name):
            size += file_info.size
    return size
```

### Compatibility

`vm.layoutEx` has been available since vSphere API 4.0 (ESXi 4.0, 2009). All currently supported vSphere versions (6.5, 6.7, 7.0, 8.0) fully support this property. The only version-gated type is `snapshotMemory` (requires 6.0+), which is optional for this use case.

## Current Exporter State

The exporter currently collects two snapshot metrics:

- `vmware_vm_snapshots` — total count of snapshots per VM
- `vmware_vm_snapshot_timestamp_seconds` — creation timestamp per snapshot

These are collected by fetching the `snapshot` property from `VirtualMachine` objects (see `vmware_exporter.py` lines 761–762, 1158–1169, 1611–1623).

## Proposed New Metric

- **`vmware_vm_snapshot_delta_size_bytes`** — delta disk size in bytes, per snapshot
  - Labels: `vm_name`, `ds_name`, `host_name`, `dc_name`, `cluster_name`, `vm_snapshot_name`

### Trade-offs

| Consideration | Detail |
|---|---|
| Additional API load | Fetching `layoutEx` for every VM adds one more property to the batch fetch |
| Implementation complexity | Requires correlating snapshot keys to file entries |
| Value | Directly signals stale/problematic snapshots; complements existing age metric |

## Metric Value in Practice

Combining delta size with the existing snapshot age metric enables alerting rules such as:

- **Alert on large deltas**: snapshot delta exceeds a size threshold
- **Alert on old + large snapshots**: snapshot is both stale and large
- **Capacity planning**: track delta size growth rate over time to predict datastore exhaustion

### Sample PromQL Alert Expressions

#### Alert when any snapshot delta exceeds 10 GB

```promql
vmware_vm_snapshot_delta_size_bytes > 10 * 1024 * 1024 * 1024
```

#### Alert when a snapshot is older than 72 hours AND its delta exceeds 5 GB

```promql
(time() - vmware_vm_snapshot_timestamp_seconds) > 72 * 3600
and on (vm_name, vm_snapshot_name)
vmware_vm_snapshot_delta_size_bytes > 5 * 1024 * 1024 * 1024
```

#### Alert when total snapshot delta per VM exceeds 50 GB

```promql
sum by (vm_name, dc_name, cluster_name) (vmware_vm_snapshot_delta_size_bytes) > 50 * 1024 * 1024 * 1024
```

#### Alert when a VM has a snapshot chain longer than 3 (VMware recommended max)

```promql
vmware_vm_snapshots > 3
```

#### Track delta growth rate (bytes/hour, useful for capacity planning)

```promql
rate(vmware_vm_snapshot_delta_size_bytes[1h]) > 0
```

## Sources

1. [Performance Best Practices for VMware Snapshots — VMware Blog](https://blogs.vmware.com/cloud-foundation/2021/06/16/performance-best-practices-for-vmware-snapshots/)
2. [Performance Impact of Snapshots in VMware vSphere 7 — 4sysops](https://4sysops.com/archives/performance-impact-of-snapshots-in-vmware-vsphere-7/)
3. [Best Practices for Using VMware Snapshots — Broadcom KB](https://knowledge.broadcom.com/external/article/318825/best-practices-for-using-vmware-snapshot.html)
4. [VMware Snapshots Explained: Internals, Pitfalls, and Deep Dive — DEV Community](https://dev.to/mohideen_sahib_79f5f9e8de/vmware-snapshots-explained-internals-pitfalls-and-deep-dive-into-base-delta-mechanics-301k)
5. [VMware vSphere Snapshots: Performance and Best Practices — VMware Docs](https://www.vmware.com/docs/vsphere-vm-snapshots-perf)
6. [Snapshot Storage Size — pyVmomi Issue #241](https://github.com/vmware/pyvmomi/issues/241)
7. [FileLayoutEx — vSphere API Reference](https://dp-downloads.broadcom.com/api-content/apis/API_VMA_001/8.0U2/html/vim.vm.FileLayoutEx.html)
8. [FileLayoutEx.FileType Enum — vSphere API Reference](https://dp-downloads.broadcom.com/api-content/apis/API_VMA_001/8.0U2/html/vim.vm.FileLayoutEx.FileType.html)
