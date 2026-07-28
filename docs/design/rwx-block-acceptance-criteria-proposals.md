# Proposed Acceptance Criteria Refinements

This document proposes two additions to the RFE-9540 acceptance criteria.
It is a discussion aid, not part of the design or implementation spec.

## 1. Failover Time: "comparable" -> "within same order of magnitude"

### Current wording

> Failover time is comparable to the existing RWX File path.

### Proposed wording

> Failover time is within the same order of magnitude as the existing RWX
> File path.

### Rationale

SBR failover time is determined by fixed, mode-independent constants:

```
failover_time ~ heartbeat_interval * maxConsecutiveFailures
             = (sbrTimeoutSeconds / 2) * maxConsecutiveFailures
             = (30 / 2) * 7
             = 105 seconds (with defaults)
```

These constants are identical in both Filesystem and Block mode. The only
variable is raw I/O latency for a single 4 KB `ReadAt`/`WriteAt` — Ceph
RBD (RADOS direct) vs CephFS (MDS + RADOS). Both are sub-millisecond in
practice.

Even a 100% increase in per-I/O latency (e.g. 0.5 ms to 1.0 ms) changes
total failover time by fractions of a second against a ~105 s baseline.
The difference is noise, not a behavioral change.

"Comparable" is ambiguous — it invites debate about acceptable percentage
thresholds (5%? 10%? 20%?) for a difference that cannot meaningfully
affect remediation outcomes. "Same order of magnitude" sets a clear bar
that catches real regressions (e.g. Block mode accidentally adding seconds
of overhead per heartbeat cycle) without creating false failures from
irrelevant micro-benchmark variance.

### Impact on testing

The E2E test measures failover time on both backends and asserts they are
within the same order of magnitude. No timing-sensitive flaky assertions
on sub-second differences.

## 2. Init Job Idempotency

### Current state

There is no acceptance criterion covering Init Job re-runs.

### Proposed criterion

> The Block mode Init Job is idempotent: re-running it against a device
> with a valid, initialized superblock exits successfully without
> modifying the device.

### Rationale

The Block mode Init Job zeroes the first 3 MB of the device and writes a
fresh superblock. If the job runs against a device that agents are already
using, it destroys all heartbeat slots, fence slots, and node maps. The
result: CRC failures across the cluster, unnecessary self-fences on every
node.

This can happen because:

- **Controller retry logic.** If a job fails (OOM kill, node eviction,
  timeout), the controller deletes it and creates a new one. A transient
  failure followed by a re-run hits an already-initialized device if
  agents started between the first run and the retry.

- **TTL cleanup race.** The init job has `ttlSecondsAfterFinished: 3600`.
  If the job's completion record is garbage-collected before the
  controller checks it (e.g. controller restart after >1 hour), the
  controller may re-create the job.

- **Manual intervention.** An operator debugging a stuck deployment may
  delete and re-create the job, not realizing agents are already running.

In Filesystem mode this is benign — re-creating files with `dd` and
`chmod` on a mounted directory does not corrupt existing file content
that agents hold open. Block mode does not have this safety property;
zeroing the device is unconditionally destructive.

### Proposed implementation

The init job reads the first 80 bytes (one superblock) before any writes:

```
1. Read sector 0 (4 KB, O_DIRECT aligned)
2. Check magic == "SBRBLK" AND CRC32 is valid AND version is recognized
3. If all pass: log "device already initialized", exit 0
4. Otherwise: proceed with zero + format
```

This is ~10 lines of Go code using the same superblock deserialization
the agent already has. It makes the job safe to re-run at any point
without disrupting a running cluster.

### Impact on testing

Add a unit test: call the init binary twice on the same device, assert
the second run exits 0 without modifying any bytes after the superblock.
