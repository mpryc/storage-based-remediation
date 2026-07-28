# Block Device Format Specification v1

This document specifies the binary on-disk format for SBR Block mode. It is
the implementation companion to
[rwx-block-volume-support.md](rwx-block-volume-support.md).

All integer fields use **little-endian** byte order, consistent with the
existing SBD message format (`message.go`). All CRC32 checksums use the
**IEEE polynomial** (`crc32.ChecksumIEEE` in Go, polynomial `0xEDB88320`),
matching the existing SBD protocol.

## Device Layout

```
Sector(4K)   Offset (bytes)   Size          Region
────────────────────────────────────────────────────────────────────────────
0            0                4,096         Superblock
1-16         4,096            65,536        Node Map Buffer A
17-32        69,632           65,536        Node Map Buffer B
33-287       135,168          1,044,480     Heartbeat Slots (255 x 4096)
288-542      1,179,648        1,044,480     Fence Slots (255 x 4096)
────────────────────────────────────────────────────────────────────────────
Total                         2,224,128     (~2.12 MB)
```

Go constants:

```go
const (
    BlockSectorSize            int64 = 4096
    BlockSuperblockOffset      int64 = 0
    BlockSuperblockSize        int64 = 4096
    BlockNodeMapAOffset        int64 = 4096
    BlockNodeMapBOffset        int64 = 69632
    BlockNodeMapRegionSize     int64 = 65536   // 16 sectors per buffer
    BlockHeartbeatRegionOffset int64 = 135168
    BlockFenceRegionOffset     int64 = 1179648
    BlockSlotSize              int64 = 4096    // per-slot I/O granularity
)
```

## Superblock (Sector 0)

The superblock is formatted once by the Init Job and is **strictly
immutable** thereafter. If a new layout version is ever required, the Init
Job formats a completely new superblock.

```
Byte Offset   Size   Field
───────────────────────────────────────────
0             6      Magic ("SBRBLK")
6             2      Version (uint16 LE, value 1 → bytes 0x01 0x00)
8             8      Node Map A Offset
16            8      Node Map A Length
24            8      Node Map B Offset
32            8      Node Map B Length
40            8      Heartbeat Region Offset
48            8      Heartbeat Region Length
56            8      Fence Region Offset
64            8      Fence Region Length
72            4      Flags (reserved, zero)
76            4      CRC32 (bytes 0–75)
───────────────────────────────────────────
Total: 80 bytes (padded to 4096 by zeroes)
```

### Version Validation

On startup, the agent reads the superblock and validates the magic and
version fields:
- If the magic does not match `SBRBLK`, the agent refuses to start.
- If the version is unrecognized, the agent refuses to start.
- The controller surfaces the resulting agent failure through status
  conditions and events.

## Heartbeat and Fence Slots

Each slot is 4096 bytes. The protocol payload is unchanged:
- Heartbeat: 33 bytes (8 magic + 2 version + 1 type + 2 nodeID +
  8 timestamp + 8 sequence + 4 CRC)
- Fence: 36 bytes (33 header + 2 targetNodeID + 1 reason)

The remaining bytes in each 4 KB slot are zero padding. This satisfies
O_DIRECT alignment requirements on both 512e and 4Kn block devices.

### Slot Offset Formula

NodeIDs range from 1 to 255. Within each region:

```
slot_offset = (nodeID - 1) * BlockSlotSize
```

Slot 0 does not exist (NodeID 0 is invalid per `IsValidNodeID()`). The
`OffsetDevice` base offset translates region-relative offsets to absolute
device offsets.

The existing `SBD_SLOT_SIZE` (512) remains the protocol constant. In Block
mode, `BlockSlotSize` (4096) is used for offset calculations.

## Node Map Buffer Format (64 KB each)

```
Byte Offset   Size       Field
───────────────────────────────────────────────────
0             8          Generation (uint64, LE)
8             16         WriterUUID (raw bytes)
24            4          PayloadLength (uint32, LE)
28            variable   Payload (JSON data)
28+N          variable   Padding (zeroes)
65532         4          CRC32 (uint32, LE, bytes 0–65531)
───────────────────────────────────────────────────
Total: 65,536 bytes (64 KB)
```

The Payload field contains the output of `NodeMapTable.Marshal()`: a 4-byte
CRC32 prefix (the existing node map checksum) followed by the JSON-encoded
`NodeMapTable`. This is the same byte sequence that Filesystem mode writes
to the `sbr-device-nodemap` file. The outer envelope (Generation,
WriterUUID, PayloadLength, outer CRC32) is Block-mode specific; the inner
payload is mode-independent.

Maximum payload: 255 nodes x ~150 bytes = ~38 KB. Safely fits within 64 KB.

## O_DIRECT Requirements

1. **I/O alignment and length:** All reads and writes are padded and aligned
   to 4096 bytes.

2. **Memory alignment:** I/O buffers must be page-aligned. Use
   `golang.org/x/sys/unix.Mmap` or `github.com/ncw/directio` to allocate
   page-aligned byte slices, replacing `make([]byte, size)` for raw block
   writes.

3. **Implementation note:** Every `ReadAt()` and `WriteAt()` path in Block
   mode — including node map I/O, heartbeat slots, fence slots, and any
   helper paths — must use page-aligned buffers. It is easy to accidentally
   allocate with `make()` in one path and lose O_DIRECT compatibility.

In Filesystem mode, the current 512-byte padding via `PadToSlotSize()`
continues to work unchanged.

## OffsetDevice

```go
type OffsetDevice struct {
    device     SBDDevice
    baseOffset int64
    regionSize int64  // bounds limit; 0 = unbounded
}

func (d *OffsetDevice) ReadAt(p []byte, off int64) (int, error) {
    if d.regionSize > 0 && off+int64(len(p)) > d.regionSize {
        return 0, fmt.Errorf("read at offset %d + %d bytes exceeds region size %d",
            off, len(p), d.regionSize)
    }
    return d.device.ReadAt(p, d.baseOffset+off)
}

func (d *OffsetDevice) WriteAt(p []byte, off int64) (int, error) {
    if d.regionSize > 0 && off+int64(len(p)) > d.regionSize {
        return 0, fmt.Errorf("write at offset %d + %d bytes exceeds region size %d",
            off, len(p), d.regionSize)
    }
    return d.device.WriteAt(p, d.baseOffset+off)
}

func (d *OffsetDevice) Sync() error { return d.device.Sync() }
```

The agent opens the device once and creates three logical views:
- Heartbeat: `OffsetDevice{baseOffset: BlockHeartbeatRegionOffset}`
- Fence: `OffsetDevice{baseOffset: BlockFenceRegionOffset}`
- Node map: direct reads/writes at `BlockNodeMapAOffset` / `BlockNodeMapBOffset`

### Sync() Semantics

`blockdevice.Device.Sync()` issues an `fsync()` system call, requesting
that buffered data for the device be committed to stable storage according
to the storage stack's durability guarantees. Note that `fsync()` guarantees
local durability; visibility to other nodes depends on the distributed
storage backend's consistency model (Ceph RBD provides strong consistency
for overlapping I/O regions).

## NodeMapStore Interface

```go
type NodeMapStore interface {
    Load() ([]byte, error)
    Save(data []byte) error
}
```

### FileNodeMapStore

Wraps the current file-based logic unchanged:

```go
type FileNodeMapStore struct {
    filePath string
}

func (s *FileNodeMapStore) Load() ([]byte, error) {
    return os.ReadFile(s.filePath)
}

func (s *FileNodeMapStore) Save(data []byte) error {
    tmpPath := s.filePath + ".tmp"
    if err := os.WriteFile(tmpPath, data, 0644); err != nil {
        return err
    }
    return os.Rename(tmpPath, s.filePath)
}
```

### BlockNodeMapStore

Uses double-buffering with optimistic write-verify conflict detection.

## Write-Verify Protocol

In Filesystem mode, `flock()` on a cluster filesystem (CephFS, NFSv4)
provides cross-node serialization via the storage backend's metadata server
or distributed lock manager. Raw block devices bypass this infrastructure
entirely — there is no MDS or DLM to broker locks. The write-verify
protocol replaces distributed locking with optimistic conflict detection.
It does not provide linearizable updates; correctness relies on the node
map being non-authoritative (see Safety Analysis).

### Algorithm

1. **Prepare:** Node reads both buffers, verifies CRCs, selects the one
   with the highest Generation.
2. **Write:** Node increments the generation (`Gen = Gen + 1`), generates a
   random WriterUUID, constructs the 64 KB block, and writes it to the
   *inactive* buffer via `WriteAt()`.
3. **Flush:** Node issues `Sync()` to request that the write be committed
   to stable storage before the verification read. Without this, the
   readback in step 5 could return stale cached data on some storage
   backends.
4. **Wait:** Node sleeps for a randomized delay (1.5–2.5 seconds). This
   provides a bounded conflict detection window for in-flight concurrent
   writes to land on disk before verification. Correctness does not depend
   on storage latency assumptions; a failed verification causes retry. The
   delay is chosen to remain comfortably below the hardware watchdog timeout
   (typically 10–60 seconds).
5. **Verify:** Node reads back the inactive buffer from disk.
   - If CRC is valid AND Generation matches AND WriterUUID matches: success.
   - If mismatch: another node wrote concurrently. Back off with randomized
     jitter (50–200 ms) and retry from step 1.

### Crash Consistency (Torn Write Protection)

The CRC32 covers the entire 64 KB payload including the Generation and
WriterUUID. CRC32 verification detects torn writes with high probability
(~1 in 2³² chance of a false match). If a write mixes newly written sectors
with old sectors from a previous generation, the CRC check fails. On
restart, the agent detects the invalid CRC and falls back to the other
buffer.

### First Boot and Reinitialization Safety

If both buffers are invalid (freshly formatted device), the agent writes an
empty node map directly to Buffer A with Generation=1. This initialization
write does not follow the write-verify protocol (there is no valid prior
state to conflict with). Subsequent updates use the full protocol.

**Reinitialization guard:** Before initializing, the agent verifies that
both buffers are truly empty (all zeroes or invalid CRCs with Generation=0).
If either buffer has a non-zero Generation with an invalid CRC, this
indicates a torn write from a previous operation rather than a fresh device.
In this case, the agent logs a warning and uses the other buffer if valid,
or refuses to start if both are corrupted with non-zero generations
(requiring operator intervention via the Init Job).

### Equal-Generation Conflict Resolution

When two nodes both read the same current generation G and both write G+1,
each with a different WriterUUID, the result depends on timing:

- If both writes target the same buffer, one overwrites the other. The
  verification step detects this for the overwritten writer (UUID mismatch).
- If both writes target different buffers (should not happen — both select
  the same inactive buffer based on generation), Load() resolves by
  selecting the buffer with the highest valid generation. If both are G+1,
  the buffer with the valid CRC that is read first wins (effectively
  arbitrary but deterministic for a given read).

In all cases, the losing writer retries. The generation counter is
monotonically increasing; after resolution, the winner's generation G+1 is
the new baseline.

### Safety Analysis

The write-verify protocol provides **eventual convergence with bounded stale
state**, not linearizable updates. The following analysis explains why this
is sufficient for SBR.

**The node map is non-authoritative advisory metadata.** It is used for
slot assignment and nodeID-to-name resolution. It is **not** consulted
during fencing decisions. Fencing correctness does not depend on node map
consistency.

**Fencing decisions use heartbeat slot content directly:** The peer monitor
reads each slot, validates the CRC, checks the embedded NodeID against the
expected slot owner (`main.go:1079`), and evaluates timestamps/sequences.
Conflicting slot ownership can result in stale or ambiguous heartbeat
state. The protocol's CRC, NodeID, and sequence validation prevents
accepting invalid data as a healthy signal. The impact is **loss of
availability** (unnecessary self-fences), not a **safety violation** (wrong
node fenced).

**Defense in depth — verification delay and watchdog:** The verification
delay (1.5–2.5 seconds) provides a bounded conflict detection window. The
hardware watchdog provides an upper bound on process pauses. These are
defense-in-depth measures that reduce the probability of conflicts reaching
the heartbeat layer, but **correctness does not depend on them**. Even if
the write-verify protocol fails to detect a conflict, the heartbeat
protocol's own validation (CRC, NodeID, sequence) prevents incorrect
fencing. The watchdog is not a correctness proof; it is a reliability
optimization.

**Heartbeat slot ownership self-check:** In Block mode, each agent
periodically reads back its own heartbeat slot and verifies the embedded
NodeID matches its assigned ID. If a mismatch is detected, the agent
triggers a node map reload and re-assignment, providing an additional
recovery mechanism beyond write-verify.

**Reconciliation:** Node map state is periodically reconciled. Any
duplicate or stale assignment created by a missed conflict is detected
during the next reconciliation cycle and corrected (one node observes its
slot is occupied by another and re-assigns).

### WriterUUID Logging

Agents log the WriterUUID on every write attempt and include both the local
and observed UUIDs in error messages when verification fails. If a collision
occurs in production, the UUIDs in pod logs allow direct correlation between
storage state and node behavior.

### Implementation Note: Heartbeat Loop Independence

The write-verify Wait step (1.5–2.5 seconds) must not block the heartbeat
loop. The `NodeManager` registration/update should run in a separate
goroutine or be invoked outside the heartbeat critical path to avoid falsely
triggering peer timeouts during delayed map updates.

## Init Job

The Init Job validates device capacity, zeroes the layout, and writes the
V1 Superblock. Because the superblock contains computed fields and a CRC32,
the init job is a Go binary (`cmd/sbr-init`) that reuses the superblock
serialization code from the agent. This mirrors the existing Filesystem
init job pattern (a container running in the same PVC) but replaces shell
`dd`/`printf` with type-safe Go serialization.

Steps:
1. Open device via `blockdevice.OpenWithTimeout()`
2. Verify size ≥ 2,224,128 bytes (`blockdev --getsize64` equivalent via `unix.IoctlGetInt(fd, BLKGETSIZE64)`)
3. Zero the first 3 MB (`WriteAt` zeroed 4 KB pages in a loop)
4. Serialize and write V1 Superblock to offset 0

The init job uses `volumeDevices` with the same device path as the agent.
The container image is the same SBR agent image with a different entrypoint.

## StorageClass Capability Testing

Known provisioners (`rbd.csi.ceph.com`) are validated via a fast path. For
all other provisioners, the controller creates a temporary PVC:

- Generated name (e.g. `sbr-capability-test-<hash>`) with `ownerReference`
  pointing to the `StorageBasedRemediationConfig` for automatic garbage
  collection.
- 60-second timeout for PVC binding, cleanup regardless of outcome.
- Result cached per StorageClass name for the lifetime of the controller
  process to avoid repeated probing.

## Storage Topologies

The structural equivalence between the two modes:

**Filesystem Mode:**
```
/dev/sbr/ (directory)
  ├── sbr-device           (Heartbeat slots, implicit locking via OS)
  ├── sbr-device-fence     (Fence slots)
  └── sbr-device-nodemap   (JSON, crash-safe via rename(), concurrent via flock())
```

**Block Mode:**
```
/dev/sbr-block (raw device)
  ├── [Superblock]         (Immutable layout definitions)
  ├── [Node Map A & B]     (Double-buffered write-verify, replaces rename/flock)
  ├── [Heartbeat Slots]    (I/O adapted via OffsetDevice)
  └── [Fence Slots]        (I/O adapted via OffsetDevice)
```
