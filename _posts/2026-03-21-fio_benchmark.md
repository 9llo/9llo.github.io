---
title: "fio_benchmark: A wrapper to stop writing the same fio commands over and over"
date: 2026-03-20 10:00:00 -0300
categories: Storage Troubleshooting
tags: storage fio benchmark linux sysadmin performance # TAG names should always be lowercase
description: A Bash script that compiles the standard fio test profiles sysadmins and storage engineers run manually every time into a single repeatable tool — with structured output and optional sweeps for finding bottlenecks.
---

Every storage troubleshooting session starts the same way: you SSH into a box, pull up your notes (or browser history), and reconstruct the same handful of `fio` commands you've run a hundred times before. Sequential read, sequential write, 4K random read, mixed R/W — each with slightly different block sizes, queue depths, and job counts depending on what you're hunting.

That repetitive scaffolding is exactly what `fio_benchmark` eliminates.

> **WARNING — THIS SCRIPT WRITES DIRECTLY TO THE BLOCK DEVICE YOU SPECIFY.**
>
> Pointing it at any disk that contains data — your OS drive, a mounted volume, an in-use datastore — **will corrupt or destroy that data, immediately and without any confirmation prompt.** There is no undo.
>
> Before running, confirm your target with `lsblk` or `fdisk -l`. If `/dev/sda` is your system disk, `/dev/sdb` is your VM datastore, or you are not 100% certain what is on the device, **do not run this script against it.** Use a blank, dedicated test disk or a test file on a disposable filesystem (`-t /mnt/testdir/testfile`).
{: .prompt-danger }

---

## What it is (and what it isn't)

[fio_benchmark](https://github.com/9llo/fio_benchmark) is a Bash wrapper around `fio`. It doesn't replace `fio` — it doesn't try to. `fio` is already the right tool. What it does is compile the test profiles most sysadmins and storage professionals run separately into a single repeatable execution, with consistent parameters and structured output you can actually compare across systems.

The target audience is straightforward: if you're a sysadmin or storage engineer who needs to quickly characterize a device — whether it's a new NVMe on a bare-metal server, a SAN LUN, or a VM disk you suspect is being throttled — this gets you there without building the test matrix from scratch every time.

---

## The 9 profiles it runs

All tests use `--direct=1`, `--ioengine=libaio`, and `--group_reporting`. Random tests additionally disable repeat patterns (`--randrepeat=0 --norandommap`) to avoid cache-warming artifacts.

| # | Test | Pattern | Block Size | Jobs | iodepth | Size |
|---|---|---|---|---|---|---|
| 1 | `seqread` | Sequential Read | 8k | 8 | 4 | 1G |
| 2 | `seqwrite` | Sequential Write | 32k | 4 | 4 | 2G |
| 3 | `seq128kread` | Sequential Read (max BW) | 128k | 8 | 4 | 2G |
| 4 | `seq128kwrite` | Sequential Write (max BW) | 128k | 4 | 4 | 2G |
| 5 | `randread` | Random Read | 8k | 16 | 8 | 1G |
| 6 | `randwrite` | Random Write | 64k | 8 | 8 | 512m |
| 7 | `rand4kread` | 4K Random Read | 4k | 16 | 8 | 1G |
| 8 | `rand4kwrite` | 4K Random Write | 4k | 8 | 8 | 1G |
| 9 | `randrw` | Random R/W (90% read) | 16k | 8 | 8 | 1G |

For each test, it collects IOPS, bandwidth (MiB/s), average latency, latency standard deviation, p50/p99/p99.9 percentiles, and CPU utilization. It also calculates an IOPS-per-CPU-percent efficiency ratio — useful when you're comparing a bare-metal result against a VM running on the same hardware and want to quantify hypervisor overhead.

---

## Installation

Requirements: Bash 4.4+, fio 3.0+, POSIX `awk`. No `jq`.

```bash
git clone https://github.com/9llo/fio_benchmark.git
chmod +x fio_benchmark/fio_benchmark.sh
```

---

## Running it

The simplest invocation against a block device (requires root):

```bash
sudo ./fio_benchmark.sh -t /dev/sdb
```

If you want a dry-run first to verify the setup without touching the disk:

```bash
./fio_benchmark.sh -t /dev/sdb -n
```

No arguments drops you into interactive mode, which prompts for the target and runtime.

Output format is plain text by default. For scripting or long-term tracking, use JSON or CSV — progress and status messages always go to stderr, so the structured output on stdout stays clean for redirection:

```bash
sudo ./fio_benchmark.sh -t /dev/sdb -o json > results.json
sudo ./fio_benchmark.sh -t /dev/sdb -o csv  > results.csv
```

Runtime per test defaults to 600 seconds. For a faster pass during initial triage you can shorten it:

```bash
sudo ./fio_benchmark.sh -t /dev/sdb -r 60
```

---

## The two sweeps that actually find bottlenecks

The standard profiles give you a baseline. The sweeps are where you diagnose *why* a device behaves the way it does.

### iodepth sweep (`-s`)

Runs `randread` with `bs=4k`, `numjobs=1`, and steps through iodepth values of 1, 2, 4, 8, 16, 32, 64, 128.

```bash
sudo ./fio_benchmark.sh -t /dev/sdb -s
```

The IOPS curve tells you where the device's internal parallelism saturates. A consumer NVMe typically plateaus around iodepth 8–16. An enterprise SSD or a SAN LUN with a deep command queue can keep climbing well past 32. If you see IOPS plateau early and then degrade, that's usually a firmware or queue management issue worth investigating.

### RW-mix sweep (`-m`)

Runs `randrw` with `bs=4k`, `numjobs=8`, `iodepth=8`, stepping write percentage from 0% to 100% in 10% increments.

```bash
sudo ./fio_benchmark.sh -t /dev/sdb -m
```

This maps the write-penalty curve. On a healthy NVMe, IOPS should degrade fairly gradually as write % increases. On a heavily overprovisioned SSD near capacity, or a thin-provisioned datastore being over-committed, the curve drops sharply as writes start triggering garbage collection and write amplification. If you're running a database and wondering why performance cliffs under write-heavy workloads, this sweep will show it.

---

## Output example (text)

```
Device: /dev/sdb | Samsung SSD 870 QVO | SSD | 1000GB | 512B blocks | mq-deadline
┌──────────────────┬──────────┬──────────┬────────────┬────────────┬────────────┬────────────┬────────────┬───────┬───────┬──────────────┐
│ Test             │    IOPS  │  BW MB/s │   Lat avg  │   Lat std  │     p50    │     p99    │   p99.9    │ CPU u │ CPU s │ IOPS/CPU%    │
├──────────────────┼──────────┼──────────┼────────────┼────────────┼────────────┼────────────┼────────────┼───────┼───────┼──────────────┤
│ seqread          │    52341 │    409.7 │    0.61 ms │    0.12 ms │    0.58 ms │    0.94 ms │    1.24 ms │  3.1% │  2.1% │    10018.7   │
│ seqwrite         │    14832 │    462.9 │    1.08 ms │    0.31 ms │    0.99 ms │    1.97 ms │    3.51 ms │  2.2% │  1.8% │     3707.8   │
...
```

---

## When to use it

- Qualifying new storage hardware before it enters production
- Comparing VM disk performance against bare-metal to measure virtualization overhead
- Checking whether a "slow storage" complaint from an application team is real or a red herring
- Building a baseline before and after a firmware update or storage configuration change
- Investigating an SSD that feels slower than expected after months of use (wear and write amplification)

It won't tell you *why* something is slow on its own — that still takes interpretation. But having the full test matrix, with consistent parameters, in under an hour, is a better starting point than assembling it by hand every time.

---

## A word on how this was built

Full disclosure: a good chunk of the iteration on this script was done with [Claude Code](https://claude.ai/code) as a pair programmer.

The domain knowledge — which tests matter, what block sizes to use for which workload profile, why the iodepth sweep is more useful than just running a single queue depth, how to parse fio's JSON output without pulling in `jq` as a dependency — that all came from years of actual storage troubleshooting. Claude didn't know that `randrepeat=0` matters for avoiding cache-warming artifacts, or that an IOPS-per-CPU efficiency ratio is useful specifically in VM environments. That context had to be fed in.

What it *did* accelerate: the Bash boilerplate nobody enjoys writing. The argument parser, the cleanup traps, the timeout wrapper around each `fio` invocation, the `awk` pass that extracts percentile fields from JSON without a dependency. Things that are tedious and error-prone to get right, but not intellectually interesting once you know what they need to do.

The honest summary: it took a task that would've been a full weekend of intermittent hacking and compressed it into a focused evening. The thinking didn't get outsourced — it got unblocked.

AI isn't going to replace sysadmins or storage engineers. It has no idea what your SAN topology looks like, why your latency spikes at 3am, or what "it was fine before the firmware update" actually means in context. But if you can describe what you want precisely, it's a remarkably good collaborator for the parts of the job that are just execution. The people who figure out how to use it well aren't going to be replaced — they're going to get more done.

---

The repository is at [github.com/9llo/fio_benchmark](https://github.com/9llo/fio_benchmark). Issues and PRs are open.
