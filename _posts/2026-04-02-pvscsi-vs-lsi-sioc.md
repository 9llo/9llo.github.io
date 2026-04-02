---
title: "PVSCSI vs LSI on vSphere: does the controller choice actually matter?"
date: 2026-04-02 18:00:00 -0300
categories: Storage VMware
tags: vsphere vmware storage fio benchmark pvscsi lsi sioc performance virtualization # TAG names should always be lowercase
description: Five VMs, two controllers, SIOC on and off. The numbers show that what you think is a controller problem is usually a storage fairness problem.
---

The VMware docs say to use PVSCSI for storage-intensive workloads. The internet agrees. And it's not wrong — but after running the same fio workload across five concurrent VMs with both controllers, the controller type turned out to be the less interesting variable. SIOC was doing all the heavy lifting.

This is a follow-up to [the fio_benchmark post](https://9llo.github.io/posts/fio_benchmark/). Same script, same 200 GiB virtual disks, five VMs hitting storage simultaneously across four configurations: LSI without SIOC, LSI with SIOC, PVSCSI without SIOC, and PVSCSI with SIOC.

---

## The test setup

Five VMs (`gus_fio_1` through `gus_fio_5`), all on the same vSphere cluster, all running `fio_benchmark` simultaneously against a dedicated 200 GiB virtual disk. Each test ran for 300 seconds per profile. The target was `/dev/sdb` — a dedicated disk with no OS or data on it.

**VM configuration:** Ubuntu Server 24.04, 2 vCPU, 4 GB RAM. Identical across all five VMs for each test run.

Each VM ran the following command simultaneously:

```bash
sudo ./fio_benchmark.sh -t /dev/sdb -r 300 -s -m
```

`-r 300` sets 300 seconds per test profile, `-s` runs the iodepth sweep, `-m` runs the RW-mix sweep.

Four configurations:
- **LSI Logic SAS (LSI)** — the default controller vSphere assigns when you click through the VM creation wizard
- **VMware Paravirtual (PVSCSI)** — VMware's purpose-built paravirtual SCSI controller
- Each repeated with **SIOC (Storage I/O Control) disabled and enabled**

> **WARNING — fio_benchmark writes directly to the block device you specify.** Running it against any disk containing data will destroy that data immediately. Test disks only.
{: .prompt-danger }

---

## Without SIOC: the storage lottery

With five VMs running simultaneously and SIOC off, what you get is a race. Whoever submits I/O first gets the queue — and the results reflect exactly that.

### Sequential 128K read — LSI, SIOC off

| VM | Bandwidth | Latency (avg) |
|---|---|---|
| gus_fio_1 | 401 MiB/s | 9.95 ms |
| gus_fio_2 | **1,148 MiB/s** | 3.47 ms |
| gus_fio_3 | 203 MiB/s | 19.63 ms |
| gus_fio_4 | 326 MiB/s | 12.22 ms |
| gus_fio_5 | 204 MiB/s | 19.60 ms |

gus_fio_2 pulled 1,148 MiB/s while gus_fio_3 and gus_fio_5 were stuck at ~204 MiB/s. Five identical VMs, same workload, same datastore — and one of them got **5.6× more bandwidth** than another. Not a configuration difference. Just queue timing.

The latency spread confirms it: gus_fio_2 at 3.47 ms average, gus_fio_3 at 19.63 ms. The "winner" VM was completing I/O 5.7× faster because it had the queue.

PVSCSI has the same problem:

### Sequential 128K read — PVSCSI, SIOC off

| VM | Bandwidth | Latency (avg) |
|---|---|---|
| gus_fio_1 | 191 MiB/s | 20.85 ms |
| gus_fio_2 | 382 MiB/s | 10.44 ms |
| gus_fio_3 | **648 MiB/s** | 6.15 ms |
| gus_fio_4 | 382 MiB/s | 10.45 ms |
| gus_fio_5 | 191 MiB/s | 20.85 ms |

Different winner, same dynamic. gus_fio_3 grabbed 648 MiB/s while gus_fio_1 and gus_fio_5 sat at 191 MiB/s — a 3.4× gap. The controller changed; the unfairness didn't.

This is what happens when multiple VMs share storage without any fairness mechanism. It's not predictable, it's not proportional, and it changes every time you run the benchmark.

### The iodepth sweep reveals another layer

The iodepth sweep (4K randread, single job, stepping from depth 1 to 128) shows something interesting about how the different VMs behave under contention.

**gus_fio_4 iodepth sweep — LSI, SIOC off:**

| iodepth | IOPS | Avg Latency |
|---|---|---|
| 16 | 32,470 | 0.49 ms |
| 32 | 40,120 | 0.80 ms |
| 64 | 40,319 | 1.59 ms |
| 128 | 40,735 | 3.14 ms |

gus_fio_4 could keep scaling well past iodepth 32 and held over 40K IOPS through depth 128. Meanwhile:

**gus_fio_2 iodepth sweep — LSI, SIOC off:**

| iodepth | IOPS | Avg Latency |
|---|---|---|
| 16 | 22,399 | 0.71 ms |
| 32 | 30,885 | 1.03 ms |
| 64 | **19,845** | 3.22 ms |
| 128 | 19,274 | 6.64 ms |

gus_fio_2 peaked at depth 32 (30,885 IOPS) then fell off a cliff at depth 64 — down to 19,845 IOPS. That's the storage queue saturating: when gus_fio_2 pushed too many concurrent I/Os, they backed up and latency climbed enough to reduce effective throughput. The command queue hit its ceiling, and adding more depth made things worse.

This is the most useful diagnostic the sweep provides. If your IOPS curve peaks then drops as you increase depth, you've found the point where your storage (or hypervisor queue) is saturated.

---

## What SIOC actually does

SIOC (Storage I/O Control) adds a congestion management layer at the datastore level. When it detects that latency is climbing above a threshold (configurable, defaults to 30ms for HDD datastores), it starts throttling VMs that are issuing disproportionately large amounts of I/O.

### Sequential 128K read — LSI, SIOC on

| VM | Bandwidth | Latency (avg) |
|---|---|---|
| gus_fio_1 | 406 MiB/s | 9.82 ms |
| gus_fio_2 | 401 MiB/s | 9.95 ms |
| gus_fio_3 | 292 MiB/s | 13.67 ms |
| gus_fio_4 | 331 MiB/s | 12.07 ms |
| gus_fio_5 | 293 MiB/s | 13.64 ms |

gus_fio_2's 1,148 MiB/s became 401 MiB/s — a 65% reduction. gus_fio_3, which had been starved to 203 MiB/s, came up to 292 MiB/s. The range compressed from **203–1,148 MiB/s down to 292–406 MiB/s**.

SIOC didn't increase total aggregate throughput — the datastore has the same physical capacity either way. What it did was stop one VM from taking 5× the fair share.

gus_fio_3 and gus_fio_5 still lag behind gus_fio_1 and gus_fio_2 even with SIOC enabled. This is consistent across all test runs and suggests they're on a different physical disk or VMFS extent within the same datastore — a placement issue that SIOC can't fix.

PVSCSI with SIOC shows the same pattern:

### Sequential 128K read — PVSCSI, SIOC on

| VM | Bandwidth | Latency (avg) |
|---|---|---|
| gus_fio_1 | 285 MiB/s | 14.00 ms |
| gus_fio_2 | **568 MiB/s** | 7.02 ms |
| gus_fio_3 | 576 MiB/s | 6.93 ms |
| gus_fio_4 | 569 MiB/s | 7.01 ms |
| gus_fio_5 | 285 MiB/s | 14.02 ms |

SIOC compressed the PVSCSI variance from a 3.4× spread (191–648 MiB/s) down to 2× (285–576 MiB/s). gus_fio_1 and gus_fio_5 are still consistently half the bandwidth of the other three — the same two VMs that lagged with LSI. The controller changed; the placement didn't.

### The RW-mix sweep — SIOC's fairness is most visible here

The RW-mix sweep (4K randrw, 8 jobs, iodepth 8, stepping write % from 0% to 100%) shows how IOPS degrade as write pressure increases. The write penalty curve tells you a lot about storage health.

**LSI, SIOC off — gus_fio_4 RW-mix sweep:**

| Write % | IOPS | Avg Latency |
|---|---|---|
| 0% (pure read) | 40,585 | 1.56 ms |
| 50% | 33,997 | 1.92 ms |
| 100% (pure write) | 29,838 | 2.13 ms |

Clean, gradual degradation. gus_fio_4 was the lucky VM in this run.

**LSI, SIOC off — gus_fio_3 RW-mix sweep:**

| Write % | IOPS | Avg Latency |
|---|---|---|
| 0% | 19,189 | 3.32 ms |
| 50% | 16,043 | 4.05 ms |
| 100% | 14,888 | 4.29 ms |

gus_fio_3 started at 19K IOPS where gus_fio_4 started at 40K. Same datastore, same moment in time. Without SIOC, the spread between the "winning" and "losing" VM at pure read is more than 2× — and that gap persists across the entire write penalty curve.

---

## PVSCSI vs LSI: what the numbers actually say

The paravirtual SCSI controller eliminates hardware emulation overhead. Instead of the hypervisor pretending to be an LSI Logic SAS card, PVSCSI uses a shared ring buffer that the guest driver talks to directly. Fewer context switches, more efficient I/O batching, theoretically better throughput.

In practice, with these VMs and this datastore, the difference was workload-specific.

### Throughput and latency (gus_fio_1 only, SIOC off)

| Test | Controller | IOPS | Bandwidth | Avg Latency |
|---|---|---|---|---|
| 4K rand read | LSI | 19,238 | 75.2 MiB/s | 6.63 ms |
| 4K rand read | PVSCSI | 20,193 | 78.9 MiB/s | 6.32 ms |
| 4K rand write | LSI | 17,357 | 67.8 MiB/s | 3.67 ms |
| 4K rand write | PVSCSI | 15,451 | 60.4 MiB/s | 4.13 ms |
| Random write | LSI | 5,146 | 321.6 MiB/s | 12.37 ms |
| Random write | PVSCSI | 1,653 | 103.3 MiB/s | 38.69 ms |
| Seq write | LSI | 8,960 | 280.0 MiB/s | 1.76 ms |
| Seq write | PVSCSI | 5,450 | 170.3 MiB/s | 2.92 ms |

Random reads are essentially a wash. The write picture is harder to read cleanly: 4K random writes are close, but the larger-block random write (64K bs, 8 jobs) shows PVSCSI at 3× fewer IOPS and 3× higher latency than LSI. Sequential writes follow the same direction — PVSCSI about 40% lower bandwidth.

These tests ran simultaneously across all five VMs, so the results aren't just a controller comparison — they're a controller-under-shared-contention comparison. The I/O batching behavior of PVSCSI's ring buffer can shift contention dynamics when multiple VMs hit the same datastore concurrently.

### CPU overhead: where PVSCSI actually delivers

The throughput picture above is mixed. The CPU picture is not.

| Test | Controller | CPU (usr / sys) | IOPS/CPU% |
|---|---|---|---|
| Seq read | LSI | 0.68% / 2.09% | 8,563 |
| Seq read | PVSCSI | 0.48% / 1.51% | 9,851 |
| Seq write | LSI | 2.85% / 2.10% | 1,807 |
| Seq write | PVSCSI | 1.01% / 1.00% | 2,701 |
| 4K rand read | LSI | 0.26% / 0.77% | 18,640 |
| 4K rand read | PVSCSI | 0.20% / 0.63% | 24,260 |
| 4K rand write | LSI | 0.75% / 1.40% | 8,104 |
| 4K rand write | PVSCSI | 0.44% / 1.01% | 10,666 |

PVSCSI consistently burns less CPU per IOPS across every test:
- Sequential write: LSI uses 4.95% total CPU for 8,960 IOPS. PVSCSI uses 2.01% total CPU for 5,450 IOPS. The bandwidth regression is real, but so is the 59% CPU reduction.
- 4K random read: PVSCSI is 30% more efficient (24,260 vs 18,640 IOPS/CPU%). Here PVSCSI also wins on throughput, so it's a clean win.
- 4K random write: 31% better efficiency on PVSCSI, and nearly equivalent throughput.

This is the original promise of the paravirtual driver: eliminate the emulation overhead, get more I/O per CPU cycle. The CPU numbers confirm it works. Whether the throughput trade-off on large-block writes matters depends on your workload.

---

## The main takeaways

**SIOC matters more than controller type.** Without SIOC, a single VM in a five-VM cluster pulled 5.6× more bandwidth than its neighbors. With SIOC, the variance compressed to roughly 1.4×. If you're running multiple storage-intensive VMs on a shared datastore and SIOC is disabled, you have a fairness problem regardless of what controller you're using.

**PVSCSI's real advantage is CPU efficiency, not raw throughput.** The throughput differences between controllers are workload-specific and not consistently in PVSCSI's favor. What is consistent: PVSCSI burns 30–60% less CPU per IOPS across every test. On a host running many storage-heavy VMs, that overhead adds up. If you're CPU-constrained, PVSCSI matters even when the IOPS numbers look similar.

**The iodepth sweep is diagnostic, not just informational.** The queue saturation behavior (IOPS peak then drop as depth increases) tells you where your storage ceiling is. gus_fio_2 peaked at depth 32 and degraded from there — that's the point where adding more queue depth hurts rather than helps.

**The practical recommendation:** Enable SIOC on every shared datastore — the fairness argument is unambiguous regardless of controller. For controller choice, PVSCSI is the right default: the CPU efficiency win is consistent across all workloads, and for most mixed or read-heavy workloads the throughput numbers are equivalent or better. The one case worth benchmarking before committing is large-block bulk writes (backups, ETL, video ingest) — the regression there was significant enough that if that's your primary workload, you should verify it on your specific storage before assuming PVSCSI is the right call.

**VM placement inside a datastore has real effects.** gus_fio_3 and gus_fio_5 consistently underperformed gus_fio_1 and gus_fio_2 across every configuration, including with SIOC. If VMs on the same datastore are getting systematically different performance, suspect disk extent or spindle layout before blaming the controller or the guest.

---

All tests used [fio_benchmark](https://github.com/9llo/fio_benchmark) — the benchmark script covered in the [previous post](https://9llo.github.io/posts/fio_benchmark/). 300-second runtime per test, libaio, direct I/O, no filesystem caching.

Raw results from all five VMs across all four configurations are available for download if you want to verify the numbers or run your own analysis: [results_fio.txt](/assets/files/results_fio.txt).
