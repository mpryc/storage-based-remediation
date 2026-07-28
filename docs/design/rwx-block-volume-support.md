# RWX Block Volume Support for Heartbeat Backend

## Problem

SBR currently requires an RWX Filesystem StorageClass for its shared heartbeat
backend. The controller creates a PVC with `ReadWriteMany` access mode and
default `volumeMode: Filesystem`, mounts it as a directory at `/dev/sbr/`, and
the agent operates on three files inside that directory: `sbr-device`
(heartbeat slots), `sbr-device-fence` (fence slots), and `sbr-device-nodemap`
(JSON node map).

Kubernetes virtualization platforms (KubeVirt and distributions built on it)
typically use a StorageClass backed by Ceph RBD that provisions RWX Block
volumes for VM disks. Users want to reuse this StorageClass for SBR rather
than deploying a second StorageClass solely for SBR. Today this is not
possible: if a user points `sharedStorageClass` at their VM StorageClass, the
controller creates a Filesystem PVC, the CSI driver cannot provision it, and
the PVC stays Pending.

### Why This Matters

Requiring a separate RWX Filesystem StorageClass adds operational overhead.
Supporting RWX Block removes this deployment barrier and aligns SBR with the
storage topology that virtualization-enabled clusters already have.

## Design Principles

- Filesystem mode behavior remains unchanged.
- Block mode uses the same SBD protocol semantics.
- The shared device remains the sole source of truth; Kubernetes API is not
  used for heartbeat coordination.
- The block layout is versioned and self-describing.
- Safety is preferred over availability during ambiguous storage states.

## Current Limitations

Three areas of the codebase assume Filesystem semantics:

1. **PVC creation** (`storagebasedremediationconfig_controller.go:566-665`):
   The controller creates the PVC without setting `volumeMode` (defaults to
   `Filesystem`).

2. **DaemonSet volumes** (`storagebasedremediationconfig_controller.go:1801-1864`):
   The shared storage is mounted via `volumeMounts` (directory). Block PVCs
   require `volumeDevices` (raw block path).

3. **Node map persistence** (`sbdprotocol/nodemanager.go:438-674`): The
   `NodeManager` stores the node-to-slot mapping using `os.ReadFile()`, and
   `os.WriteFile()` + `os.Rename()`. `rename()` provides atomic, crash-safe
   updates on POSIX filesystems. Raw block devices do not support these
   operations and lack native atomicity for arbitrary-sized writes.

The heartbeat and fence I/O already use positioned `ReadAt`/`WriteAt` via the
`blockdevice.Device` type with `O_RDWR|O_SYNC|O_DIRECT`. While these work on
block devices, their current hardcoded 512-byte buffer sizes and alignments do
not safely guarantee O_DIRECT compatibility across all block device topologies
(especially 4K-native devices).

## Proposed Design

Add a `sharedStorageVolumeMode` field to `StorageBasedRemediationConfigSpec`.
When set to `Block`, the controller creates a Block PVC, exposes it as a raw
device via `volumeDevices`, and the agent partitions that single device into
logical regions.

To safely replace filesystem atomicity (`rename()`) and concurrency
(`flock()`), Block mode introduces device layout versioning and a
double-buffered node map with optimistic write-verify conflict detection.

The existing Filesystem path is unchanged. All behavioral differences are
gated on the volume mode.

### Architecture Overview

```
              StorageBasedRemediationConfig
                           |
                sharedStorageVolumeMode
                      /            \
              Filesystem            Block
                  |                   |
           PVC (Filesystem)    PVC (Block Device)
                  |                   |
           volumeMounts         volumeDevices
                  |                   |
         /dev/sbr/ (directory)  /dev/sbr-block (raw device)
           |     |     |          |    |    |    |
         hb   fence  nodemap    super nmap  hb  fence
        file   file   file      block bufs  slots slots
```

### CRD Changes

**File:** `api/v1alpha1/storagebasedremediationconfig_types.go`

```go
// SharedStorageVolumeMode defines the volume mode for the shared storage PVC.
// +kubebuilder:validation:Enum=Filesystem;Block
type SharedStorageVolumeMode string

const (
    SharedStorageVolumeModeFilesystem SharedStorageVolumeMode = "Filesystem"
    SharedStorageVolumeModeBlock      SharedStorageVolumeMode = "Block"
)
```

On the spec:

```go
// SharedStorageVolumeMode specifies the volume mode for the shared storage PVC.
// "Filesystem" (default): mounts the PVC as a directory.
// "Block": exposes the PVC as a raw block device.
// +optional
SharedStorageVolumeMode *SharedStorageVolumeMode `json:"sharedStorageVolumeMode,omitempty"`
```

Helper methods:

```go
func (s *StorageBasedRemediationConfigSpec) GetSharedStorageVolumeMode() corev1.PersistentVolumeMode {
    if s.SharedStorageVolumeMode != nil && *s.SharedStorageVolumeMode == SharedStorageVolumeModeBlock {
        return corev1.PersistentVolumeBlock
    }
    return corev1.PersistentVolumeFilesystem
}

func (s *StorageBasedRemediationConfigSpec) IsBlockMode() bool {
    return s.GetSharedStorageVolumeMode() == corev1.PersistentVolumeBlock
}
```

### PVC and Pod Changes

**PVC creation:** Set `VolumeMode` on the PVC spec based on the new field.
The 10Mi PVC request guarantees enough space for the ~2.12 MB block layout
regardless of CSI driver rounding.

**DaemonSet volumes:**
- **Filesystem mode** (unchanged): `volumeMounts` with `mountPath: /dev/sbr`.
- **Block mode**: `volumeDevices` with `devicePath: /dev/sbr-block`.

**Agent arguments in Block mode:**
```
--sbr-device=/dev/sbr-block
--sbr-volume-mode=block
--sbr-file-locking=false
```

`--sbr-file-locking=false` because `flock()` is not meaningful on a raw
block device. In Filesystem mode, when the PVC is backed by a cluster
filesystem (e.g. CephFS, NFSv4), `flock()` provides cluster-wide
serialization brokered by the storage backend's metadata server or
distributed lock manager. Raw block devices bypass this infrastructure
entirely, so `flock()` reverts to host-local semantics only.

**Init Job:** In Block mode, the Init Job validates device capacity,
zeroes the device, and writes the V1 Superblock. Unlike the Filesystem
init job (which uses simple `dd` + `chmod` shell commands), the Block init
job should be a Go binary because the superblock contains computed fields
(region offsets/lengths) and a CRC32 checksum that are fragile to produce
in shell. The Go binary reuses the superblock serialization code from the
agent. The agent handles node map initialization on first startup.

### Block Device Layout (High Level)

Block mode reserves regions for a superblock, two node map buffers,
heartbeat slots, and fence slots. The layout uses 4 KB sectors to support
O_DIRECT on both 512-byte and 4K-native block devices. The layout is
versioned via a self-describing superblock to allow future changes.

```
Region             Size          Content
──────────────────────────────────────────────────────────────
Superblock         4 KB          Magic, Version, Layout Descriptors
Node Map Buffer A  64 KB         Double-buffered node map (primary)
Node Map Buffer B  64 KB         Double-buffered node map (secondary)
Heartbeat Slots    ~1 MB         255 slots x 4 KB
Fence Slots        ~1 MB         255 slots x 4 KB
──────────────────────────────────────────────────────────────
Total              ~2.12 MB
```

NodeIDs range from 1 to 255. Within each region, the slot offset is
`(nodeID - 1) * BlockSlotSize`. The heartbeat and fence protocol payloads
remain unchanged (33 bytes for heartbeat, 36 bytes for fence); the
additional bytes in each 4 KB slot are padding.

The detailed binary format (superblock byte layout, node map buffer wire
format, endianness, CRC coverage) is specified in
[rwx-block-volume-format.md](rwx-block-volume-format.md).

### Agent Changes

**OffsetDevice:** The single `blockdevice.Device` is wrapped in an
`OffsetDevice` that applies a region base offset to all `ReadAt`/`WriteAt`
calls. This reuses the existing heartbeat/fence I/O logic without
modification.

**O_DIRECT alignment:** Block mode uses 4 KB aligned I/O with page-aligned
memory buffers to support both 512e and 4Kn devices.

**Storage coherence prerequisite:** Block mode correctness relies on the CSI
backend providing coherent shared access semantics for concurrent reads and
writes to the same volume. Ceph RBD provides this guarantee.

### Runtime Code Path

No new goroutines are introduced. The heartbeat loop, peer check loop, and
watchdog loop are identical in both modes. The volume mode is resolved once
at agent startup in `main.go`: Filesystem mode opens three files, Block
mode opens one device and wraps it in three `OffsetDevice` views. The
resulting device handles are passed to the same loops — they call
`ReadAt`/`WriteAt` without knowing which mode they are in.

### Agent Device Opening

Currently the agent opens three separate files by deriving paths from the
base `--sbr-device` path (appending `-fence` and `.nodemap` suffixes).
In Block mode, the agent opens a single device and creates `OffsetDevice`
views. The path derivation logic in `main.go` is gated on
`--sbr-volume-mode`:

- **Filesystem:** Unchanged — three `blockdevice.OpenWithTimeout()` calls.
- **Block:** One `blockdevice.OpenWithTimeout()` call on the raw device,
  then three `OffsetDevice` wrappers (heartbeat, fence, node map).

The existing `blockdevice.Device` retry config (3 attempts, exponential
backoff) and I/O timeout (from `--io-timeout`) apply to the underlying
device in both modes. `OffsetDevice` delegates all I/O to the wrapped
`Device`.

### Node Map Persistence

The `NodeMapStore` interface extracts the **storage layer only** from
`NodeManager`. The existing CAS logic (version checking, retry with
backoff), corruption recovery, and stale node cleanup remain in
`NodeManager` unchanged. `NodeManager` calls `store.Load()` / `store.Save()`
where it currently calls `os.ReadFile()` / write-and-rename directly.

```go
type NodeMapStore interface {
    Load() ([]byte, error)
    Save(data []byte) error
}
```

**`FileNodeMapStore`** wraps the current file I/O: `os.ReadFile` for Load,
`os.WriteFile`+`os.Rename` for Save. `flock()` coordination remains in
`NodeManager` (called before `store.Save()`), not in the store.

**`BlockNodeMapStore`** uses double-buffering with optimistic write-verify
conflict detection, replacing `rename()` crash-safety and `flock()`
concurrency:

- **Crash consistency:** CRC32 covers each 64 KB buffer. CRC32 verification
  detects torn writes with high probability (~1 in 2³² chance of a false
  match). On restart, the agent falls back to whichever buffer has a valid
  CRC and the highest generation.

- **Concurrency:** Because raw block devices bypass the cluster filesystem's
  distributed lock manager, Block mode uses a write-verify protocol with
  generation counters, WriterUUIDs, and a bounded verification delay. The
  full algorithm is specified in
  [rwx-block-volume-format.md](rwx-block-volume-format.md).

- **Safety:** The node map is **non-authoritative advisory metadata**. It
  is not consulted during fencing decisions. Fencing is based entirely on
  heartbeat slot content: CRC validity, embedded NodeID, timestamps, and
  sequence numbers. The peer monitor already rejects heartbeats where the
  embedded NodeID does not match the expected slot owner
  (`main.go:1079`). Conflicting slot ownership can result in stale or
  ambiguous heartbeat state, but the protocol's validation prevents
  accepting invalid data as a healthy signal. The impact of a lost node
  map update is loss of availability (unnecessary self-fences), not
  incorrect fencing (wrong node fenced).

- **Heartbeat slot ownership validation:** In Block mode, each agent
  periodically reads back its own heartbeat slot and verifies the embedded
  NodeID matches its assigned ID. If a mismatch is detected (another node
  is writing to the same slot), the agent triggers a node map reload and
  re-assignment. This provides an additional detection layer beyond the
  write-verify protocol.

- **First boot:** If both buffers are invalid (freshly formatted device),
  the agent writes an empty node map directly to Buffer A without the
  write-verify protocol (no prior state to conflict with).

The full write-verify protocol analysis, including the race window
characterization and watchdog bounding proof, is in
[rwx-block-volume-format.md](rwx-block-volume-format.md).

### StorageClass Validation

Known provisioners (`rbd.csi.ceph.com`) are validated via a fast path
without runtime probing. For all other provisioners, the controller creates
a temporary PVC with `volumeMode: Block` and `ReadWriteMany` to verify
capability dynamically.

On validation failure (PVC stays Pending past timeout, or provisioner
rejects the request), the controller sets the existing
`SharedStorageReady` condition to `False` with reason
`StorageClassIncompatible` and emits a Warning event. This uses the
existing condition/event infrastructure — no new conditions are needed.

### Webhook Validation

- `SharedStorageVolumeMode` must be nil, `"Filesystem"`, or `"Block"`.
- If `SharedStorageVolumeMode` is `"Block"`, `SharedStorageClass` must be
  set.
- On update: reject mutations to `SharedStorageVolumeMode` on an existing
  CR.

## Compatibility and Upgrade

### What Does Not Change

- The SBD protocol message format (33-byte header, CRC32 checksum)
- `SBD_SLOT_SIZE` (512) as the protocol constant, `SBD_MAX_NODES` (255),
  `SBD_HEADER_SIZE` (33)
- The node map JSON payload format (the storage envelope differs in Block
  mode but the JSON content is identical)
- The heartbeat loop, peer monitor loop, and watchdog loop
- The fence message flow (write fence -> target reads own slot -> self-fence)
- Existing Filesystem-mode deployments: on-disk format, atomic rename
  persistence, and flock coordination are completely unchanged
- `SBRTimeoutSeconds`, `MaxConsecutiveFailures`, timing calculations
- The `blockdevice.Device` type, Prometheus metrics, security context, RBAC

### Upgrade Path

- The new field defaults to `nil` (resolves to `Filesystem`). Existing
  resources are unaffected — no migration required.
- Changing modes requires deleting the old config and creating a new one.
  The webhook rejects mutations to `SharedStorageVolumeMode` on existing
  resources.

## Alternatives Considered

### 1. Auto-detect volume mode from StorageClass

CSI capability discovery is not standardized in Kubernetes. An explicit
field is simpler, predictable, and matches how users already know their
storage topology.

### 2. Two separate PVCs (one for heartbeat, one for fence)

Doubles storage resource consumption and controller complexity. The
single-device-with-regions approach is how traditional SBD works.

### 3. Layer a filesystem on the block device

Single-node filesystems (ext4, xfs) on a shared RWX block device cause data
corruption. Cluster filesystems (GFS2, OCFS2) add a dependency that defeats
the purpose of reusing the existing StorageClass.

### 4. Store node map in a ConfigMap/Secret

Introduces API server dependency for heartbeat coordination, violating
SBD's design principle that the shared device is the sole coordination
mechanism.

### 5. Use 512-byte slot alignment instead of 4 KB

On 4K-native (4Kn) devices, 512-byte-aligned offsets fail with `EINVAL`.
4 KB alignment works on both 512e and 4Kn devices. The space overhead
(~2 MB vs ~319 KB) is negligible within the 10Mi PVC.

## Testing Strategy

1. **Unit Tests:** OffsetDevice offset arithmetic, BlockNodeMapStore
   double-buffer fallback and conflict detection, O_DIRECT buffer alignment.
2. **E2E Tests:** Deploy RWX Block (Ceph RBD) and verify fencing succeeds.
   Deploy RWX Filesystem and verify no regression.
3. **Failover Time E2E:** Measure time from heartbeat stop to fence
   completion on both Block and Filesystem backends. Assert Block failover
   is within 20% of Filesystem (same timing constants apply).
4. **Webhook Tests:** Reject `SharedStorageVolumeMode` mutations on existing
   resources. Reject Block mode without `SharedStorageClass`.
5. **Superblock Tests:** Reject unrecognized version, invalid magic/CRC,
   fully zeroed device.
6. **First-boot Tests:** Initialize fresh node map when both buffers are
   invalid.

## Edge Cases

| Scenario | Outcome |
|----------|---------|
| Block mode without `sharedStorageClass` | Webhook rejects |
| Node map write interrupted (power loss) | Torn write yields invalid CRC; agent loads alternate buffer |
| Concurrent node map writers | Verify delay + WriterUUID detects conflicts; writers back off and retry |
| StorageClass does not support RWX Block | PVC stays Pending; controller sets error condition |
| Agent reads unrecognized superblock version | Agent refuses to start; controller surfaces failure via status/events |
| Both node map buffers invalid (first boot) | Agent initializes empty node map in Buffer A |
| Volume mode mutation on existing config | Webhook rejects — immutable once set |

## Documentation

The `StorageBasedRemediationConfig` CRD description and the user guide are
updated to document `sharedStorageVolumeMode`. For OCP Virt deployments,
add a one-line recommendation: *"Use the same StorageClass as your virtual
machine disks."*

## Out of Scope

- **CSI driver certification matrix.** This feature targets Ceph RBD as the
  primary driver.
- **Multiple StorageClasses per node.** The current CRD is a singleton per
  namespace. Supporting multiple configs is a separate RFE; this design
  does not preclude it.
- **Changes to RWX Filesystem behavior.** The only cross-cutting change is
  the `NodeMapStore` interface extraction.
- **Performance benchmarking.** The I/O sizes (4 KB) are trivial for any
  modern storage backend.
