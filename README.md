# CXL-page-migration
Performance characterization of AutoNUMA, TPP, and DAMON under emulated CXL memory tiering in QEMU — four workloads, custom Linux 6.11 kernel

# Hot Page Tracking under CXL Memory Tiering

**Course:** EECE5643 — Simulation and Performance Evaluation, Northeastern University
**Authors:** Jashwanth, Venkatesh, Sathwik

---

## Overview

This project characterizes three Linux kernel hot-page migration tools — AutoNUMA (PTE-scan), TPP, and DAMON_RECLAIM — on an emulated two-tier memory system built inside QEMU. One node holds fast DRAM, the other emulates slower CXL memory. Four workloads are run across all three tools to identify which performs best and under what conditions.

No physical CXL hardware is required. Everything runs in a QEMU VM with a custom Linux 6.11 kernel and is fully reproducible.

---

## Key findings

- The CXL penalty is not a fixed number. It ranges from nearly zero on streaming workloads to **5.82×** on random graph traversal, depending entirely on access pattern.
- Migration tools can exceed the raw hardware ratio. STREAM achieves a **3.28× speedup** from a 1.73× hardware gap, because promoting pages to DRAM also warms the L3 cache.
- No single tool wins on every workload. TPP is the safest default; PTE-scan is better for Zipfian-skewed access; DAMON combined with PTE-scan gives a 10% improvement on graph traversal.
- DAMON without real DRAM pressure actively hurts performance. On the KV workload it fell below the all-Node-2 baseline by churning hot pages unnecessarily.

---

## Testbed

| | Node 0 (DRAM) | Node 2 (CXL) |
|---|---|---|
| Capacity | 4 GB | 4 GB |
| Latency | 10 ns | 100 ns |
| Bandwidth | 30 GB/s | 7.5 GB/s |
| Backend | anonymous RAM | `/dev/shm` tmpfs |

Host: AMD Ryzen 7 6800U, Ubuntu 24.04. VM: QEMU q35 with HMAT enabled.

---

## Workloads

| # | Name | Size | Pattern |
|---|---|---|---|
| W1 | Pointer-chase | 256 MB | 90% hot / 10% cold, random |
| W2 | STREAM | 3 × 128 MB | Sequential copy/triad |
| W3 | KV Zipfian | 238 MB | Zipf s=0.99, ~2.4 MB hot set |
| W4 | Graph traversal | ~160 MB | Random walk, hub-leaf structure |

---

## Latency breakdown

The 411 ns Node 2 access time breaks down into three components:

| Component | Latency | Notes |
|---|---|---|
| Bare DRAM | 121 ns | Same on both nodes |
| KVM page-table walk | 117 ns | Virtualization overhead |
| CXL fabric penalty | **173 ns** | The only part any migration tool can recover |

---

## Tool recommendation

| Workload type | Best tool | Why |
|---|---|---|
| Latency-bound, irregular | TPP | Fewer false promotions |
| Sequential / streaming | TPP | Fastest cache warming |
| Zipfian-skewed KV | PTE-scan | DAMON causes thrash here |
| Graph with hub-leaf structure | DAMON + PTE | Push-pull: promote hubs, demote leaves |

Do not enable demotion unless there is measured DRAM pressure.

---

## Reproducibility notes

- `CONFIG_CXL_REGION_INVALIDATION_TEST=y` must be compiled into the kernel — region commit fails silently without it
- `auto_online_blocks` must be set to `online_movable` before creating the CXL region, or pages land in `ZONE_NORMAL` and cannot be migrated
- `get_mempolicy()` reports wrong node IDs for Node 2 pages in Linux 6.11 — use `move_pages()` instead
- QEMU silently drops HMAT entries if bandwidth values are not MiB-aligned

---

## Report

Full write-up with setup scripts, kernel config, migration diagnostics, and result tables:
[EECE5643_ProjectReport.pdf](./EECE5643_ProjectReport.pdf)
