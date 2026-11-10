# Storage-Arbitrated Remediation for OpenShift Virtualization

| Item | Value |
|---|---|
| Status | Proposal for review |
| Target release | 5.9.0 |
| Tracking | `RHWA-1839`, `RHWA-1837`, `RFE-9581` |

Contributors: Daniel Froehlich ([@DanielFroehlich](https://github.com/DanielFroehlich)), Wesley Hayutin ([@weshayutin](https://github.com/weshayutin)), Pranav Gaikwad ([@pranavgaikwad](https://github.com/pranavgaikwad)), and Michal Pryc ([@mpryc](https://github.com/mpryc)).

## Index

- [1. Introduction](#1-introduction)
- [2. Motivation](#2-motivation)
- [3. Goals](#3-goals)
- [4. Non-Goals](#4-non-goals)
- [5. Design](#5-design)
  - [5.1. Use-case matrix](#51-use-case-matrix)
  - [5.2. Components and ownership](#52-components-and-ownership)
  - [5.3. Configuration API](#53-configuration-api)
  - [5.4. Private probe volume](#54-private-probe-volume)
  - [5.5. Storage and API decision policy](#55-storage-and-api-decision-policy)
  - [5.6. Host journal and repeated-request checks](#56-host-journal-and-repeated-request-checks)
  - [5.7. Remediation CR status and reboot evidence](#57-remediation-cr-status-and-reboot-evidence)
  - [5.8. Requested remediation, escalation, and cleanup](#58-requested-remediation-escalation-and-cleanup)
  - [5.9. NHC policy and reboot limits](#59-nhc-policy-and-reboot-limits)
  - [5.10. Startup, shutdown, and observability](#510-startup-shutdown-and-observability)
  - [5.11. Default-deny boot protection and controlled rejoin](#511-default-deny-boot-protection-and-controlled-rejoin)
- [6. Open Questions](#6-open-questions)
- [7. Upgrade / Downgrade Concerns](#7-upgrade--downgrade-concerns)
- [8. Test Plan](#8-test-plan)
- [Alternatives considered](#alternatives-considered)
- [Terminology](#terminology)

## 1. Introduction

Storage Based Remediation (SBR) currently uses shared storage for heartbeats and fence messages. Node Healthcheck Operator (NHC) creates a `StorageBasedRemediation` custom resource (remediation CR) when its policy permits remediation.

This proposal adds `BlockPerNode` to the existing `sharedStorageVolumeMode` field in the SBR config CR. Each selected node uses a private Block volume to check storage access. SBR stores remediation history in a file on the host. NHC uses SBR as its remediation provider. The host file is called the journal in this document. The local agent writes this file.

The local SBR agent reboots its host and reports boot information. Peer controllers apply OutOfService and check workload cleanup through the API until a fixed deadline. Default-deny host boot protection prevents workload execution after an unclean or remediation-triggered reboot until controlled rejoin under Section 5.11. Automatic rejoin relies on standard control-plane reconciliation and delegates physical crash recovery to the configured storage stack; AdminAssisted adds an external recovery prerequisite. Reboot evidence alone does not authorize destructive cleanup or rejoin. Existing `Filesystem` and `Block` configurations retain their shared-storage behavior.

See the [Terminology table](#terminology) for definitions of the resource names and design terms used in this document.

## 2. Motivation

The current shared Block volume requires concurrent writes from multiple nodes. Some storage drivers cannot provide this access. Private probe volumes remove that shared-write requirement.

A storage-focused policy should preserve running virtual machines during API or network loss when storage remains usable. When the API remains reachable, NHC should decide whether storage failure permits remediation. When both storage and API access fail, the local agent needs to decide whether to reboot without an API request.

Every held unclean or remediation boot requires reconciliation of stale workload instances before rejoin, even if storage recovered during reboot. This can require evacuation, replacement capacity, and operator intervention. It does not guarantee relocation elsewhere; a new workload instance may later run on the recovered node. Clean maintenance boots do not enter this evacuation protocol.

## 3. Goals

- Preserve existing `Filesystem` and shared `Block` behavior. Keep `Filesystem` as the effective default when the volume mode is absent.
- Introduce new mode `BlockPerNode` that will use private probe volumes per node instead of a shared volume.
- Distinguish temporary input/output (I/O) errors from sustained failure through a configurable probe deadline and consecutive failed health evaluations.
- Keep the watchdog active during isolated API loss when storage remains healthy.
- Reboot autonomously after sustained combined storage and API loss.
- Enforce one local agent per host to control the watchdog and write the journal.
- Persist request identity and boot information across agent and host restarts.
- Retain SBR's OutOfService taint without direct Pod deletion. Report the results of cleanup checks within the fixed period separately from reboot success.
- Report saved information that proves a recorded boot ended, without depending on the probe volume.
- For enrolled worker nodes, persist holds before requested/autonomous reboot and detect unclean host boots even before an SBR decision was recorded. Valid clean boots and admitted same-boot service restarts must not wait for the API.
- Support Automatic and AdminAssisted rejoin through standard Kubernetes/CSI state, without backend-specific fencing APIs or custom CSI addons; state the delegated storage crash-recovery assumptions explicitly.
- Keep reboot completion, continuing isolation, cleanup observation, safe rejoin, and autonomous-budget rearming as separate facts.
- Continue tracking an accepted remediation request even if NHC stops the request or someone deletes its `StorageBasedRemediation` CR. SBR must still confirm that the request’s recorded initial boot ended (`BootEnded`). Incomplete workload cleanup alone must not prevent CR-holder deletion after the required boot confirmation and Node updates. Safe rejoin remains a separate prerequisite for later execution.

## 4. Non-Goals

- Remove or redesign the existing `Filesystem` and shared `Block` modes.
- In the new mode, inspect another node's private probe volume or exchange fence messages through it.
- Perform API cleanup while the API is unavailable. Once connectivity returns, Section 5.11 reconciles held autonomous/unclean incidents without manufacturing a requested-remediation CR.
- Provide general hard-dead-node VM high availability without a separately qualified external fencing topology.
- Support multiple StorageClasses or multiple probes per node in this release.
- Independently prove physical storage revocation or universal data integrity across all CSI backends. Automatic rejoin delegates backend crash recovery; a healthy probe or empty attachment list alone is not fencing proof.
- Enable the boot-hold profile on control-plane nodes, single-node OpenShift, or nodes eligible to host recovery dependencies under the placement contract in Section 5.11.
- Guarantee remediation CR lifecycle progress when no peer that meets Section 5.2 requirements remains running with API access.

## 5. Design

### 5.1. Use-case matrix

These rows describe the proposed BlockPerNode behavior. They distinguish supported decisions from deferred guarantees. They do not claim that every failure leads to automatic workload recovery.

Each requested remediation below requires an eligible peer to offer and process its ordinary CR lifecycle. Held-node rejoin and validation of its boot evidence can proceed independently through Section 5.11. Local watchdog and boot-hold enforcement do not depend on peer availability. If no such peer is available, ordinary request processing retains its state; the operator rejoin controller can still reconcile a held incident without waiting for a held worker to start.

| # | Scenario | Proposed behavior | Limit or dependency |
|---|---|---|---|
| 1 | Healthy storage and API | The local agent probes storage, feeds the watchdog, and reports storage as healthy. | Probe results describe the selected backend, not every workload volume. |
| 2 | Cluster-wide API outage with healthy storage | The local agent continues feeding the watchdog. It does not reboot its host for API loss alone. | Other Kubernetes and workload controllers have independent behavior. |
| 3 | Broad storage outage with reachable API | The local agent reports unhealthy storage after `maxConsecutiveFailures` consecutive failed health evaluations. It waits for a valid request from NHC. Running peers reconcile through the API despite their own probe failures. | NHC thresholds can block new requests. Reports can arrive late, and agents can already have decided to reboot. Therefore, these thresholds cannot guarantee that no node reboots. Peer startup can remain blocked by CSI. |
| 4 | Storage and API unavailable | After storage becomes unhealthy and the total-outage grace expires, the local agent performs at most one autonomous reboot during the unresolved outage period. SBR skips workload cleanup for this attempt and preserves any existing requested-remediation record. | The agent cannot reboot autonomously again if it has used its one attempt for the outage or cannot establish its history. After reboot, the host remains held even if the API returns. Recovery of Pods elsewhere is not guaranteed. |
| 5 | One node loses API or cluster-network access | The local agent keeps feeding the watchdog while its storage remains healthy. | Configure NHC for the storage condition only. |
| 6 | One node loses storage with API available | NHC creates a request. A peer records it as Offered on the Node. The target agent accepts the request and reboots. After the request reaches `BootEnded`, a peer applies OutOfService and checks cleanup until completion or the deadline. | Processing can stop while an accepted request has not reached BootEnded or the required boot information is unavailable. Rebooted targets remain held under Section 5.11; cleanup additionally requires qualified isolation evidence. |
| 7 | Kernel panic, host freeze, or power loss | A qualified armed watchdog can reset a powered host that stops feeding it. On an unclean host boot, durable boot admission blocks workload startup even without a pre-crash SBR decision. | The watchdog cannot restore power. Without trustworthy reset/hold evidence SBR cannot confirm this fencing path. A held returning worker is recovered by the operator rejoin controller. |
| 8 | Planned agent restart or upgrade | Before a planned agent stop, the local agent disarms its watchdog if it has not recorded a reboot decision. It preserves request history. Another peer that meets Section 5.2 requirements resumes the same API lifecycle state. After startup, the local agent compares the current host boot ID with the saved beforeBootID. | Preserve peer coverage during rollout. Check that the hardware supports watchdog disarm. A replacement local agent must acquire the host lock before controlling the watchdog. |
| 9 | Kubelet failure with healthy storage | The local agent continues feeding the watchdog while it remains functional. | This storage policy does not remediate every kubelet failure. |
| 10 | Slow or hung storage I/O | Bound each complete probe with `ioTimeout`. Count failed health evaluations without blocking watchdog decisions or peer processing of remediation CRs. | A deadline cannot cancel blocked kernel I/O. Limit unfinished probe operations within each process. Do not share locks that can block these independent tasks. |
| 11 | Network partition with healthy storage | Do not reboot solely for the partition. This NHC policy does not request relocation solely for it. | This is not a guarantee against concurrent workloads created by other controllers. |
| 12 | API or storage recovers after acceptance | The local agent completes the accepted reboot. It preserves the request’s initial boot ID and attempt history. If the local boot ID changes, the local agent clears only the FenceLatched flag for the previous boot. The host reporter publishes boot information while kubelet is held. Clearing the old FenceLatched flag does not clear the boot hold. A peer validates it before recording BootEnded. | When NHC stops a request or someone deletes its remediation CR, a peer can cancel the request only in Offered. Cleanup checks remain optional and have a fixed deadline. |
| 13 | Different nodes require different StorageClasses | Deferred. Add explicit node-group to StorageClass mapping. | See Section 6. |
| 14 | One node uses several storage backends | Deferred. Add multiple probes and a health aggregation policy. | See Section 6. |
| 15 | Stuck or terminating workload Pod | The peer uses OutOfService only with qualified isolation. At the fixed deadline, it finishes bounded observations and records incomplete or unknown cleanup separately; success still requires isolation. When someone deletes the remediation CR, the peer removes that CR holder; an independent rejoin hold continues to retain restrictions. | Workload finalizers cannot retain execution indefinitely. Cleanup expiry does not authorize rejoin; the independent boot hold remains. |
| 16 | Host journal unavailable or recovered | The local agent preserves known reboot decisions and compares them with trusted API records. If autonomous reboot history is unknown, do not permit another autonomous reboot. Peers continue their independent API work. | Missing history cannot prove that no autonomous reboot occurred during the outage. The read order in Section 5.7 can still establish BootEnded for a known accepted request. |
| 17 | Repeated remediation on persistent failure | Each new valid remediation CR permits one requested reboot after ownership release. Permit at most one autonomous reboot per incident. | Restore permission for one autonomous reboot after continuous healthy storage and reachable API for `autonomousRebootRecoveryPeriod`, which defaults to five minutes. |

### 5.2. Components and ownership

**Budget** means the durable permission for at most one autonomous reboot per unresolved outage period. Consuming it records an autonomous attempt; only continuous recovery and reconciled history rearm it. Requested reboots do not consume this budget and remain subject to their separate execution and boot-hold protocol.

NHC watches `SBRStorageUnhealthy=True` and creates an SBR remediation CR. Its name identifies the target node.

Peer agents run the existing `SBRRemediationReconciler`. In this document, a peer means this controller running on another node. Peer controllers process requests concurrently without a lifecycle leader or a Lease per CR. The target's peer controller skips its own Node UID. The target’s local monitoring loop accepts requests for its own host and performs the requested reboot.

The operator’s configuration controller manages SBR configurations and deployments. Its existing leader election does not coordinate remediation requests. An SBRRejoinReconciler in the existing operator manager handles held-node recovery under Section 5.11 using the operator's existing leader election; it does not select requested reboot ownership. A host-installed reporter starts without kubelet or the probe volume. It reads the host boot ID and publishes observations through its own Node status under Section 5.11. It cannot accept requests, control the watchdog, authorize rejoin, or perform cleanup.

`BlockPerNode` uses these components:

```mermaid
flowchart TB
  NHC["NHC: storage health policy"] --> CR["SBR remediation CR"]
  CR --> Peers["Concurrent eligible peer controllers"]
  Peers <-->|conditional updates| Record["Node annotation: execution, budget, restrictions"]
  subgraph Target["Target node"]
    Local["Local agent"] --> Probe["Private Block probe"]
    Local --> Journal["Durable host journal"]
    Local --> Watchdog["Watchdog and software reboot"]
    Reporter["Host boot reporter and gate"] --> Boot["Current host boot ID"]
    Reporter -->|read-only| Journal
  end
  Record --> Local
  Local -->|acceptance and observations| Record
  Reporter -->|own Node status observations| Record
  Peers --> OOS["OutOfService and bounded API observations"]
  Operator["Operator: configuration and deployments"] --> Local
  Operator --> Reporter
```

A peer can process a target’s requests when it runs on a different Node and uses the configuration UID recorded for the target. It needs API access and permission to process requests and update their records. Its probe need not be healthy, its watchdog need not be ready, and its Pod need not report Ready. Peers process requests across namespaces.

At least one eligible peer with API access is required for ordinary requested-remediation lifecycle work; held-node rejoin has the independent controller described in Section 5.11. The peer controller remains in the probe Pod. CSI attachment failure can therefore prevent a replacement peer from starting. The host reporter does not replace a peer or rejoin controller. A held node cannot run a peer Pod; the independently hosted rejoin controller can reconcile its held incident and ordered boot evidence without one. Ordinary requested offers and CR lifecycle work retain their eligible-peer requirement.

#### Storage-independent reconciliation

The agent process starts the BlockPerNode peer manager before probe initialization and storage preflight. The peer manager’s lifetime remains independent of probe and journal errors. When a local error occurs, the local agent follows the watchdog shutdown rules. The error must not cancel the peer controller.

Peer reconciliation uses API objects only. It does not read host journals, inspect probe devices, resolve shared NodeMap entries, or wait for storage locks. Limit the duration of API calls and the delay before retrying reconciliation. Keep probe I/O, journal I/O, watchdog feeding, and reconciliation on separate execution paths.

The peer determines the target’s configuration before selecting the shared-storage or BlockPerNode path. If configuration is missing or conflicting, peers cannot record a new request as Offered. Record the Node UID, configuration UID, volume mode, and initial boot ID permanently for each request. Never interpret missing status as permission for shared-storage fencing.

All state writers use conditional updates with `resourceVersion`. After a conflict, read current objects and check identities and permitted transitions again. Preserve unrelated conditions and fields assigned to other writers. Node conditions require strategic merge or an optimistic read-modify-write update.

#### One authoritative execution record

Reuse `sbr.medik8s.io/remediation-owner` on the Node. Its version-2 JSON records the current request, autonomous reboot counts and limits, CRs that require scheduling restrictions, and a coordination revision. Version 2 adds per-holder outOfServiceReleased; version-1 state must be migrated under the upgrade rules before this profile is enabled. CR status reports this record. The target cannot accept or execute a request based only on CR status.

The remediation CR UID identifies a requested attempt. The request records one Node UID and one `beforeBootID`. Neither value can change for this request. A changed generation, new configuration, or later boot cannot give that CR another attempt.

An illustrative record is:

```json
{
  "version": 2,
  "nodeUID": "node-uid",
  "coordinationRevision": 12,
  "execution": {
    "requestNamespace": "sbr-system",
    "requestName": "worker-0",
    "requestUID": "request-uid",
    "configUID": "config-uid",
    "beforeBootID": "boot-a",
    "phase": "Accepted",
    "observedBootID": "",
    "cleanupClosed": false
  },
  "budget": {
    "phase": "Known",
    "outageEpoch": "",
    "autonomousAttempts": []
  },
  "restrictions": {
    "preExistingUnschedulable": false,
    "preExistingOutOfService": null,
    "holders": [
      {
        "namespace": "sbr-system",
        "name": "worker-0",
        "uid": "request-uid",
        "configUID": "config-uid",
        "outOfServiceReleased": false
      }
    ]
  }
}
```

`coordinationRevision` is a Node-wide counter. Initialize it to zero with a valid new record. Preserve it when replacing a request record or removing a CR from the restrictions list. The peer increments it in every required release update during remediation CR deletion, including retries. Other writers preserve it. This changes the Node’s `resourceVersion`, even when no CR needs removal from the list.

The counter does not replace `resourceVersion` preconditions or identify a particular request. It requires no retained list of deleted CR UIDs. The release sequence below makes a peer’s earlier Offered update conflict with remediation CR deletion.

The budget fields retain autonomous attempt identities, not an unlimited event log. Retain attempts that still require resolution. Summarize older reboot counts only after the configured recovery period and required history checks.

If autonomous reboot history is missing, record `Unknown`. Do not assume that another autonomous reboot is permitted. Initialize this history only after proving first installation or obtaining complete trusted history. Accepting a requested reboot does not change whether another autonomous reboot is permitted. After API recovery, add the autonomous attempts and their outage period IDs to the API record.

The local agent and peer controllers change the Node execution record through these phase transitions:

| From | To | Writer and prerequisite |
|---|---|---|
| No current execution, or reusable terminal execution | `Offered` | Peer selects one request that passes the required checks and preserves all autonomous reboot and restriction records. |
| `Offered` | `Accepted` | Target confirms the local boot identified by beforeBootID and persists its request reservation. The Node update accepts the request without a budget charge. |
| `Offered` | `Cancelled` | Peer withdraws permission through a conditional update. No target acknowledgement is required. |
| `Accepted` | `BootEnded` | Peer, or the operator rejoin controller for a held incident, validates the ordered boot evidence required by Section 5.7. Both use the same conditional Node transition. |

In Offered, the target may accept the request but cannot yet reboot for it. Acceptance cannot be reversed for the recorded initial boot ID. The peer cancels the request and the target accepts it through conditional updates to the same Node record. Only one update can succeed. A peer cannot cancel an accepted request when NHC stops it or someone deletes its remediation CR. Storage recovery and elapsed time also cannot cancel acceptance.

The target confirms `Accepted` before requesting reboot or stopping watchdog feeds. A lost update response requires a live read. Before acceptance, the local host boot ID must equal the request’s initial boot ID in the API record. If they differ, the target must not accept or execute the request. The target can report the mismatch so a peer cancels the request, as Section 5.7 describes.

#### Select and finish requests

Before recording Offered, the peer verifies enrolled boot protection and that Section 5.11 permits a new request, then installs the remediation CR’s finalizer. It also records the Node UID, configuration UID, volume mode, and initial boot ID. These values cannot change for this request. Use one conditional Node update to record Offered, establish the incident-bound rejoin owner, add the CR to the restrictions list, and cordon the Node. Save the original cordon and OutOfService state only when adding the first CR to the list.

Before replacing a completed request record, the peer saves its terminal phase in the remediation CR if that CR still exists. A `Cancelled` or `BootEnded` CR never receives another offer. Keep the old terminal execution in the annotation until a successor replaces it.

To record a new request as Offered, the peer performs these steps in order:

1. Read the Node identified by the recorded UID through the uncached API. Check its UID and retained record.
2. Read the CR through the uncached API. Check its UID, recorded identities that cannot change, finalizer, deletion timestamp, stop state, and terminal state.
3. Use a conditional update against that Node version to record Offered, add the CR to the restrictions list, and cordon the Node.

After a conflict, the peer repeats this complete sequence. It reads the remediation CR again after reading the new Node version. Do not use an earlier or cached check to decide whether the request can enter Offered. Finalizer removal requires the Node update below. Therefore, a peer’s update prepared before normal remediation CR deletion cannot record Offered after that deletion completes.

No other request can replace an accepted request until the execution reaches the `BootEnded` phase. A peer can then replace the record when cleanup checks finish, NHC stops the request, or its remediation CR is deleting or absent, but only after the independent rejoin handshake permits a new request. Cleanup observations retain a fixed deadline; unresolved rejoin can independently delay a later request indefinitely. An accepted request that has not reached BootEnded can still block another request.

NHC can retain a stopped remediation CR as history after its request finishes. The peer preserves the stopped outcome in that CR. Do not wait for its deletion before accepting a different request that passes the required checks. NHC's own health thresholds, leases, template order, and existing CR names remain independent constraints.

#### Restrictions that outlive execution

The Node record’s `holders` field lists the CRs that still require SBR’s scheduling restrictions. Each entry identifies a CR by UID, not just by name. Keep the original cordon and OutOfService state separately from the current request. Retain this saved state while any CR remains in the list. A later request must not treat an earlier SBR restriction as an administrator’s restriction.

A remediation CR remains in the restrictions list while it exists, even if NHC stopped its request or SBR reported success. Finishing cleanup checks does not remove OutOfService. Section 5.11 can explicitly discharge a resolved holder's OutOfService requirement by setting outOfServiceReleased=true while retaining its CR and cordon bookkeeping; this breaks the NHC-health/held-kubelet deletion dependency. When someone deletes the remediation CR, the peer removes only that CR’s entry after its request reaches Cancelled or BootEnded.

While any CR or unresolved rejoin record requires a restriction, retain that restriction. A holder with outOfServiceReleased=true no longer requires OutOfService, and cannot cause its reapplication; preserve any remaining cordon requirement. Removing the last CR does not release a rejoin hold. Remove only SBR-added restrictions after Section 5.11 permits release, and retain the original snapshot until then. Preserve request history, rejoin state, and autonomous reboot records in every update.

A peer handling delayed deletion of remediation CR A cannot remove remediation CR B’s entry or restrictions. If a CR is absent, apply the same removal rules using its saved UID. A replacement CR with the same name has a different UID and needs a separate entry.

Preserve any pre-existing cordon, OutOfService taint, and unrelated taints. The OutOfService snapshot identifies the exact key and NoExecute effect that SBR manages. Preserve taints with other effects separately. Check the current taint value before removing an SBR-added taint. The snapshot cannot detect an administrator reasserting the same cordon later. Concurrent administrators must follow the documented ownership convention.

The restrictions list can grow as CRs remain after processing. Remove entries only after checking that their restrictions were released or their CRs are absent, and their requests are resolved. Limit record size. Reject new Offered requests with a visible reason before reaching annotation limits. Never discard CR entries that still require restrictions or autonomous reboot history to make space.

#### Automatic execution recovery

Peer controllers watch Nodes as well as remediation CRs. They process a saved request record even if someone forcibly deleted its remediation CR. For an Offered request, a peer uses a conditional Node update to change its phase to Cancelled. For an Accepted request, the peer waits for the boot information required by Section 5.7. The host reporter publishes into Node status, so CR absence does not remove its evidence channel.

The unchanged initial boot ID prevents the target from executing the same request again after `BootEnded`. Cancellation changes the Node record from Offered to Cancelled. The target can no longer change that record from Offered to Accepted, so the cancelled request cannot cause a reboot.

A listed CR other than the current request has already reached Cancelled or BootEnded. Replacing its request record requires that result. If this CR disappears, a peer can remove its list entry without using the later request’s phase as the earlier request’s result.

Peers use the Node record to retry interrupted removal of CR entries and restrictions. If a CR has no list entry, no restrictions need removal for it. However, an earlier update that has not completed might still add its entry and record the request as Offered. Do not remove restrictions belonging to CRs added after the earlier list became empty. A later listed CR must not delay removal of the earlier CR’s finalizer. Incomplete workload cleanup does not prevent this removal.

Before removing a remediation CR’s finalizer, the peer performs the following sequence. This includes requests without an Offered record or restrictions entry:

1. Read the deleting CR through the uncached API. Check its UID.
2. Read the Node identified by the recorded UID through the uncached API. Check its UID and retained record. Cancel or resolve any execution belonging to this CR before release.
3. Conditionally remove the matching CR from the list. Remove SBR-added restrictions only when the list is empty and no independent rejoin hold requires them. Increment `coordinationRevision` in that same Node update, even when this CR has no entry in the restrictions list. Preserve other executions, rejoin state, listed CRs, restriction snapshots, and budget history.
4. Confirm that the Node update succeeded. Remove the finalizer using CR UID and `resourceVersion` checks. Do not wait for global schedulability or removal of the controller-managed unschedulable taint.

An update that leaves Node data unchanged does not satisfy this sequence. If the release update wins, an earlier offer update fails its Node version check. If the offer wins, the releasing peer must read it and cancel or resolve execution before completing release. Both paths repeat their complete checks after a conflict.

After an unknown response or peer restart, the peer repeats the conditional Node release update before removing the remediation CR’s finalizer. Another increment is safe. An absent list entry alone does not confirm that the required Node update succeeded. Failed API writes do not establish release. Never perform this update on a replacement Node UID.

On each retry, check which CRs currently require restrictions. If this CR has no list entry, preserve later CRs’ restrictions and their saved original state. Never repeat an earlier decision to uncordon the Node or remove a later CR’s OutOfService taint. A later cordon does not change the fact that this CR’s entry was removed.

A missing or invalid record cannot be replaced merely to perform release. Apply the existing history recovery rules.

When someone deletes the Node object, Kubernetes also deletes its annotation containing the request record. A replacement Node UID does not prove that the old host stopped. Never mutate the replacement through the old request. Recover only from trusted retained history or external evidence. Missing or conflicting identity and budget records cannot create permission for another autonomous reboot.

Permanent loss of both required local history and API history cannot resolve automatically. Likewise, an accepted request cannot reach BootEnded without trustworthy boot information. These cases need restored evidence or external recovery, rather than a timeout-based claim of success.

### 5.3. Configuration API

Extend `StorageBasedRemediationConfig.spec.sharedStorageVolumeMode` with `BlockPerNode`. Expose `spec.ioTimeout` as an optional duration in all modes. Reuse the existing `spec.sbrTimeoutSeconds` and `spec.maxConsecutiveFailures` fields. Add a total-outage grace, an autonomous recovery period, and rejoinPolicy for BlockPerNode. Storage detection uses the existing failure threshold.

| Field | Effective default | Validation | Meaning |
|---|---|---|---|
| `sharedStorageVolumeMode` | `Filesystem` | `Filesystem`, `Block`, or `BlockPerNode` | Existing shared Filesystem, existing shared Block, or one private Block probe per selected node. |
| `ioTimeout` | `15s` in `BlockPerNode`, `2s` in existing modes | Duration from `1s` through `10m`. Valid in all modes. | Complete probe deadline in BlockPerNode. Existing per-operation timeout behavior in shared modes. |
| `sbrTimeoutSeconds` | Existing `30` | Existing integer range `10–300` | Supplies the BlockPerNode evaluation and probe interval through `GetSBRUpdateInterval()`. |
| `maxConsecutiveFailures` | Existing `7` | Existing integer range `2–32` | Consecutive failed health evaluations required before BlockPerNode reports unhealthy storage. |
| `totalOutageGracePeriod` | `60s` in `BlockPerNode` | Duration from `0s` through `1h`. Valid only in `BlockPerNode`. | The API must remain unavailable for this period while storage is unhealthy before the agent records an autonomous reboot decision. |
| `autonomousRebootRecoveryPeriod` | `5m` in `BlockPerNode` | Positive duration. Valid only in `BlockPerNode`. | Storage must remain healthy and the API reachable for this period before another autonomous reboot is permitted. |
| `rejoinPolicy` | `Automatic` in `BlockPerNode` | `Automatic` or `AdminAssisted`; rejected in other modes. Effective value immutable for a configuration UID and snapshotted per incident. | Automatic requires verified isolation and standard API reconciliation under the storage crash-recovery contract. AdminAssisted additionally requires recorded external-runbook completion. Neither uses backend integrations or probe-only release. |

The BlockPerNode defaults do not replace existing shared-storage thresholds. Omitted `ioTimeout` preserves the existing two-second timeout in Filesystem and Block. An omitted volume mode remains equivalent to explicit `Filesystem`. Explicit `Block` continues to select shared storage.

Use effective getters for defaults. Do not assign an unconditional CRD default to `ioTimeout`, because its effective default depends on the mode. Reject `ioTimeout: 0s`. Explicit `totalOutageGracePeriod: 0s` removes only the additional combined-outage delay. It does not disable detection or autonomous remediation. The autonomous recovery period must be greater than zero.

Proposed additions in [configuration types](../../api/v1alpha1/storagebasedremediationconfig_types.go):

```go
package v1alpha1

// Existing imports, type, constants, and other fields are omitted.
const (
	SharedStorageVolumeModeBlockPerNode SharedStorageVolumeModeType = "BlockPerNode"
	DefaultIOTimeout                                                = 2 * time.Second
	DefaultBlockPerNodeIOTimeout                                    = 15 * time.Second
)

type StorageBasedRemediationConfigSpec struct {
	// +kubebuilder:validation:Enum=Filesystem;Block;BlockPerNode
	// +optional
	SharedStorageVolumeMode *SharedStorageVolumeModeType `json:"sharedStorageVolumeMode,omitempty"`

	// +optional
	IOTimeout *metav1.Duration `json:"ioTimeout,omitempty"`

	// +optional
	TotalOutageGracePeriod *metav1.Duration `json:"totalOutageGracePeriod,omitempty"`

	// +optional
	AutonomousRebootRecoveryPeriod *metav1.Duration `json:"autonomousRebootRecoveryPeriod,omitempty"`

	// +kubebuilder:validation:Enum=Automatic;AdminAssisted
	// +optional
	RejoinPolicy *string `json:"rejoinPolicy,omitempty"`
}

func (s *StorageBasedRemediationConfigSpec) IsBlockPerNodeMode() bool {
	return s.GetSharedStorageVolumeMode() == SharedStorageVolumeModeBlockPerNode
}

func (s *StorageBasedRemediationConfigSpec) GetRejoinPolicy() string {
	if s.RejoinPolicy != nil {
		return *s.RejoinPolicy
	}
	if s.IsBlockPerNodeMode() {
		return "Automatic"
	}
	return "" // Not applicable in shared modes.
}

func (s *StorageBasedRemediationConfigSpec) GetIOTimeout() time.Duration {
	if s.IOTimeout != nil {
		return s.IOTimeout.Duration
	}
	if s.IsBlockPerNodeMode() {
		return DefaultBlockPerNodeIOTimeout
	}
	return DefaultIOTimeout
}
```

Keep `GetSharedStorageVolumeMode()` and its `Filesystem` fallback. Keep `IsBlockMode()` specific to shared `Block` behavior. Its existing callers select shared devices, RWX claims, and shared-volume cleanup. Use `IsBlockPerNodeMode()` before those paths instead of treating both values as shared Block.

`BlockPerNode` is an SBR configuration value. It is not a Kubernetes PV or PVC volume mode. The generated private claim still uses `volumeMode: Block` and `ReadWriteOnce`.

Update the `sharedStorageClass` description and validation for both Block values. Preserve the existing StorageClass requirement for shared `Block`. `BlockPerNode` requires a class that supports the private RWO Block claims described in Section 5.4.

Update the custom resource definition (CRD), generated code, webhook, samples, and agent arguments together. The controller currently pins `--io-timeout` to the two-second [agent constant](../../internal/agent/constants.go). Pass `GetIOTimeout()` through `buildSBRAgentArgs()` and the existing `buildBlockInitJob()` in [the configuration controller](../../internal/controller/storagebasedremediationconfig_controller.go).

The [preflight checks](../../cmd/sbr-agent/preflight.go) also use a separate fixed two-second timeout. Pass the effective value into applicable initialization and preflight operations. A configured runtime timeout must not leave startup checks using the previous constant. Keep the BlockPerNode peer-manager startup independent of these storage checks.

Existing modes retain per-operation timeout and retry behavior. Their complete operations can therefore exceed `ioTimeout`. The Filesystem `ReopenDevice()` fallback does not currently enforce a timeout. Exposing the field does not add a deadline guarantee to that fallback. See [device operations](../../internal/blockdevice/blockdevice.go) and [device selection](../../cmd/sbr-agent/main.go).

For BlockPerNode, use one deadline for the complete verification probe, including required flush operations and retries. A nested timeout must not shorten this configured deadline unintentionally. An actual I/O error can still finish the probe early. Keep retries internal and within the original deadline. Section 5.5 defines the health result.

The configured storage timeout does not set API, journal, cleanup, or hardware watchdog timeouts. Keep `rebootMethod` internal. Keep the `10Mi` probe size as a qualified internal value.

#### Field interactions

| Configuration | Existing `Filesystem` or `Block` outcome | `BlockPerNode` outcome |
|---|---|---|
| `sharedStorageVolumeMode` omitted or `Filesystem` | Existing shared Filesystem behavior. | Not selected. |
| `sharedStorageVolumeMode: Block` | Existing shared ReadWriteMany (RWX) Block behavior. | Not selected. |
| `sharedStorageVolumeMode: BlockPerNode` | Not selected. | One private ReadWriteOnce (RWO) Block claim per agent Pod. |
| `sharedStorageClass` | Existing shared-storage validation and provisioning. | Required. Selects the single StorageClass for private probes. It must support dynamic RWO Block provisioning. |
| `watchdogPath` | Existing behavior. | Same device path. One local monitoring loop owns the watchdog. |
| `nodeSelector` omitted or empty | Existing worker-node selection. | Same worker-node selection. The effective selector includes qualified Linux workers only; Section 5.11 excludes control-plane and recovery-dependent nodes. |
| `sbrTimeoutSeconds` | Existing timeout and derived intervals. | Reuse `GetSBRUpdateInterval()` for probe scheduling and failed-state evaluation. It divides the duration by six, with a one-second minimum. |
| `maxConsecutiveFailures` | Existing counter behavior. | Report unhealthy at this many consecutive failed health evaluations. Count evaluations, not individual read, write, or retry errors. |
| `ioTimeout` | Existing per-operation behavior with a configurable value. Omission retains `2s`. | Complete probe deadline. Omission selects `15s`. Expiry starts a failed interval and does not bypass the failure threshold. |
| `detectOnlyMode: Enabled` | Existing detection-only behavior. | Block new remediation actions. Continue reporting boot information and checking cleanup for a request with an already recorded reboot decision. |
| `detectOnlyMode` omitted or `Disabled` | Existing behavior. | Enable requested and autonomous actions under the policy below. |
| `totalOutageGracePeriod` or `autonomousRebootRecoveryPeriod` is present | Admission rejects the inapplicable field. | Apply the effective duration. |
| `rejoinPolicy` | Admission rejects the inapplicable field. | Omitted means Automatic; AdminAssisted adds external-runbook approval. Snapshot the policy per incident; no timeout bypasses either policy. |

The existing timeout defaults remain `30` seconds and seven consecutive failures. Their schema ranges remain unchanged. BlockPerNode does not use shared-heartbeat expiry or the existing mixed operation counters. Multiplying `ioTimeout` by `maxConsecutiveFailures` does not define its detection duration.

Qualify the fifteen-second BlockPerNode default and supported overrides on CSI storage that represents supported workloads. The `1s–10m` range matches current device validation. It does not establish suitable latency for every backend. Do not require `ioTimeout` to be shorter than the probe interval or hardware watchdog timeout. Independent observations and watchdog feeding must continue while a probe runs.

An example BlockPerNode configuration is:

```yaml
apiVersion: storage-based-remediation.medik8s.io/v1alpha1
kind: StorageBasedRemediationConfig
metadata:
  name: sbr-block-per-node
  namespace: sbr-system
spec:
  sharedStorageVolumeMode: BlockPerNode
  sharedStorageClass: example-block
  ioTimeout: 15s
  sbrTimeoutSeconds: 30
  maxConsecutiveFailures: 7
  totalOutageGracePeriod: 60s
  autonomousRebootRecoveryPeriod: 5m
  rejoinPolicy: Automatic
```

#### Validation and immutability

Extend `ValidateCreate()` and `ValidateUpdate()` in the [configuration webhook](../../api/v1alpha1/storagebasedremediationconfig_webhook.go).

1. Accept `Filesystem`, `Block`, and `BlockPerNode` in the existing volume-mode validation.
2. Validate duration syntax and bounds, including a positive autonomous recovery period. Require a StorageClass for `BlockPerNode`.
3. Accept `ioTimeout` in all modes. Reject total-outage and autonomous recovery fields outside BlockPerNode.
4. Reject changes to the effective volume mode, total-outage grace, autonomous recovery period, and rejoinPolicy. Reject unknown/empty explicit rejoinPolicy and any rejoinPolicy in shared modes; omitted and explicit Automatic are equivalent only in BlockPerNode. Retain the snapshotted incident policy across configuration replacement.
5. Reject configurations that overlap a BlockPerNode configuration on selected nodes.
6. Reject different StorageClasses across BlockPerNode configurations in this release.

Compare durations as values, not strings. Omitted volume mode and explicit `Filesystem` are equivalent. Omitted durations and explicit effective defaults are equivalent. Reject removal of a nondefault immutable duration. Permit valid `ioTimeout` updates through the controlled replacement rules below.

The existing volume-mode immutability rule covers transitions to and from `BlockPerNode`. Switching between any two distinct effective volume modes requires configuration replacement. Existing field update rules remain unchanged. Mutable fields can still require an agent rollout. For BlockPerNode, the configuration controller waits until accepted requests reach BootEnded and peers remove the required CR entries and restrictions. Only then does the controller replace affected local monitoring loops. Incomplete observations alone are not a replacement gate, but unresolved rejoin ownership must retain compatible protection, configuration, and recovery permissions.

Apply this replacement rule to selector, StorageClass, watchdog path, detect-only, `ioTimeout`, `sbrTimeoutSeconds`, and `maxConsecutiveFailures` changes. A selector update must protect removed nodes as well as newly selected nodes. The configuration controller retains the effective configuration until its requests are resolved and peers remove its CRs from the restrictions list. Peers cancel requests still in Offered through the Node record.

Each started probe retains its original deadline. Timing changes take effect in the replacement monitor. The replacement restarts transient detection as Section 5.10 describes.

Current production code does not enforce configuration overlap consistently. The webhook and controller each refer that check to the other component. The new checks must cover namespaces, effective worker selectors, and the Linux restriction. Admission alone cannot prevent concurrent creates or later node-label changes.

The configuration controller must reconcile assignments after admission, including concurrent creates and later label changes. If the effective configuration is missing, conflicting, unreadable, or ambiguous, do not prepare requests, record reboot decisions, or start destructive actions. An action whose reboot decision is already recorded continues only with the configuration saved and retained for that request.

One host process holds permission to perform local actions through a non-blocking exclusive `flock` on `/var/lib/medik8s/state/sbr-executor.lock`. The lock file is separate from the atomically replaced journal and is never unlinked during operation. The local agent acquires the lock before opening or arming the watchdog, loading mutable journal state, or accepting an offer. It holds the lock for its lifetime.

If locking fails or is unsupported, the local agent reports `ExecutorConflict`. That process performs no watchdog or journal writes. Its peer controller can continue API reconciliation if it meets the requirements in Section 5.2. Never break the lock by Pod UID, Lease expiry, peer replacement, or age. Never disarm a watchdog opened by an unknown process.

Set DaemonSet `maxSurge: 0` and `maxUnavailable: 1` explicitly. These settings do not replace the host lock. Assignment checks select the configuration. Node ownership selects the request. Neither selects the process that holds the watchdog.

#### Configuration replacement

To change an immutable value, the administrator deletes the old SBR config CR. The administrator creates its replacement after the shutdown and resource-removal steps below complete. An in-place DaemonSet rollout cannot change these immutable values. Mutable field updates follow the safe rollout rules above.

When the administrator deletes a BlockPerNode SBR config CR, the components complete these steps:

1. The configuration controller prevents new requests from using the old configuration.
2. Peer controllers wait until accepted requests reach BootEnded. They remove associated CR entries as Section 5.2 requires. Unresolved rejoin holds retain restrictions and block retirement of required host protection and recovery permissions; cleanup timeout is not permission to remove them.
3. Each local agent without a recorded reboot decision verifies watchdog disarm before stopping its monitoring loop.
4. The configuration controller waits for the old local agents and reporters to terminate before releasing the nodes from the old configuration.
5. The configuration controller removes resources owned by the old configuration and permits a replacement configuration.

An unreachable node can delay SBR configuration replacement. Do not start a second watchdog owner to avoid that delay. Preserve the host journal across replacement. Configuration deletion must not erase completed request identities.

The configuration controller retains the agent DaemonSet, its service account, and required API permissions while assigned requests need further processing. This includes cancellation, confirmation of BootEnded, and removal of CR entries and restrictions. When the administrator removes a configuration, peers stop its cleanup observations without waiting for successful workload cleanup. The configuration controller includes saved Node execution records and restrictions entries when the associated remediation CR is missing. Keep a peer that meets Section 5.2 requirements for each affected target during planned rollout or retirement. A terminating configuration prevents new requests but does not disqualify peers from processing its retained requests.

If no peer remains, the configuration controller reports that configuration replacement is blocked and waits for a peer.

The current configuration deletion handler removes role-based access control (RBAC) and relies on garbage collection for the DaemonSet. It does not establish this shutdown order. Add a separate BlockPerNode deletion path in [the configuration controller](../../internal/controller/storagebasedremediationconfig_controller.go). Do not apply shared-PV cleanup to private probe claims.

### 5.4. Private probe volume

Each selected node runs one local SBR agent with one generic ephemeral Block volume. The Pod owns its PersistentVolumeClaim (PVC). The controller uses a named `10Mi` request value rather than an embedded manifest literal. Driver allocation granularity can produce a larger volume. A supported backend must accept that request or define its qualified minimum.

The StorageClass must support raw Block, dynamic provisioning, and the probe's direct I/O operations. The claim requests `ReadWriteOnce`. SBR does not require concurrent writes from other nodes.

`SBRStorageUnhealthy` describes this selected probe path, not every VM disk or the whole storage array. Each supported policy documents how the probe StorageClass, backend, and failure domain represent the workloads eligible for remediation. Block destructive remediation if tests have not established that the probe is suitable, or if it does not represent the affected workload storage. Distinguish `ProbeNotProvisioned`, `ProbeNotAttached`, `ProbeIOFailed`, `StoragePathUnhealthy`, and `BackendUnqualified` in status and events.

Add a BlockPerNode branch before `validateStorageClass()`, `ensurePVC()`, and `ensureSBRDevice()` in [the configuration controller](../../internal/controller/storagebasedremediationconfig_controller.go). Their current shared-volume path rejects some RWO provisioners and creates an RWX PVC. Selecting `Block` alone does not change that behavior. Prefer `WaitForFirstConsumer` for topology-constrained storage. Warn, rather than reject, when `Immediate` binding is combined with topology constraints.

The proposed DaemonSet fragment is:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
  template:
    spec:
      containers:
        - name: sbr-agent
          # Existing image, arguments, and host mounts are omitted.
          volumeDevices:
            - name: storage-probe
              devicePath: /dev/sbr-probe
      volumes:
        - name: storage-probe
          ephemeral:
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                volumeMode: Block
                storageClassName: example-block
                resources:
                  requests:
                    storage: 10Mi
```

Keep the watchdog device mount separate from `/dev/sbr-probe`. Mounting the entire host `/dev` over that path can hide the CSI device.

A container restart reuses the Pod's PVC. A node reboot can also retain the same Pod and PVC. A replacement Pod receives its own PVC. Kubernetes deletes the old PVC with its owning Pod. The PersistentVolume (PV) reclaim policy controls what happens to the underlying volume. See [generic ephemeral volume lifecycle](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#lifecycle-and-persistentvolumeclaim).

The probe contains disposable test data. It contains no authoritative reboot evidence, shared NodeMap, peer heartbeat, or fence message. SBR does not mount another node's probe.

The local agent must select the private prober before shared-storage initialization in `NewSBRAgentWithWatchdog()`. Disable shared heartbeat writes, peer checks, and fence-slot reads only in BlockPerNode. Keep the existing shared-storage initialization and runtime paths intact. See [the agent constructor and loops](../../cmd/sbr-agent/main.go).

The BlockPerNode entry point must branch before the current storage preflight and constructor. Initialize API clients and start the peer manager without opening the probe. Then initialize the local agent independently. The existing constructor initializes storage and NodeMap before its manager, so it cannot provide this ordering unchanged. Section 5.10 defines the required lifetime separation.

Run the evidence reporter as the host service in Section 5.11, with read-only journal access and no watchdog device. Neither a sidecar nor a separate DaemonSet can report while kubelet is blocked. Report missing host-reporter coverage independently of local agent Pod readiness. The Pod-backed CSI probe is unavailable during the boot hold and cannot be its release prerequisite.

The initial implementation keeps one StorageClass across BlockPerNode configurations. Admission rejects different selections, and the controller checks concurrent creations before starting agents. Section 6 describes the deferred extension.

### 5.5. Storage and API decision policy

The local agent’s monitoring loop evaluates storage health and live API reachability independently. The local agent performs the probe, health-reporting, and watchdog actions in this section. Probe I/O, API calls, journal writes, and watchdog feeding must not share a blocking execution path.

The local agent writes data to the probe volume and reads it to check storage access. Use one deadline for the complete probe, including required flush operations and retries. Define these values:

- `I` is the effective `ioTimeout`, defaulting to fifteen seconds.
- `P` is `GetSBRUpdateInterval()`, defaulting to five seconds.
- `M` is `maxConsecutiveFailures`, defaulting to seven.

The local agent applies this failure-count policy:

1. On each scheduled tick, check for unfinished probes or probe I/O in the current process. Start a probe only if none remain.
2. A completed probe error or expiry of `I` starts a failed interval with count one.
3. Schedule the next failed-state evaluation no earlier than `P` after that first failure.
4. While the failed state remains, increment the count once per evaluation, with at least `P` between increments.
5. Additional probe errors during that interval do not increment the count between evaluations or restart its schedule.
6. When the count reaches `M`, treat storage as unhealthy and publish `SBRStorageUnhealthy=True` when API access permits.
7. A fresh complete success clears the failed state and resets the count to zero, before any recorded reboot decision.

These are failed health evaluations, not separate failed I/O attempts. Timeout expiry contributes to the threshold. It does not make storage unhealthy immediately. Pending I/O before its deadline is not a new failure. During an existing failed interval, pending I/O does not establish recovery.

Do not reset the count merely because time elapsed. Do not apply the existing ten-minute reset of mixed operation counters. Use a monotonic clock within a boot. Do not manufacture missed evaluations after scheduling delays.

For hung I/O, the deadline marks failure even if the kernel operation continues. A Go timeout does not cancel an operation blocked in the kernel. Bound outstanding operations to one probe operation per device within the current process. Continue failed-state evaluation while that operation remains blocked. Do not launch another operation on each tick or retry while it remains outstanding.

When an expired operation eventually finishes, its late result cannot clear the failed state. Only a subsequent fresh complete probe can restore health. Process a fresh successful result before a simultaneous failed-state evaluation. After recording a reboot decision, preserve the current boot's decision and condition as Section 5.9 requires.

This timing example starts with healthy storage and a zero failure count. With default values, continuous failure needs six further evaluations after the first failure. This takes at least thirty seconds after that first failure. For the blocked-probe example, its deadline produces the first failure. That probe first consumes its fifteen-second deadline. It therefore takes at least forty-five seconds from probe submission to unhealthy storage.

Scheduling and status publication can add delay. An immediate error starts counting earlier.

If API access remains unavailable, the additional default total-outage grace is sixty seconds. In the same initially healthy example, an autonomous decision is permitted no earlier than 105 seconds after probe submission. This is not a reboot-completion deadline. Request acceptance and autonomous history checks remain separate.

For BlockPerNode, each agent writes `SBRStorageUnhealthy` only on its own Node. It does not infer another node's storage health from a missing heartbeat. It never writes `Unknown`. The existing `Unknown` value is a shared-storage peer-monitor signal and is not part of BlockPerNode.

The local agent updates its Node’s storage condition through a strategic merge patch or an optimistic-lock read-modify-write retry. This preserves unrelated Node conditions. Preserve `LastTransitionTime` while the status is unchanged. Combine duplicate writes. Vary the delay before publishing a failure transition, within a fixed limit. NHC's condition duration adds a separate delay after this condition becomes True.

| Storage state | Live API result | Local action before any recorded reboot decision |
|---|---|---|
| Healthy | No valid offered request | Feed the watchdog. Do not reboot solely because API access failed. |
| Failed evaluations below threshold | No valid offered request | Continue observation and feed the watchdog. |
| Unhealthy | Reachable, no authorized request | Publish True when possible. Feed the watchdog while NHC decides. |
| Any state | Reachable, valid offered request | Persist the request reservation, conditionally accept, persist the reboot decision, and execute the requested reboot without a budget check. |
| Unhealthy | Unavailable within total-outage grace | Feed the watchdog while the grace runs. |
| Unhealthy | Unavailable through total-outage grace | Check that the autonomous history is known and the one attempt for this outage has not been used. If both checks pass, persist an autonomous attempt. Then execute one local reboot operation. Skip cleanup. |
| Unhealthy | Unavailable, autonomous attempt already used or history unknown | Record exhaustion or uncertainty when possible and keep feeding. Block further autonomous reboots until the recovery period and required history checks permit another attempt. |
| Any state | Detect-only enabled, no previously recorded reboot decision | Report observations. Do not perform remediation actions. |

Acceptance requires matching request UID, Node UID, configuration UID, and initial boot ID. Each valid externally created CR permits one requested reboot, regardless of current storage health or whether another autonomous reboot is permitted. Existing execution, detect-only, identity, and durability checks still apply. Once the Node record becomes `Accepted`, later health recovery cannot cancel that decision.

Storage recovery alone does not cause a peer to cancel an Offered request. NHC can request CR deletion under its health policy. When NHC stops a request or someone deletes its remediation CR, a peer tries to change Offered to Cancelled. This conditional Node update competes with the target agent’s update to Accepted. A separate CR read cannot make this cross-object race atomic.

If acceptance succeeds first, the peer must wait for the accepted request to reach BootEnded. It cannot cancel that request.

To check API reachability, send an authenticated live request with a time limit. A cached informer result is insufficient. The routine check uses the small SBR remediation CR, or `/readyz` where RBAC permits. A full uncached Node read is reserved for recording request identities, recording a reboot decision, and updating evidence.

A successful empty result or `NotFound` means the endpoint is reachable. However, Node absence or name reuse is an identity mismatch. Authorization, certificate, proxy, and client-configuration failures prevent request authorization. These failures must not become combined-outage triggers.

Connection failures and network deadlines contribute to API unavailability. For this policy, a received `429` or `5xx` means the configured endpoint is reachable. It cannot authorize a request. Keep feeding and reset the API-unavailability interval for those responses. Credential, certificate-validation, and client-configuration errors report degradation instead of triggering autonomous remediation. Neither a failed Node status write nor a missing remediation CR alone establishes API unavailability.

The local agent starts `totalOutageGracePeriod` when storage is unhealthy and API access is unavailable together. It clears this interval if storage recovers or the API becomes reachable. Its duration starts after the failed-evaluation threshold is reached, rather than at the first probe error. It does not replace storage detection.

```mermaid
flowchart TD
  Tick["Independent observations"] --> Committed{"Already committed in this boot?"}
  Committed -->|Yes| Finish["Keep watchdog feeding stopped"]
  Committed -->|No| Detect{"Detect-only enabled?"}
  Detect -->|Yes| Observe["Report only"]
  Detect -->|No| Request{"Valid live offer for this Node and boot?"}
  Request -->|Yes| Accept["Persist request reservation and confirm API acceptance"]
  Accept --> Commit["Persist local commitment before reset"]
  Request -->|No| Storage{"Failed evaluation count reached threshold?"}
  Storage -->|No| Feed["Feed watchdog"]
  Storage -->|Yes| API{"Live API reachable?"}
  API -->|Yes| Wait["Report storage failure and await NHC"]
  Wait --> Feed
  API -->|No| Grace{"Total-outage grace expired?"}
  Grace -->|No| Feed
  Grace -->|Yes| Budget{"Autonomous budget available?"}
  Budget -->|No| Feed
  Budget -->|Yes| Commit
  Commit --> Reset["Software reboot with watchdog fallback"]
```

NHC’s health thresholds can prevent new requests during a broad storage outage. They cannot stop an already recorded reboot decision or an autonomous reboot during API loss. Reports can arrive at different times. An agent that already recorded its reboot decision continues the reboot.

### 5.6. Host journal and repeated-request checks

This design requires a new `github.com/medik8s/common/pkg/nodestate` package. It does not exist in `common v1.17.0`, which the reviewed projects currently pin. Installation requires a common release that provides this contract:

- Node-UID request keys and `ReadBootID`.
- Versioned schemas that preserve unknown top-level fields.
- Required identity and budget records that diagnostic pruning cannot remove.
- Explicit recovery and durability outcomes.
- Safe path handling.
- Atomic temporary writes, file `fsync`, rename, and directory `fsync`.

The local agent writes `/var/lib/medik8s/state/sbr.json` on the host. Mount its directory at `/var/lib/medik8s/state` inside the local agent container. This path is independent of the probe volume and Pod identity. Only the process holding the common host executor lock writes the journal; this includes serialized host boot/shutdown/release transactions when the local agent does not own it. The reporter can read completed snapshots.

Use the proposed common envelope below. These names specify the required API, not an existing `v1.17.0` implementation.

| SBR data | Common field | JSON location |
|---|---|---|
| Format version | `Document.SchemaVersion` | `schemaVersion` |
| Node name and immutable Kubernetes Node UID | `Document.Node`, `Document.NodeUID` | `node`, `nodeUID` |
| Current local kernel boot ID | `Document.BootID` | `bootID` |
| Effective configuration | `Document.Config` | `config`, with string values |
| Recorded reboot decision for the current boot | `Agent.FenceLatched` | `agent.fenceLatched` |
| Agent shutdown state | `Agent.Shutdown` | `agent.shutdown` |
| Required SBR state | `Agent.Extra` | `agent.extra`, with string values |
| Diagnostic reboot records | `Document.Reboots` | `reboots` |
| Diagnostic samples and events | `Document.IO`, `Document.Conditions`, `Document.Events` | `io`, `conditions`, `events` |

Use seven SBR keys inside `agent.extra`: `sbrStateVersion`, `activeAttempt`, `resetLedger`, `outageState`, `bootHold`, `bootAdmission`, and `autonomousInhibit`. Set `sbrStateVersion` to the string `"2"`. Section 5.11 defines bootHold, bootAdmission, and migration; older readers must reject this version rather than infer a clean boot. Encode each object or array as a JSON string because `Agent.Extra` has type `map[string]string`. The local agent and serialized host boot/shutdown/release helpers validate and change their assigned records. The common library preserves them without interpreting their contents.

The requested action identity is `(StorageBasedRemediation UID, Node UID)`. Use the request UID as its `attemptID`. Bind it permanently to the Node UID, configuration UID, and `beforeBootID`. Generation changes and configuration replacement do not permit another attempt for that request. A new valid request UID permits another requested attempt after execution ownership becomes available, without a limit on the number of requested reboots.

The Node execution record is authoritative for `Offered`, `Accepted`, `Cancelled`, and `BootEnded`. A journal entry cannot create acceptance or reverse cancellation. Retain local records to count autonomous attempts and recover the state of requested reboots. They do not create another requested-execution owner. The separate bootHold controls permission to start kubelet as specified in Section 5.11.

For autonomous remediation, the local agent generates one local attempt ID before recording the reboot decision. Do not fabricate a remediation CR identity. The local agent preserves autonomous history when a requester creates or replaces a remediation CR, NHC stops its request, or someone deletes it. A later CR cannot adopt an earlier autonomous attempt as its requested action.

The following fragment shows a requested attempt whose reboot decision the agent has durably recorded. Common also records its timestamps, revision, and writer metadata.

```json
{
  "schemaVersion": 1,
  "node": "worker-0",
  "nodeUID": "node-uid",
  "bootID": "11111111-1111-4111-8111-111111111111",
  "config": {
    "uid": "config-uid",
    "sharedStorageVolumeMode": "BlockPerNode"
  },
  "agent": {
    "component": "sbr",
    "shutdown": "running",
    "fenceLatched": true,
    "extra": {
      "sbrStateVersion": "2",
      "autonomousInhibit": "{\"version\":1,\"nodeUID\":\"node-uid\",\"active\":false,\"revision\":0}",
      "activeAttempt": "{\"attemptID\":\"request-uid\",\"source\":\"NHC\",\"requestUID\":\"request-uid\",\"nodeUID\":\"node-uid\",\"configUID\":\"config-uid\",\"beforeBootID\":\"11111111-1111-4111-8111-111111111111\",\"phase\":\"Committed\"}",
      "resetLedger": "[{\"attemptID\":\"request-uid\",\"source\":\"NHC\",\"requestUID\":\"request-uid\",\"nodeUID\":\"node-uid\",\"beforeBootID\":\"11111111-1111-4111-8111-111111111111\",\"phase\":\"Committed\"}]",
      "outageState": "{\"phase\":\"Idle\"}",
      "bootHold": "{\"version\":1,\"phase\":\"Required\",\"source\":\"NHC\",\"incidentID\":\"hold-uuid\",\"nodeUID\":\"node-uid\",\"configUID\":\"config-uid\",\"beforeBootID\":\"11111111-1111-4111-8111-111111111111\",\"attemptID\":\"request-uid\"}",
      "bootAdmission": "{\"version\":1,\"phase\":\"Running\",\"nodeUID\":\"node-uid\",\"bootID\":\"11111111-1111-4111-8111-111111111111\",\"admissionID\":\"admission-uuid\"}"
    }
  },
  "reboots": []
}
```

#### Reserve a request and record its reboot decision

The local agent uses one `Store` and one decision loop in the process holding the host lock. `Store.Update` serializes each journal transition. The loop also prevents requested and autonomous decisions from overlapping while an API acceptance remains unresolved.

The local agent performs these steps for a requested reboot:

1. Check the exact Node execution record, its permanent identities, and the current local boot ID.
2. Persist a local request reservation and a bootHold in phase Required with the request UID and boot binding. Keep `FenceLatched=false`. This closes the crash window between API acceptance and local commitment. A confirmed cancellation can clear an uncommitted hold only through the same-boot checks in Section 5.11.
3. Conditionally change the matching Node record from `Offered` to `Accepted`. Preserve the autonomous reboot count and limit. Do not count this requested attempt as an autonomous attempt.
4. Confirm that acceptance succeeded. After an unknown API response, read the live record before making another decision.
5. Persist the accepted attempt, its execution history, and `FenceLatched=true` in one durable journal update.
6. Request bounded best-effort kubelet stop after durable commitment, then invoke the software reboot and watchdog fallback defined in Sections 5.5 and 5.11. Neither a failed stop nor a stopped kubelet proves fencing.

The Node update uses the request UID to accept the attempt once. A retry must adopt the existing acceptance without authorizing another attempt. Local reconciliation records that same acceptance without changing the permission for one autonomous reboot. Section 5.2 defines the Node record and its conditional updates.

If the local agent cannot determine whether its Accepted update succeeded, it must not start a new reboot action. Continue feeding unless the agent has already saved a durable decision to reboot. Preserve the reservation and retry the API read. Do not release a reservation merely because a call timed out or the CR disappeared.

A trusted successor execution on the same Node UID also proves that the earlier execution resolved. The record must retain the valid schema and budget history. Annotation loss, reinitialization, or conflicting history does not establish this proof.

If the local boot ID still equals the earlier reservation’s beforeBootID, the previous request must have reached Cancelled before replacement. Remove its reservation, for which no reboot decision was recorded. If the boot IDs differ, do not execute the previous request. Compare its saved history with the trusted records. Never discard recorded history or uncertain outcomes merely because the current request changed.

After confirming `Cancelled`, the agent can remove the reservation if no reboot decision was recorded. It cannot clear an earlier recorded reboot decision. Confirmed `Accepted` is irreversible, including after health recovery or CR deletion. If persistence fails before the local record of the reboot decision is durable, preserve the accepted state and report degradation. Resume persistence when possible. Do not change it to `Cancelled`.

#### Preserve journal durability and watchdog state

The common library writes a temporary journal file, synchronizes its contents, renames it, and synchronizes the directory. Readers cannot observe a partial replacement. The separate host lock prevents competing writers.

The local agent limits host I/O waits and retries without blocking watchdog feeding. It permits only one outstanding journal operation. An expired wait does not prove that the operation stopped. Do not start another transition while its result remains unresolved.

Before the reboot decision becomes durable, a persistent journal failure prevents the local agent from starting a new reboot action. The local agent reports degradation when possible. If the qualified device supports safe disarm, verify disarm before stopping the monitoring loop. This does not cancel an Accepted request.

For nowayout or failed-disarm devices, the local agent continues feeding in a degraded state until a qualified shutdown or recovery path is available. Never exit and imply a reboot-free stop. A device without a supported fatal-error policy fails qualification before it is armed.

`ErrDurabilityUnconfirmed` means readers can see the replacement journal file, but the file might not survive a crash. It does not authorize a new reboot. Common must provide a tested contract for resolving this outcome.

After the agent durably records its reboot decision, journal or API failures must not restore watchdog feeding for the recorded beforeBootID. During its shutdown, the local agent must not disarm the watchdog for this recorded beforeBootID. Later health recovery cannot reverse the recorded reboot decision.

Software reboot and watchdog expiry are two mechanisms for one reboot operation. Reuse `executeSelfFencing()` in [the agent](../../cmd/sbr-agent/main.go). Persist the reboot decision before invoking the command or stopping feeds. A command return or elapsed timer does not prove that the boot ended.

#### Recover after a process or host restart

Before normal monitoring, the local agent loads the journal and reads the local kernel boot ID through `ReadBootID`. It compares this ID with the attempt’s permanent `beforeBootID`.

`FenceLatched` applies only to the boot identified by the attempt’s recorded beforeBootID. If the current local boot ID is valid and different, update the journal atomically before arming the watchdog for the current boot. Record the new boot ID and clear only the FenceLatched flag for the previous boot. Preserve the attempt, bootHold, and reboot-count history. The host gate retains the workload hold independently; the local agent can resume watchdog monitoring only after controlled rejoin allows its Pod to start.

| Record and boot comparison | Action |
|---|---|
| Reservation, acceptance unresolved | Read the authoritative API record. Keep the reservation and block another attempt until its outcome is known. |
| Valid reservation, confirmed Accepted, same boot | Durably record the reboot decision in the journal before acting. Retain the same requested attempt without changing the autonomous reboot count and limit. |
| Committed attempt, same boot | Resume the committed watchdog state. Do not issue another software reboot command. |
| Attempt, different valid local boot | Never execute that request in the new boot. Preserve its history and report the boot observation. |
| Clean shutdown marker, same boot | Treat it as an agent restart. The marker does not prove a host reboot. |
| Invalid or unavailable boot ID | Block new attempts. Preserve any known recorded reboot decision and report degradation. |
| `NeverInitialized` | Initialize only after proving first installation and checking the API identities and budget records. |
| Unreadable, corrupt, unsupported, or uncertain history | Enter `HistoryReconciliationRequired`. Reconstruct only facts that complete authoritative API records establish. |

A local comparison alone does not set `BootEnded` or authorize cleanup. Section 5.7 requires reading the Accepted record before reading the current host boot ID. The host reporter publishes evidence in its own Node status against that exact Node and request record, including when the CR no longer exists; peers validate it before updating the execution annotation.

The peer does not require a separate acknowledgement from the local agent that it has stopped processing the old request. `Cancelled` prevents acceptance. Valid `BootEnded` evidence proves that the boot identified by the accepted request’s beforeBootID has ended. A process in a different boot must reject that old request.

#### Preserve and reconcile the autonomous reboot count and limit

Keep a durable record of requested reservations and every requested or autonomous attempt whose reboot decision the agent recorded. Use stable attempt IDs. Assign an outage period ID, called an epoch ID, only to autonomous attempts. Before recording an autonomous reboot decision, use the existing outage period if its recovery requirements are not yet met. If no such period exists, save the proposed outage period before acting. Requested attempts retain their history but do not change whether another autonomous reboot is permitted.

Before rebooting autonomously, the local agent atomically records the attempt, marks the outage period as unresolved, and persists bootHold=Required with the same incident binding. The outage period ends only after storage remains healthy and the API remains reachable for `autonomousRebootRecoveryPeriod`. This period defaults to five minutes.

The first failed probe, including deadline expiry, restarts this recovery interval. Do not wait for the unhealthy threshold. API failure also restarts it. Start measuring storage recovery again only after a fresh complete probe succeeds. A brief recovery, new CR, requested reboot, agent restart, or configuration replacement does not permit another autonomous reboot.

The local agent preserves the used autonomous attempt across agent restarts, host reboots, and configuration replacement. It restarts recovery measurement when it cannot establish continuous storage and API health.

Permit at most one autonomous reboot attempt per outage period until its recovery requirements are met. After this attempt is used, continue feeding while the local agent is running and report `BudgetExhausted` when possible. This rule never releases a post-reboot hold; the host gate remains independent of watchdog feeding and budget accounting. Do not reject or cancel a valid requested attempt because an autonomous attempt was already used or its history is unknown. Requested attempts remain subject to execution ordering and their own identity and durability checks, without a limit on the number of requested reboots.

After API recovery, the local agent reconciles its autonomous attempts with the Node’s API budget using attempt IDs and epoch IDs. Repeated reconciliation must not count an attempt twice. Preserve attempts already recorded in the API from other observations. Complete this reconciliation before restoring the permission for one autonomous reboot. It does not impose a budget check on requested acceptance.

Distinguish first installation from lost, corrupt, incompatible, or uncertain history. An API record can reconstruct only the facts it contains. It cannot prove that no unreported autonomous attempt occurred during an API outage. Unknown autonomous history blocks a fresh autonomous attempt until the missing facts are established.

Keep this uncertainty durable across restarts. Never replace it with an empty ledger. A surviving Accepted record can establish that request's acceptance. It cannot reconstruct missing watchdog state or unrelated autonomous history.

Remove old diagnostics separately from required attempt and outage period records. Retain records needed to avoid counting attempts twice, determine their outcomes, or establish whether another autonomous reboot is permitted. CR deletion does not authorize their removal. If required records cannot be retained, block new attempts and report degradation.

### 5.7. Remediation CR status and reboot evidence

Add optional fields to [remediation status](../../api/v1alpha1/storagebasedremediation_types.go). Existing conditions remain available. Filesystem and shared Block requests do not require these fields.

Proposed additions are shown below. Other existing fields are omitted.

```go
package v1alpha1

type StorageBasedRemediationStatus struct {
	NodeUID              types.UID      `json:"nodeUID,omitempty"`
	ConfigUID            types.UID      `json:"configUID,omitempty"`
	VolumeMode           string         `json:"volumeMode,omitempty"`
	BeforeBootID         string         `json:"beforeBootID,omitempty"`
	ExecutionPhase       string         `json:"executionPhase,omitempty"`
	ObservedBootID       string         `json:"observedBootID,omitempty"`
	Cleanup              *CleanupStatus `json:"cleanup,omitempty"`
	RestrictionsReleased bool           `json:"restrictionsReleased,omitempty"`
}

type CleanupStatus struct {
	Phase               string       `json:"phase,omitempty"`
	StartedAt           *metav1.Time `json:"startedAt,omitempty"`
	Deadline            *metav1.Time `json:"deadline,omitempty"`
	OutOfServiceApplied bool         `json:"outOfServiceApplied,omitempty"`
	Reason              string       `json:"reason,omitempty"`
}
```

The peer records the request’s Node UID, configuration UID, volume mode, and initial boot ID once in remediation CR status. The CR UID comes from metadata. The initial boot ID comes from a live Node read. None of these identities can change during this CR's lifetime.

Peers publish `executionPhase` as `Offered`, `Accepted`, `Cancelled`, or `BootEnded` from the authoritative Node record. They publish the accepted `observedBootID` with `BootEnded`. CR status cannot grant another reboot.

`CleanupStatus` reports one operation with a fixed deadline. Its phases are `Pending`, `Observing`, `Completed`, `Incomplete`, `Unknown`, and `Skipped`. The peer records why cleanup ended: deadline expiry, NHC stopping the request, remediation CR deletion, or replacement by a later request. It also records missing permissions or failed API operations when applicable. Events and logs describe remaining resources without storing an unbounded target list.

Peers must not change the recorded deadline after the operation starts. `outOfServiceApplied` records confirmed application, not an inference from elapsed time. Missing cleanup status never means completion.

`restrictionsReleased` means that this CR no longer has an entry requiring SBR’s scheduling restrictions. Another listed CR or an administrator can still keep the Node cordoned or tainted. Do not call this field proof that the Node is schedulable.

#### Ordered boot observations

The local agent accepts only while its actual host boot ID equals `beforeBootID`. It checks the live Node UID and configuration identity. It persists its request reservation before the conditional acceptance transition, without a budget check.

For an Offered request, the target can report that its local boot ID differs from beforeBootID through a conditional observation update. A peer then changes the request to `Cancelled` with reason `InitialBootChanged`. This prevents the target from accepting the previous request and rebooting for it. The peer must not change the CR’s initial boot ID or report `BootEnded` for a request that the target never accepted.

After the target agent accepts the request, either the local agent or the separate reporter can publish boot evidence. The publishing component performs these steps:

1. Read the exact `Accepted` Node execution record through the live API.
2. Check its Node UID and recorded request identity.
3. Read the current local host boot ID with `ReadBootID`.
4. Require a valid nonempty ID different from `beforeBootID`.
5. Conditionally publish that ID and the exact execution binding in the SBRBootHeld Node status observation against the Node version read in step 1. Do not let the reporter patch the execution annotation.
6. On conflict, repeat the identity and phase checks before retrying.

The reporter cannot change execution phase or budget. A peer that meets Section 5.2 requirements, or the operator rejoin controller handling the held incident, validates the observation against the current accepted record. It then changes `Accepted` to `BootEnded` through a conditional Node update. The transition and accepted observed boot ID persist together.

The host reporter reads the Node record even when the CR no longer exists. It publishes the ordered observation through SBRBootHeld in Node status, not by patching the main Node resource; Section 5.11 defines the payload and validation. It does not require the probe device to attach. Journal observations can provide diagnostic context, but an authoritative accepted record already supplies the requested identity and initial boot.

Reading a different local boot ID after reading Accepted establishes that the boot identified by beforeBootID ended. It does not establish which reset mechanism caused the change. It also does not prove continuing workload isolation.

A change in `Node.status.nodeInfo.bootID` alone does not establish the required order. A later API resource version does not prove when the host booted. Node status can support the target’s observation. It cannot replace the local read in the required order. See [the Kubernetes Node status patch implementation](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/component-helpers/node/util/status.go).

Delayed observations must still match the accepted request, Node UID, and initial boot. Reject observations for another execution or replacement Node. An accepted observation remains valid if a later boot occurs. Never replace its initial boot ID.

If neither the local agent nor reporter can publish boot evidence, the peer keeps execution `Accepted` with reason `EvidenceUnavailable`. NHC stopping a request, remediation CR deletion or absence, timeout expiry, and Node replacement cannot substitute for boot evidence. After validating BootEnded, the peer needs no old-boot execution acknowledgement, but destructive cleanup still requires the independent isolation evidence in Section 5.11.

### 5.8. Requested remediation, escalation, and cleanup

BlockPerNode retains SBR's OutOfService approach. The BlockPerNode remediation path does not directly delete workload Pods, VMIs, VolumeAttachments, PVCs, or PVs. Kubernetes, KubeVirt, and CSI controllers perform their normal resource handling.

A peer cordons the Node, and the target accepts one reboot request. The agent or reporter then publishes information showing that the recorded initial boot ended. After a peer validates this information and qualified continuing isolation, it applies OutOfService and checks resource removal. CR deletion removes that CR holder, but independent rejoin ownership retains required restrictions until Section 5.11 authorizes release.

#### Required isolation and controlled rejoin

A changed boot ID establishes only BootEnded. Before adding OutOfService, validate the current incident's continuing execution isolation under Section 5.11. Kubelet stop, API recovery, elapsed time, and VolumeAttachment absence do not establish isolation. Host reset and default-deny startup protection provide the local fence; the control plane validates evidence before acting.

Kubernetes shutdown guidance requires verifying shutdown/power-off before destructive OutOfService cleanup. Qualify this design's reset-plus-held-host execution-isolation implementation on supported platforms rather than equating a boot-ID change with that prerequisite. This is host/platform qualification, not a vendor fencing API or custom CSI extension. The operation delegates storage-side crash recovery to the documented storage stack semantics. See [Kubernetes shutdown guidance](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#non-graceful-node-shutdown-handling).

Requested CR completion/deletion and observation expiry do not authorize rejoin. Independent incident reconciliation may continue under Section 5.11 without reopening the expired requested cleanup operation. Automatic release requires all common reconciliation checks; AdminAssisted additionally requires its external-runbook attestation. No timeout certifies physical detach or opens the gate.

#### One cleanup operation with a fixed deadline

After the request reaches `BootEnded`, a peer starts one optional cleanup operation if the request is still active. Here, optional means that cleanup completion is not required for successful completion of the remediation policy with a fixed cleanup deadline. It is not a new configuration switch.

The peer saves `startedAt` and `deadline` through a conditional remediation CR status update before trying to apply OutOfService. Use a provisional five-minute internal observation limit. Do not add another configuration field in this proposal. Qualification can adjust this constant.

```go
package controller

// Existing imports and other declarations are omitted.
const blockPerNodeCleanupObservationTimeout = 5 * time.Minute
```

The peer applies `node.kubernetes.io/out-of-service=nodeshutdown:NoExecute` using the existing `ensureOutOfServiceTaint()` behavior with conditional ownership checks. Preserve the original snapshot and all current listed CRs.

Check Node UID, current execution UID, `BootEnded`, valid Section 5.11 isolation evidence, no release in progress, `cleanupClosed=false`, outOfServiceReleased=false, and this request’s entry in the restrictions list inside the Node update. Before new cleanup work, the peer handles any NHC stop request or deletion of the remediation CR. A delayed update cannot apply the taint for an execution that a successor replaced.

Retry failed API operations only within the recorded cleanup period. A peer restart or lost response does not extend that period. Read current state to resolve an uncertain taint update. Record `outOfServiceApplied=true` only after confirming application. A pre-existing matching taint can satisfy application without becoming SBR-owned.

The peer may repeat cleanup observations. It must not start another cleanup operation after the deadline expires or a terminal result exists.

Before publishing a terminal cleanup result, the peer conditionally sets the Node execution's `cleanupClosed=true`. The peer also closes cleanup when NHC stops the request, someone deletes its remediation CR, or a peer replaces the request record. A delayed taint update based on an earlier Node version then fails. Keep this flag independent of whether OutOfService remains present. If cleanup was closed without a saved CR result, the peer reports `Unknown` with reason `ClosureResultUnavailable` when it resumes processing. Finish the policy with a fixed cleanup deadline without reopening cleanup or repeating scans.

The requested-operation deadline stops new cleanup attempts by that request; it does not terminate the independent rejoin reconciliation in Section 5.11. It cannot cancel an API request already in flight. Close the Node permission before reporting the operation finished. If a delayed taint write wins first, read its result and retain the taint under the listed CR rules.

The peer records cleanup results as follows:

| Observation | Cleanup result |
|---|---|
| Confirmed taint application and complete fresh scans find no relevant resources | `Completed` |
| The deadline expires after confirmed taint application, with a complete final scan showing remaining resources | `Incomplete` |
| The deadline expires without a complete final scan, or taint application remains unconfirmed | `Unknown` |
| NHC stops the request or someone deletes its remediation CR before the peer starts cleanup | `Skipped` |
| The peer stops observations because NHC stopped the request, someone deleted its remediation CR, or a peer replaced the request record | `Incomplete` or `Unknown`, according to the available evidence |

Timeout does not establish successful taint application or resource removal. Preserve the application result and error reason separately. A temporary API failure can delay publication of the terminal result, but it cannot restart the observation period.

#### Observation scope and exclusions

The peer discovers current workload resources through the live API. It does not retain a target list. The peer performs these checks:

- List Pods assigned to the target node name, while checking that the recorded Node UID still exists.
- List VMIs assigned to that node or linked to relevant virt-launcher Pods.
- List VolumeAttachments independently of Pods by target node name.
- Treat remaining terminating objects as present.
- Treat a missing optional KubeVirt API as not applicable only after API discovery confirms that it is absent.
- Treat permission errors, failed lists, incomplete pages, and expired continuation tokens as unknown results.

Exclude only verified SBR agent Pods; the host reporter has no Pod. Check namespace, Node identity, Pod UID, DaemonSet owner UID, and configuration ownership. A label alone does not establish an exclusion.

Exclude private probe attachments only when the PV, PVC, and Pod ownership chain establishes their purpose. Missing or unverifiable objects do not establish an exclusion. An excluded volume has no Kubernetes toleration of its own. Its consuming Pod must tolerate OutOfService.

Add this toleration to the agent Pod template:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node.kubernetes.io/out-of-service
          operator: Exists
          effect: NoExecute
```

Omit `tolerationSeconds` so these Pods do not expire under this taint. Preserve other required tolerations. Tolerations protect against taint eviction, not every deletion or detach path. Test terminating Pods, rolling updates, and probe replacement explicitly.

Ordinary workload Pods remain in scope even when they tolerate OutOfService. Those Pods, workload finalizers, and unavailable controllers can prevent complete cleanup. Report that result without adding direct deletion or stripping finalizers.

Complete fresh scans are required for `Completed`. Separate lists do not form an atomic cluster snapshot. Recheck the Node UID and request state before publishing the result. If the deadline expires before a complete scan finishes, record `Unknown` rather than an empty result.

#### Conditions and NHC behavior

After `BootEnded`, verified continuing isolation, and either cleanup completion or deadline expiry, the peer sets these conditions on an active remediation CR. If isolation cannot be established, do not report FencingSucceeded or Succeeded; retain the hold and report IsolationUnverified. CR deletion after BootEnded can still follow the independent-holder rules without claiming success:

| Condition | Value | Meaning |
|---|---|---|
| `FencingSucceeded` | `True` | The accepted boot ended, isolation remains enforced, and bounded cleanup processing finished; rejoin is not authorized. |
| `FencingInProgress` | `False` | No requested reboot remains in progress. |
| `Succeeded` | `True` | The policy with a fixed cleanup deadline completed, including its accepted cleanup limitations. |
| `Processing` | `False` | SBR finished active processing for this request. |
| `Ready` | `True` | Preserve the existing terminal condition convention. |

Keep `cleanup.phase=Incomplete` or `Unknown` when appropriate. Use a reason such as `CleanupObservationExpired` and report the incomplete work. These success conditions do not assert complete resource removal, permission to rejoin, or replacement workload startup. Isolation must remain enforced until the separate release protocol permits rejoin.

A cleanup timeout before `BootEnded` cannot set these success conditions. An accepted request without reboot evidence remains unresolved.

The existing [`handleFencingSuccess()`](../../internal/controller/storagebasedremediation_controller.go) sets the terminal conditions after OutOfService. The BlockPerNode branch adds observation within a fixed period and preserves its separate result.

NHC does not use `FencingSucceeded` to trigger remediation CR deletion. It requests deletion of that CR after Node health recovers, subject to `healthyDelay`. It reads `Succeeded` during escalation. A successful CR can still time out in NHC while the Node remains unhealthy. See [NHC remediation handling](https://github.com/medik8s/node-healthcheck-operator/blob/dc02045b23a8a0d091116a4f4d679d26e9ef3004/internal/controller/nodehealthcheck_controller.go) and [healthy-node handling](https://github.com/medik8s/node-healthcheck-operator/blob/dc02045b23a8a0d091116a4f4d679d26e9ef3004/internal/controller/resources/manager.go).

#### Handling stopped requests and remediation CR deletion

NHC can stop a remediation request, and NHC or another API client can request deletion of its remediation CR. The peer handles these changes before returning success or checking workload cleanup. Neither change causes the peer to start another scan.

| Execution state | Peer behavior when NHC stops the request or someone deletes its remediation CR |
|---|---|
| No offer | The peer stops preparing the request. During remediation CR deletion, it performs the required Node release update even if the CR has no restrictions entry. |
| `Offered` | The peer competes with the target agent’s acceptance by conditionally changing the same Node record to `Cancelled`. |
| `Accepted` | The peer stops cleanup work. It keeps the request in Accepted until the required boot information arrives. |
| `Cancelled` or `BootEnded` | The peer stops cleanup observations and permits a later request to replace the current record. It preserves the terminal state. |

For a CR that NHC stopped but retained, the peer permits a later request after cancellation or validated boot evidence. Preserve its stopped condition outcome and its entry in the restrictions list. Do not wait for workload objects to disappear. Do not overwrite an NHC-stopped failure with normal success.

During remediation CR deletion, the peer removes only that CR’s entry from the restrictions list. When the list becomes empty, an unresolved rejoin record still retains restrictions and the original snapshot. Incomplete or unknown cleanup does not block removal of the CR entry, but cannot authorize rejoin or removal of the independent hold. Retry required API changes until their result is known.

The peer removes the remediation CR’s finalizer only after the release sequence in Section 5.2 succeeds. Its conditional Node update must increment `coordinationRevision`, including when an earlier reconciliation already removed the CR’s entry. Finalizer completion does not require the Node to become schedulable or the controller-managed unschedulable taint to disappear. Another listed CR or an administrator can retain scheduling restrictions. Do not remove that taint directly to complete finalization.

The current [`handleDeletion()`](../../internal/controller/storagebasedremediation_controller.go) restores scheduling without scanning workload resources, but waits for unschedulable taint removal. The BlockPerNode path omits that wait. It requires ownership-aware release and the coordination update before finalizer removal. Existing Filesystem and shared Block deletion behavior remains unchanged.

If someone forcibly removes the remediation CR’s finalizer, this bypasses the required order. SBR then cannot guarantee that a peer will not record Offered after CR deletion. Peers still reconcile retained Node records through the cancellation and release rules. A replacement Node or missing restriction snapshot never authorizes reconstruction from current taints. During normal remediation CR deletion, the peer retains the finalizer if it cannot establish cancellation or BootEnded.

Failed required Node updates can also prevent the peer from removing the finalizer. Cleanup observations alone cannot prevent finalizer removal.

Proposed routing in the existing controller is:

```go
package controller

// Existing fields and shared-storage implementation are omitted.
func (r *SBRRemediationReconciler) Reconcile(
	ctx context.Context, req ctrl.Request,
) (ctrl.Result, error) {
	state, err := r.resolveRequestOrRetainedNodeRecord(ctx, req)
	if err != nil {
		return ctrl.Result{}, err
	}
	if state.IsBlockPerNode() {
		return r.reconcileBlockPerNode(ctx, state)
	}
	return r.reconcileSharedStorage(ctx, req)
}

func (r *SBRRemediationReconciler) reconcileBlockPerNode(
	ctx context.Context, state *blockPerNodeState,
) (ctrl.Result, error) {
	if state.StoppedOrDeletingOrMissing() {
		return r.reconcileExecutionAndRelease(ctx, state)
	}
	if state.ExecutionCancelled() {
		return r.finishCancelledRequest(ctx, state)
	}
	if !state.BootEnded() {
		return r.reconcileExecution(ctx, state)
	}
	if state.CleanupTerminal() {
		return r.publishBoundedPolicyResult(ctx, state)
	}
	return r.reconcileBoundedCleanup(ctx, state)
}
```

These are proposed helpers, not existing interfaces. Node watches supply retained request keys when the CR is absent. Conditional updates remain mandatory inside every helper.

#### API loss and autonomous remediation

Autonomous remediation preserves local autonomous history, any accepted request, and a durable boot hold. During API loss it performs no API cleanup. After recovery, the operator rejoin controller reconciles that held incident under Section 5.11, including standard OutOfService processing and applicable CSI state, without manufacturing a requested CR or restarting an expired requested-cleanup operation. API recovery alone never releases workloads.

An autonomous reboot can end the boot identified by an already accepted request’s beforeBootID. Boot information read and published as Section 5.7 requires can then establish BootEnded for that request. It does not prove SBR's requested reboot caused the reboot. An unaccepted request cannot claim an earlier autonomous reboot.

Existing requested cleanup keeps its original deadline during API loss. After API recovery, peers publish that operation's result or handle CR deletion without resuming its expired scans. The independent rejoin controller may continue incident reconciliation under its own persisted permission. Failed writes never establish release.

```mermaid
sequenceDiagram
  participant NHC
  participant P as Eligible peers
  participant N as Node record
  participant A as Target agent
  participant R as Target reporter
  participant K as Kubernetes controllers
  NHC->>P: Create SBR CR
  P->>N: CAS offer, restriction holder, snapshot, cordon
  A->>A: Check local boot and persist request reservation
  A->>N: CAS Offered to Accepted without budget charge
  A->>A: Confirm acceptance, persist latch, reset
  R->>N: Read exact Accepted record
  R->>R: Read current local host boot ID
  R->>N: CAS own-status boot and persistent-hold observation
  P->>N: CAS Accepted to BootEnded
  P->>P: Persist fixed cleanup deadline
  P->>N: Apply OutOfService only with qualified isolation
  K->>K: Process Pod deletion and volume detach
  P->>K: Observe Pods, VMIs, and attachments
  P->>P: Complete or expire observation with truthful result
  Note over P,N: Execution can finish while restrictions remain
  NHC->>P: Delete CR after health policy permits
  P->>N: CAS holder release and increment coordinationRevision
  Note over P,N: Preserve independent rejoin hold and restrictions
  N-->>P: Confirm persisted release update
  P->>P: Remove finalizer with CR UID and resourceVersion checks
```

### 5.9. NHC policy and reboot limits

The administrator configures NHC with `SBRStorageUnhealthy=True` and one SBR template. Do not add `Ready=False` or `Ready=Unknown`. NHC combines unhealthy conditions with a logical OR, so it would route a network-only failure to SBR.

BlockPerNode cannot detect a powered-off host that cannot publish its storage condition. By itself, BlockPerNode does not provide general hard-dead-node VM high availability. Any external power-fencing topology is a separate architecture and qualification decision.

NHC controls creation of new requests through its condition duration, `minHealthy`, pause setting, and optional `stormCooldownDuration`. The storm cooldown is inactive when unset. Size `minHealthy` against storage failure domains. These controls cannot stop an already recorded reboot decision or an autonomous reboot during API loss. See [the reviewed NHC API](https://github.com/medik8s/node-healthcheck-operator/blob/dc02045b23a8a0d091116a4f4d679d26e9ef3004/api/v1alpha1/nodehealthcheck_types.go).

Two NodeHealthCheck config CRs selecting the same node do not provide independent alternatives. NHC’s per-node lease permits one at a time. NHC skips the second request while the lease is held.

The reference policy shape is shown below. Names, thresholds, and durations are deployment-specific. This policy must watch only the storage condition.

```yaml
apiVersion: remediation.medik8s.io/v1alpha1
kind: NodeHealthCheck
metadata:
  name: sbr-storage
spec:
  selector:
    matchExpressions:
      - key: node-role.kubernetes.io/worker
        operator: Exists
  minHealthy: "51%"          # size against the storage failure domain
  stormCooldownDuration: 10m # inactive when unset
  healthyDelay: 2m           # independent of the cleanup observation deadline
  unhealthyConditions:
    - type: SBRStorageUnhealthy
      status: "True"
      duration: 30s
  remediationTemplate:
    apiVersion: storage-based-remediation.medik8s.io/v1alpha1
    kind: StorageBasedRemediationTemplate
    name: sbr-block-per-node
    namespace: sbr-system
```

After recording its reboot decision, the local agent does not clear `SBRStorageUnhealthy` during the boot identified by the attempt’s beforeBootID. After reboot, the local agent reports `False` when a fresh probe succeeds. The remediation CR's `cleanup.phase` reports observation progress. The storage condition does not remain untruthfully `True` to represent that work.

NHC requests remediation CR deletion after the node no longer matches its unhealthy condition, subject to `healthyDelay`. The peer then stops cleanup observations and completes the release sequence in Section 5.2, including its mandatory Node update. Incomplete cleanup does not retain the finalizer. An accepted request that has not reached BootEnded or failed required release operations can still retain it. Surviving CRs can keep a node outside NHC's healthy count and affect `minHealthy`.

A finite `healthyDelay` can reduce immediate deletion churn but is not a safety mechanism. A negative value must not be used for this policy. NHC advances to another remediation only when `escalatingRemediations` is configured. It follows the configured fixed order.

BlockPerNode is supported only where another remediation provider cannot act concurrently on the same Node or failure domain. The SBR owner annotation is not understood by SNR, Fence Agents Remediation (FAR), Machine Deletion Remediation (MDR), MachineHealthCheck (MHC), or manual procedures. Report detectable overlaps and document the remaining administrator responsibility.

Maintenance requires an explicit workflow. A cordon, generic maintenance taint, or NHC `pauseRequests` is insufficient for these reasons:

- SBR cordons the node itself.
- Autonomous action can occur without the API.
- Work already in progress continues.

Before deliberately disrupting storage or API access, the administrator enables detect-only mode or the durable autonomous-inhibit handshake below. The administrator verifies that it is enabled before causing the disruption. The agent cannot reverse an already recorded reboot decision.

#### Durable local autonomous inhibit

`autonomousInhibit` is a versioned journal object with nodeUID, active, monotonically increasing revision, commandID, and a bounded reason. A root-only local administrative command submits set/clear to the agent's serialized decision loop over an authenticated local Unix socket (root peer credentials, mode 0600). If the agent is absent, the command must acquire the same executor lock and use the same Store validation and durable transaction; it must never break a live lock or edit JSON directly. The socket and its parent directory require restrictive ownership and permissions and persistent SELinux labeling appropriate to the qualified platform. Authorize client process domains as well as socket-file access and Unix-stream connection permissions; a label alone is not authorization. Enforce peer credentials independently of SELinux and reject unapproved clients. The host release service receives no inhibit permission by default: release handling and maintenance inhibition are separate privileges. If a deployment explicitly grants that capability, document and test the narrow policy exception. This does not establish a security boundary against host root bypassing the supported protection model. No API is required. Acknowledgement includes the durable revision and commandID; an uncertain response requires reading that command's result, not assuming success.

If inhibit commits before an autonomous decision, no new autonomous decision is permitted. If a reboot commitment wins, return `TooLateForCurrentAttempt`; a future inhibit may be recorded but cannot cancel that commitment, resume feeding, or clear a hold. Check inhibit in the same serialized transaction as consuming the autonomous budget. Clear requires an explicit administrative action; there is no expiry. Preserve inhibit across agent restarts, host boots and configuration replacement on the same Node UID. Missing/corrupt inhibit state on an enrolled node is not permission to reboot autonomously. It neither resets history nor suppresses requested remediation, panic, or watchdog expiry caused by a failed agent. While inhibited, a healthy running agent continues normal watchdog feeding. Report `AutonomousInhibited` and its revision; maintenance must verify acknowledgement before disruption.

### 5.10. Startup, shutdown, and observability

The agent process starts the BlockPerNode peer manager after initializing API clients and schemes, before storage preflight or local agent initialization. The current `main()` performs storage preflight before constructing the agent. `NewSBRAgentWithWatchdog()` initializes devices and NodeMap before its controller manager. Add an earlier BlockPerNode path in [the agent entry point](../../cmd/sbr-agent/main.go). Keep the existing shared-storage startup path unchanged.

The agent process ends the peer manager during process shutdown or removal of its configuration, following the shutdown rules in Section 5.3. The current `StartWithContext()` links the agent cancellation context to manager shutdown. Do not reuse that coupling for BlockPerNode storage errors. A probe or journal worker must not cancel the peer manager through a shared context or error group. It reports degradation while the peer manager continues API reconciliation.

Keep local execution safety separate from the requirements for processing another node’s requests. A local probe failure or `HistoryReconciliationRequired` prevents unsafe actions on that host. It does not prevent that process from reconciling another target's API state. Never wait for local storage recovery before processing a peer request.

The local agent starts normal watchdog monitoring only after acquiring the host lock, validating its watchdog, and initializing its probe and journal. A fresh journal requires proved first installation and checked API and local history. When history is unknown, the local agent enters `HistoryReconciliationRequired`. Unknown autonomous reboot history blocks autonomous attempts, but does not impose a budget check on requested execution. Missing execution or watchdog history still requires reconciliation before new actions. Do not treat initial provisioning delay as a new storage failure.

On process restart or controlled replacement, restart the transient failed-evaluation count at zero. Discard prior process timers and results. This can postpone initial detection. Resetting the count does not establish healthy storage. Do not publish False without a fresh complete probe. Preserve an existing True condition until such a probe succeeds, subject to recorded reboot decisions. Do not treat absent probes or provisioning delay as successful observations.

A new process starts a fresh probe after completing the startup checks above. It does not track or wait for probe operations submitted by the previous process. The limit of one outstanding probe per device applies within each process.

Restart recovery measurement when continuity cannot be established. Preserve accepted requests, durable reboot decisions, and autonomous attempts. Restarting the transient count cannot grant another autonomous reboot.

After a reboot whose decision the agent recorded, the local agent preserves request history before starting another decision cycle. Before a planned agent stop without a recorded reboot decision, the local agent records its shutdown and verifies watchdog disarm. The agent shutdown marker is diagnostic information, not reboot evidence or a host CleanShutdown authorization. Only the Section 5.11 shutdown protocol can create the latter.

Report desired nodes, ready peer controllers, ready local agents, healthy host reporter/release services, ready probes, and armed watchdogs through metrics. Configuration status uses concise coverage conditions instead of duplicating these counts. Report controller readiness independently of probe and watchdog readiness. Do not require the whole Pod to report Ready before its peer controller can process another node’s requests. A peer-ready count includes the target itself. It does not prove that another peer that meets Section 5.2 requirements exists for that target.

Conditions and events expose unavailable peer coverage. Keep per-node detail outside an unbounded configuration list. Do not reuse `ConcurrentWriteable`, which describes shared-storage readiness.

The existing `LeadershipAcquired` condition and `HasLeadership()` helper are unused outside tests in the reviewed source and are not part of BlockPerNode.

Expose mode, probe identity, and bounded reasons through status or events. Include `WaitingForExecution`, `Accepted`, `WaitingForBootEvidence`, `EvidenceUnavailable`, `CleanupIncomplete`, `CleanupUnknown`, `ExecutorConflict`, `HistoryUnknown`, `BudgetExhausted`, and `BackendUnqualified`. Metrics cover phase age, reboot attempts, coverage, writer conflicts, removal of CR entries from the restrictions list, and observation results. Never label them with tokens or UIDs.

Report history and writer conflicts. Retain diagnostic timestamps during API loss. Publish current conditions after recovery rather than replaying stale conditions. Treat a finalizer-held request reducing NHC's healthy count as an active alert.

Keep separate service accounts for the operator and the agent that runs both the peer controller and local monitoring. The host reporter uses the existing node certificate identity under Section 5.11, not a cluster-wide reporter service account. The peer controller and local agent share one process identity and therefore share its API permissions. Kubernetes RBAC cannot separate those roles inside that process.

Update [agent RBAC markers](../../cmd/sbr-agent/main.go) and the generated [agent role](../../config/rbac/sbr_agent_generated_role.yaml) for lifecycle work:

| Resource | Required peer access | Purpose |
|---|---|---|
| SBR remediation CRs, status, and finalizers | Read and conditional update or patch | Bind requests and advance lifecycle state across namespaces. NHC retains request creation and deletion. |
| SBR config CRs | Read | Resolve assignments and retained configuration identities. |
| Nodes and Node status | Read and conditional update or patch | Manage request ownership, cordon, taints, and own-node storage reporting. |
| Pods | Read | Observe workloads and validate SBR exclusions across namespaces. |
| PVCs, PVs, and VolumeAttachments | Read | Discover target-node attachments, verify probe exclusions, and check cleanup completion. |
| DaemonSets | Read | Check the ownership chain before excluding SBR Pods. |
| VMIs | Read | Check relevant KubeVirt workloads during cleanup. |

Retain main-resource Node mutation in the agent role. Add required attachment and workload reads, without Pod-delete permission for this path. Neither peers nor the reporter directly delete workload resources. The operator does not need workload deletion permissions.

Register `storage/v1` and `apps/v1` in the agent schemes used by lifecycle clients. Register KubeVirt types if using a typed VMI client. The current `getScheme()` registers core and SBR types only. Check `initializeKubernetesClients()` as well as manager setup. Use the live API reader for ownership decisions and complete cleanup scans.

Retain the request watch in `SetupWithManager()`. Watch Node execution records and sets of listed CRs, including initial synchronization when the CR is absent. Map their stored namespace, name, and UID to recovery work. Retry pending Offered requests and boot checks with limited delays. Apply the same retry rule to cleanup checks and removal of CR entries from the restrictions list. A peer must not wait for a cleanup scan before handling an NHC stop request or remediation CR deletion.

Cached clients require list and watch permissions for their informers. Do not restrict workload scans or remediation CR watches to the configuration namespace.

Only the local agent component controls the watchdog. The agent, boot-admission/shutdown helper, and host release helper serialize journal updates through the same host executor lock; the reporter is read-only. The early-boot gate may durably consume a clean record before agent/runtime startup, but same-boot service checks do not contend indefinitely with the running agent's lifetime lock: they read a validated complete atomic snapshot and fail closed on inconsistent state. During a boot hold, the agent Pod cannot run. The reporter uses GET on its own Node and PATCH on its own nodes/status, never main-resource Node patching. Preserve unrelated status conditions with optimistic concurrency.

Existing peer agents retain their lifecycle permissions and remain trusted privileged components; this change does not claim to remove that existing risk. Do not copy their service-account credentials to the host. The host must not receive new permission to patch Node labels, taints, or the rejoin authorization. The operator rejoin controller needs the peer's Node/status and workload/attachment discovery permissions in the table above plus authorization-field mutation; it requires no workload-delete, vendor API, or extra host privileges. Gate field ownership separately from RBAC and keep controller credentials off the target host. Section 5.11 requires admission enforcement for controller/admin-owned rejoin records.

Update manifests and OpenShift Security Context Constraints (SCC) bindings for the `/var/lib/medik8s/state` host mount. Require directory mode `0700`, file mode `0600`, expected ownership, no-follow opens, and protection against symlink and path replacement. Avoid storing credentials. A compromised host root can forge local state and is outside this trust boundary.

#### Watchdog validation contract

Before arming the watchdog in BlockPerNode, the local agent performs these checks:

1. Require a character device.
2. Correlate its major and minor numbers with `/sys/class/watchdog/*/dev`.
3. Obtain a positive timeout.
4. Record supported identity and capabilities through `WDIOC_GETSUPPORT` and `WDIOC_GETTIMEOUT` or an explicitly qualified driver contract.

Record nowayout, magic-close, disable-card, process-exit, and disarm behavior. Never use an unrelated sysfs timeout. Never infer successful disarm from `close()`.

For an invalid or unmatched device, the local agent reports `InvalidWatchdogDevice` and zero coverage. The agent does not arm that device. Write-based keepalive is allowed only after positive device qualification.

A non-watchdog path is permitted only behind an explicit test-only flag. The local agent reports `UnsafeTestWatchdogDevice` when it uses that path. The production DaemonSet never emits that path. Preflight uses the same validation and never silently substitutes softdog. It checks the magic-`V` or disable operation result.

Runbooks cover unavailable peers, missing reporter or probe, unknown history, inconsistent ownership, and stuck cleanup or CSI detach. Expose BootProtectionUnqualified, BootHeld, IsolationUnverified, RejoinAwaitingApproval, RejoinAuthorized, RejoinFailed, and HostCredentialExpired with incident audit references and bounded metrics; never put credentials or high-cardinality IDs in metric labels. They also cover local agent lock or watchdog failure, Node UID replacement, CRs retained in the restrictions list, configuration deletion, and downgrade. Forced finalizer removal, API-object absence, and Node deletion are never fencing evidence.

### 5.11. Default-deny boot protection and controlled rejoin

`bootAdmission` records whether the current kernel boot was admitted to run protected workloads. `bootHold` records an unresolved remediation incident that requires a hold. Together they prevent startup until local clean-boot admission or a boot-bound recovery authorization is durably recorded.

| Component | Responsibility and availability |
|---|---|
| Agent Pod | Only watchdog opener, armer and feeder; available only after kubelet can start the Pod. Probes storage and executes requested/autonomous resets. |
| Host service | One reporter/release service, independent of kubelet and runtime; reports held evidence and consumes authorization. Never opens or feeds the watchdog. |
| Host helper subcommands | Short-lived boot gate and shutdown coordination using the common journal protocol; these are not additional operators or watchdog daemons. |
| Existing SBR operator | Runs the rejoin reconciler and controls recovery authorization under existing leader election. |
| Platform administrator | Installs and qualifies host wiring, ensures independent recovery capacity, repairs credentials and performs AdminAssisted runbooks. |

```mermaid
flowchart TD
  Boot["Protected service start"] --> Facts{"Durable local facts"}
  Facts -->|"Admitted same boot"| Run["Allow startup"]
  Facts -->|"Unused clean record"| Consume["Persist current Running"]
  Consume --> Run
  Facts -->|"Unclean, explicit hold or uncertainty"| Held["Keep workloads stopped"]
  Held -->|"Valid boot-bound authorization"| Consume
```

#### Contract and separation of responsibilities

SBR MUST prevent protected workload execution following an unclean or remediation-triggered host reboot until a boot-bound rejoin authorization is durably accepted. Fencing is local: a genuine hardware/software host reset ends the old execution, and installed host boot protection prevents workload execution in the new boot without requiring the API, network, or CSI. A changed kernel boot ID is evidence that the old boot ended, not the mechanism that maintains isolation. A stopped kubelet, missing heartbeat, or elapsed watchdog timeout is not proof of a completed reset.

Automatic rejoin means verified continuing host isolation, completion of applicable Kubernetes reconciliation, and closure of conflicting SBR actions. It does NOT attest to physical storage revocation, flushed backend caches, or universal application crash consistency. SBR uses standard Kubernetes/CSI objects only: no vendor fencing API, array integration, custom CSI addon, or driver-name allowlist is required. Physical recovery, locking, and access semantics remain the responsibility of the configured storage stack under its documented crash-recovery contract. Selecting Automatic accepts that prerequisite; it does not absolve SBR of ordering, identity, evidence, or reconciliation defects. API errors and missing evidence never satisfy a prerequisite.

AdminAssisted uses the same host isolation and reconciliation prerequisites, plus recorded completion of the deployment's external recovery runbook. An approval button alone creates no additional storage guarantee. SBR does not implement that runbook's vendor-specific actions. Neither policy releases on API reachability alone, successful probe alone, attachment absence alone, a cleanup timeout, CR deletion, or configuration replacement.

Host protection applies to explicitly enrolled Linux workers and all their protected workload execution paths. Qualify kubelet/runtime startup and independent VM/autostart services; unsupported paths reject enrollment. Malicious host root, privileged administrators bypassing protection, and silent rollback to a valid old disk snapshot are outside the guarantee. Restore/clone procedures must isolate and reenroll the host. A real kernel reboot is required; a container restart, userspace-only soft reboot, or unexplained boot-ID change is not an equivalent fence.

Control-plane/SNO nodes and recovery infrastructure that depends on held workers are not supported by this profile. The API, operator rejoin controller, admission enforcement, and CSI controller services must remain independently recoverable during a full selected-worker outage. Missing recovery capacity preserves isolation but delays recovery; no availability timeout opens the gate.

#### Managed host installation

On OpenShift, supply administrator-managed MachineConfig artifacts for the selected worker MachineConfigPool: versioned host binaries, additive kubelet/runtime systemd drop-ins, a host reporter/release service, shutdown handling, and required SELinux labels/policy. Use the supported Ignition version for the deployed release. Respect immutable /usr: use the supported writable executable location such as /usr/local where provided by the RHCOS layout, or a qualified image-layering method. Store runtime state under /var/lib/medik8s/state and never overwrite it with MachineConfig. Do not replace vendor units, write into immutable /usr, or mutate MCO-managed files from an agent Pod. MCO rollout may drain/reboot nodes; MCO is not the incident-time fencing actuator.

On other Kubernetes distributions, install equivalent artifacts through the supported node-image/package/configuration mechanism. An init container is too late to enforce pre-kubelet boot admission. Discover and qualify the actual kubelet, CRI-O/containerd, socket activation, and other workload-start paths; do not assume one universal unit name or runtime behavior.

Every protected service start executes a synchronous non-optional ExecStartPre gate. Failure blocks ExecStart; never prefix the check with `-` or treat missing state as success. A successful RemainAfterExit oneshot from an earlier start is insufficient. The gate performs local checks and bounded local persistence only; it never waits for API access. A separate host service retries reporting/release using bounded calls and backoff, and starts/retries the normal service units only after durable authorization. It has no kubelet/runtime/CSI dependency. A failed gate must not prevent administrative host access or complete unrelated boot targets.

Require the real journal filesystem before checking state. Use the configured node identity and supported shared nodestate parser, not uname, grep, or optional Python availability. Binaries, trust roots, and configuration must already be present for disconnected boot. Validate protection before allowing either requested or autonomous remediation and recheck drift; missing protection blocks new actions.

Enrollment is explicit and durable. An installation may begin with an explicit valid Disabled record, never an absent-record default. Under the executor lock, enrollment validates identity and boot wiring, initializes the journal and current-boot admission state, fsyncs them, then atomically activates protection. The agent cannot remediate before activation. A crash after activation with missing/inconsistent state blocks startup. Once enrolled, power loss before any SBR decision is covered by the boot protocol below.

#### Durable boot admission, not a reusable clean Boolean

Add bootAdmission beside bootHold in agent.extra. It is a versioned JSON string containing nodeUID, the recorded kernel bootID, phase, and a unique admissionID. Phases are Running, ShuttingDown, CleanShutdown, and Held. CleanShutdown additionally carries a single-use cleanShutdownID; Held carries an incidentID and previousBootID. Update the common document bootID atomically with the admission transition; permanent attempt beforeBootID values never change. bootHold separately records an explicit remediation hold, so a graceful SBR software reboot cannot be mislabeled as a clean maintenance reboot.

All first-start decisions are serialized under the existing executor lock before any protected service can run. The gate checks the common and SBR schema, identities, bootAdmission, bootHold, reservations, and required attempt history together. Missing/unreadable/unsupported/corrupt/inconsistent state, invalid boot IDs, or uncertain write durability blocks startup. Do not infer clean state from absent JSON fields or outageState.phase=Idle.

| Local facts | Boot-admission action |
|---|---|
| Same kernel boot, durably admitted Running, no active hold | Permit an ordinary kubelet/runtime service restart locally; no API request. |
| Different kernel boot, valid unconsumed CleanShutdown from the recorded previous boot, no unresolved hold/reservation | Atomically consume the clean record and persist Running for the new boot BEFORE starting any protected service. Only then permit startup without API access. |
| Different kernel boot after Running/ShuttingDown, or an explicit Required hold | Persist/reuse Held and an incidentID before reporting; keep all protected services stopped. An unclean boot needs no pre-crash SBR decision. |
| Same boot already Held, or shutdown already committed locally | Keep the gate closed. A process/service restart must not manufacture a new boot or clean marker. |
| Released authorization bound to this exact boot and incident | Permit startup only after the local release transaction also durably records Running for this boot. Preserve the consumed release identity. |
| Released authorization bound to an earlier boot | Hold the new boot and require fresh recovery authorization. |

The critical ordering is **consume the previous clean record and durably mark the current boot Running before any workload starts**. If power fails after consumption, the next boot is conservatively held. If consumption is not durable, the current boot must not start workloads. No repeated service start consumes another clean record. Record application must use the common atomic write, file fsync, rename, directory fsync, and uncertain-durability handling contract.

The shutdown helper and agent must cooperatively transfer the executor lock: when the agent is running it first records ShuttingDown, follows its qualified non-committed watchdog-disarm rules, and yields ownership through a bounded host-local shutdown handshake. If it cannot safely yield, no clean marker is written. The helper never unlinks/breaks the lock or becomes a second watchdog owner. An agent Pod restart alone is not a request for host shutdown.

The shutdown helper may write CleanShutdown only as the final result of a verified shutdown sequence: persist ShuttingDown to deny new protected-service starts; stop/quiesce all protected workloads and confirm they cannot restart; verify no unresolved remediation/reservation/hold; then durably record CleanShutdown for this boot while the journal remains writable. Merely running a service Before=shutdown.target, receiving SIGTERM, stopping kubelet, stopping the runtime daemon, or executing agent shutdown hooks is insufficient proof that containers/VMs stopped. Ensure platform-specific stop ordering and all autostart paths are covered. A failure, deadline, journal error, or forced shutdown leaves the boot unclean. If shutdown is aborted after recording clean state, re-admit the same boot by consuming that record and persisting Running before restarting any protected service; never leave clean state while workloads execute.

The existing agent shutdown marker remains diagnostic and does not authorize clean host boot. An SBR Required hold always wins over a clean record, including a software reboot that follows graceful OS shutdown. No systemd timeout, service retry, host reboot, detect-only switch, or config deletion clears it.

#### Remediation hold and local reset

bootHold is a versioned JSON string with Idle, Required, and Released phases, nodeUID, configUID, and bounded release history (the current incident and last completed release only). Never prune unresolved execution, incident or budget evidence; their existing limits fail closed. Old completed release IDs need no unbounded archive because admission checks exact Node UID, incident, authorization revision and current kernel boot. Required/Released bind incidentID, beforeBootID, source, and attemptID when one exists; autonomous attempts also bind outageEpoch. Unclean boots without an SBR attempt create a new local incident with source=UncleanBoot and no fabricated request/attempt UID. Generate an unpredictable incidentID once and retain it across reporter restarts. Reuse an existing unresolved incident rather than overwriting it. bootAdmission and bootHold changes are one atomic journal transaction.

For requested actions, persist Required with the reservation BEFORE accepting the API offer; preserve it when acceptance is uncertain. A confirmed cancellation can clear an uncommitted reservation only with matching identity, unchanged boot, and no unresolved side effects under the executor lock. For autonomous actions, atomically persist Required, the unresolved outage, and consumed attempt before acting. A crash earlier is still caught by bootAdmission=Running from the current boot.

After durable commitment, request bounded best-effort stop of kubelet to reduce further reconciliation, then execute the software reboot/watchdog fallback. Do not wait for API drain or CSI unmount; do not stop the runtime first and kill the watchdog owner. A failed/hung stop does not postpone the committed watchdog reset. Stopping kubelet alone leaves running Pods/VMs and MUST NOT establish isolation. Never restore old-boot feeding after commitment.

Clearing FenceLatched for the old boot does not clear either boot hold. **The agent Pod is the sole watchdog opener, armer and feeder.** No host gate, reporter, shutdown or release helper may open, disarm, pet or take ownership of the device, including opening it for validation. Opening a watchdog device can itself arm it. Platform qualification must establish that each supported reset path leaves the watchdog disabled until the new agent opens it. `nowayout` prevents stopping an armed watchdog through ordinary close/disarm; it does not by itself establish persistence across a host reset. Reset behavior depends on driver, firmware and hardware. A platform that keeps the watchdog armed during the held boot is unsupported by this design; no host feeding/handoff fallback is provided.

Publish a qualification record for each supported OS/kernel/driver/firmware profile, covering cold boot, software reboot, watchdog reset, panic, timeout behavior, and agent startup. Verify exclusive ownership and exclude competing systemd/BMC/device feeders. This proposal does not assert any hardware has already passed. During boot hold and after release before the agent successfully initializes and arms its watchdog, no SBR watchdog monitoring exists. Host execution isolation remains effective during a hold; recovery liveness and bounded fencing during the unarmed startup interval are not promised. A stuck held host stays isolated and may need administrator reset. Once armed, agent failure must retain the qualified watchdog behavior. A repeating watchdog reset loop is a failed qualification, not a recovery strategy.

#### Host evidence and security boundary

The host reporter uses the existing certificate-based kubelet identity and configured CA to GET its own Node and optimistically PATCH its own nodes/status only. SBRBootHeld carries a bounded versioned payload with Node UID, incidentID/source, recorded previous boot, current observedBootID, protection version, hold state, and request/attempt binding where applicable. It reports True only after verifying that this boot has never admitted protected workload execution and the effective hold remains enforced. It never sets Ready or renews the kubelet Lease. False/Released does not erase incident history.

For requested BootEnded, preserve Section 5.7's ordering: read the exact Accepted record, then read current local boot/hold, then publish against that Node resourceVersion; repeat after conflicts. Both peers and the rejoin controller validate these bindings. For a new boot after an unclean Running/ShuttingDown record, the local gate supplies current-boot hold evidence even when no SBR attempt existed. An observation after this boot already ran workloads cannot retroactively establish continuous boot isolation. An unclean-boot incident is reported without pretending a requested reboot occurred.

No API access is needed to maintain the hold. For gate-discovered unclean incidents, persist the previous/current boot binding and hold before publication so repeated reporter restarts cannot create competing incidents. Missing/expired node certificates, unavailable API, or rejected status patches keep it active; repair credentials through the supported administrative path without starting held kubelet. Do not disable TLS verification, auto-approve arbitrary CSRs, or copy privileged agent/admin service-account credentials to the host. This uses permitted node status access; it does not bypass NodeRestriction or grant Node taint/label write privileges.

Use a versioned `sbr.medik8s.io/rejoin` annotation on the existing Node to coordinate recovery. Enforce field ownership with a qualified validating admission policy or fail-closed webhook: node identities cannot create/change/remove authorization; peers may establish incident ownership but cannot authorize release; only the operator rejoin identity advances automatic release; only explicitly authorized recovery administrators record additional AdminAssisted approval. The controller must not manufacture human approval. Protect CREATE/UPDATE/PATCH/removal and avoid label-based match bypasses; preserve unrelated Node/status operations. RBAC alone cannot restrict annotation fields. Audit actor, policy, incident, boot, approval/runbook reference, and release identity.

A releaseID is an unpredictable correlation/replay identifier, not a cryptographic attestation of physical storage safety. Bind it to Node UID, configuration/policy snapshot, incidentID, current boot, and a durable authorization revision. Missing or replaced Node identities, cloned state, and conflicting history block release. Existing privileged peers remain trusted components; this design adds no broad host Node-patch credential.

#### Rejoin controller, reconciliation, and release

Run SBRRejoinReconciler in the existing operator manager under its existing leader election, independent of probe Pods and target kubelet. Retain peer ownership of ordinary requested remediation. The rejoin controller additionally validates ordered BootEnded evidence (or cancels an unaccepted offer after a verified boot mismatch) when necessary to recover a held incident, using the same conditional Node transitions as peers. It never creates/accepts a new requested reboot or controls a watchdog. It can therefore recover a fleet of held workers without waiting for an SBR peer Pod on one of those workers to start. Requested CR status/finalizer work may await peers; that is not a prerequisite for the rejoin handshake once the authoritative execution has been reconciled.

Requested offers create a pending independent incident owner in the same Node CAS as Offered/cordon; the host reservation adopts that incidentID. Import autonomous and UncleanBoot holds from valid host evidence after API recovery, without fabricating a remediation CR. Preserve existing attempt/outage history. The annotation phases are Held, Reconciling, Authorized, and Completed; recording Held in the API is not itself fencing evidence. Snapshot the effective rejoinPolicy when establishing the incident. A configuration edit or replacement cannot relax an active AdminAssisted incident into Automatic.

An independent incident owner retains cordon, relevant taint ownership, and original restriction snapshots after CR deletion or bounded requested cleanup expiry. Every peer/rejoin update preserves this owner and checks current rejoin phase on the SAME Node resourceVersion as execution/taint changes. Do not offer a new requested reboot while a release handshake is unresolved. Confirmed unaccepted same-boot cancellation can remove the pending owner after the matching local acknowledgement; uncertainty retains it. No CR finalizer is needed to keep the host held.

The rejoin controller performs these steps in order, persisting progress and retrying idempotently:

| Step / writer | Required evidence and durable transition |
|---|---|
| 1. Operator | Validate Node/configuration identity, current held-boot evidence, host-protection qualification, effective policy, and relevant execution history. Resolve Accepted to BootEnded only with its ordered evidence. Unknown autonomous history must not become a fresh reboot budget. Rejoin may preserve an explicitly Unknown budget after trusted reconstruction establishes the incident, identities, and current hold; unknown execution/hold identity cannot be waived. Do not make rejoin depend on proving unused autonomous budget. API reachability, node NotReady, and presumed watchdog expiry are not substitutes for isolation. |
| 2. Operator | Establish cordon and incident ownership. Close the old requested cleanup permission using its existing CAS protocol and resolve uncertain taint writes. Acquire the separate incident reconciliation permission under the same Node version; peers cannot later reopen old cleanup. Only after verified continuing isolation apply/retain SBR-owned OutOfService and let Kubernetes/KubeVirt/CSI controllers perform standard reconciliation. SBR does not directly delete workload Pods, VMIs, PVs/PVCs, or VolumeAttachments and never strips their finalizers. |
| 3. Operator | Discover current target workload and volume identities through complete live API reads: Pod/VMI/PV UIDs, CSI driver and volume handle where available, and attachment identity. Preserve bounded diagnostic evidence and the incident binding; do not introduce an unbounded historical inventory or require reconstruction of objects already legitimately removed by standard controllers before the first observation. An initially empty complete scan establishes only that no relevant API objects remain, never physical storage safety. Unknown ownership or applicability of a resource that still exists blocks the decision. Exclude only verified non-workload infrastructure and disposable probe resources as Section 5.8 specifies. |
| 4. Operator | Reconcile until stale target workload Pods/VMIs are gone and applicable controller-attachment objects have completed deletion. Include terminating resources; use complete paginated live lists, handle optional KubeVirt API absence through discovery, and treat RBAC/API/discovery errors or ambiguous ownership as Unknown. Identify driver attach behavior from CSIDriver and the relevant volume type, not from an empty list. `attachRequired=false` is NotApplicable for attachment checks, not a failed check or proof of physical detachment. For a resolved CSI driver identity and applicable persistent CSI volume type, honor Kubernetes defaults: an omitted attachRequired field, or an authoritative NotFound for its CSIDriver object, means attachRequired=true. Absence of registration is not permission to skip attachment checks. Distinguish that confirmed absence from failed reads, denied access or unresolved identities, which remain Unknown and block authorization. NotApplicable requires positive volume-type semantics or explicit attachRequired=false; it is never a fallback for uncertainty. Conflicting metadata or a custom driver that contradicts these Kubernetes semantics requires investigation, not an inferred safe default. Do not create synthetic VolumeAttachments or require backend calls. |
| 5. Operator | An empty attachment list is only standard control-plane recovery evidence for the identified applicable scope. No observation claims physical detach, backend-session revocation, cache flushing, or termination of every in-flight backend operation. Those semantics are delegated to the supported storage stack, including its normal retries and serialization. Suspected forced finalizer removal, inconsistent identities, or contradictory evidence requires investigation; do not label it successful CSI processing. Preserve audit evidence without claiming universal detection of administrator bypasses. |
| 6. Operator | Before release, repeat complete live workload/attachment checks and revalidate the Node incident version. Freeze further SBR incident cleanup/reboots via the conditional authorization transition. Keep the node cordoned during the transition. The supported operating contract prohibits direct bindings, tolerating workload placement, independent VM/autostart, and competing recovery actions that bypass this incident's placement restriction. Standard scheduling must respect the cordon; deployments requiring bypass-capable workload controllers must enforce a placement barrier for them. Separate resource lists are not an atomic snapshot: absence is not sufficient when those concurrent writers are uncontrolled. Do not claim a guarantee against privileged actors bypassing this contract. |
| 7. Operator | In Automatic, authorize when the common checks pass, with no vendor integration, manual acknowledgement, or pre-kubelet probe requirement. In AdminAssisted, additionally require an audited administrator attestation of the specified out-of-band runbook for the SAME incident and current boot; it cannot waive the common checks. Lost approval, a new boot, or a changed incident requires fresh approval. Report the precise blocked prerequisite rather than timing out into release. |
| 8. Operator | In one conditional Node update record Authorized and its boot-bound releaseID, irrevocably close incident cleanup permission, and discharge the resolved holders' OutOfService requirement (`outOfServiceReleased=true`). Remove only this incident's SBR-owned OOS; preserve pre-existing/other-owner restrictions. A remaining independently owned OOS blocks release. Retain cordon as needed through startup. Do not wait for NHC to observe healthy storage or delete the remediation CR: that would deadlock with the blocked kubelet. A stale taint/offer update must conflict and recheck authorization before retrying. |
| 9. Host service | The host release service performs a live authenticated GET, verifies the exact identity/incident/current boot and local hold under the executor lock, then atomically persists bootHold=Released and bootAdmission=Running with the consumed releaseID. Only after confirmed durability may systemd start runtime/kubelet. Service starts in that admitted boot can then proceed offline. An API error before consumption or uncertain local durability leaves the gate closed; time-based expiry is not used to revoke a possibly consumed authorization. |
| 10. Host writer / operator | The host service reports durable consumption bound to releaseID and current boot. The controller records Completed from that acknowledgement, without requiring storage health or agent readiness. The current journal writer (host helper before agent ownership, otherwise the agent through its serialized loop) clears bootHold to Idle while retaining consumed identity and publishes the exact incident-bound localIdle acknowledgement. Process this finalization before potentially blocking probe initialization. Completed plus localIdle permits a new Offered transition once ordinary execution ordering permits it; it does not rearm the autonomous budget. Preserve bootAdmission=Running. Keep a separate `schedulingRecoveryPending=true` restriction owner and the original restriction snapshot until fresh storage health and all other restriction owners permit uncordon; this owner must not block new requested execution. New incidents inherit the original snapshot rather than treating an SBR cordon as pre-existing. A failed probe after release therefore permits a new NHC-requested attempt without pretending that the node is still held. Failed agent scheduling/startup can still prevent requested execution and watchdog arming; report that availability limit. RejoinFailed never authorizes old cleanup or a second reboot under the old request. |

Rejoin reconciliation is independent of the existing fixed requested-cleanup observation deadline. It can continue normal control-plane reconciliation after that deadline or CR deletion; this does not reopen the expired request's cleanup or fabricate a new requested reboot. It also applies to autonomous and unclean incidents. There is no fail-open rejoin deadline. Remaining tolerating workloads/finalizers or unsupported topology may require intervention; progress is not guaranteed at the expense of safety.

Authorized is a commitment, not a revocable lease. The host may already be running after a lost response: do not roll back to Held, trust a stale SBRBootHeld=True, or resume destructive cleanup on timeout. A new failure needs a new coordinated incident and fresh isolation evidence. If the node reboots before completion, default-deny catches it, reports the new held boot, and the controller repeats reconciliation before issuing a new releaseID; AdminAssisted also requires new approval. A partial release never grants reusable permission for later boots.

#### Recovery placement and enrollment prerequisites

Reserve recovery nodes outside the enrolled set. The SBR operator deployment MUST use required node affinity or an equivalent hard placement constraint to that set, with appropriate tolerations and sufficient capacity. A preferred affinity or current Pod location is insufficient. Administrators supply installation configuration listing required recovery workloads (namespace, kind, name), applicable CSI controllers/attachers and, where used, KubeVirt/finalizer controllers. The operator validates its own placement, declared dependencies' hard placement constraints, enrollment disjointness and current availability; periodically repeat these checks and expose `RecoveryInfrastructureReady`. There is no universal CSIDriver-to-Deployment discovery API: an incomplete dependency declaration is an installation error, not evidence that no dependency exists. Do not silently rewrite third-party deployments. Their own storage, network and authentication dependencies must also recover independently of held workers. Loss of this prerequisite blocks new enrollment/remediation and alerts; it never opens existing holds or cancels committed resets. Admission and enrollment validation must reject recovery-node/enrolled-node overlap. Cluster administrators remain responsible for preventing conflicting placement changes.

Bootstrap is staged to avoid requiring a blocked agent to enroll itself. First install inactive artifacts on an unprotected staging pool, with no claim of boot protection. A verified first-provisioning operation initializes an explicit Disabled enrollment record and valid journal using the existing live Node identity; missing files alone never prove first installation. Prepare each node before moving it into the protected MachineConfigPool (or equivalent Kubernetes rollout group). Then enable the gate drop-ins; they permit only a valid explicit Disabled record or the normal admission protocol. The agent activates enrollment under the executor lock only after checking the installed wiring, identity and recovery prerequisites and durably initializing current-boot Running. Use a durable enrollment generation bound to Node UID; journal readiness is written before activating that generation. Activation is the commit point. Once active, Disabled is not an automatic fallback. Missing, erased, mismatched or partially written enrollment/journal state fails closed and requires identity-verified repair. MachineConfig must never overwrite live enrollment state with a Disabled template. Newly provisioned nodes enter staging first; do not add an unprepared node directly to the protected pool.

Enrollment and v1-to-v2 migration trust the currently running boot as a one-time baseline; they do not prove that it was previously held or clean. Refuse activation during unresolved execution/hold uncertainty. Crash safety starts at durable activation. Detect-only prevents new SBR decisions but does not disable enrolled boot protection: a later unclean boot still holds. A no-impact evaluation must use inactive, unenrolled protection. Removing configuration, downgrading or switching detect-only cannot unenroll an active hold.

#### Evidence schemas and bounded state

Publish `SBRBootProtectionReady` as a Node condition whose message contains versioned JSON: version, nodeUID, configUID, enrollmentID, observedBootID, admissionID, protectionVersion and profileDigest. True means the installed effective gate paths, enrollment and profile match on this boot; it does not mean the watchdog is armed. The local agent rechecks wiring before committing any reset. Peers validate the live Node UID, selected configuration, current boot/admission binding and supported profile before Offered; a stale True condition alone is insufficient. Conditions are observations from trusted node software, not cryptographic attestation against host root.

`SBRBootHeld` uses standard condition fields (type, status, reason, lastTransitionTime, message). Example message for an unclean boot:

```json
{
  "version": 1,
  "nodeUID": "node-uid",
  "configUID": "config-uid",
  "enrollmentID": "enrollment-uuid",
  "incidentID": "incident-uuid",
  "source": "UncleanBoot",
  "previousBootID": "11111111-1111-4111-8111-111111111111",
  "observedBootID": "22222222-2222-4222-8222-222222222222",
  "protectionVersion": 1,
  "profileDigest": "qualified-profile-digest",
  "holdState": "Held",
  "neverAdmitted": true
}
```

Requested evidence additionally carries requestUID/attemptID and the ordered Accepted observation required by Section 5.7; autonomous evidence carries attemptID/outageEpoch. Omit inapplicable identities, never fabricate them. Reject unknown versions, invalid types, mismatched identities and messages over 8 KiB. Release-consumption and localIdle acknowledgements bind the same incident, current boot, releaseID and authorization revision. Keep only the current incident and last completed release in bootHold; unresolved history follows existing bounded-ledger rules. Keep bounded blocker summaries (at most 20 references plus total/truncated), with complete diagnostics retrieved on demand. Metrics expose service health, certificate expiry, placement readiness and phase age without UID/token labels.

Node CAS conflicts use bounded exponential backoff with jitter, fresh reads and context deadlines. Each retry revalidates authorization and evidence; it never blindly replays an old taint mutation. Kubelet Lease renewals normally occur separately from Node status updates, so do not assume a universal ten-second Node-update cadence. Measure contention before introducing another coordination resource.

#### Verified shutdown barrier

Before CleanShutdown, the shutdown coordinator must complete all of these steps on a qualified platform:

1. Through the current journal writer, persist ShuttingDown and close local admission. Serialize with remediation commitment; an existing Required hold prevents a clean marker. The agent yields the executor lock only under its safe shutdown protocol; a helper never steals it or assumes watchdog feeding.
2. Install temporary runtime masks for protected start/socket/autostart units, preserving pre-existing masks. Cancel pending start/restart jobs, stop active or activating kubelet and independent launchers, and wait for their jobs to finish. A gate check that already returned success must be drained by this barrier; a lock around ExecStartPre alone does not close that race.
3. Terminate/quiesce protected containers and VMs using the qualified local runtime/platform procedure, then verify no protected workload processes/cgroups remain and no restart source remains active. Stopping the runtime daemon is not this verification. Do not kill the agent first and silently lose its required watchdog behavior. If safe watchdog shutdown/lock transfer or workload termination cannot complete, leave state unclean and follow the platform's bounded shutdown/reset path; no host feeder is introduced.
4. Under the executor lock, recheck identities, absence of reservations/holds and the effective shutdown barrier, then write and fsync CleanShutdown. Use bounded deadlines; uncertainty, D-state processes, journal failure or barrier failure never produces a clean marker.

Runtime masks live under /run and disappear on real reboot. If shutdown is aborted, consume any clean marker and durably restore Running before removing only SBR-created masks, reloading units and admitting starts. If this cannot be completed, retain the barrier. Qualify these operations against the actual systemd units and runtime; Before=shutdown.target alone is insufficient. Root bypass of the managed units remains outside the threat model.

#### Held-node operations and credential repair

Alert on hold age, exact blocked prerequisite, dependency failures, certificate NotAfter and failed reporting. Inspect actual certificate validity; do not assume a fixed OpenShift rotation interval. Configure an advance expiry warning appropriate to deployment response times (default seven days, capped at one quarter of the issued certificate lifetime). Add a configurable non-negative clock-uncertainty margin to that advance warning window (after applying the lifetime cap); warn when locally observed remaining validity is at most the warning window plus margin. This is an early-warning policy, not a relaxation of certificate validity. Report ClockUncertain separately when time synchronization is known unhealthy, a wall-clock discontinuity is detected, or trustworthy time evidence is unavailable; prolonged isolation can exceed any configured margin. Use monotonic time for in-process retry/backoff intervals, never as a substitute for X.509 wall-clock validity or as a cross-boot clock. HostCredentialExpired denotes expiry according to the local clock, with ClockUncertain reported alongside it where applicable; also diagnose not-yet-valid credentials and TLS failures. If the host cannot report, the controller must retain the last observed NotAfter, evaluate it against its own clock, and alert on stale credential telemetry without claiming current host validity. Never disable TLS time checks, roll back the clock, or release a hold to repair authentication. Expiry does not end a hold. The host service reloads renewed native node credentials but does not create/approve CSRs automatically or start kubelet to rotate them.

The supported administrative repair runbook is: verify the held host's registered identity and Node UID; pause release consumption while retaining all local gates; generate a private key and node-client CSR on the host; use an off-node authorized administrator to validate and submit/approve it through the distribution's supported certificate process; return only the signed certificate, checking public-key match, node subject, issuer and validity; atomically install it through the qualified native kubelet credential path with correct ownership, permissions and SELinux labels; resume reporting/release processing. The private key stays on the host; privileged cluster credentials never go there. A changed Node UID requires reenrollment/recovery of identity, not reuse of an old release. Each supported distribution profile must publish and test its exact credential paths and approval procedure before production enablement. API outage means repair waits, with the hold intact.

For stuck cleanup: identify exact Pod/VMI/attachment UID and blocking finalizer/controller; restore that controller or complete its documented obligation; verify fresh reconciliation before retrying release. Manual approval cannot bypass this. Do not blanket-remove finalizers, disable gates or repeat resets of an already-held host as a substitute. Any exceptional administrator object repair must satisfy the owning controller's storage obligations and be audited; SBR itself never strips finalizers. Indefinite hold is an explicit availability cost when evidence cannot be obtained.

#### Release handoff and delivery scope

```mermaid
sequenceDiagram
  participant H as Host service
  participant O as SBR operator
  participant C as Standard controllers
  participant A as Agent Pod
  H->>O: Publish bound Held evidence
  O->>C: Permit isolated-node reconciliation
  C-->>O: Stale objects reconciled
  Note over O: AdminAssisted also needs runbook approval
  O->>H: Persist Authorized release
  H->>H: Persist Released and Running
  H-->>O: Report consumption
  O-->>H: Record Completed
  H->>H: Persist Idle and acknowledge
  Note over H,A: Runtime and kubelet may start after durable Running
  A->>A: Initialize and arm watchdog
  A-->>O: Report fresh health when available
```

Completed records execution handoff, not healthy storage. If the agent acquires the executor lock before local finalization, it completes that same idempotent transition before waiting for probe initialization. Cordon recovery and autonomous rearm remain separate. Once Authorized is committed, no old cleanup or old hold inference may resume even if acknowledgement is lost.

Implement in inactive stages: schemas and installers; the complete gate/report/release protocol, admission enforcement and operations; integrated Automatic/AdminAssisted fault testing and platform qualification; then enable enrollment for qualified profiles. Do not defer watchdog reset qualification, start-path coverage, admission enforcement or recovery placement until after a production-enabled phase. AdminAssisted is not a shortcut around these mechanisms. No new operator is introduced, and no host watchdog owner is introduced. This design specifies acceptance criteria; qualification results and executable runbooks are release prerequisites, not claims already demonstrated by this document.

#### Qualification and references

Production qualification covers host reset/boot ordering, all workload start paths, systemd/SELinux integration, identity/admission enforcement, evidence ordering, and standard Kubernetes/CSI reconciliation. Driver-independent rejoin adds no vendor-specific prerequisite; it preserves the existing probe capability requirements (including a suitable raw Block probe StorageClass), rather than claiming that every CSI driver necessarily implements raw Block probing. It does not require a per-vendor fencing integration or certify universal physical storage semantics. Publish the delegated storage contract and deployment responsibilities, including administrator-assisted runbooks where needed. Do not describe a design-only patch as a tested production implementation. Existing shared Filesystem/Block behavior remains unchanged.

- [Kubernetes Node authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction): own-status reporting does not grant host taint mutation.
- [Pod behavior during kubelet restarts](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-behavior-during-kubelet-restarts): stopping kubelet does not stop running Pods.
- [Node shutdown and OutOfService](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#non-graceful-node-shutdown-handling): isolation must precede destructive recovery. Qualify the held-host execution-isolation profile against supported platform behavior; a mere changed boot ID is insufficient.
- [CSI specification](https://github.com/container-storage-interface/spec/blob/master/spec.md), [external-attacher](https://github.com/kubernetes-csi/external-attacher), and [CSIDriver API](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/csi-driver-v1/): controller attachment is distinct from node-side operations and can be inapplicable.
- [systemd service](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) and [shutdown targets](https://www.freedesktop.org/software/systemd/man/latest/systemd.special.html): ordering alone is not workload-quiescence evidence.
- [OpenShift MachineConfig](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/machine_configuration/machine-configs-configure) and [validating admission policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/): qualify the actual supported platform releases and failure behavior.

## 6. Open Questions

### 6.1. Residual limitation: every evidence path unavailable

The kubelet/CSI-independent host reporter can observe the boot identified by beforeBootID ending without the private probe. If every reporting path that follows the read order in Section 5.7 is unavailable, execution remains `Accepted` with reason `EvidenceUnavailable`.

A healthy peer cannot manufacture target boot evidence. The accepted request can retain its finalizer and prevent a later request until a peer establishes BootEnded. Cleanup timeout does not resolve it. NHC can also exclude that node from its healthy count while a remediation CR remains.

### 6.2. Isolation, delegated storage recovery, and availability

Default-deny boot protection covers enrolled workers after unclean boots, including panic/power loss before an SBR reboot decision. The durable boot-admission protocol prevents stale clean markers from allowing later crashes through. Explicit remediation holds survive graceful SBR software reboot. Same-boot service restarts are not host-crash events.

Automatic rejoin completes standard Kubernetes reconciliation and then delegates physical recovery semantics to the configured storage stack. It does not certify backend revocation or application crash consistency. Support for standard CSI drivers does not imply that every backend supplies every workload's required crash guarantees. AdminAssisted adds external-runbook completion for deployments that require it; approval alone does not create safety.

The host/platform, identity, and ordering mechanisms require production qualification. Held workers depend on independently available API/operator/CSI controllers for recovery. Recovery remains blocked when evidence, reconciliation, or policy prerequisites are incomplete, with no fail-open deadline.

### 6.3. API loss and incident recovery

No API cleanup runs while the API is unavailable. Local fencing and boot protection remain independent of it. After API recovery, the operator rejoin controller handles held requested, autonomous, and unclean-boot incidents without inventing remediation CRs.

Requested cleanup retains its original deadline/result. Rejoin reconciliation is a separate persisted operation that can continue after that deadline and CR deletion, with no time-based release. Its mutations share the Node CAS and authorization checks so expired or delayed peer actions cannot race a released boot.

### 6.4. Reset-budget qualification

The durable outage record permits at most one autonomous reboot attempt per outage period until its recovery requirements are met. Another autonomous reboot requires healthy storage and reachable API continuously for `autonomousRebootRecoveryPeriod`, which defaults to five minutes. Qualify this default under intermittent failures. NHC thresholds cannot replace offline local accounting.

SBR does not limit the number of requested reboots across distinct valid CRs. Accepting a request does not change whether another autonomous reboot is permitted. Each valid CR permits one attempt after execution ownership becomes available. NHC or another external requester controls repeated requested reboots. Local reservations prevent duplicate execution of the same CR.

### 6.5. Deferred multiple StorageClasses

Scenario 13 needs an explicit mapping between node groups and StorageClasses. The configuration controller must create the matching probe templates and report their identities. Validation must reject overlap and conflicting watchdog ownership. NHC thresholds may need separate backend failure domains.

Scenario 14 needs several probes per node and a health aggregation policy. Candidate policies include failure of any required backend or every backend. The policy must describe which workload storage each probe represents.

Both extensions need probe-specific status, attachment handling, cleanup exclusions, and tests. This proposal adds neither their API fields nor their controller behavior.

### 6.6. Unknown history and missing Node records

Missing or corrupt local history must remain distinguishable from first installation. Recover only facts supported by surviving authoritative records. Missing autonomous history cannot create permission for another autonomous reboot.

Deleting a remediation CR does not remove the Node annotation. Deleting the Node object removes that annotation. A replacement Node UID does not prove that the old host stopped. Never execute an old request on the replacement Node or change its restrictions using old identities.

If both the host journal and Node API history are lost, SBR cannot automatically establish the missing facts. Restore trusted history or use an external recovery procedure. This design introduces no retirement ConfigMaps. Section 5.11 adds a narrowly controlled rejoin authorization in the existing Node coordination record; it cannot reconstruct unknown history or override missing isolation evidence.

### 6.7. Local process and restriction ownership limits

The host `flock` excludes cooperating local agents. It does not control processes that ignore the lock. Compromised host root can forge local state and remains outside the trust boundary.

The original restriction snapshot preserves pre-existing restrictions. It cannot detect another actor reasserting the same cordon after SBR added it. Document the required administrator coordination.

Keep the UID of each CR that still requires scheduling restrictions. Qualify a record-size limit and visible refusal of new offers before annotation capacity is exhausted. Do not remove list entries while their requests or required restriction updates remain unresolved.

### 6.8. Accepted limitation: peer availability

Ordinary requested-remediation lifecycle work requires a peer that meets Section 5.2 requirements. Its probe can be unhealthy. Independent execution paths prevent storage I/O from blocking a running peer.

CSI attachment failure can prevent replacement peer Pods from starting. A shared outage or rollout can therefore remove every controller that meets Section 5.2 requirements. A single selected node has no peer for its own request. Requested-remediation coverage requires an independently recoverable eligible peer. Operator rejoin coverage is separate and must survive held-worker outages; losing it blocks release rather than relaxing checks.

Without a peer, ordinary requested offers and CR status/finalizer processing wait. For a held incident, the independently hosted operator rejoin controller may validate BootEnded or cancel an unaccepted boot-mismatched offer using the same CAS rules and complete rejoin without a target peer Pod. On return, peers adopt the persisted execution/rejoin state and original requested-cleanup deadline rather than reopening completed work.

### 6.9. Remaining qualification questions

- Which supported CSI drivers accept the `10Mi` Block request and direct I/O pattern?
- Are the fifteen-second probe deadline, failed-evaluation threshold, and derived interval appropriate under measured storage latency?
- Are the total-outage grace, five-minute autonomous recovery default, and five-minute requested-observation limit appropriate under measured failures? None is a rejoin-release timeout.
- Does boot/shutdown ordering reject stale clean markers and preserve admission across same-boot service restarts on every supported host image?
- Can the operator/API/CSI controllers recover all selected held workers without starting workloads on those workers first?
- Are delegated storage semantics and AdminAssisted runbooks documented without requiring SBR vendor-specific integrations?
- Which watchdog and RHCOS versions satisfy the nowayout, process-exit, and verified-disarm contract?
- Which KubeVirt and CSI versions exhibit the expected OutOfService behavior?
- Can complete API scans finish within the supported node density and request deadlines?
- Do agent tolerations and host-reporter lifecycle management preserve coverage during termination and rolling updates?
- Which recovery procedure establishes missing history without granting permission for another autonomous reboot?
- How many CRs should the supported policy allow in the restrictions list?

The design does not guarantee that a replacement VM or Pod starts successfully. Cleanup observations and replacement workload availability remain different results.

## 7. Upgrade / Downgrade Concerns

An upgrade preserves omitted volume mode as `Filesystem`. Omitted `ioTimeout` retains two seconds in existing modes and selects fifteen seconds in BlockPerNode. Existing modes retain their counter and retry behavior. Explicit timeout overrides change their per-operation wait value. Only BlockPerNode selects failed-state evaluations and a complete probe deadline. Total-outage grace and autonomous recovery remain optional BlockPerNode fields.

Install and qualify host protection, the rejoin admission policy, compatible CRDs, webhooks, operator, agent, and host reporter before enabling BlockPerNode. Enroll each node explicitly with a validated initial journal; MachineConfig content must never overwrite the runtime journal. Migrate SBR state version 1 to 2 under the host lock after reconciling pending requests, initializing bootAdmission for the current boot without fabricating a prior clean shutdown, and migrate the Node owner record to version 2 with outOfServiceReleased=false for retained holders before enabling new writers. Never run older peer writers against the new record; rollout compatibility is a release prerequisite. A missing bootHold or bootAdmission in an enrolled version-2 journal is not clean. Do not remove the gate, admission policy, recovery permissions, or compatible reader while an incident remains unresolved; block downgrade and configuration retirement with a visible reason. Pin a common release that implements the nodestate contract. The feature is unavailable with common v1.17.0.

All components must understand the Node record version, execution phases, boot observations, set of listed CRs, and budget accounting. Never send BlockPerNode arguments to an older agent. Unknown mode and record versions must block execution rather than select a shared-storage path.

Install API permissions and OutOfService tolerations before enabling cleanup. Compatible peers can reconcile concurrently. Preserve their field ownership and conditional update rules.

Use `maxSurge: 0`, `maxUnavailable: 1`, and the host lock during replacement. These settings alone do not prove peer coverage. Keep a peer that meets Section 5.2 requirements for each unfinished target while planned changes occur.

Publish a qualification matrix for OpenShift, Kubernetes, KubeVirt, NHC, SBR, common, CSI, RHCOS, and watchdog versions. Document qualified host isolation, the delegated storage crash-recovery contract, both rejoin policies, and the external runbook requirements for AdminAssisted. Re-anchor source references when implementation begins.

The administrator must replace the SBR config CR to migrate modes because the effective mode is immutable. The administrator pauses new NHC requests and waits for peers to establish BootEnded for accepted requests. The administrator then requests deletion of remediation CRs that remain in the restrictions list. Cleanup checks need not complete successfully. However, the required removal of CR entries and restrictions must succeed, and the watchdog must shut down correctly.

Preserve the host journal and Node budget across configuration replacement. Do not delete the lock file while a process can hold it. Unknown history must remain unknown across restart.

Before downgrade, the administrator ensures peers resolve BlockPerNode requests and remove their CR entries and restrictions under Section 5.2. The administrator deletes BlockPerNode SBR config CRs through their cleanup path. The administrator confirms that the agents stopped before installing incompatible binaries or CRDs. Preserve the durable budget records.

Keep peer Pods and permissions until required release operations finish. A retained historical CR or unresolved incident can delay configuration removal if it still holds restrictions; retain the rejoin controller and snapshotted policy even if the original config is deleting. Report that dependency. Incomplete observations alone must not cause the delay, but unresolved rejoin safety explicitly can.

Existing shared-storage configurations can coexist only on disjoint node sets. Admission and reconciliation must check assignment conflicts. The host lock prevents a replacement process from controlling the watchdog while an old process still holds it.

## 8. Test Plan

Extend the existing API, controller, agent, and end-to-end suites. Instructions below describe actions by the test code and the component behavior it must check. The tests below are proposed. No test in this document has been executed. Fake clients cannot establish webhook, watchdog, CSI, or KubeVirt behavior.

### 8.1. API and compatibility

- Test omitted volume mode, explicit `Filesystem`, and explicit `Block`. Preserve their existing defaults and shared-storage behavior.
- Accept explicit `BlockPerNode`. Reject unknown values, a missing required StorageClass, and invalid durations.
- Test mode-dependent `ioTimeout` defaults, explicit overrides, and the `1s–10m` bounds in every mode. Reject zero.
- Test total-outage grace bounds and positive autonomous recovery periods. Reject those fields in existing modes. Accept zero only for total-outage grace.
- Check the effective timeout in agent arguments, initialization, and preflight. Preserve omission behavior in existing modes.
- Test immutable effective values, omitted/default equivalence, and equivalent duration strings. Reject transitions between distinct effective volume modes.
- Test controlled timing updates. Keep the original deadline on an active probe and apply new values only in the replacement monitor.
- Require private PVCs to use Kubernetes `volumeMode: Block` and RWO access. Never copy `BlockPerNode` into a Kubernetes volume-mode field.
- Test configuration overlap across namespaces, concurrent creation, node-label changes, and conflicting StorageClasses. Permit coexistence on disjoint node sets.
- Test `detectOnlyMode` independently of detection. Vary `maxConsecutiveFailures` and `sbrTimeoutSeconds` to check the BlockPerNode failure threshold and cadence.
- Validate the execution states and observation identities. Reject unknown states and invalid transitions.

### 8.2. Probe, local decisions, and watchdog

- Test transient failure, sustained failure, recovery, and intermittent failure against the failed-evaluation threshold.
- Require count one at the first error or deadline. Prevent extra increments from intervening errors, retries, or simultaneous callbacks.
- Require at least one evaluation interval between increments. Do not manufacture counts after delayed scheduling or reset them after elapsed time.
- Test complete-probe deadlines, including write, flush, read, and retries. Prevent nested timeouts from shortening the configured deadline unintentionally.
- Test the fifteen-second timeout default and configured overrides. A timeout contributes to the failure threshold without bypassing it.
- Prevent overlapping logical probes during retry waits and between their I/O operations.
- Keep one probe blocked after its deadline. Require evaluations to reach the threshold without another outstanding operation in that process.
- Reject its late success as recovery. Require a subsequent fresh complete success to reset the failed state.
- Test process restart during detection. Reset transient counts without publishing healthy or discarding known unhealthy status.
- Restart the process. Check that it starts a fresh probe without tracking or waiting for probe operations from the previous process.
- Test API-only loss, storage-only loss, and combined loss with independent observations. Start total-outage grace only after storage reaches the unhealthy threshold.
- Test live `NotFound`, authentication failures, authorization failures, 429, 5xx, certificate errors, transport failures, DNS failures, and deadlines.
- Test recovery before execution acceptance and after acceptance. Storage recovery alone must not block a valid requested reboot or permit another reboot for the same CR.
- Test detect-only behavior for requested and autonomous reboots.
- Start peer reconciliation before probe initialization. Block probe or journal I/O while the peer processes another node's request.
- Require the local agent to report local failures without cancelling the peer manager or failing the peer manager’s liveness check.
- Test exclusive host-lock ownership across process and Pod replacement. A losing process must not access the watchdog or write the journal.
- Exercise qualified watchdogs under software reboot failure, hardware timeout, planned shutdown, nowayout, and process termination. Check disarm results explicitly.
- Reject invalid or mismatched watchdog devices. Confine null-device behavior to explicit test support.
- Preserve existing shared-storage heartbeat and fencing tests.

### 8.3. Execution records, boot evidence, and autonomous history

- Race peers that offer different requests for the same Node UID. Require one accepted request record and preservation of unrelated Node fields.
- Race `Offered` cancellation against target acceptance. Require exactly one conditional transition to `Cancelled` or `Accepted`.
- Require the target to validate the request identity and current boot before accepting. Stale requests and replacement Nodes must not inherit permission.
- Restart peers and the local agent at every execution transition. Reuse the persisted record without issuing another reboot for the same accepted request.
- Require the local agent or reporter to read the exact `Accepted` record before reading the current local boot ID.
- Publish observations conditionally with the matching request UID, initial boot ID, and observed boot ID. Reject a changed record during publication.
- Require a peer that meets Section 5.2 requirements to validate the accepted record and observation before setting `BootEnded`.
- Reject same-boot observations, empty boot IDs, mismatched identities, and observations from a previous request.
- Change `Node.status.nodeInfo.bootID` without a valid observation made in the order required by Section 5.7. It must not establish `BootEnded` by itself.
- Test an unrelated boot before acceptance, a boot during observation, and delayed observation publication. None may establish incorrect execution order.
- Keep the target probe unavailable after reboot. Require the independent reporter to publish a valid observation through the API.
- Force-delete the remediation CR in each execution state. The reporter must still process the Node record. An unresolved accepted record must not permit another request.
- Replace the Node with another UID but the same name. Old execution records and observations must not change the replacement Node.
- Test multiple generations of one CR and successive CR UIDs. Each valid new CR can authorize one reboot without a reboot-count limit, even when the autonomous attempt for the outage was already used. Neither retries nor a new UID may bypass execution ordering or change whether another autonomous reboot is permitted.
- Preserve autonomous journal state across process, Pod, and host restart. Test full disks, read-only filesystems, late writes, and uncertain write durability.
- Test lost, corrupt, and unsupported journal history. Unknown history must not reset the autonomous counter or permit an automatic fresh start.
- Keep storage and API unavailable across boots. Require at most one autonomous reboot per unresolved outage period.
- Test the five-minute recovery default, configured durations, interrupted recovery, restarts, and wall-clock changes. Interrupt recovery on the first failed probe or timeout, before the unhealthy threshold. Restore permission for one autonomous reboot only after continuous healthy storage and reachable API for the configured interval and completed history reconciliation. Diagnostic pruning must preserve required budget history.
- Keep requested execution records unchanged during an autonomous reboot. Do not assign an earlier autonomous reboot to a later remediation CR.

### 8.4. Restrictions, observation within a fixed period, and request completion

Use envtest for races between peers recording Offered and processing remediation CR deletion. Require actual API conflicts and finalizer behavior, rather than relying only on fake clients.

- Have peers try to add different CRs to the restrictions list at the same time. Preserve the initial cordon and OutOfService snapshot while any listed CR remains.
- Remove one CR’s entry from the list while another request remains active. Retain restrictions needed by the remaining listed CR.
- Remove the last CR entry. If no rejoin holder remains, remove only SBR-added restrictions; otherwise preserve the independent hold and snapshot. Preserve pre-existing restrictions and unrelated taints.
- Concurrently remove a CR’s entry, add another CR, and submit delayed updates from an older request. Check Node UID and resource version on each mutation.
- Pause a peer after it reads the Node and checks that a CR can enter Offered. The request has no Offered record or restrictions entry yet. Delete the CR through normal finalizer handling. Require a persisted coordination increment before finalizer removal. Resume the claim after storage fails again, with the condition write delayed. Its stale Node update must fail.
- Reverse that order so the offer wins before the release update. Require cancellation before release, or resolution if target acceptance wins. Never remove the finalizer while that execution remains unresolved.
- Test remediation CR deletion when the CR has no restrictions entry, including when an earlier peer removed its entry. Require an actual Node mutation. An unchanged update must not satisfy release.
- With no unresolved rejoin holder, remove CR A’s entry as the last entry in the restrictions list. Uncordon the Node. Lose the release response or restart the peer before finalizer removal. Add CR B to the restrictions list and cordon the Node again. Retry A's release with the guarded coordination increment. Remove A's finalizer while the unschedulable taint remains. Preserve B’s execution, restrictions entry, reboot-count records, cordon, taints, and saved original state.
- With no unresolved rejoin holder, delay unschedulable taint removal after the final listed CR leaves. Permit finalizer removal after the required release update succeeds. Repeat with an administrator cordoning the Node before the retry. Preserve that cordon without waiting for the taint to disappear.
- After a peer’s Offered update conflicts, change the remediation CR’s deletion state, NHC stop state, or terminal state, or replace its UID. After reading the Node, read the CR without the cache and repeat the checks for entering Offered. Never reuse cached checks with a new Node version.
- Replace the Node before a release retry. Reject the old binding without changing the replacement. Check CR UID and resource version during finalizer removal.
- Require accepted BootEnded evidence and separately validated, qualified isolation before requested OutOfService processing. Reject same-boot kubelet-stop observations and stale or mismatched hold evidence.
- Confirm that agent Pods tolerate OutOfService and that the host reporter remains independent of Pod eviction. Observe workload removal through Kubernetes controllers.
- Assert that SBR never directly deletes Pods, VMIs, VolumeAttachments, PVCs, or PVs. The BlockPerNode cleanup path must not require Pod-delete permission.
- Persist the internal five-minute deadline before the first OutOfService attempt. Preserve it across conflicts, peer replacement, and process restart.
- Inject OutOfService update failures and lost API responses. Resolve uncertain writes through live reads. Deadline expiry must not imply successful taint application.
- Test skipped optional observations, empty complete scans, remaining objects, partial scans, API errors, and unavailable optional resource APIs.
- Treat partial or failed observations as unknown. Never report workload cleanup completed because an observation failed or its deadline expired.
- After BootEnded and verified continuing isolation, expire the deadline with resources remaining or observations unknown. Permit bounded policy completion while retaining the independent hold. Without isolation evidence, do not report fencing or remediation success.
- Keep taint application unconfirmed through deadline expiry. Record cleanup as unknown and `outOfServiceApplied=false` with a reason. Permit the same completion policy only after BootEnded and verified continuing isolation, without claiming that the taint was applied.
- After requested-cleanup expiry, do not extend its deadline or add the taint from that operation. Independent incident reconciliation can act only under its own current permission. Expiry without valid BootEnded/isolation evidence must not produce success.
- Retain OOS while a CR or rejoin record requires it. Test CR deletion with incomplete observations: remove only the CR holder, preserve the boot hold, restriction snapshot, and required restrictions.
- Have NHC stop a remediation request and retain its CR for escalation history. Check that the peer permits a later request only after the required execution transition and completed rejoin handshake. Require the peer to preserve the earlier CR’s recorded failure outcome.
- Create a later request while earlier observations remain incomplete. Require both the execution transition and the independent rejoin handshake before offering it; expired observations alone are insufficient.
- Close cleanup permission in the Node record before publishing terminal observation status. Reject delayed taint writes after closure or successor replacement. Crash the peer process after it closes cleanup but before it publishes CR status. Require the resumed peer to report unknown cleanup without reopening the operation.
- Cancel an offer after a reported local boot mismatch. Never rebind the same CR to the new boot.
- Recover a local reservation after a trusted successor replaced its cancelled record and the old CR disappeared. Preserve execution history and recorded autonomous attempts.
- Remove the restrictions entry for a deleted CR whose request is no longer current. Do not use the later request’s phase as its result.
- Have NHC request CR deletion while cleanup remains incomplete. Remaining resources must not block CR-holder removal after the required execution transition; they must still prevent unsafe rejoin.
- Test API unavailability separately from incomplete workload cleanup. Failed required Node mutations must remain retryable and must not be reported as completed.
- Remove every peer that meets Section 5.2 requirements, then restore one. Resume the persisted execution record, set of listed CRs, and original deadline.
- Test administrator deletion and replacement of an SBR config CR while remediation CRs remain in the restrictions list. Preserve the peers and permissions needed to remove those entries correctly.

### 8.5. End-to-end coverage

Exercise the use-case matrix in Section 5.1. Retain Filesystem and shared Block coverage. Run private Block tests on representative RWO-capable CSI drivers.

Run at least two peers against one target. Stop a peer controller while the target accepts a request, the peer checks cleanup, or the peer removes a restrictions entry. Require another peer to continue without repeating the reboot or extending the observation deadline.

Keep one running peer's probe I/O blocked while it reconciles another target. Separately prevent all peer Pods from starting: ordinary requested lifecycle work waits, but the operator rejoin controller must still recover held incidents from host evidence and standard API state. Loss of the operator/API/CSI controllers must keep release blocked.

Test Pod and PVC replacement while preserving host state. Test requested host reboot, NHC remediation timeout, remediation CR deletion, SBR config CR replacement, and operator restart. Verify that incomplete Pod, VMI, or attachment observations do not prevent the specified completion and deletion behavior.

Attempt to start workloads after reboot before OutOfService processing. Assert that the installed host protection prevents execution and preserves the hold through API recovery. Any such execution is a qualification failure, not an accepted risk.

Scenarios 13 and 14 remain deferred. Scenario 17 includes requested execution ordering, autonomous reboot counts and limits, history uncertainty, and exclusion of a second local agent. A power-loss test must not expect the watchdog to restore power.

### 8.6. Kind cluster testing in CI

Extend [the Block Kind workflow](../../.github/workflows/kind-e2e-block.yaml) with `sharedStorageVolumeMode: BlockPerNode` coverage. Preserve the existing shared Block job and Filesystem coverage.

Extend [the block setup script](https://github.com/medik8s/tools/blob/main/dev/kind-block.sh) and its storage manifest with a separate BlockPerNode fixture. The existing fixture shares one loop device across two workers and exposes one static RWX Block PV.

1. Determine the Kind node count instead of assuming two workers.
2. For N Kind nodes, create N distinct backing images, loop devices, and RWO Block volumes.
3. Map each device only into its assigned node. Give its PV the corresponding node affinity.
4. Verify distinct device, PV, and Pod-owned PVC identities. Require Kubernetes `volumeMode: Block` and RWO access.
5. Provide claim reclamation or provisioning for replacement Pods. A retained PV bound to a deleted claim cannot satisfy its replacement automatically.
6. Use at least three selected workers to test two concurrent peers against another target.
7. Test single-node storage failure and hung I/O without disrupting another node's private probe.
8. Test Pod and claim replacement while retaining required host history.

Extend [the setup tests](https://github.com/medik8s/tools/blob/main/dev/tests/test_kind_block.py) for variable node counts, independent devices, partial setup failure, replacement claims, and repeated teardown. Test failed detach recovery. Collect device mappings, PV/PVC identities, agent and reporter logs, execution records, restrictions lists, and observation results.

Always run teardown after failures. Remove fixture devices and backing images only after the Kind nodes are gone. Preserve undetached devices, their backing files, and cleanup records for retry.

The current reboot watcher waits for `FencingSucceeded` before restarting a Kind container. BlockPerNode requires boot evidence before that condition. Use explicit test-only reboot and boot-evidence adapters with the required observation order. A forced watcher timeout must not count as successful remediation.

Kind shares the host kernel. Its null watchdog cannot qualify hardware watchdog behavior. Keep production checks enabled. Run real-host, CSI, and KubeVirt qualification separately from these simulations.

### 8.7. Default-deny boot protection and rejoin qualification

Additional acceptance cases for the reviewed protocol:
- Prove agent-only watchdog ownership, no host helper device opens, reset-path disabled state, late/failed agent scheduling, and rejection of persistent-armed profiles. Test nowayout behavior separately from across-reset behavior.
- Hold all enrolled workers simultaneously; recover with operator, admission and CSI/KubeVirt dependencies outside that set. Reject conflicting affinity/enrollment and alert on dependency drift.
- Race an already-passed ExecStartPre and pending restart against shutdown; no CleanShutdown until start jobs, workload processes and restart sources are quiescent. Crash/abort each shutdown barrier step and preserve unrelated masks.
- Complete release while agent absent or probe blocked/unhealthy; acknowledge Idle without healthy storage; retain cordon and original snapshot; allow a new requested attempt without reopening old cleanup.
- Race durable inhibit against autonomous commitment, lose its response, restart/change config, and verify explicit clear, requested-action independence and no cancellation of committed reset.
- Exercise staged enrollment, deleted journal versus true first provisioning, enrollment activation crashes, migration baseline, detect-only on enrolled and unenrolled nodes, bounded release history and invalid condition payloads.
- Expire host client certificates during a hold, run isolated credential repair, reject mismatched key/subject/Node UID, and resume reporting without starting kubelet.
- Retain holds for foreign finalizers and failed dependency controllers; diagnostics identify exact blockers without unbounded API payloads. Test Node CAS conflicts, stale evidence and authorization response loss.


Use deterministic transition tests, envtest for real resourceVersion/finalizer conflicts, and real-host OpenShift/Kubernetes qualification. Kind cannot establish systemd ordering, watchdog reset behavior, or absence of resumed VM execution. Validate this feature's execution/reconciliation contract separately from backend crash semantics.

- Valid clean boot during API outage: consume the prior clean record, durably persist current Running, then start workloads without API calls. Ordinary kubelet/runtime restart in that admitted kernel boot is permitted; it does not create an unclean incident.
- Power cut/panic before an SBR decision, during normal Running, or during shutdown before clean commit: the new boot is held before protected services start. No pre-crash remediation flag is required.
- Replay a clean marker across two host boots, crash after clean consumption but before workload startup, and abort shutdown after clean commit. Never allow workloads while a reusable clean marker remains. Persist Running before readmission after an aborted shutdown.
- Perform an SBR software reboot through normal graceful systemd shutdown: the explicit Required hold overrides clean-shutdown handling. A failed kubelet stop, hung workload termination, or shutdown timeout must not create a clean record or delay the committed reset.
- Requested reservation crash before/after Accepted, including lost update responses: hold remains. An Idle outage with a committed requested attempt is not a clean boot. Same-boot cancellation requires matching identity and cannot clear a successor incident.
- Combined API/storage outage, autonomous reboot, API-only recovery: hold remains until all common reconciliation prerequisites pass. Host reporting and operator rejoin require no target kubelet/runtime/probe/CSI node plugin.
- Stop kubelet in the original boot while a VM keeps writing: do not assert post-reset isolation or authorize OutOfService. Test independent VM autostart/runtime/socket paths and prohibit unsupported paths before enrollment.
- Inject crashes around fsync/rename/directory fsync, enrollment activation, Held, Authorized, Released/Running, Completed, and localIdle. Uncertain durability fails closed; recovery preserves incident IDs and does not consume another autonomous attempt.
- Remove gate binaries, fail journal mounts, change service ordering, retry manual service starts, and perform MCO rollouts under SELinux enforcement. Missing state cannot pass via ConditionPathExists or cached oneshot success. Gate artifacts require no network download.
- Expire node certificates or deny own-status patches: isolation remains enforced, credential repair does not require releasing kubelet. Reporter cannot patch Node taints/labels or rejoin authorization. Validate field-admission rules against create/update/patch/removal and label-bypass attempts while preserving unrelated status writes.
- Remove every eligible peer Pod while leaving operator/API/CSI controllers available: operator rejoin can process ordered evidence, reconcile held incidents, and release without waiting for a peer on a held worker. Remove the operator or required CSI controller: retain the hold and expose the dependency.
- Exercise omitted attachRequired, confirmed absent CSIDriver, RBAC denial, failed reads, contradictory metadata and unresolved driver identity as distinct cases. Defaults retain attachment checks; uncertainty blocks release and never becomes NotApplicable.
- Test socket access with SELinux enforcing: approved administrative domain succeeds; unauthorized users/domains and the host release service without an explicit grant fail. Validate labels after reboot/recreation and peer-credential checks independently.
- Test certificate warnings with forward/backward wall-clock jumps, unsynchronized time, margins larger than remaining validity, not-yet-valid/expired credentials and loss of telemetry. Monotonic backoff is unaffected; TLS validation and boot holds remain enforced.
- Exercise attach-required and attachRequired=false CSI fixtures, multiple drivers on one target, ordinary non-CSI volume types, legitimate absence of historical objects, optional KubeVirt API absence, remaining terminating objects, failed/paginated lists, missing driver metadata, and stale VolumeAttachments. Empty lists alone never establish fencing or physical detach; attachRequired=false does not require a synthetic attachment.
- Keep stale workload Pods/VMIs present, including those tolerating OOS; deny release even if attachments disappear. Verify supported workload controllers respect the incident placement restriction; exercise and reject/configure direct-binding or independent-autostart bypasses.
- Expire the requested five-minute cleanup observation, delete/force-delete its CR, or remove the config. The independent incident remains and can reconcile under its own permission; expired requested cleanup never reopens. Do not wait for NHC health/CR deletion before discharging the resolved OOS holder during release.
- Race stale peer taint/offer updates, incident reconciliation, and Authorized. Exactly one compatible conditional transition wins. After authorization, old cleanup cannot restart; lost responses or elapsed time cannot revoke possibly consumed authorization.
- Automatic releases after the common checks without manual approval or backend calls. AdminAssisted remains held after identical API checks until matching external-runbook completion is recorded; approval cannot waive missing isolation/reconciliation. Configuration edits cannot relax an active incident's snapshotted policy.
- Reboot after authorization but before consumption, after Released but before kubelet startup, and before completion acknowledgement. Repeat reconciliation for the new boot and generate a fresh releaseID; reject previous-boot approval/replay. Ordinary same-boot retries remain idempotent.
- Fail the probe after startup: retain scheduling restrictions and report RejoinFailed, but do not claim the node is still isolated or repeat old-incident cleanup. Rearm autonomous attempts only through the existing recovery period and history checks.
- Validate Node UID replacement, restored/cloned disk reenrollment, malformed/unknown schema, and protection downgrade. A local parser cannot detect every silent rollback to a valid old snapshot; isolated administrative restore procedures are mandatory.
- Verify independent recovery placement, control-plane/SNO rejection, watchdog state after reset, retained artifacts during unresolved configuration deletion, and the explicit delegated-storage contract in installation/runbook documentation.

## Alternatives considered

- Run the BlockPerNode lifecycle controller in the elected operator manager or a separate Pod without a probe volume. This removes the probe-volume startup dependency from lifecycle reconciliation. Ordinary requested remediation retains the existing peer controller and its availability requirement; Section 5.11 places only held-node rejoin in the operator so recovery does not depend on starting a held worker.
- Elect one peer or acquire a Lease per remediation CR. Either can reduce duplicate reconciliation, but neither selects between competing requests for one Node. Conditional transitions and the durable Node owner remain required. This proposal uses concurrent peers without another election mechanism.
- Keep the existing shared-storage SBR design. This preserves the current implementation, but it does not support the CSI drivers targeted by this proposal. These drivers cannot provide the required shared Block volume with concurrent writes.
- Chain NHC, SNR, and SBR. This reuses SNR reboot and cleanup behavior, but divides one remediation across two providers. The providers must also coordinate watchdog control, boot information, and responsibility for cleanup.
- Trigger remediation from the Node `Ready` condition or make out-of-band fencing the primary path. `Ready` would allow an API or network failure with healthy storage to disrupt a VM. Out-of-band fencing is useful where available, but is not present in every target environment and remains a separately qualified isolation mechanism.

This design retains one remediation provider and one watchdog owner. Host boot/shutdown protection and an operator rejoin reconciler add durable post-reboot isolation and policy-controlled recovery without moving CSI probing or vendor fencing to the host. It replaces the shared probe with a private per-node Block volume. It preserves existing shared-storage behavior, but boot protection is a substantial host integration and recovery protocol, not a minimal implementation change. Enablement requires the complete qualified safety path described in Section 5.11.

## Terminology

A **remediation CR** is a `StorageBasedRemediation` resource. An **SBR config CR** selects nodes and configures agents. An **NHC config CR** defines the health policy that creates remediation requests.

| Term | Meaning in this design |
|---|---|
| Local agent | The part that checks storage, controls the watchdog, reboots its host, and writes the journal. |
| Peer controller | The API controller in an agent that processes requests for other nodes. |
| Reporter | The host reporter/release service publishes ordered boot observations without kubelet or the probe; it never feeds the watchdog. |
| Budget | Permission for one autonomous attempt per unresolved outage; requested attempts do not consume it. |
| Incident / incidentID | One held recovery episode, including an unclean reboot without an SBR attempt. |
| releaseID | Unpredictable identifier for one authorization bound to a Node UID, incident and current boot; not storage-safety attestation. |
| admissionID / cleanShutdownID | Local admission transaction identity / single-use clean-shutdown identity. |
| outageEpoch | Durable identity of the outage period used for autonomous-attempt accounting. |
| SBRBootHeld | Node condition carrying validated current-boot hold evidence. |
| SBRBootProtectionReady | Node condition identifying installed and checked protection; distinct from watchdog readiness. |
| outOfServiceReleased | Per-holder acknowledgement that its OOS requirement was discharged; the CR may still exist. |
| Probe | This private Block volume contains disposable test data. |
| Journal | This durable host file records attempts and autonomous reboot counts. |
| Request identity | The CR UID identifies a request. Its Node UID and initial boot ID cannot change. |
| Offered | A peer recorded a request that the target may accept. The target cannot reboot for the request yet. |
| Accepted | The target successfully changed Offered to Accepted through a conditional Node update, without a reboot-count check. This decision cannot be cancelled. |
| Cancelled | A peer changed Offered to Cancelled in the Node record. The target can no longer accept or execute this request. |
| BootEnded | A peer validated information showing that the boot identified by the accepted request’s beforeBootID ended. |
| Recorded reboot decision | The agent durably recorded that it must reboot during the attempt’s beforeBootID. It may invoke software reboot and must stop feeding the watchdog. |
| Attempt | This is one requested or autonomous reboot operation. A requested attempt uses its CR UID. An autonomous attempt uses a local ID. |
| Execution ownership | Only the current request in the Node record may be accepted and completed on the Node. |
| Listed CR | This CR has an entry in the Node record’s holders field because it still requires SBR’s scheduling restrictions. The entry is separate from execution ownership. |
| Restriction snapshot | This records the cordon and OutOfService state before SBR added the first CR to the current restrictions list. |
| Cleanup observation | A peer checks relevant Pods, VMIs, and VolumeAttachments through the API after reboot, within a fixed period. |
| Rejoin | Section 5.11 authorizes a held boot after verified execution isolation and applicable API reconciliation, plus external-runbook completion in AdminAssisted. Physical storage recovery is delegated; CR deletion alone is not rejoin. |
| Boot hold | Durable, incident-bound host state that blocks protected workload starts until boot-bound authorization is consumed; independent of FenceLatched and reboot budget. |
| Boot admission | Per-boot durable state and single-use clean-shutdown record; covers unclean boots even before an SBR decision and permits admitted same-boot service restarts. |
| Automatic rejoin | Host isolation plus completed applicable Kubernetes reconciliation; physical crash recovery is delegated to the storage stack. |
| AdminAssisted rejoin | The same common checks plus audited completion of the deployment's external recovery runbook. |
| Autonomous reboot count and limit | SBR records autonomous attempts and their limit per node. A new CR, process, or configuration cannot reset these records. Requested reboots have no such limit. |
| Outage period | After an autonomous attempt, no further autonomous attempt is permitted until storage and API health satisfy the configured recovery period. |
| Rearm | SBR permits another autonomous reboot after the configured recovery period and required history checks. The recovery period defaults to five minutes. |

Recording Offered, accepting the request, recording the local reboot decision and hold, establishing BootEnded, proving continuing isolation, checking cleanup, and authorizing rejoin are separate steps. Bounded processing can finish while cleanup remains incomplete; the independent hold remains until controlled rejoin completes.
