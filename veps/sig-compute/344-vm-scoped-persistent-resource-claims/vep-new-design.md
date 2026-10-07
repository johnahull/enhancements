# VEP #344: VM-Scoped Persistent ResourceClaims for DRA Device Passthrough

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: v1.10.0
- This VEP targets beta for version: TBD
- This VEP targets GA for version: TBD

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting a milestone or release*.

- [x] (R) Enhancement issue created, which links to the VEP directory in
  [kubevirt/enhancements](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/344-vm-scoped-persistent-resource-claims)
- [ ] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Overview

This VEP defines VM-scoped persistence for Kubernetes Dynamic Resource
Allocation (DRA) ResourceClaims used by KubeVirt.

A user declares a direct claim reference once in the `VirtualMachine`
template, using the existing `resourceClaimName` field, and sets an
`allocationPolicy` on that entry. A new, dedicated reservation controller adds
the VM as a consumer in `ResourceClaim.status.reservedFor` according to that
policy. The VM reservation is retained across VMI and launcher Pod
replacement.

In alpha, KubeVirt creates, names, owns, and deletes no claim objects. It
writes exactly one field — the consumer reference — on claims the user already
owns. ResourceClaimTemplate-backed claims and VEP-300 managed claims extend
this same mechanism at beta, once claim creation and ownership are added; this
document describes their shape so the alpha mechanism is designed to extend
cleanly into them, but they are not part of the alpha surface.

Standalone VMIs retain their existing VMI-scoped behavior.

## Motivation

DRA claims are normally consumed by Pods. KubeVirt creates and deletes
launcher Pods as VMIs are started, stopped, restarted, or recovered. If a
claim is reserved only for the launcher Pod, deleting that Pod can release the
allocation before the replacement Pod is scheduled.

Releasing and reallocating the device can cause:

1. A VM to receive a different GPU, host device, or network device after a
   restart.
2. Device-specific state or data to become unavailable.
3. Network identity, such as a MAC address or DHCP reservation, to change.
4. A restart to fail because another workload acquired the device during the
   allocation gap.

The VM, rather than the launcher Pod, is the logical owner of a device attached
to the VM. The VM therefore needs to be represented as a DRA claim consumer.

## Goals

- Retain VM-scoped DRA allocations across VMI and launcher Pod replacement.
- Require users to declare a VM claim only once in the VM template.
- Allow users to choose an allocation policy that controls whether an
  allocation is retained across an explicit VM stop.
- Preserve existing standalone VMI claim behavior exactly, as the default
  policy.
- Isolate the privilege required to modify `status.reservedFor` in a single,
  narrowly-scoped component rather than granting it to virt-controller.
- Prevent KubeVirt from modifying or deleting user-owned direct claims.
- Clean up KubeVirt-owned reservations during VM deletion.
- Design the mechanism so that ResourceClaimTemplate-backed and VEP-300
  managed claims can adopt it at beta without a second persistence model.

## Non Goals

- Changing Kubernetes DRA allocation or scheduling behavior.
- Supporting live migration of DRA-backed devices.
- Preserving VM schedulability to other nodes while a non-`Ephemeral`
  allocation is retained. A retained allocation is node-specific; see
  [Scheduling and Migration Constraints](#scheduling-and-migration-constraints).
- Eagerly allocating claims for halted VMs.
- Creating, naming, or owning ResourceClaim objects in alpha. This is
  explicitly deferred to beta; see
  [ResourceClaimTemplates (Beta)](#resourceclaimtemplates-beta) and
  [Managed Claims (Beta)](#managed-claims-beta).
- Defining device-specific request semantics owned by VEP-152, VEP-183, or
  other device-specific proposals.
- Changing the managed claim generation model defined by VEP-300 beyond the
  additions required for VM scope.
- Mutating an allocated `ResourceClaim.spec` in place.
- Defining a general-purpose shared-claim policy for multiple VMs.

## Definition of Users

- **VM user:** A person who declares devices in a VirtualMachine template.
- **VMI user:** A person who creates and manages a standalone
  VirtualMachineInstance.
- **Cluster administrator:** A person who installs DRA drivers, configures
  ResourceClaimTemplates, and grants RBAC to the reservation controller.
- **Provisioner author:** A person who implements a VEP-300 managed claim
  provisioner (beta).

## User Stories

- As a VM user, I want a GPU allocation to survive VM restart so that the VM
  receives the same device after its launcher Pod is recreated.
- As a VM user, I want to declare a direct claim in one place and choose
  whether it is released on explicit stop.
- As a VM user, I want to release a device on explicit stop when resource
  efficiency is more important than device stability.
- As a VM user, I want to retain a device on explicit stop when the device has
  persistent state or identity requirements, understanding that doing so
  pins the VM to the node where the device was allocated.
- As a VM user, I want to force a fresh allocation for a stopped VM when I
  need to replace failed hardware or change placement.
- As a cluster administrator, I want the privilege to modify claim
  reservations confined to one small, auditable component rather than spread
  across virt-controller.
- As a provisioner author, I want the managed claim framework to identify
  whether a claim belongs to a VM or a standalone VMI, once VEP-300 adopts
  this mechanism at beta.
- As a KubeVirt developer, I want the generated VMI to be an implementation
  detail rather than a second user-facing claim configuration surface.

## Repos

- `kubevirt/enhancements` — this VEP and the VEP-300 design contract.
- `kubevirt/kubevirt` — API types, validation, the reservation controller,
  virt-controller rendering changes, RBAC, and tests.

## Design

### Design Principle

Claim provisioning and claim persistence are separate concerns:

```text
VM template claim entry (direct reference, alpha)
        |
        v
DRA allocates the claim for the launcher Pod
        |
        v
reservation controller maintains status.reservedFor
```

In alpha this is literally all there is: KubeVirt does not create, name, own,
or delete any claim object. It resolves the user's existing
`resourceClaimName` reference and maintains one status field. At beta, when
template-backed and managed claims are added, a claim source resolver is
introduced ahead of this pipeline to produce the concrete claim — but the
reservation controller downstream of it does not change. There must not be one
persistence implementation for direct claims and another for generated claims.

### Feature Gate

All VM-scoped claim behavior is gated behind `PersistentDRAClaims` during
alpha. The feature gate controls the new API validation, the reservation
controller's reconciliation, and VM finalizer behavior.

### API Changes

The existing `VirtualMachineInstanceResourceClaim` entry is extended with
`allocationPolicy`:

```go
type ResourceClaimAllocationPolicy string

const (
	// ResourceClaimAllocationPolicyEphemeral is the default. The VM is
	// never added to the claim's reservedFor. Behavior is identical to
	// today's VMI-scoped claims: the allocation is released when the
	// launcher Pod is deleted, for any reason.
	ResourceClaimAllocationPolicyEphemeral ResourceClaimAllocationPolicy = "Ephemeral"

	// ResourceClaimAllocationPolicyWhileRunning adds the VM to reservedFor
	// once the claim is allocated. The allocation is retained across VMI
	// and launcher Pod replacement (reboot, crash recovery). On an
	// explicit VM stop, the VM reservation is removed and the allocation
	// may be released.
	ResourceClaimAllocationPolicyWhileRunning ResourceClaimAllocationPolicy = "WhileRunning"

	// ResourceClaimAllocationPolicyPersistent behaves like WhileRunning,
	// except the VM reservation is also retained across an explicit VM
	// stop.
	ResourceClaimAllocationPolicyPersistent ResourceClaimAllocationPolicy = "Persistent"
)

type VirtualMachineInstanceResourceClaim struct {
	// Name uniquely identifies this claim inside the VM or VMI.
	Name string `json:"name"`

	// Exactly one of ResourceClaimName or ResourceClaimTemplateName must
	// be set. (ManagedClaimProvisionerName is added at beta, see VEP-300.)
	ResourceClaimName *string `json:"resourceClaimName,omitempty"`

	ResourceClaimTemplateName *string `json:"resourceClaimTemplateName,omitempty"`

	// AllocationPolicy controls whether the VM reservation controller adds
	// the VM as a consumer of this claim, and whether that reservation
	// survives an explicit VM stop. It has no effect on a standalone VMI,
	// where only the default, Ephemeral, is accepted: a standalone VMI has
	// no VM object to act as the reservation's consumer reference.
	//
	// Defaults to Ephemeral, so enabling the feature gate does not change
	// the behavior of any existing VM or VMI.
	// +optional
	// +kubebuilder:validation:Enum=Ephemeral;WhileRunning;Persistent
	AllocationPolicy *ResourceClaimAllocationPolicy `json:"allocationPolicy,omitempty"`
}
```

The following table defines each value precisely, in terms of
`status.reservedFor`:

| `allocationPolicy` | VM added to `reservedFor`? | VMI / launcher Pod replacement | Explicit VM stop |
|---|---|---|---|
| `Ephemeral` (default) | never | allocation released with the Pod, as today | released |
| `WhileRunning` | yes, once allocated | allocation retained | VM reservation removed; DRA may deallocate |
| `Persistent` | yes, once allocated | allocation retained | VM reservation retained |

`Ephemeral` is specified as exactly today's VMI-scoped behavior, so that it is
a true no-op: the reservation controller never touches a claim whose entry has
the default policy. This is deliberate — the alternative of defaulting to
`WhileRunning` would make the motivating restart-stability guarantee the
default behavior, but it would also mean a controller bug on day one of the
feature gate could change the behavior of VMs that never opted in. See
[Alternatives](#alternatives) for the beta default discussion.

The VM user declares the entry once in:

```text
VirtualMachine.spec.template.spec.resourceClaims
```

The VMI generated from the template contains a copy of the claim entry,
including `allocationPolicy`, but the VM template is authoritative; users do
not maintain a second claim entry on the VMI.

The `VirtualMachineSpec.resourceClaimTemplates` API proposed by the initial
VEP-344 design is not used by this design.

### Scope Determination

Scope is inferred from the object containing the declaration:

| Declaration location | Scope | Lifecycle owner |
|---|---|---|
| `VirtualMachine.spec.template.spec.resourceClaims` | VM | VirtualMachine |
| `VirtualMachineInstance.spec.resourceClaims` | VMI | VirtualMachineInstance |

No `scope` field is required. A VM-generated VMI's copied entry is recognized
as belonging to a VM-managed VMI (see
[Reservation Controller](#reservation-controller)) and is not treated as an
independent VMI-scoped declaration.

### Alpha: Direct Claims Only

Alpha supports exactly one claim source: `resourceClaimName`, a claim the user
already created. For a direct claim:

- The reservation controller resolves `resourceClaimName`.
- The claim must be in the VM's namespace.
- KubeVirt never changes the claim spec.
- KubeVirt never deletes the claim.
- KubeVirt adds and removes only its VM consumer reference.

The user remains responsible for creating and deleting the direct claim. If
the claim is already reserved for an incompatible consumer, the VM reports a
reservation conflict and does not start.

No claim is created, named, or owned by KubeVirt in alpha. The remainder of
this section describes the beta extensions; they are included here so the
alpha reservation mechanism — described in
[Reservation Controller](#reservation-controller) — is designed to extend to
them without modification.

### ResourceClaimTemplates (Beta)

For a VM-scoped `resourceClaimTemplateName`:

1. A claim source resolver reads the ResourceClaimTemplate.
2. It creates one concrete ResourceClaim for the VM.
3. The concrete claim receives a controller owner reference to the VM.
4. The launcher Pod references the concrete claim name.
5. The reservation controller manages the VM reservation exactly as it does
   for a direct claim — the mechanism from alpha is unchanged.

The launcher Pod must not reference the ResourceClaimTemplate directly. This
prevents the claim from being tied to the lifetime of an individual launcher
Pod. Kubernetes ResourceClaimTemplate-generated claims are normally associated
with the consuming Pod; VM scope requires a stable direct ResourceClaim.

The generated claim is created lazily when the VM starts. It is retained as an
object for the VM lifetime, but its allocation may be released when
`allocationPolicy` is `Ephemeral` or `WhileRunning`. This makes the policy
about allocation retention, not about whether a generated object exists.

**Generated VMI entry rewrite.** virt-launcher resolves device metadata by
reading the VMI's own claim entry (VEP-10): a `resourceClaimName` entry
resolves to `resourceclaims/<claim-name>/<request>/`, while a
`resourceClaimTemplateName` entry resolves to
`resourceclaimtemplates/<podClaimName>/<request>/`. Because the launcher Pod
always references a concrete claim name (step 4 above), the VMI entry
generated from the VM template is rewritten from
`resourceClaimTemplateName` to the concrete `resourceClaimName` before the VMI
is created. If this rewrite is skipped, virt-launcher looks for metadata under
the template path, finds none, and the device never reaches the libvirt
domain. This rewrite is required for beta template support; it has no alpha
analog, since alpha has no template-backed source to rewrite.

### Managed Claims (Beta)

VEP-300 ([kubevirt/enhancements#432](https://github.com/kubevirt/enhancements/pull/432),
open, unmerged) is extended to support managed claims originating from a VM
template. This extension is sequenced after VEP-300 merges; the API deltas
below belong in that proposal and are shown here only to describe the shape
the reservation mechanism must accommodate.

For a VM-scoped managed claim:

- the managed claim provisioner watches the VirtualMachine;
- device declarations are read from `spec.template.spec`;
- the generated ResourceClaim is owned by the VM;
- the generated ResourceClaim name is stable across VMI replacement;
- the managed claim provisioner creates and maintains the claim spec;
- virt-controller renders the generated claim name into the launcher Pod;
- the reservation controller maintains `status.reservedFor`, unchanged from
  alpha.

Virt-controller does not invoke the provisioner and does not generate the
managed ResourceClaim spec.

VEP-300's managed claim context would need to identify the owning workload:

```go
type ManagedClaimContext struct {
	VM          *v1.VirtualMachine
	VMI         *v1.VirtualMachineInstance
	Claim       *v1.VirtualMachineInstanceResourceClaim
	Provisioner *v1alpha1.ManagedClaimProvisioner
	Devices     ManagedClaimDevices
}
```

For a VM-scoped claim, `VM` is populated and the VM template is authoritative.
For a standalone VMI claim, `VMI` is populated and the existing VEP-300
behavior is preserved.

The managed claim framework must watch both VirtualMachines and standalone
VMIs. It must skip the VMI provisioning path for a VMI generated from a VM
template, preventing duplicate claims.

VEP-300 should add a capability declaration to `ManagedClaimProvisioner` so a
VM-scoped request can fail early when the provisioner does not support it:

```go
type ManagedClaimProvisionerSpec struct {
	Provisioner    string                   `json:"provisioner"`
	DeviceTypes    []ManagedClaimDeviceType `json:"deviceTypes"`
	SupportedScopes []ResourceClaimScope    `json:"supportedScopes,omitempty"`
}
```

When `supportedScopes` is omitted, the provisioner supports only VMI scope for
backward compatibility. Beta also requires enabling both `ManagedDRAClaims`
(VEP-300) and `PersistentDRAClaims` (this VEP) together for VM-scoped managed
claims; neither gate alone is sufficient.

### Claim Naming and Ownership (Beta)

This section applies only to the beta ResourceClaimTemplates and Managed
Claims sources; alpha creates no claim objects.

Direct claims retain their user-provided name.

Generated VM-scoped claims use a deterministic name based on the VM name and
logical claim name, combined with a content hash to avoid collisions — not
merely to shorten names that exceed the Kubernetes DNS length limit. A naive
`<vm-name>-<claim-name>` scheme collides: VM `foo` with claim `bar-baz`, and VM
`foo-bar` with claim `baz`, both produce `foo-bar-baz`. The naming function
must therefore incorporate a hash of the (VM name, claim name) pair, not just
truncate when the result is too long.

The owner UID must be included in labels and verified before an existing
generated claim is adopted. If a generated claim with the expected name exists
but carries a different owner UID, the controller must not adopt or overwrite
it; it must report a naming-collision condition and event instead.

VM-scoped generated claims have a controller owner reference to the VM. They
also carry labels identifying the logical claim and source:

```yaml
kubevirt.io/resource-claim: <claim-name>
kubevirt.io/resource-claim-scope: VM
kubevirt.io/resource-claim-vm: <vm-name>
kubevirt.io/resource-claim-source: template|managed
```

The managed claim provisioner and virt-controller must use the same naming
function.

### Launcher Pod References

For VM-scoped claims, the launcher Pod always references a concrete
`resourceClaimName`. In alpha this is simply the user's own claim name. At
beta, for template-backed and managed claims, it is the generated claim's
name. The Pod does not reference a `resourceClaimTemplateName`, and it does
not need to know whether the concrete claim was direct, template-backed, or
managed.

### Reservation Controller

A dedicated reservation controller — not virt-controller — owns all
`status.reservedFor` writes. It is deployed and managed by virt-operator as
its own component, with its own ServiceAccount, and participates in leader
election like the other virt-* controllers.

This separation exists because writing `status.reservedFor` or
`status.allocation` requires the synthetic `resourceclaims/binding`
authorization, which Kubernetes checks cluster-wide — the check carries no
namespace or claim-name scoping (see
[Security Considerations](#security-considerations)). Confining that grant to
a single, small, auditable controller whose entire reconcile loop is "add or
remove one `reservedFor` entry" is materially safer than granting it to
virt-controller, which has a much larger attack surface.

The controller:

- watches ResourceClaims and VirtualMachines (and, at beta,
  VirtualMachineInstances for the rewritten template references);
- for every VM-scoped claim entry with a non-`Ephemeral` `allocationPolicy`,
  adds the VM reservation once the claim is allocated;
- retains the reservation across VMI and launcher Pod replacement;
- preserves Pod and other consumer references already on the claim;
- removes the VM reservation on explicit stop when `allocationPolicy` is
  `WhileRunning`, and on VM deletion regardless of policy;
- updates the claim with conflict retries;
- removes only the VM's own reservation, never another consumer's;
- owns a dedicated VM finalizer (for example,
  `dra.kubevirt.io/reservation-protection`) so that virt-controller requires
  no coordination logic for VM deletion.

virt-controller's only remaining responsibility for VM-scoped claims is
resolving the claim name (direct in alpha; generated at beta) and rendering it
into the launcher Pod spec — work it already performs for VMI-scoped claims
today.

**Controller availability.** If the reservation controller is unavailable:

- VM start still succeeds — the Kubernetes scheduler allocates the claim and
  adds the launcher Pod to `reservedFor` on its own, independent of this
  controller.
- The VM reservation is installed late, once the controller resumes,
  widening the window described in
  [Allocation-to-Reservation Race](#allocation-to-reservation-race). A VM
  stopped or its Pod replaced during this window does not get the retention
  guarantee for that cycle.
- VM deletion blocks on the reservation-protection finalizer until the
  controller resumes and removes the VM's reservations. This is bounded by
  controller restart time under normal operation; see
  [Reservation Leak and Recovery](#reservation-leak-and-recovery) for the
  case where the controller never resumes.

### Allocation-to-Reservation Race

There is a small unavoidable race between DRA allocation and the first VM
reservation update, because Kubernetes rejects a reservation write against an
unallocated claim — this is enforced by the Kubernetes API server, not merely
a convention the controller must follow. The reservation controller must
minimize this window and ensure the VM reservation is installed before
deleting a launcher Pod for an intentional stop. It must report reservation
failures and retry them.

**Unplanned Pod loss.** The mitigation above only covers an intentional stop.
If the launcher Pod is lost before the reservation is installed — node
failure, OOM-kill, eviction, or any other unplanned deletion — the allocation
is released with the Pod regardless of `allocationPolicy`, and restart may
receive a different device. This is a known alpha limitation, not a design
gap the mechanism claims to close: it is the same gap described in Motivation
item 4, narrowed to the specific window between allocation and the first
reservation write. The window is expected to be short (one reconcile cycle)
but is not zero. Tests must cover and report its observed duration.

### Explicit Stop

When a VM is explicitly stopped:

- `allocationPolicy: Persistent`: retain the VM reservation and allocation.
- `allocationPolicy: WhileRunning`: remove the VM reservation.
- `allocationPolicy: Ephemeral` (default): no VM reservation exists to
  remove; behavior is unchanged from today.

When the VM reservation is removed and no other reservation remains, the DRA
controller may deallocate the claim. A direct claim remains entirely
user-owned and is never deleted by KubeVirt.

### Scheduling and Migration Constraints

A retained allocation (`WhileRunning` or `Persistent`) is node-specific: once
a claim's `status.allocation.nodeSelector` is set, Kubernetes restricts any
future consumer of that claim — including a replacement launcher Pod — to the
nodes matching that selector. This has two consequences, and the first is
easy to overlook:

1. **Scheduling is pinned, independent of migration.** While a non-`Ephemeral`
   allocation is retained, the VM cannot be scheduled to any other node, even
   via a normal stop/start, until the allocation is released. If the original
   node is drained, cordoned, or becomes unavailable, a VM with
   `allocationPolicy: Persistent` cannot start elsewhere unless the user frees
   the claim (see [Manual Reallocation](#manual-reallocation)). This is a
   direct tradeoff against the restart-stability goal and must be weighed by
   the user choosing a non-`Ephemeral` policy.
2. **Live migration is not supported.** VM-scoped DRA allocations are not
   live-migratable by this VEP. VMs using a non-`Ephemeral` VM-scoped DRA
   claim must be marked as not live-migratable unless a future DRA-aware
   migration design (see VEP-109 for the analogous vGPU case) provides
   equivalent guarantees.

### Manual Reallocation

A user who wants a different physical device for a stopped VM — for example,
to work around failed hardware, or to change node placement — deletes the
user-owned claim directly and starts the VM again:

```text
1. Ensure the VM is stopped.
2. kubectl delete resourceclaim <claim-name>
3. virtctl start <vm-name>
```

Because alpha claims are always user-owned direct claims, this requires no
controller-side support beyond the ordinary reservation reconciliation: once
the claim is gone, there is no reservation to preserve, and the next start
resolves a fresh (or newly user-created) claim as normal. Deleting the claim
while the VM is running has no effect until the next replacement of the
launcher Pod, since the running Pod already holds the device.

A single-step API (for example, a `reallocateOnNextStart` field) is a
candidate for beta once generated claims exist, where the controller — not
the user — would own the delete step. For alpha, the manual approach is
sufficient; reallocation is an exceptional operation.

### VM Deletion

KubeVirt adds a VM finalizer, owned by the reservation controller, while
VM-scoped claim reservations exist.

During VM deletion:

1. The reservation controller removes the VM from every VM-scoped claim's
   `reservedFor`.
2. It retries until the reservation is removed or records a failure (see
   [Reservation Leak and Recovery](#reservation-leak-and-recovery) for what
   happens if it cannot).
3. The VM finalizer is removed.
4. For beta-only generated claims, Kubernetes garbage collection removes the
   KubeVirt-owned claim via its owner reference. Alpha has no generated
   claims to collect.

KubeVirt never deletes direct claims. It only removes the VM reservation from
them.

### Reservation Leak and Recovery

The mechanism this VEP relies on — that Kubernetes never prunes a
`reservedFor` entry it does not recognize (see
[Security Considerations](#security-considerations) for the citation) — has a
failure-mode cost that must be designed for, not merely disclaimed. If a VM's
reservation is never removed, two things become permanently true:

- the device remains allocated and cannot be used by any other workload;
- the ResourceClaim object itself cannot be deleted, because Kubernetes only
  clears the builtin allocation finalizer once `reservedFor` is empty.

This can happen when:

- the reservation controller fails repeatedly while processing VM deletion
  and exhausts retries without operator intervention;
- a VM is force-deleted (`kubectl delete vm --force --grace-period=0`),
  which removes the VM object before any finalizer-driven cleanup completes;
- a cluster administrator manually clears the
  `dra.kubevirt.io/reservation-protection` finalizer to unblock a deletion;
- the cluster is downgraded to a KubeVirt build that predates this VEP and
  does not know to remove VM reservations.

**Detection.** The reservation controller must expose a metric counting
claims with a VM reservation whose owning VM no longer exists (that is, a
dangling consumer reference it can detect by looking up the VM by name/UID
and finding it gone or recreated with a different UID), and must set a
condition on the VM while a reservation removal is outstanding and retrying.

**Recovery.** A cluster administrator can manually remove the dangling entry
from `status.reservedFor` using the same `resourceclaims/binding`
authorization the controller itself requires (see
[Security Considerations](#security-considerations)); this is a documented,
supported operator procedure, not an unsupported workaround. The alpha
documentation must include this procedure explicitly, including the RBAC an
administrator needs to perform it.

**Downgrade.** A controller version that does not understand VM reservations
must not attempt to delete or mutate a ResourceClaim that still carries one;
doing so could violate the reservation the newer controller relied on. The
safe downgrade path is: before downgrading, ensure no VM-scoped claim has a
pending non-`Ephemeral` reservation (stop any such VMs with
`allocationPolicy: WhileRunning`, or accept that `Persistent` reservations
remain and will not be understood by the older controller, which will simply
leave them alone, matching its existing behavior with unrecognized consumer
references).

### ResourceClaim Spec Immutability

Kubernetes defines `ResourceClaim.spec` as immutable. KubeVirt must not update
the spec of a direct claim or an allocated generated claim in place.

At beta, if a VM template change would change the desired spec of a generated
claim, the controller must:

- detect the difference;
- report a clear condition and event;
- replace the claim only after it is no longer allocated or reserved; or
- require an explicit reallocation operation (see
  [Manual Reallocation](#manual-reallocation)).

Changing only `allocationPolicy` does not change the ResourceClaim spec, in
either alpha or beta.

### Relationship to VEP-10

[VEP-10](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/10-dra-devices)
removed virt-controller's ResourceClaim and ResourceSlice informers in v1.8,
moving device-attribute discovery to CDI-mounted metadata files read directly
by virt-launcher, specifically to avoid virt-controller needing to watch DRA
objects.

This VEP reinstates a ResourceClaim watch — but in the new, dedicated
reservation controller described in
[Reservation Controller](#reservation-controller), not in virt-controller.
The two changes are not in tension: VEP-10 removed the watch because
device-attribute reporting no longer needed it, not because watching
ResourceClaims is inherently undesirable. Reservation reconciliation
genuinely needs live claim state — specifically, whether a claim is allocated
yet, which cannot be known from CDI metadata alone — so a watch is
unavoidable here. Isolating it in its own controller also keeps it out of
virt-controller's already-large reconcile surface.

The VMI-entry rewrite required for beta ResourceClaimTemplate support (see
[ResourceClaimTemplates (Beta)](#resourceclaimtemplates-beta)) also interacts
directly with VEP-10's virt-launcher metadata-path resolution; that
interaction is described there rather than repeated here.

### Validation

The validating webhook enforces, in alpha:

1. `resourceClaimName` must be set (it is the only supported source in
   alpha).
2. `allocationPolicy`, if set, must be one of `Ephemeral`, `WhileRunning`, or
   `Persistent`.
3. `allocationPolicy` other than `Ephemeral` is only accepted for a claim
   declared in a VM template; a standalone VMI may not request
   `WhileRunning` or `Persistent`, because it has no VM object to serve as
   the reservation's consumer reference.
4. The referenced claim is in the VM's namespace.
5. Claim names are unique within the VM template.
6. A VM-generated VMI does not independently change the VM-scoped claim
   source or allocation policy.

Additional rules apply once ResourceClaimTemplates and managed claims are
added at beta: mutual exclusion across the three source fields, template/
provisioner namespace checks, managed claim provisioner existence and scope
support, and the requirement that every claim entry be referenced by at
least one device declaration.

### RBAC

The reservation controller's ServiceAccount requires permission to:

- get, list, and watch ResourceClaims;
- update the `resourceclaims/status` subresource;
- update the `resourceclaims/binding` subresource — both are required on
  Kubernetes 1.36+, where `DRAResourceClaimGranularStatusAuthorization` is
  beta and enabled by default: that feature gate enforces a dedicated
  authorization check against the synthetic `resourceclaims/binding`
  subresource specifically for writes to `status.allocation` and
  `status.reservedFor`, in addition to the ordinary `resourceclaims/status`
  update permission. Clusters on an older minor version only need
  `resourceclaims/status` update;
- get, list, and watch VirtualMachines, and update them for its own
  finalizer;
- emit events.

This is the entire RBAC footprint for the reservation controller, and it is
not granted to virt-controller. virt-controller's existing RBAC is
unchanged by this VEP; it only needs to read the resolved claim name to
render it into the launcher Pod, which does not require any new permission.

At beta, the reservation controller additionally needs create and delete
permission on generated ResourceClaims (for template-backed and managed
sources), and get/list/watch on ResourceClaimTemplates. Managed claim
provisioners require the permissions defined by VEP-300.

### Security Considerations

Writing `status.reservedFor` (and `status.allocation`) requires the synthetic
`resourceclaims/binding` authorization check that Kubernetes performs on any
update to those fields. This check is performed cluster-wide: it does not
scope to a namespace or a specific claim name, so a grant of
`resourceclaims/binding: update` authorizes modifying `status.allocation` and
`status.reservedFor` on *every* ResourceClaim in the cluster, not only
VM-scoped ones. In upstream Kubernetes, exactly two identities hold this
permission by default: kube-scheduler and the resource-claim-controller
component of kube-controller-manager.

Granting this permission to virt-controller — which has a comparatively large
reconcile surface and RBAC footprint already — would mean that any
vulnerability in virt-controller could be used to release or re-target any
workload's DRA device, not only KubeVirt's own. This VEP instead isolates the
grant: it is held only by the dedicated reservation controller described in
[Reservation Controller](#reservation-controller), whose entire reconcile
loop is adding or removing a single `reservedFor` entry per VM-scoped claim.
This directly narrows the blast radius of the new privilege to one small,
auditable component.

Namespace RBAC remains the boundary for who may create VMs and who may
reference an existing claim from one (claim creation and claim reference are
unaffected by this VEP's new privilege). It is not the boundary for who may
modify `status.reservedFor` — that authority is held by the reservation
controller's ServiceAccount alone, cluster-wide, as described above.

Cross-namespace claim references are rejected.

### Kubernetes DRA Scheduler and Kubelet

Kubernetes remains responsible for:

- allocating ResourceClaims;
- adding and removing Pod consumer references;
- enforcing reservation limits;
- making allocated devices available to the launcher Pod.

## API Examples

### Direct Claim, Persistent

The user creates the claim separately:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-for-vm
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: example.com/gpu
        count: 1
```

The VM references it once:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: gpu-vm
spec:
  template:
    spec:
      resourceClaims:
      - name: gpu
        resourceClaimName: gpu-for-vm
        allocationPolicy: Persistent
      domain:
        devices:
          gpus:
          - name: gpu0
            claimName: gpu
            requestName: gpu
```

### Direct Claim, Default (Ephemeral)

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: gpu-vm-ephemeral
spec:
  template:
    spec:
      resourceClaims:
      - name: gpu
        resourceClaimName: gpu-for-vm
        # allocationPolicy omitted: defaults to Ephemeral, today's behavior
      domain:
        devices:
          gpus:
          - name: gpu0
            claimName: gpu
            requestName: gpu
```

### ResourceClaimTemplate (Beta)

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: gpu-vm
spec:
  template:
    spec:
      resourceClaims:
      - name: gpu
        resourceClaimTemplateName: gpu-template
        allocationPolicy: Persistent
      domain:
        devices:
          gpus:
          - name: gpu0
            claimName: gpu
            requestName: gpu
```

KubeVirt creates a VM-owned concrete ResourceClaim from `gpu-template` and
places that concrete claim name into the launcher Pod.

### Managed Claim (Beta)

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: aligned-gpu-vm
spec:
  template:
    spec:
      resourceClaims:
      - name: aligned-devices
        managedClaimProvisionerName: gpu-default
        allocationPolicy: Persistent
      domain:
        devices:
          gpus:
          - name: gpu0
            claimName: aligned-devices
            requestName: gpu
```

The VEP-300 provisioner creates the VM-owned concrete claim. The user does not
add a second entry to the VMI.

### Standalone VMI

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: standalone-vmi
spec:
  resourceClaims:
  - name: gpu
    resourceClaimName: gpu-for-vmi
  domain:
    devices:
      gpus:
      - name: gpu0
        claimName: gpu
        requestName: gpu
```

This remains VMI-scoped and is not affected by VM-scoped persistence. Setting
`allocationPolicy` to anything other than `Ephemeral` here is rejected by the
webhook.

## Alternatives

### persistWhenStopped Boolean

An earlier revision of this design used a single boolean,
`persistWhenStopped`, rather than the three-valued `allocationPolicy`.

Rejected in favor of the enum because a boolean cannot express the
distinction between "retain across restart, release on stop" and "retain
across restart and stop" without an implicit, undocumented default for the
restart case. The enum makes all three states explicit and gives a name
(`Ephemeral`) to the no-op default, which a boolean's `nil`/`false` state does
not.

### WhileRunning or Persistent as the Beta Default

`Ephemeral` is the alpha default specifically because it is a guaranteed
no-op: enabling `PersistentDRAClaims` cannot change the behavior of any
existing VM. The counter-argument is that restart stability is the entire
motivation for this VEP, so defaulting to the inert value means most users
must discover and set the field explicitly to get the benefit.

This is left for beta: once the mechanism has operated in alpha clusters and
the leak-detection and recovery tooling in
[Reservation Leak and Recovery](#reservation-leak-and-recovery) has been
exercised, the SIG should revisit whether `WhileRunning` becomes the default
for claims declared in a VM template. Changing the default after alpha is a
behavior change and must go through the normal graduation review, not be
decided unilaterally in this document.

### Separate VM ResourceClaimTemplate List

Add a second VM-level list that declares claims to create, while also adding a
matching claim reference to the VMI template.

Rejected because it requires duplicate user configuration and creates two
sources of truth. The existing claim entry in the VM template already provides
the logical name, source, and lifecycle policy.

### Let Each Claim Type Implement Persistence

Rejected because reservation updates, conflict handling, VM deletion cleanup,
and finalizers would be duplicated across multiple controllers. This is the
core justification for the [Design Principle](#design-principle): a single
reservation controller must apply the same lifecycle rules to every
VM-scoped claim, regardless of source.

### Keep ResourceClaimTemplate References in the Launcher Pod

Rejected because a Pod-generated template claim is tied to Pod lifetime. A
VM-scoped template claim must first become a stable direct ResourceClaim.

### Let Virt-controller Generate Managed Claims

Rejected because VEP-300 assigns claim generation and provisioner-specific
device policy to managed claim provisioners. Virt-controller should only
resolve and render the generated claim name.

### Use an Anchor Pod

Rejected because an anchor Pod adds a second workload, complicates scheduling,
and makes VM deletion and failure handling harder. The VM consumer reference is
the Kubernetes-native mechanism for retaining the allocation.

### Grant `resourceclaims/binding` to virt-controller

Considered and rejected; see
[Reservation Controller](#reservation-controller) and
[Security Considerations](#security-considerations). Confining the
cluster-wide authorization to one small, single-purpose controller is a
materially smaller attack surface than adding it to virt-controller's
existing, much broader RBAC footprint.

## Does it belong to core KubeVirt?

The reservation mechanism depends on two things only core KubeVirt can
provide: a VM finalizer that blocks VM deletion until claim reservations are
cleaned up, and identity of which Pod is the current launcher Pod for a given
VMI, which the reservation controller needs in order to distinguish "pod
replaced, VM reservation should persist" from "VM deleted, VM reservation
should be removed." Both require privileged knowledge of VM/VMI/Pod lifecycle
that an external controller would otherwise have to reconstruct by watching
the same objects virt-controller already watches, duplicating that logic
outside the project.

An external controller (for example, as a VEP-190 structured plugin) was
considered given the narrow, auditable scope of the reservation controller
described above. It was not chosen for the initial proposal because the VM
finalizer must be coordinated with virt-controller's own VM finalizers to
avoid races during VM deletion, which is simpler to get right as a
first-party component with direct access to the same VM reconcile dependency
graph. If the reservation controller's interface proves stable, revisiting it
as an externally pluggable component remains an option for a future VEP.

## Scalability

- Each VM contributes at most one reservation per VM-scoped claim.
- `ResourceClaim.status.reservedFor` has a Kubernetes-defined maximum of 256
  entries (`ResourceClaimReservedForMaxSize`). A VM normally contributes one
  entry, so this design does not introduce a per-restart accumulation of
  consumers.
- In alpha, the number of claims KubeVirt reconciles is exactly the number of
  direct claim entries across all VM templates; no new objects are created.
- Reconciliation must be idempotent and use live reads or conflict retries for
  ResourceClaim status updates.
- VMs without VM-scoped claims, and claims using the default `Ephemeral`
  policy, incur no additional claim-status updates.
- No anchor Pods or polling loops are required.

## Update/Rollback Compatibility

- The feature is gated behind `PersistentDRAClaims` and is disabled by
  default during alpha.
- Existing standalone VMI claims are unchanged.
- Existing direct VM claim references remain valid; they gain VM reservation
  behavior only when `allocationPolicy` is explicitly set to `WhileRunning`
  or `Persistent` in a VM template with the feature gate enabled. The
  default, `Ephemeral`, changes nothing.
- The initial VEP-344 `VirtualMachineSpec.resourceClaimTemplates` API and the
  intermediate `persistWhenStopped` field are replaced by the
  `allocationPolicy` API in this design. Because the feature is alpha and not
  a released stable API, these changes are intentionally breaking before
  implementation is finalized.
- Downgrade safety and the reservation-leak recovery procedure are described
  together in [Reservation Leak and Recovery](#reservation-leak-and-recovery);
  that section is the authoritative statement of this requirement, not a
  restatement of it.
- Changes to an allocated ResourceClaim spec require explicit reallocation;
  they are not silently applied during a VM update.

## Functional Testing Approach

### Unit Tests

- Validate that `resourceClaimName` is the only accepted source in alpha.
- Validate that `allocationPolicy` other than `Ephemeral` is rejected for
  standalone VMIs.
- Validate that `allocationPolicy` defaults to `Ephemeral` and that an
  `Ephemeral` entry never causes a reservation-controller write.
- Validate that VM-template claims are not processed as independent VMI
  claims.
- Validate reservation merge behavior and conflict retries.
- Validate that direct claims are never deleted or spec-mutated.
- Validate reservation-controller RBAC is exactly the set described in
  [RBAC](#rbac) — no broader grant.

### Integration Tests

- Start a VM with a direct claim and `allocationPolicy: Persistent`, and
  verify the VM appears in `reservedFor`.
- Recreate the VMI and verify that the same claim and allocation are reused.
- Stop with `allocationPolicy: WhileRunning` and verify that the VM
  reservation is removed.
- Stop with `allocationPolicy: Persistent` and verify that the VM reservation
  remains.
- Verify that a VM with the default `Ephemeral` policy behaves identically to
  a VM with no `allocationPolicy`-aware controller present at all.
- Delete the VM and verify that its reservation is removed before the
  reservation-protection finalizer is removed.
- Verify that direct claims survive VM deletion.
- Simulate reservation-controller unavailability during VM deletion and
  verify the VM blocks on its finalizer, then completes once the controller
  resumes (see [Reservation Controller](#reservation-controller)).
- Verify the manual reallocation sequence in
  [Manual Reallocation](#manual-reallocation) end-to-end.

### End-to-End Tests

With a real DRA driver:

- allocate a GPU or host device to a VM with `allocationPolicy: Persistent`;
- restart the VM repeatedly;
- verify that the claim remains allocated and the device identity is stable;
- stop and restart with all three allocation policies;
- verify that KubeVirt's webhook, not an upstream guarantee, is what prevents
  a second VM from independently requesting the same retained claim — a
  plain Pod or another workload may still reserve the same claim if nothing
  in KubeVirt prevents it, since Kubernetes itself permits sharing a claim
  across up to 256 consumers;
- verify that deletion releases the device.

Tests must also cover the allocation-to-reservation race described in
[Allocation-to-Reservation Race](#allocation-to-reservation-race), including
measuring its observed duration, and verify that reservation failures are
surfaced and retried.

## Implementation History

- 2026-09-29: Drafted the revised design as an alternative to the initial
  VEP-344 API and implementation, introducing `status.reservedFor` as the
  persistence mechanism.
- 2026-10-07: Revised again to scope alpha to direct claims only, replace
  `persistWhenStopped` with the `allocationPolicy` enum, and move
  `status.reservedFor` management to a dedicated reservation controller.
- Hardware validation (prior `resourceClaimTemplates`-based prototype): Dell
  XE9680 with 4x Samsung PM1745 NVMe drives, 4 VMs each holding a unique NVMe
  device across 10 stop/start cycles and 10 reboot cycles (80/80 total, zero
  device swaps). This validated the underlying `reservedFor`-retention
  mechanism that this design also relies on; it predates the controller
  split and `allocationPolicy` API described here.
- Prior art: [GSoC 2024: Persistent Device Claims for KubeVirt](https://github.com/kubevirt/community/issues/254),
  proposed by Alice Frosi, Victor Toso de Carvalho, and Luboslav Pivarc, which
  first identified the device-allocation-loss problem and proposed DRA
  ResourceClaims as the persistence mechanism.
- Earlier implementation reference (initial `resourceClaimTemplates` design,
  superseded by this document):
  [kubevirt/kubevirt#17957](https://github.com/kubevirt/kubevirt/pull/17957).

## Graduation Requirements

### Alpha

- [ ] Feature gated behind `PersistentDRAClaims`.
- [ ] `allocationPolicy` field added to `VirtualMachineInstanceResourceClaim`,
  defaulting to `Ephemeral`.
- [ ] Direct VM-scoped claims supported; no claim objects created, named, or
  owned by KubeVirt.
- [ ] Dedicated reservation controller deployed with the minimal RBAC
  described in [RBAC](#rbac), separate from virt-controller's RBAC.
- [ ] VM reservation retained across VMI and launcher Pod replacement.
- [ ] Explicit stop behavior implemented for `WhileRunning` and `Persistent`.
- [ ] VM deletion removes VM reservations before the reservation-protection
  finalizer is removed.
- [ ] Reservation-leak detection metric and VM condition implemented.
- [ ] Manual reallocation procedure documented and tested.
- [ ] Standalone VMI behavior remains unchanged.
- [ ] Unit and integration tests cover direct claims and all three
  allocation policies.
- [ ] API examples and user documentation are published, including the
  scheduling-pinning consequence of `WhileRunning`/`Persistent`.

### Beta

- [ ] ResourceClaimTemplate-backed VM-scoped claims supported, including the
  generated-VMI-entry rewrite described in
  [ResourceClaimTemplates (Beta)](#resourceclaimtemplates-beta).
- [ ] VEP-300 VM-scoped managed claim integration is implemented, sequenced
  after VEP-300 merges.
- [ ] Managed claim provisioner capability handling is implemented.
- [ ] Deterministic, collision-resistant generated claim naming is
  implemented and tested.
- [ ] End-to-end tests cover direct, template-backed, and managed claims.
- [ ] Allocation-to-reservation race behavior is tested and its duration
  documented.
- [ ] Upgrade and rollback behavior, including the downgrade procedure in
  [Reservation Leak and Recovery](#reservation-leak-and-recovery), is tested.
- [ ] Feedback from alpha users is incorporated, including a decision on the
  beta default for `allocationPolicy` (see
  [Alternatives](#whilerunning-or-persistent-as-the-beta-default)).
- [ ] Scheduling and live migration restrictions are documented in the user
  guide.

#### On-By-Default Readiness

- [ ] No known data-loss or allocation-leak bugs remain; the leak-recovery
  procedure has been exercised against a real leak at least once.
- [ ] VM deletion reliably removes VM reservations.
- [ ] RBAC is validated against supported Kubernetes versions, including both
  the pre- and post-1.36 `DRAResourceClaimGranularStatusAuthorization`
  behavior.
- [ ] ResourceClaim status update conflicts are retried correctly.
- [ ] The API and claim lifecycle have remained stable across at least one
  release.

### GA

- [ ] Stable API and lifecycle semantics across multiple releases.
- [ ] Direct, template-backed, and managed claim paths are supported.
- [ ] Upgrade, downgrade, and controller restart behavior is validated.
- [ ] No outstanding reservation leaks or unintended claim deletions remain.
- [ ] User documentation covers allocation retention, explicit stop behavior,
  scheduling pinning, and live migration restrictions.

## References

- [VEP-300: Managed DRA Claims](https://github.com/kubevirt/enhancements/pull/432)
- [VEP-183: DRA Network Devices](https://github.com/kubevirt/enhancements/tree/main/veps/sig-network/183-dra-network)
- [VEP-152: CPU DRA](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/152-cpu-dra)
- [VEP-109: vGPU Live Migration](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/109-vgpu-live-migration)
- [VEP-10: DRA Devices](https://github.com/kubevirt/enhancements/tree/main/veps/sig-compute/10-dra-devices)
- [Kubernetes DRA API Objects](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/dra-api/)
- [Kubernetes ResourceClaim API](https://kubernetes.io/docs/reference/kubernetes-api/resource/resource-claim-v1/)
- [GSoC 2024: Persistent Device Claims for KubeVirt](https://github.com/kubevirt/community/issues/254)
