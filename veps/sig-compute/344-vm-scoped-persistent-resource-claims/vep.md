# VEP #344: VM-Scoped Persistent ResourceClaims for DRA Device Passthrough

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: v1.11.0
- This VEP targets beta for version: TBD
- This VEP targets GA for version: TBD

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting a milestone or release*.

- [x] (R) [Enhancement issue](https://github.com/kubevirt/enhancements/issues/344) created
- [ ] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Overview

A VM may be restarted by replacing its VirtualMachineInstance (VMI) and
virt-launcher Pod. Today, a `ResourceClaimTemplate` referenced by the VMI is
passed to that Pod. Kubernetes creates a Pod-owned `ResourceClaim`, then deletes
it with the Pod. The replacement Pod may receive different hardware.

This VEP lets a VM request that a template-backed hardware allocation remain
stable across launcher Pod replacement. An intentional stop releases the
allocation by default; the VM may instead retain it while stopped.

Existing VMs and standalone VMIs keep their current behavior unless the VM
opts in.

## Motivation and Goals

A replacement GPU can disrupt guest drivers or device-bound licensing. A
replacement NVMe device can make guest data unavailable. A replacement NIC can
change hardware identity. Releasing a device between launcher Pods also lets
another workload allocate it before the VM restarts.

This VEP aims to:

- Preserve the same allocated DRA device across intentional VMI and launcher
  Pod replacement.
- Release the allocation on intentional VM stop by default, while offering an
  option to keep it through stop/start.
- Keep existing VM and standalone VMI claim behavior unchanged unless opted in.
- Release the device allocation when its VM is deleted.

This VEP does not cover direct `resourceClaimName` references, VEP-300 managed
claims, eager allocation before a VM starts, or live migration of DRA devices.
Those sources may adopt the reservation lifecycle in a later proposal.

## API

The existing `VirtualMachineInstanceResourceClaim` entry gains an optional
`allocationPolicy` field. Non-default policies are valid only in
`VirtualMachine.spec.template.spec.resourceClaims` for a
`ResourceClaimTemplate` source. Users declare each claim once:

```go
type ResourceClaimAllocationPolicy string

const (
    ResourceClaimAllocationPolicyEphemeral    ResourceClaimAllocationPolicy = "Ephemeral"
    ResourceClaimAllocationPolicyWhileRunning ResourceClaimAllocationPolicy = "WhileRunning"
    ResourceClaimAllocationPolicyPersistent   ResourceClaimAllocationPolicy = "Persistent"
)

type VirtualMachineInstanceResourceClaim struct {
    Name                      string                         `json:"name"`
    ResourceClaimName         *string                        `json:"resourceClaimName,omitempty"`
    ResourceClaimTemplateName *string                        `json:"resourceClaimTemplateName,omitempty"`
    AllocationPolicy          *ResourceClaimAllocationPolicy `json:"allocationPolicy,omitempty"`
}
```

| Policy | Generated claim owner | Launcher replacement | Intentional VM stop |
|---|---|---|---|
| Omitted or `Ephemeral` | Pod, as today | New claim and allocation | Released |
| `WhileRunning` | VM | Keep claim and allocation | Remove VM reservation, then delete claim |
| `Persistent` | VM | Keep claim and allocation | Keep claim and allocation |

A VM user adds the policy to the existing template-backed claim entry:

```yaml
spec:
  template:
    spec:
      resourceClaims:
      - name: gpu
        resourceClaimTemplateName: gpu-template
        allocationPolicy: WhileRunning
```

The VM template is authoritative for ownership and retention. A claim entry
without a policy continues to pass its template reference to the launcher
Pod. Switching between `Ephemeral` and VM ownership, or changing the template
source, requires the VM to be stopped and the old claim to be gone. For a
stopped `Persistent` VM, first change to `WhileRunning` and wait for claim
deletion before switching to `Ephemeral` or changing the template source.
Changing between `WhileRunning` and
`Persistent` while running changes the next stop's behavior; it does not
replace the running VMI's claim. Changing from `Persistent` to `WhileRunning`
while stopped releases the existing allocation.

The field is in a type shared with standalone VMIs because VM templates embed
the VMI spec. Admission rejects VM-owned policies on standalone VMIs. A
separate VM-level template list would require users to repeat the logical
claim name and template source in two places.

## Design

### Claim creation and launcher references

For `WhileRunning` and `Persistent` entries, KubeVirt reads the named
`ResourceClaimTemplate` in the VM's namespace and lazily creates one concrete
`ResourceClaim` before creating the VMI. It copies the template's claim spec,
sets the VM as controller owner, and records the logical claim name and source
on the generated claim. The claim persists across launcher replacement. It is
deleted at explicit stop for `WhileRunning`; `Persistent` retains both
the claim and its allocation through stop/start.

The generated name is deterministic and includes a hash of the VM name and
logical claim name. KubeVirt truncates the readable prefix to fit the claim
name limit while retaining the hash suffix. A plain concatenation can collide
when names contain hyphens or exceed the limit. Before reusing a claim with
the expected name, KubeVirt verifies its
VM owner UID and logical claim identity. It reports a conflict instead of
adopting or overwriting another claim.

KubeVirt rewrites the generated VMI's entry from `resourceClaimTemplateName`
to the concrete `resourceClaimName` and clears `allocationPolicy` on that VMI
entry. The launcher Pod references the concrete name, not the template. The
VMI rewrite is also needed by virt-launcher: it reads DRA device metadata
from a path determined by the VMI claim source, which must match the Pod's
concrete claim reference. The VM template remains authoritative for the
policy. Standalone VMIs and VM entries with an omitted or `Ephemeral` policy
retain the existing Pod-owned template flow.

### Allocation reservation

Kubernetes allocates the claim while scheduling the first launcher Pod and
adds that Pod to `status.reservedFor`. A dedicated KubeVirt reservation
controller adds a second consumer reference for the VM:

```text
{apiGroup: kubevirt.io, resource: virtualmachines, name: <vm>, uid: <vm UID>}
```

The controller adds this reference only after it sees a non-nil allocation
and the VM's own launcher Pod in `reservedFor`. This prevents attaching the VM
to an allocation made for an unrelated consumer. It preserves other consumer
references and retries status-update conflicts. Kubernetes removes the old
Pod's reference when that Pod is done; it leaves the VM reference alone. As
long as the VM reference remains, Kubernetes does not deallocate the claim.
The replacement Pod is then scheduled against the same allocation.

`ReservationEstablished` is true only when every current VM template entry
with a VM-owned policy resolves to a claim owned by the current VM UID, that
claim has a non-nil allocation, and its `reservedFor` contains the VM UID.
Initial establishment also requires the current launcher Pod's reservation.
The condition is invalidated when the VM entry, claim UID, or allocation
changes, or the VM reference disappears. Before an intentional restart,
virt-controller checks the current claims again; a stale condition cannot
authorize launcher deletion.

The reservation controller owns all KubeVirt writes to `status.reservedFor`.
Claim creation and VMI rendering remain separate from that privilege. The
controller watches VMs, VMIs, Pods, and claims; reconciles desired VM entries
and existing VM reservations; and removes a reservation on explicit stop,
release from `Persistent` to `WhileRunning` while stopped, or VM deletion.
Claim deletion follows reservation removal. Admission blocks source changes
and removal of VM ownership until that cleanup completes.
Only a matching VM UID may retain or delete a generated claim.

### Restarts, stops, and deletion

A guest reboot within an existing launcher Pod does not change its Pod
reservation. `virtctl restart`, automatic VMI recovery, and launcher Pod
replacement do. The VM reservation survives these replacements for both
VM-owned policies. `virtctl restart` is encoded as paired `StopRequest` and
`StartRequest` actions. The pair is a restart, not an explicit stop: under
every run strategy, virt-controller must establish and recheck the VM
reservation before acting on the stop request, and must skip claim cleanup
for that pair. If the reservation cannot be established, it reports the
problem and leaves the old launcher in place. Unexpected Pod loss during the
initial allocation-to-reservation gap remains an alpha limitation.

| Transition | `WhileRunning` | `Persistent` |
|---|---|---|
| Guest reboot in the same launcher | Keep Pod and VM reservations | Keep Pod and VM reservations |
| `virtctl restart`, automatic recovery under `Always` or `RerunOnFailure`, or launcher replacement | Keep claim and VM reservation | Keep claim and VM reservation |
| `virtctl stop`, `runStrategy: Halted`, or a stopped `Manual` VM | Remove VM reservation, then delete claim | Keep claim and VM reservation |
| Completion under `Once`, or successful completion under `RerunOnFailure` with no replacement | Remove VM reservation, then delete claim | Keep claim and VM reservation |
| VM deletion | Remove VM reservation, then delete claim | Remove VM reservation, then delete claim |

For `WhileRunning`, cleanup reconciles the VM's desired state and terminal
VMI status, including when the VMI is already absent. It does not depend on
a particular `stopVMI` call path. A stop waits until the old VMI and launcher
Pod can no longer use the claim, then removes the VM reservation and deletes
the claim. The next start creates a new claim from the current template and
may get different hardware. A restart is never processed as a stop merely
because its VMI is temporarily absent. The transition mapping must be
confirmed with sig-compute before alpha.

A reservation on a VM-owned claim requires deletion cleanup: Kubernetes does
not prune unknown consumer types, and its allocation finalizer can keep the
claim in `Terminating` while a VM reference remains. The reservation
controller adds a VM finalizer while reservations exist. On VM deletion it
removes each VM reference, then removes its VM finalizer; Kubernetes can then
deallocate and garbage-collect the VM-owned claim. The controller also removes
orphaned references whose VM is missing or has a different UID. Disabling the
feature gate must not disable this cleanup for already-created reservations.

### Scheduling and migration

A retained node-local allocation pins future launcher Pods to the node in the
allocation's node selector. If that node is unavailable, the replacement Pod
remains Pending until the node returns or the allocation is explicitly
released. This VEP does not make DRA devices live-migratable. VMs using a
VM-owned DRA claim must have `VirtualMachineInstanceIsMigratable=False` with
a DRA reason. Migration creation admission must reject it, and automatic
evacuation must respect the same condition. This applies regardless of device
type until a migration design provides the same device guarantees.

The claim spec is immutable. Updating the `ResourceClaimTemplate` does not
change an existing VM-owned claim. To request a different device or apply a
new template spec, stop the VM. For `WhileRunning`, wait for the old claim
to be deleted, then start the VM. For `Persistent`, change it to
`WhileRunning` while stopped, wait for reservation removal and claim deletion,
restore `Persistent` if continued retention is desired, then start. This
also lets a user manually reallocate a stopped VM without deleting the VM.
If the device's node is unavailable while the VM is meant to run,
explicitly stop and release the claim before starting again. KubeVirt does
not silently reallocate on another node. Deleting an allocated claim while
its VM reservation remains leaves it in `Terminating`.

## Validation and Security

The `PersistentDRAClaims` gate is disabled by default during alpha. Admission
accepts new `WhileRunning` and `Persistent` entries only in a VM's VMI template
while the gate is enabled. Direct `resourceClaimName` references and standalone
VMIs cannot opt in. The template must be in the VM's namespace. Claim entry
names must be unique and have exactly one source. Generated VMIs have a
resolved direct reference and no policy; standalone VMIs cannot request VM
ownership by supplying a VM owner reference. Admission rejects
switching between `Ephemeral` and a VM-owned policy, or changing the template
source, while a VMI exists. A stopped VM may change `Persistent` to
`WhileRunning` to release the claim, including after the gate is disabled.
Admission rejects changing the template source or switching to `Ephemeral`
while the old VM-owned claim still exists. Reconciliation must finish deleting
it before reusing its name for a different template source.

Updating `status.reservedFor` requires `resourceclaims/status` update and, on
Kubernetes versions with granular DRA status authorization,
`resourceclaims/binding` update. The latter is a synthetic cluster-wide
permission. It belongs to a dedicated, auditable reservation controller
ServiceAccount, not virt-controller. Claim creation, template reads, and
claim deletion use a separate identity without binding permission. The
reservation controller changes only the VM's own consumer reference, reports
write failures, and retries them. A metric and VM condition expose orphaned
or stuck reservations.

Kubernetes upstream confirmed that third-party VM references in
`reservedFor` are supported, as [recorded in the earlier VEP review](https://github.com/kubevirt/enhancements/pull/345#discussion_r3805771749).
Kubernetes preserves non-Pod entries in its claim controller, with existing
controller tests for that behavior. KubeVirt must still test reservation
creation, launcher replacement, stop, and deletion against supported
Kubernetes versions. This proposal requires no Kubernetes API or scheduler
change.

A VM reservation keeps the allocation but does not make the claim exclusive:
Kubernetes currently permits another Pod in the namespace to reference and
reserve the same named claim. This design assumes namespace users with Pod
creation rights are trusted to use one another's claims. Environments that
need exclusive VM use must enforce Pod claim references with a separate
admission policy. KubeVirt's VM controller cannot provide that boundary.

There is a short gap after the scheduler allocates a claim and before the VM
reservation can be added: Kubernetes rejects a `reservedFor` entry on an
unallocated claim. Gating intentional restart protects that operation, but
unexpected launcher loss or terminal Pod status in this interval may release
the allocation, even if the Pod object has a finalizer. The alpha guarantee
starts once `ReservationEstablished` is true. This gap must be measured and
documented; closing it completely requires a Kubernetes-side reservation
mechanism before first allocation.

## Alternatives

The original `VirtualMachineSpec.resourceClaimTemplates` list follows
`dataVolumeTemplates`, but requires the user to repeat each claim's logical
name and template source alongside the VMI template. [Reviewers objected to
the parallel API](https://github.com/kubevirt/enhancements/pull/345#discussion_r3791748826). The single-entry policy avoids that duplication.

A generic Kubernetes API could manage claims across Pod replacement, but the
VM lifecycle policy and device identity requirement are specific here. This
VEP uses existing DRA primitives for template-backed VMs and leaves a generic
workload design to Kubernetes.

An external controller selected by a VM label would make opt-in apply to the
whole VM. One VM may need to retain an NVMe allocation while stopped but
release a GPU allocation; a label would need separate configuration mapping
each claim to a retention policy. That mapping would duplicate the claim
references in the VM template and could drift from them. The per-claim field
keeps the policy with its reference. A dedicated reservation controller still
isolates `reservedFor` write permission, while the feature gate controls new
opt-ins. Cleanup must remain active for existing claims even if the gate is
later disabled, so removing an add-on controller at that point would strand
reservations.

## Compatibility

Existing VMs and standalone VMIs remain unchanged when `allocationPolicy` is
omitted. If the feature gate is disabled after opted-in VMs exist, running
VMIs may continue and guest-reboot, but KubeVirt does not create a replacement
VMI or new claim for them. Intentional restart is rejected; unexpected loss
leaves the VM Pending with a condition until the gate is re-enabled. Existing
VMs may remove or weaken a policy to release a claim; new opt-ins are rejected.
Stop, deletion, and orphan cleanup remain active so reservations can be
released. A downgrade to a KubeVirt version without reservation cleanup may
leave VM references that hold claims
and allocations indefinitely; the upgrade and rollback procedure must remove
those references before downgrading. The original VEP's top-level
`VirtualMachineSpec.resourceClaimTemplates` and `persistWhenStopped` are
replaced by the single-entry policy. Both VM-owned policies require a VM
reservation; claim ownership alone does not preserve the allocation.

## Functional Testing Approach

- Unit tests: policy validation; generated claim naming and owner UID checks;
  VMI claim rewrite; VM/Pod reservation reconciliation; stop policy; deletion
  and orphan cleanup; gate-disabled cleanup; claim-specific restart gating;
  paired restart requests; terminal `Once` and `RerunOnFailure` cleanup;
  and status conflict retries.
- Integration tests: template claim creation before VMI; same claim UID and
  allocation across `virtctl restart`; deletion on `WhileRunning` stop;
  retained claim and allocation on `Persistent` stop; stale condition blocks
  restart; source change is rejected until a `Persistent` claim is released;
  gate disablement leaves replacement Pending; unavailable node leaves
  replacement Pending; and VM deletion removes reservations and claims.
- End-to-end tests with a DRA driver: confirm physical device identity across
  launcher replacements under both policies and across stop/start only for
  `Persistent`. Confirm `WhileRunning` releases its allocation on stop. Exercise
  node loss, explicit release and recreation, rejected migration and
  evacuation, and controller restart. Measure the allocation-to-reservation
  interval.

## Graduation Requirements

### Alpha

- [ ] API and feature gate added without changing existing VMs or VMIs.
- [ ] Template-backed VM claims and generated-VMI rewrite implemented.
- [ ] Dedicated reservation controller and cleanup deployed with isolated RBAC.
- [ ] Intentional restart waits for `ReservationEstablished`.
- [ ] Unit and integration tests cover allocation identity, run strategies,
      and both stop modes.
- [ ] User documentation explains node pinning, reallocation, and the initial
  reservation gap.

### Beta

- [ ] End-to-end device identity tests run with a real DRA driver.
- [ ] Upgrade, rollback, controller restart, and orphan recovery are tested.
- [ ] The reservation gap and cleanup behavior meet reliability requirements.
- [ ] Scheduling and migration restrictions are documented in the user guide.

### GA

- [ ] API and lifecycle semantics are stable across multiple releases.
- [ ] No known allocation leaks or unintended device changes remain.

## References

- [VEP-344 issue](https://github.com/kubevirt/enhancements/issues/344)
- [VEP-10: DRA Devices](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/10-dra-devices)
- [Kubernetes DRA](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/)
- [Prior implementation PR](https://github.com/kubevirt/kubevirt/pull/17957)
- [GSoC 2024: Persistent Device Claims for KubeVirt](https://github.com/kubevirt/community/issues/254)
