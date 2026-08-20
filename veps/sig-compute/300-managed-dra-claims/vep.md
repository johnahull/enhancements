# VEP #300: Managed DRA Claims

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: v1.10
- This VEP targets beta for version: v1.11
- This VEP targets GA for version: TBD

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [ ] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## Overview

This proposal adds managed DRA claim generation to KubeVirt via a new
`ManagedClaimProvisioner` CRD. Admins create provisioner objects that
encode DeviceClass names, topology alignment policy, and the name of
the controller responsible for claim generation. Users reference a
provisioner by name in the VMI spec and declare their devices as they
do today. The provisioner controller generates the `ResourceClaim`
automatically.

The design extends the approach sketched in
[VEP-10 Appendix C](../10-dra-devices/vep.md#c-managed-resource-claims)
and incorporates feedback from the VEP-300 tracking issue discussion.

## Motivation

DRA claim authoring is complex. A user who wants a GPU and SR-IOV NIC
co-placed on the same PCIe root must:

1. Know the exact DeviceClass names deployed on their cluster
2. Know topology attribute names (`resource.kubernetes.io/pcieRoot`)
3. Author a `ResourceClaim` or `ResourceClaimTemplate` with per-device
   requests and `matchAttribute` constraints
4. Wire `claimName` and `requestName` references from the VMI spec into
   the claim

The VMI spec already says "I have a GPU and a NIC." The topology intent
("put them on the same PCIe root") should not require re-expressing that
in a separate Kubernetes object.

A `ManagedClaimProvisioner` closes this gap: the admin encodes the
infrastructure details once, and users express only their device intent.
The mechanics are described in the Design section.

## Goals

- Let users express topology-aligned multi-device claims without
  hand-authoring `ResourceClaim` objects
- Separate admin concerns (DeviceClass selection, topology policy) from
  user concerns (device declarations) via the provisioner CRD
- Support extensibility via pluggable provisioner controllers
- Reuse existing device declaration patterns (`gpus[]`, `hostDevices[]`,
  `networks[]`, `cpu.dra`) as the source of truth for claim generation

## Non Goals

- Replace explicit `resourceClaimName` or `resourceClaimTemplateName`
  references (these remain as power-user escape hatches)
- Define CPU consumption behavior (owned by VEP-152)
- Define guest NUMA topology construction (owned by VEP-115)
- Support live migration of DRA-backed devices

## Definition of Users

- **User:** a person who wants DRA devices in a VM without writing
  ResourceClaim YAML
- **Admin:** a person who deploys DRA drivers, creates DeviceClass
  resources, and creates `ManagedClaimProvisioner` objects
- **Developer:** a person building custom provisioner controllers or
  automation on top of KubeVirt DRA APIs

## User Stories

- As a user, I want to request a GPU and NIC co-placed on the same PCIe
  root without learning DRA claim syntax or knowing DeviceClass names
- As a user, I want to request GPUs, NICs, and CPUs on the same NUMA
  node with a single provisioner reference
- As an admin, I want to define DeviceClass mappings and topology policy
  once and have all users reference them by name
- As a developer, I want to implement a custom provisioner controller
  with my own claim generation logic

## Repos

[KubeVirt](https://github.com/kubevirt/kubevirt)

## Design

### Feature Gate

All changes are gated behind `ManagedDRAClaims` (alpha, off by default).

### Responsibility Boundary

- **User owns:** device declarations (`gpus[]`, `hostDevices[]`,
  `networks[]`, `cpu.dra`)
- **Admin owns:** `ManagedClaimProvisioner` objects (DeviceClass
  mappings, topology policy, provisioner controller selection)
- **Provisioner controller owns:** `ResourceClaim` generation, constraint
  assembly, claim lifecycle (creation, GC via owner references)
- **Scheduler owns:** device allocation, topology constraint satisfaction

### Scope Boundary with VEP-152

VEP-152 and VEP-300 are developed concurrently:

- **VEP-152 owns:** `cpu.dra` struct (`CPUDRASource`), `deviceClassName`
  on `CPUDRASource`, CPU consumption (virt-launcher vCPU pinning), CPU
  accounting formula, `CPUsWithDRA` feature gate, unified `dedicated` API
- **VEP-300 owns:** `ManagedClaimProvisioner` CRD, `managedClaimProvisioner`
  field on `VirtualMachineInstanceResourceClaim`, provisioner controller,
  `ManagedDRAClaims` feature gate

VEP-152's future `autoClaim` path for CPU-only claims is superseded by
VEP-300's managed claims, which handle CPUs as part of cross-device claims.

### Scope Boundary with VEP-183

`spec.networks[].resourceClaim` (the network device declaration) is owned
by VEP-183, which defines it as a `ClaimRequest` (`claimName`,
`requestName`, no `deviceClassName`). VEP-300 reads it as a claim source
during generation but does not define it.

### External Dependencies

- [KEP-6072](https://github.com/kubernetes/enhancements/issues/6072)
  (standard topology attributes): GA in Kubernetes 1.37. Standardizes
  `resource.kubernetes.io/numaNode`.
- [KEP-5491](https://github.com/kubernetes/enhancements/issues/5491)
  (list-typed attributes): alpha in Kubernetes 1.36. Required for
  `pcieRoot` alignment with CPUs. The CPU DRA driver publishes `pcieRoot`
  as a list (a CPU group is affine to multiple PCIe roots), while GPU/NIC
  drivers publish it as a scalar. KEP-5491 redefines `matchAttribute` as
  non-empty set intersection, enabling cross-device alignment. See
  [dra-driver-cpu#114](https://github.com/kubernetes-sigs/dra-driver-cpu/issues/114).
- [dra-driver-cpu](https://github.com/kubernetes-sigs/dra-driver-cpu)
  v0.2.0+: publishes `pcieRoot` list attribute in grouped mode (PR #163).

### API Changes

#### New CRD: `ManagedClaimProvisioner`

A cluster-scoped resource that encodes DeviceClass mappings, topology
alignment policy, and the provisioner controller name:

```yaml
apiVersion: kubevirt.io/v1alpha1
kind: ManagedClaimProvisioner
metadata:
  name: pcie-aligned
spec:
  # Names the controller that generates the ResourceClaim.
  # Built-in controller: policy.kubevirt.io/aligner
  # External controllers use their own provisioner name.
  provisioner: policy.kubevirt.io/aligner

  # Topology alignment policy for the built-in controller.
  # Values are shorthands (numaNode, pcieRoot) or fully-qualified
  # attribute names (see Shorthand Expansion).
  # +optional
  align: [pcieRoot]

  # DeviceClass mappings per device type.
  # One DeviceClass per type for Alpha.
  # +optional
  cpu:
    deviceClassName: cpu.dra.k8s.io
  # +optional
  gpus:
    deviceClassName: gpu.nvidia.com
  # +optional
  hostDevices:
    deviceClassName: pci.example.com
  # +optional
  networks:
    deviceClassName: sriov.mellanox.com

  # Opaque parameters passed to external provisioner controllers.
  # +optional
  parameters:
    customKey: customValue
```

#### New Type: `ManagedClaimProvisionerRef`

```go
// ManagedClaimProvisionerRef references a ManagedClaimProvisioner
// object by name.
type ManagedClaimProvisionerRef struct {
	// Name is the name of the ManagedClaimProvisioner object.
	// This field is immutable after creation.
	Name string `json:"name"`
}
```

#### Modified: `VirtualMachineInstanceResourceClaim`

Add `ManagedClaimProvisioner` as a third mutually-exclusive option:

```go
type VirtualMachineInstanceResourceClaim struct {
	// Name uniquely identifies this resource claim inside the VMI.
	Name string `json:"name"`

	// ResourceClaimName is the name of a ResourceClaim object in the
	// same namespace as this VMI.
	// Exactly one of ResourceClaimName, ResourceClaimTemplateName, or
	// ManagedClaimProvisioner must be set.
	ResourceClaimName *string `json:"resourceClaimName,omitempty"`

	// ResourceClaimTemplateName is the name of a ResourceClaimTemplate
	// object in the same namespace as this VMI.
	// Exactly one of ResourceClaimName, ResourceClaimTemplateName, or
	// ManagedClaimProvisioner must be set.
	ResourceClaimTemplateName *string `json:"resourceClaimTemplateName,omitempty"`

	// ManagedClaimProvisioner references a ManagedClaimProvisioner
	// object that controls how the ResourceClaim is generated.
	// Exactly one of ResourceClaimName, ResourceClaimTemplateName, or
	// ManagedClaimProvisioner must be set.
	// +optional
	ManagedClaimProvisioner *ManagedClaimProvisionerRef `json:"managedClaimProvisioner,omitempty"`
}
```

#### `GPU` and `HostDevice`

No new fields added to `GPU` or `HostDevice` by this VEP. The existing
structs are used as-is with their `ClaimRequest` fields (`claimName`,
`requestName`).

#### CPU (`CPUDRASource`, defined by VEP-152)

VEP-152 adds the following structure (shown here for reference):

```go
type CPU struct {
	// ... existing fields (Cores, Sockets, Threads, etc.) ...

	// DRA enables Dynamic Resource Allocation for CPU resources.
	// +optional
	DRA *CPUDRASource `json:"dra,omitempty"`
}

type CPUDRASource struct {
	// ClaimRequest references a specific request from a ResourceClaim
	// listed in vmi.spec.resourceClaims[].
	*ClaimRequest `json:",inline"`
}
```

VEP-300 scans `cpu.dra` during claim generation alongside the other
device types. The DeviceClassName is resolved from the provisioner
CRD's `cpu.deviceClassName` field. The CPU count in the generated claim
is derived from VEP-152's accounting formula; VEP-152 is the single
source of truth for that formula. See
[VEP-152 (PR #414)](https://github.com/kubevirt/enhancements/pull/414).

### DeviceClassName Resolution

DeviceClass names are defined in the `ManagedClaimProvisioner` CRD.
The controller determines device type by which VMI field the device
appears in:

- `domain.devices.gpus[]` → provisioner `spec.gpus.deviceClassName`
- `domain.devices.hostDevices[]` → provisioner `spec.hostDevices.deviceClassName`
- `spec.networks[].resourceClaim` → provisioner `spec.networks.deviceClassName`
- `domain.cpu.dra` → provisioner `spec.cpu.deviceClassName`

### Claim Generation Algorithm

```
GenerateManagedClaim(vmi, claimEntry, provisioner) → ResourceClaim:

  1. Collect device requests. For each device type, resolve the
     DeviceClassName per the DeviceClassName Resolution section and
     emit a DeviceRequest named after requestName:
     a. gpus[] where claimName == claimEntry.Name (Count=1).
     b. hostDevices[] where claimName == claimEntry.Name (Count=1).
     c. networks[] where resourceClaim.claimName == claimEntry.Name.
     d. cpu.dra (VEP-152) if claimName == claimEntry.Name. The CPU
        count comes from VEP-152's accounting formula. The claim shape
        (capacity vs count) is determined by the DeviceClass and driver
        mode, not by the user.

  2. Validate the collected requests. The full rules are enforced at
     admission (see the Validation section) and re-checked here.

  3. Build topology constraints from provisioner.spec.align. Expand
     each align value per the Shorthand Expansion section, then create
     one matchAttribute constraint per align value spanning all request
     names collected in step 1. Cross-device pcieRoot matching relies
     on KEP-5491 set intersection (see External Dependencies).

  4. Assemble the ResourceClaim:
     Name: <vmi-name>-<claim-name>
     Namespace: vmi.Namespace
     OwnerReference: VMI (controller=true, for GC)
     Labels: kubevirt.io/managed-claim: <claim-name>
     Spec.Devices.Requests: collected requests
     Spec.Devices.Constraints: collected constraints
```

### Where Generation Happens

Claim generation runs in **virt-controller** as a reconciliation loop.
The VMI is persisted with `managedClaimProvisioner` in the spec. The
controller creates ResourceClaims asynchronously, tracks them via
expectations, and waits for readiness before proceeding with pod
creation. This follows the same pattern KubeVirt uses for backend
storage PVCs (`pkg/storage/backend-storage/backend-storage.go`).

**VMI sync flow integration:**

Managed claim handling is inserted into the VMI sync loop
(`pkg/virt-controller/watch/vmi/lifecycle.go`) after topology hints
and before backend storage:

```
sync() → check deletionTimestamp → check dataVolumes →
  check topologyHints → handleManagedClaims() →
  wait claimExpectations → wait managedClaimsReady →
  handleBackendStorage() → wait backendStorageReady →
  RenderLaunchManifest() → createPod()
```

For each `spec.resourceClaims[]` entry with
`managedClaimProvisioner != nil`:

1. Read the referenced `ManagedClaimProvisioner` object.
2. If `provisioner` matches the built-in controller
   (`policy.kubevirt.io/aligner`):
   a. Call `GenerateManagedClaim` to build the `ResourceClaim`.
   b. Create the `ResourceClaim` in the API server with
      ownerReference to the VMI.
   c. Record the generated claim name in VMI status.
3. If `provisioner` names an external controller, virt-controller does
   not generate the claim. It watches for the externally created
   ResourceClaim instead, per the External Controller Contract section.
4. Once all managed claims are resolved (status shows all claims in the
   `Created` phase), proceed with pod creation.

### Status Tracking

A new `ManagedClaims` field on `VirtualMachineInstanceStatus` tracks
the lifecycle of each managed claim:

```go
type VirtualMachineInstanceStatus struct {
	// ... existing fields ...

	// ManagedClaims tracks the status of managed ResourceClaim
	// generation for each spec.resourceClaims[] entry that uses
	// managedClaimProvisioner.
	// +optional
	// +listType=map
	// +listMapKey=name
	ManagedClaims []ManagedClaimStatus `json:"managedClaims,omitempty"`
}

type ManagedClaimStatus struct {
	// Name matches the spec.resourceClaims[].name entry.
	Name string `json:"name"`

	// ResourceClaimName is the name of the generated ResourceClaim.
	ResourceClaimName string `json:"resourceClaimName,omitempty"`

	// ProvisionerName identifies which provisioner created the claim.
	ProvisionerName string `json:"provisionerName,omitempty"`

	// Phase indicates the current lifecycle phase.
	Phase ManagedClaimPhase `json:"phase"`

	// Message provides human-readable details about the current phase.
	// +optional
	Message string `json:"message,omitempty"`
}

type ManagedClaimPhase string

const (
	// ManagedClaimPending indicates the claim has not been created yet.
	ManagedClaimPending ManagedClaimPhase = "Pending"
	// ManagedClaimCreated indicates the ResourceClaim exists.
	ManagedClaimCreated ManagedClaimPhase = "Created"
	// ManagedClaimFailed indicates claim creation failed.
	ManagedClaimFailed  ManagedClaimPhase = "Failed"
)
```

The controller checks readiness before proceeding with pod creation:
all managed claims must be in the `Created` phase and the corresponding
ResourceClaim must exist with an ownerReference to the VMI.

### Expectations Pattern

The controller uses `UIDTrackingControllerExpectations` (the same
mechanism used for pod and PVC creation) to avoid reconciling before
the informer cache reflects newly created ResourceClaims:

- `claimExpectations.ExpectCreations(vmiKey, count)` before creating
  claims
- `claimExpectations.CreationObserved(vmiKey)` when the ResourceClaim
  informer sees the new object
- `claimExpectations.SatisfiedExpectations(vmiKey)` checked in the
  sync loop before proceeding

A ResourceClaim informer is added to the VMI controller, with event
handlers that observe creation expectations and enqueue the owning VMI
for reconciliation.

### External Controller Contract

When the provisioner's `provisioner` field does not match the built-in
controller, an external controller is responsible for creating the
ResourceClaim. This follows the CSI provisioner pattern: the external
controller creates the resource, and virt-controller observes it and
updates VMI status. Only virt-controller writes VMI status.

**What the external controller must do:**

1. Watch for VMIs where the referenced `ManagedClaimProvisioner` has a
   `provisioner` field matching its name
2. Create a `ResourceClaim` in the VMI's namespace
3. Set `ownerReference` on the claim pointing to the VMI with
   `controller: true` (required for GC)

The external controller does not write VMI status. It only creates the
ResourceClaim.

**What virt-controller does for external claims:**

1. Skips claim generation for entries where `provisioner` does not
   match the built-in controller
2. Watches for ResourceClaims owned by the VMI via the ResourceClaim
   informer
3. When a matching ResourceClaim appears, validates ownership and
   updates VMI `status.managedClaims[]` to phase `Created`
4. Proceeds with pod creation once all managed claims are resolved

**Timeout:** if an external controller does not create the claim within
5 minutes, virt-controller emits a `ManagedClaimTimeout` event on the
VMI explaining which provisioner has not responded. The VMI stays in a
pending state. It is not rejected, since the controller may start later.

**Conflict avoidance:** the built-in controller only processes claims
where `provisioner` matches `policy.kubevirt.io/aligner`. External
controllers must only process claims matching their own provisioner
name.

### Implementation Details

**Claim naming:** the generated claim is named
`<vmi-name>-<claim-name>`. If this exceeds the 253-character DNS subdomain
limit, the name is truncated and a short hash suffix is appended to
preserve uniqueness.

**Idempotency:** controller reconciliation is naturally idempotent. If
the ResourceClaim already exists with the correct ownerReference, the
controller skips creation and updates status to `Created`.

**RBAC:** virt-controller's ClusterRole is updated to grant `create`,
`get`, `list`, and `watch` permissions on `resourceclaims` in the
`resource.k8s.io` API group. Deletion of generated claims is handled by
owner-reference garbage collection, so `delete` is not required.
virt-controller already has broad permissions for pod and PVC management.

**Multiple managed claims per VMI:** a VMI can have multiple
`resourceClaims[]` entries, each independently using
`managedClaimProvisioner`, `resourceClaimName`, or
`resourceClaimTemplateName`. Each managed claim entry is reconciled
independently. Pod creation waits for all of them.

### Shorthand Expansion

| Shorthand  | Fully-Qualified Attribute                 |
| ---------- | ----------------------------------------- |
| `numaNode` | `resource.kubernetes.io/numaNode`         |
| `pcieRoot` | `resource.kubernetes.io/pcieRoot`         |

Fully-qualified names are passed through unchanged, enabling
forward-compatibility with future topology attributes.

### Validation

The validating webhook enforces:

1. **Mutual exclusion:** exactly one of `resourceClaimName`,
   `resourceClaimTemplateName`, or `managedClaimProvisioner` must be
   set per `resourceClaims[]` entry.
2. **Feature gate:** `ManagedDRAClaims` must be enabled when
   `managedClaimProvisioner` is used.
3. **Provisioner exists:** the referenced `ManagedClaimProvisioner`
   must exist at VMI creation time.
4. **Immutability:** `managedClaimProvisioner.name` cannot be changed
   after VMI creation.
5. **Device coverage:** at least one device declaration must reference
   each managed claim entry (no empty claims).
6. **DeviceClassName available:** the provisioner CRD must have a
   `deviceClassName` configured for every device type referenced by
   devices in the managed claim.
7. **Unique request names:** no duplicate `requestName` values within
   a managed claim.
8. **Valid align entries:** each `align` value must be a known shorthand
   or a valid fully-qualified attribute name.

### Error Handling

**Claim generation failure:** if the built-in controller cannot generate
a claim (e.g., no DeviceClassName resolvable, no devices reference the
claim), it sets the status phase to `Failed` with a descriptive message
and emits a `FailedCreateResourceClaim` event on the VMI. The VMI stays
pending; the controller retries on the next reconciliation.

**ResourceClaim deleted externally:** if a managed claim's ResourceClaim
is deleted while the VMI is running, the controller detects this via the
informer and re-creates the claim (built-in controller) or emits a
warning event (external controller).

**External controller timeout:** see the External Controller Contract
section for the 5-minute timeout behavior.

**Provisioner deleted or modified after VMI creation:** the provisioner
is validated at admission, but it is cluster-scoped and mutable. If the
controller cannot read the referenced provisioner at generation time
(deleted) or a required `deviceClassName` is missing (edited), it sets
the status phase to `Failed`, emits a `FailedCreateResourceClaim` event,
and retries. The VMI stays pending.

**VMI deletion during claim creation:** the controller checks
`vmi.DeletionTimestamp` before creating claims. If the VMI is being
deleted, the controller skips claim creation. Owner reference GC handles
cleanup of any already-created claims.

## API Examples

### Single GPU

Admin creates provisioner:

```yaml
apiVersion: kubevirt.io/v1alpha1
kind: ManagedClaimProvisioner
metadata:
  name: gpu-default
spec:
  provisioner: policy.kubevirt.io/aligner
  gpus:
    deviceClassName: gpu.nvidia.com
```

User creates VMI:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: gpu-vm
spec:
  resourceClaims:
  - name: my-gpu
    managedClaimProvisioner:
      name: gpu-default
  domain:
    devices:
      gpus:
      - name: gpu0
        claimName: my-gpu
        requestName: gpu
    resources:
      requests:
        memory: 8Gi
```

Generated `ResourceClaim`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-vm-my-gpu
  labels:
    kubevirt.io/managed-claim: my-gpu
  ownerReferences:
  - apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstance
    name: gpu-vm
    controller: true
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
```

### GPU + NIC Co-Placed on PCIe Root

Admin creates provisioner:

```yaml
apiVersion: kubevirt.io/v1alpha1
kind: ManagedClaimProvisioner
metadata:
  name: pcie-aligned
spec:
  provisioner: policy.kubevirt.io/aligner
  align: [pcieRoot]
  gpus:
    deviceClassName: gpu.nvidia.com
  networks:
    deviceClassName: sriov.mellanox.com
```

User creates VMI:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: gpu-nic-vm
spec:
  resourceClaims:
  - name: aligned-devices
    managedClaimProvisioner:
      name: pcie-aligned
  domain:
    devices:
      gpus:
      - name: gpu0
        claimName: aligned-devices
        requestName: gpu
      interfaces:
      - name: rdma-nic
        sriov: {}
    resources:
      requests:
        memory: 16Gi
  networks:
  - name: rdma-nic
    resourceClaim:
      claimName: aligned-devices
      requestName: nic
```

Generated `ResourceClaim`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-nic-vm-aligned-devices
  labels:
    kubevirt.io/managed-claim: aligned-devices
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
    - name: nic
      exactly:
        deviceClassName: sriov.mellanox.com
        count: 1
    constraints:
    - matchAttribute: resource.kubernetes.io/pcieRoot
      requests: [gpu, nic]
```

### Multi-GPU + NIC + CPU on Same NUMA Node and PCIe Root

Admin creates provisioner:

```yaml
apiVersion: kubevirt.io/v1alpha1
kind: ManagedClaimProvisioner
metadata:
  name: hgx-b200-quarter
spec:
  provisioner: policy.kubevirt.io/aligner
  align: [numaNode, pcieRoot]
  cpu:
    deviceClassName: cpu.dra.k8s.io
  gpus:
    deviceClassName: gpu.nvidia.com
  networks:
    deviceClassName: sriov.mellanox.com
```

User creates VMI:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: full-topology-vm
spec:
  resourceClaims:
  - name: all-devices
    managedClaimProvisioner:
      name: hgx-b200-quarter
  domain:
    cpu:
      cores: 16
      dra:
        claimName: all-devices
        requestName: cpus
    devices:
      gpus:
      - name: gpu0
        claimName: all-devices
        requestName: gpu0
      - name: gpu1
        claimName: all-devices
        requestName: gpu1
      interfaces:
      - name: rdma-nic
        sriov: {}
    resources:
      requests:
        memory: 64Gi
  networks:
  - name: rdma-nic
    resourceClaim:
      claimName: all-devices
      requestName: nic
```

Generated `ResourceClaim`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: full-topology-vm-all-devices
  labels:
    kubevirt.io/managed-claim: all-devices
spec:
  devices:
    requests:
    - name: gpu0
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
    - name: gpu1
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
    - name: nic
      exactly:
        deviceClassName: sriov.mellanox.com
        count: 1
    - name: cpus
      exactly:
        deviceClassName: cpu.dra.k8s.io
        count: 16
    constraints:
    - matchAttribute: resource.kubernetes.io/numaNode
      requests: [gpu0, gpu1, nic, cpus]
    - matchAttribute: resource.kubernetes.io/pcieRoot
      requests: [gpu0, gpu1, nic, cpus]
```

The `pcieRoot` constraint spans all four requests even though the CPU
group publishes `pcieRoot` as a list while the GPU and NIC publish it as
a scalar. See External Dependencies for the KEP-5491 set-intersection
semantics that make this match.

The `count: 16` on the CPU request assumes a driver operating in count
mode. A capacity-mode driver would express the same request differently
(see algorithm step 1d).

## Alternatives

### Alternative 1: Inline `managedClaim` on VMI spec

The initial design (from
[VEP-10 Appendix C](../10-dra-devices/vep.md#c-managed-resource-claims))
placed topology policy (`align`) and DeviceClass names directly on the
VMI spec via a `managedClaim` field on `resourceClaims[]` and
`deviceClassName` on each device declaration. Admin defaults were
configured in the KubeVirt CR (`managedClaimDefaults`).

Rejected because:
- DeviceClass names and topology policy are admin concerns, not user
  concerns. Putting them in the VMI leaks infrastructure details.
- No extensibility for external controllers without adding a
  `controllerName` field (which duplicates what the CRD's `provisioner`
  field does more cleanly).
- Admin defaults in the KubeVirt CR are a single global config. The
  CRD allows multiple provisioner profiles per cluster.

### Alternative 2: Webhook-based claim generation

The claim generation logic runs in a mutating admission webhook instead
of a controller. The webhook creates the ResourceClaim during VMI
admission and replaces `managedClaim` with `resourceClaimName` before
the VMI is persisted.

Rejected because:
- Creating Kubernetes objects during admission is an anti-pattern
  (blocks the request, can leak objects if a downstream webhook
  rejects the VMI).
- No retry on failure (webhook fails, VMI creation fails).
- Kubernetes made the same decision when implementing DRA extended
  resources in the scheduler instead of a webhook.

### Alternative 3: No KubeVirt involvement (external-only)

Users install an external controller (e.g., topology coordinator) that
watches VMIs and generates ResourceClaims independently. KubeVirt has
no managed claim API at all.

Rejected because:
- No standard contract between KubeVirt and external controllers
  (how does virt-controller know when to proceed with pod creation?).
- Every external implementation reinvents the same integration
  (status tracking, expectations, ownership).
- KubeVirt should provide a built-in default implementation while
  allowing external extensibility via the `provisioner` field.

## Scalability

Managed claim generation adds one `ResourceClaim` CREATE per managed
claim entry during VMI reconciliation. This follows the existing DRA
scalability model. See
[VEP-10 Scalability](../10-dra-devices/vep.md#scalability).

## Update/Rollback Compatibility

- API changes are additive and gated by `ManagedDRAClaims`. With the
  gate disabled, existing behavior is unchanged.
- Rollback: disable the feature gate and delete VMIs using managed
  claims before downgrading.
- Generated `ResourceClaim` objects are standard Kubernetes resources
  and are cleaned up by owner-reference GC.

## Functional Testing Approach

- Unit tests for `GenerateManagedClaim`: single device, multi-device,
  topology alignment, shorthand expansion, error cases
- Unit tests for validation: mutual exclusion, missing provisioner,
  invalid align values, empty claims, duplicate request names,
  provisioner existence
- Integration tests with fake ManagedClaimProvisioner objects (envtest)
- E2E: VMI with managed claim for GPU (requires DRA driver in CI)

## Graduation Requirements

### Alpha

- `ManagedClaimProvisioner` CRD (cluster-scoped)
- `managedClaimProvisioner` field on `VirtualMachineInstanceResourceClaim`
- Built-in provisioner controller (`policy.kubevirt.io/aligner`)
- API changes behind `ManagedDRAClaims` feature gate (off by default)
- GPU, HostDevice, Network, and CPU support
- Topology alignment with `align: [numaNode]` and `align: [pcieRoot]`
- Controller-based claim generation with status tracking
- External controller extensibility via `provisioner` field
- Validation
- Unit tests and mock e2e tests
- Requires KEP-6072 (GA in Kubernetes 1.37)
- `pcieRoot` alignment with CPUs requires KEP-5491 (alpha in Kubernetes
  1.36)

### Beta

- Feature gate on by default
- User documentation
- Real-driver e2e testing

### GA

- Upgrade/downgrade testing
- Scale testing on large nodes

## Future Extensions

- **Partition mode:** provisioner defines device counts and resources,
  user references provisioner name without declaring devices
- **Multiple DeviceClasses per device type:** array of DeviceClass
  entries with names, devices reference by name
- **Memory and hugepages:** when the CPU DRA driver adds memory
  allocation support, memory can participate in managed claims
- **Preference-based alignment:** `prefer` vs `require` enforcement
  for topology constraints

## Implementation History

- 2026-08-05: Initial VEP draft based on VEP-10 Appendix C design
- 2026-08-11: Redesigned with ManagedClaimProvisioner CRD based on
  feedback from Alay Patel (VEP owner)
- 2026-08-12: Added controller-based generation, external controller
  contract, status tracking
- 2026-08-13: Aligned CPUDRASource with VEP-152

## References

- [VEP-10: Support GPUs with DRA](../10-dra-devices/vep.md) (Appendix C:
  Managed Resource Claims)
- [VEP-115: PCIe NUMA Topology Awareness](../115-pcie-numa-topology-awareness/vep.md)
- [VEP-152: Support CPUs with DRA (PR #414)](https://github.com/kubevirt/enhancements/pull/414)
- [VEP-183: DRA for Network Devices](../../sig-network/183-dra-network/vep.md)
- [KEP-6072: Standard Topology Attributes](https://github.com/kubernetes/enhancements/issues/6072)
- [KEP-5491: List Types for Attributes](https://github.com/kubernetes/enhancements/issues/5491)
- [KEP-5304: DRA Device Attributes Downward API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5304-dra-attributes-downward-api)
- [dra-driver-cpu#114: NIC/CPU alignment](https://github.com/kubernetes-sigs/dra-driver-cpu/issues/114)
