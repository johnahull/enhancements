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

This proposal adds managed DRA claim generation to KubeVirt. Instead of
hand-authoring `ResourceClaim` or `ResourceClaimTemplate` objects with
vendor-specific DeviceClass names and `matchAttribute` constraints, users
declare devices in the VMI spec as they do today and add a `managedClaim`
entry. KubeVirt assembles the `ResourceClaim` automatically, including
topology alignment constraints.

The design extends the approach sketched in
[VEP-10 Appendix C](../10-dra-devices/vep.md#c-managed-resource-claims):
the VMI spec already carries device declarations (GPUs under
`domain.devices.gpus[]`, host devices under `domain.devices.hostDevices[]`,
networks under `spec.networks[]`, and CPUs under `domain.cpu.dra` via
[VEP-152](../152-cpu-dra/vep.md)). Managed claims read those declarations and
produce the `ResourceClaim` the user would otherwise write by hand.

## Motivation

DRA claim authoring is complex. A user who wants a GPU and SR-IOV NIC
co-placed on the same PCIe root must:

1. Know the exact DeviceClass names deployed on their cluster
2. Know attribute names (`resource.kubernetes.io/pcieRoot`)
3. Author a `ResourceClaim` or `ResourceClaimTemplate` with per-device
   requests and `matchAttribute` constraints
4. Wire `claimName` and `requestName` references from the VMI spec into
   the claim

The VMI spec already says "I have a GPU and a NIC." The topology intent
("put them on the same PCIe root") should not require re-expressing that
in a separate Kubernetes object.

VEP-152's `cpu.dra` field adds CPUs to the same pattern. With managed
claims, a single `ResourceClaim` can co-place GPUs, NICs, and CPUs on the
same NUMA node — the most common topology request for AI/HPC workloads.

## Goals

- Let users express topology-aligned multi-device claims without
  hand-authoring `ResourceClaim` objects
- Reuse existing device declaration patterns (`gpus[]`, `hostDevices[]`,
  `networks[]`, `cpu.dra`) as the source of truth for claim generation
- Support topology alignment via `matchAttribute` constraints using
  shorthand attribute names

## Non Goals

- Replace explicit `resourceClaimName` or `resourceClaimTemplateName`
  references (these remain as power-user escape hatches)
- Define CPU consumption behavior (owned by VEP-152)
- Define guest NUMA topology construction (owned by VEP-115)
- Support live migration of DRA-backed devices

## Definition of Users

- **User:** a person who wants DRA devices in a VM without writing
  ResourceClaim YAML
- **Admin:** a person who deploys DRA drivers and creates DeviceClass
  resources
- **Developer:** a person building automation on top of KubeVirt DRA APIs

## User Stories

- As a user, I want to request a GPU and NIC co-placed on the same PCIe
  root without learning DRA claim syntax
- As a user, I want to request GPUs, NICs, and CPUs on the same NUMA node
  with a single topology declaration
- As an admin, I want to configure default DeviceClasses so users don't
  need to know driver-specific class names
- As an admin, I want users to be able to override defaults when they need
  a specific DeviceClass

## Repos

[KubeVirt](https://github.com/kubevirt/kubevirt)

## Design

### Feature Gate

All changes are gated behind `ManagedDRAClaims` (alpha, off by default).

### Responsibility Boundary

- **User owns:** device declarations (`gpus[]`, `hostDevices[]`,
  `networks[]`, `cpu.dra`), topology alignment intent (`align`),
  optional `deviceClassName` override per device
- **KubeVirt owns:** `ResourceClaim` generation, constraint assembly,
  claim lifecycle (creation, GC via owner references), DeviceClass
  resolution (user override → admin default → reject)
- **Admin owns:** DRA driver deployment, `DeviceClass` definitions,
  default DeviceClass configuration in the KubeVirt CR
- **Scheduler owns:** device allocation, topology constraint satisfaction

### Scope Boundary with VEP-152

VEP-152 and VEP-300 are developed concurrently:

- **VEP-152 owns:** `cpu.dra` struct (`CPUDRASource`), `deviceClassName`
  on `CPUDRASource`, CPU consumption (virt-launcher vCPU pinning), CPU
  accounting formula, `CPUsWithDRA` feature gate, unified `dedicated` API
- **VEP-300 owns:** `managedClaim` on `spec.resourceClaims[]`, claim
  generation algorithm (scanning all device types including `cpu.dra`),
  topology policies (`align`), `deviceClassName` on `GPU`, `HostDevice`,
  and `NetworkClaimRequest`, `ManagedDRAClaims` feature gate

VEP-152's future `autoClaim` path for CPU-only claims is superseded by
VEP-300's managed claims, which handle CPUs as part of cross-device claims.

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

### Default DeviceClass Configuration

Admins configure default DeviceClasses in the KubeVirt CR so users don't
need to know driver-specific class names:

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
spec:
  configuration:
    managedClaimDefaults:
      gpuDeviceClassName: gpu.nvidia.com
      networkDeviceClassName: sriov.mellanox.com
      cpuDeviceClassName: cpu.dra.k8s.io
      hostDeviceClassName: ""
```

**Resolution order** for each device in a managed claim:

1. User specifies `deviceClassName` on the device declaration → use it
2. User omits `deviceClassName` → use the admin default from the KubeVirt
   CR for that device type
3. Neither is set → reject the VMI with an error explaining that no
   DeviceClass is available for the device type

This means on a cluster where the admin has configured defaults, the
simplest possible managed claim VMI is:

```yaml
spec:
  resourceClaims:
  - name: my-gpu
    managedClaim: {}
  domain:
    devices:
      gpus:
      - name: gpu0
        claimName: my-gpu
        requestName: gpu
```

No `deviceClassName`, no topology attributes, no ResourceClaim authoring.

### API Changes

#### New Type: `ManagedClaim`

```go
// ManagedClaim configures KubeVirt to generate and own a ResourceClaim
// from device declarations in the VMI spec. Devices reference this claim
// via their claimName field; KubeVirt reads their deviceClassName and
// requestName to assemble the claim's device requests and topology
// constraints.
type ManagedClaim struct {
	// Align specifies topology attributes for device co-placement.
	// Each entry becomes a matchAttribute constraint spanning all
	// device requests in the generated claim.
	//
	// Shorthand names are expanded to fully-qualified attributes:
	//   numaNode  → resource.kubernetes.io/numaNode
	//   pcieRoot  → resource.kubernetes.io/pcieRoot
	//
	// Fully-qualified attribute names are passed through unchanged.
	//
	// When omitted or empty, no topology constraints are applied.
	// +optional
	// +listType=atomic
	Align []string `json:"align,omitempty"`
}
```

#### Modified: `VirtualMachineInstanceResourceClaim`

Add `ManagedClaim` as a third mutually-exclusive option alongside
`ResourceClaimName` and `ResourceClaimTemplateName`:

```go
type VirtualMachineInstanceResourceClaim struct {
	// Name uniquely identifies this resource claim inside the VMI.
	Name string `json:"name"`

	// ResourceClaimName is the name of a ResourceClaim object in the
	// same namespace as this VMI.
	// Exactly one of ResourceClaimName, ResourceClaimTemplateName, or
	// ManagedClaim must be set.
	ResourceClaimName *string `json:"resourceClaimName,omitempty"`

	// ResourceClaimTemplateName is the name of a ResourceClaimTemplate
	// object in the same namespace as this VMI.
	// Exactly one of ResourceClaimName, ResourceClaimTemplateName, or
	// ManagedClaim must be set.
	ResourceClaimTemplateName *string `json:"resourceClaimTemplateName,omitempty"`

	// ManagedClaim configures KubeVirt to automatically generate and
	// manage a ResourceClaim for this VMI. The claim is assembled from
	// device declarations (gpus[], hostDevices[], networks[], cpu.dra)
	// that reference this entry by claimName. The generated claim is
	// owned by the VMI and deleted when the VMI is deleted.
	// Exactly one of ResourceClaimName, ResourceClaimTemplateName, or
	// ManagedClaim must be set.
	// +optional
	ManagedClaim *ManagedClaim `json:"managedClaim,omitempty"`
}
```

#### Modified: `GPU`

Add `DeviceClassName`:

```go
type GPU struct {
	Name              string       `json:"name"`
	DeviceName        string       `json:"deviceName,omitempty"`
	*ClaimRequest     `json:",inline"`
	VirtualGPUOptions *VGPUOptions `json:"virtualGPUOptions,omitempty"`
	Tag               string       `json:"tag,omitempty"`

	// DeviceClassName specifies the DRA DeviceClass for this GPU.
	// When omitted and the referenced claim uses managedClaim, the
	// default from KubeVirt CR managedClaimDefaults.gpuDeviceClassName
	// is used. Ignored when using resourceClaimName or
	// resourceClaimTemplateName.
	// +optional
	DeviceClassName string `json:"deviceClassName,omitempty"`

	// Count specifies how many GPUs of this type to request.
	// When set, the webhook expands this entry into Count individual
	// GPU entries named <name>-0 through <name>-<count-1>, each with
	// requestName <requestName>-0 through <requestName>-<count-1>.
	// Defaults to 1 when omitted.
	// +optional
	Count int `json:"count,omitempty"`
}
```

#### Modified: `HostDevice`

Add `DeviceClassName`:

```go
type HostDevice struct {
	Name          string `json:"name"`
	DeviceName    string `json:"deviceName,omitempty"`
	*ClaimRequest `json:",inline"`
	Tag           string `json:"tag,omitempty"`

	// DeviceClassName specifies the DRA DeviceClass for this host device.
	// When omitted and the referenced claim uses managedClaim, the
	// default from KubeVirt CR managedClaimDefaults.hostDeviceClassName
	// is used. Ignored when using resourceClaimName or
	// resourceClaimTemplateName.
	// +optional
	DeviceClassName string `json:"deviceClassName,omitempty"`

	// Count specifies how many host devices of this type to request.
	// When set, the webhook expands this entry into Count individual
	// entries named <name>-0 through <name>-<count-1>.
	// Defaults to 1 when omitted.
	// +optional
	Count int `json:"count,omitempty"`
}
```

#### Modified: `NetworkSource`

Replace `ResourceClaim *ClaimRequest` with a new type that adds
`DeviceClassName`:

```go
// NetworkClaimRequest extends ClaimRequest with a DeviceClassName field
// for managed claim generation.
type NetworkClaimRequest struct {
	ClaimRequest `json:",inline"`

	// DeviceClassName specifies the DRA DeviceClass for this network device.
	// When omitted and the referenced claim uses managedClaim, the
	// default from KubeVirt CR managedClaimDefaults.networkDeviceClassName
	// is used. Ignored when using resourceClaimName or
	// resourceClaimTemplateName.
	// +optional
	DeviceClassName string `json:"deviceClassName,omitempty"`

	// Count specifies how many network devices of this type to request.
	// When set, the webhook expands this entry into Count individual
	// entries named <name>-0 through <name>-<count-1>.
	// Defaults to 1 when omitted.
	// +optional
	Count int `json:"count,omitempty"`
}

type NetworkSource struct {
	Pod           *PodNetwork          `json:"pod,omitempty"`
	Multus        *MultusNetwork       `json:"multus,omitempty"`
	ResourceClaim *NetworkClaimRequest `json:"resourceClaim,omitempty"`
}
```

#### CPU (`CPUDRASource`, defined by VEP-152)

VEP-152 adds `DeviceClassName` to `CPUDRASource`. VEP-300 scans it during
claim generation alongside the other device types.

### Claim Generation Algorithm

KubeVirt generates a `ResourceClaim` from the VMI spec by scanning device
declarations that reference a managed claim entry:

```
GenerateManagedClaim(vmi, claimEntry) → ResourceClaim:

  1. Expand device counts:
     - For each GPU, HostDevice, or network device with
       count > 1, expand into individual entries: name-0
       through name-(count-1), requestName-0 through
       requestName-(count-1). Each inherits deviceClassName
       from the original entry.

  2. Collect device requests:
     a. Scan domain.devices.gpus[] (after expansion) — for each
        GPU where claimName == claimEntry.Name, resolve
        DeviceClassName (device field → CR default → error),
        create a DeviceRequest with Name=requestName,
        DeviceClassName=resolved, Count=1.
     b. Scan domain.devices.hostDevices[] — same pattern.
     c. Scan spec.networks[] — for each network where
        resourceClaim.claimName == claimEntry.Name, resolve
        DeviceClassName, create a DeviceRequest with
        Name=requestName, DeviceClassName=resolved, Count=1.
     d. Scan domain.cpu.dra (VEP-152) — if claimName matches,
        resolve DeviceClassName, create a DeviceRequest with
        Name=requestName, DeviceClassName=resolved, CPU count
        derived from VEP-152's accounting formula
        (cores × sockets × threads + emulator + IOThreads).
        The claim shape (capacity vs count) is determined by
        the DeviceClass and driver mode, not by the user.

  3. Validate:
     - At least one device must reference the claim.
     - Every referencing device must have a resolved DeviceClassName
       (either from the device field or from CR defaults).
     - No duplicate requestName values.

  4. Build topology constraints from managedClaim.align:
     - For each align entry, expand shorthands:
         numaNode → resource.kubernetes.io/numaNode
         pcieRoot → resource.kubernetes.io/pcieRoot
     - Create a matchAttribute constraint spanning ALL request
       names collected in step 1.
     - All device types (GPU, NIC, CPU) publish both numaNode and
       pcieRoot. CPUs publish pcieRoot as a list attribute
       (KEP-5491); the scheduler's set-intersection semantics
       handle list-vs-scalar matching transparently.

  5. Assemble ResourceClaim:
     - Name: <vmi-name>-<claim-name>
     - Namespace: vmi.Namespace
     - OwnerReference: VMI (controller=true, for GC)
     - Labels: kubevirt.io/managed-claim: <claim-name>
     - Spec.Devices.Requests: collected requests
     - Spec.Devices.Constraints: collected constraints
```

### Where Generation Happens

Claim generation runs in the **VMI mutating webhook**
(`pkg/virt-api/webhooks/mutating-webhook/mutators/vmi-mutator.go`), in
`ApplyNewVMIMutations`, after defaults are applied.

For each `spec.resourceClaims[]` entry with `managedClaim != nil`:

1. Call `GenerateManagedClaim` to build the `ResourceClaim`.
2. Create the `ResourceClaim` in the Kubernetes API server.
3. Replace `managedClaim` with `resourceClaimName` pointing to the
   generated claim.

After mutation, the persisted VMI contains only `resourceClaimName`
references — virt-controller and virt-launcher never see `managedClaim`.
The entire DRA pipeline (scheduler allocation, driver preparation,
virt-launcher metadata reading) works unchanged.

**Why webhook, not virt-controller:**

- Synchronous error reporting — the user sees admission errors immediately
- No reconciliation loop — the claim exists before the VMI is persisted
- Simpler lifecycle — no controller state to manage

### Implementation Details

**Claim naming:** the generated claim is named
`<vmi-name>-<claim-name>`. If this exceeds the 253-character DNS subdomain
limit, the name is truncated and a short hash suffix is appended to
preserve uniqueness.

**Idempotency:** if the webhook is retried (e.g., due to a transient
error), the second `CREATE` call may return `AlreadyExists`. The webhook
handles this by verifying the existing claim's owner reference matches the
VMI. If it does, the webhook proceeds; if it doesn't, the VMI is rejected.

**RBAC:** the virt-api ClusterRole must be updated to grant `create` and
`delete` permissions on `resourceclaims` in the `resource.k8s.io` API
group. `delete` is needed for cleanup if claim creation succeeds but the
VMI admission is later rejected by the validating webhook.

**Multiple managed claims per VMI:** a VMI can have multiple
`resourceClaims[]` entries, each independently using `managedClaim`,
`resourceClaimName`, or `resourceClaimTemplateName`. For example, one
managed claim for GPUs aligned on `pcieRoot` and a separate managed claim
for CPUs with no alignment. Each generates its own `ResourceClaim`.

### Shorthand Expansion

| Shorthand  | Fully-Qualified Attribute                 |
| ---------- | ----------------------------------------- |
| `numaNode` | `resource.kubernetes.io/numaNode`         |
| `pcieRoot` | `resource.kubernetes.io/pcieRoot`         |

Fully-qualified names are passed through unchanged, enabling
forward-compatibility with future topology attributes.

### Validation

The validating webhook (`pkg/dra/admitter/dra_admitter.go`) enforces:

1. **Mutual exclusion:** exactly one of `resourceClaimName`,
   `resourceClaimTemplateName`, or `managedClaim` must be set per
   `resourceClaims[]` entry.
2. **Feature gate:** `ManagedDRAClaims` must be enabled when
   `managedClaim` is used.
3. **Device coverage:** at least one device declaration must reference each
   managed claim entry (no empty claims).
4. **`deviceClassName` resolvable:** every device referencing a managed
   claim must have a `deviceClassName` — either set on the device or
   available as a default in the KubeVirt CR.
5. **Unique request names:** no duplicate `requestName` values within a
   managed claim.
6. **Valid align entries:** each `align` value must be a known shorthand or
   a valid fully-qualified attribute name.


## API Examples

### Single GPU (Admin Defaults Configured)

When the admin has set `managedClaimDefaults.gpuDeviceClassName` in the
KubeVirt CR, the user writes:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: gpu-vm
spec:
  resourceClaims:
  - name: my-gpu
    managedClaim: {}
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

Generated `ResourceClaim` (using admin default `gpu.nvidia.com`):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-vm-my-gpu
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

### Single GPU (Explicit DeviceClassName)

Without admin defaults, or to override them, the user specifies the
DeviceClass directly:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: gpu-vm
spec:
  resourceClaims:
  - name: my-gpu
    managedClaim: {}
  domain:
    devices:
      gpus:
      - name: gpu0
        claimName: my-gpu
        requestName: gpu
        deviceClassName: gpu.amd.com
    resources:
      requests:
        memory: 8Gi
```

### GPU + NIC Co-Placed on PCIe Root

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: gpu-nic-vm
spec:
  resourceClaims:
  - name: aligned-devices
    managedClaim:
      align: [pcieRoot]
  domain:
    devices:
      gpus:
      - name: gpu0
        claimName: aligned-devices
        requestName: gpu
        deviceClassName: gpu.nvidia.com
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
      deviceClassName: sriov.mellanox.com
```

Generated `ResourceClaim`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-nic-vm-aligned-devices
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

Using `count` to request multiple GPUs without repeating entries:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: full-topology-vm
spec:
  resourceClaims:
  - name: all-devices
    managedClaim:
      align: [numaNode, pcieRoot]
  domain:
    cpu:
      cores: 16
      dra:
        claimName: all-devices
        requestName: cpus
        deviceClassName: cpu.dra.k8s.io
    devices:
      gpus:
      - name: gpu
        claimName: all-devices
        requestName: gpu
        deviceClassName: gpu.nvidia.com
        count: 2
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
      deviceClassName: sriov.mellanox.com
```

The webhook expands `count: 2` into `gpu-0` and `gpu-1`. Generated
`ResourceClaim`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: full-topology-vm-all-devices
spec:
  devices:
    requests:
    - name: gpu-0
      exactly:
        deviceClassName: gpu.nvidia.com
        count: 1
    - name: gpu-1
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
      requests: [gpu-0, gpu-1, nic, cpus]
    - matchAttribute: resource.kubernetes.io/pcieRoot
      requests: [gpu-0, gpu-1, nic, cpus]
```

The `pcieRoot` constraint works across all device types because the CPU
DRA driver publishes `pcieRoot` as a list attribute (KEP-5491). The
scheduler's set-intersection semantics match the CPU group's PCIe root
list against the GPU and NIC scalar values.

## Scalability

Managed claim generation adds one `ResourceClaim` CREATE per managed claim
entry during VMI admission. This follows the existing DRA scalability model.
See [VEP-10 Scalability](../10-dra-devices/vep.md#scalability).

The webhook caches DeviceClass objects via an informer to avoid per-admission
API server queries.

## Update/Rollback Compatibility

- API changes are additive and gated by `ManagedDRAClaims`. With the gate
  disabled, existing behavior is unchanged.
- Rollback: disable the feature gate and delete VMIs using managed claims
  before downgrading.
- Generated `ResourceClaim` objects are standard Kubernetes resources and
  are cleaned up by owner-reference GC.

## Functional Testing Approach

- Unit tests for `GenerateManagedClaim`: single device, multi-device,
  topology alignment, shorthand expansion, error cases
- Unit tests for validation: mutual exclusion, missing `deviceClassName`,
  invalid `align` values, empty claims, duplicate request names
- Integration tests with fake DeviceClasses (envtest)
- E2E: VMI with managed claim for GPU (requires DRA driver in CI)

## Graduation Requirements

### Alpha

- API changes behind `ManagedDRAClaims` feature gate (off by default)
- GPU, HostDevice, Network, and CPU support
- Topology alignment with `align: [numaNode]` and `align: [pcieRoot]`
- Webhook-based claim generation
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

- **Default alignment policies:** cluster-wide defaults via the KubeVirt CR
  (e.g., all managed claims align on `numaNode` by default)
- **Memory and hugepages:** when the CPU DRA driver adds memory allocation
  support, memory can participate in managed claims, enabling full
  GPU + NIC + CPU + memory co-placement on the same NUMA node

## References

- [VEP-10: Support GPUs with DRA](../10-dra-devices/vep.md) (Appendix C:
  Managed Resource Claims)
- [VEP-115: PCIe NUMA Topology Awareness](../115-pcie-numa-topology-awareness/pcie-numa-topology-awareness.md)
- [VEP-152: Support CPUs with DRA](../152-cpu-dra/vep.md)
- [VEP-183: DRA for Network Devices](../../sig-network/183-dra-network/vep.md)
- [KEP-6072: Standard Topology Attributes](https://github.com/kubernetes/enhancements/issues/6072)
- [KEP-5491: List Types for Attributes](https://github.com/kubernetes/enhancements/issues/5491)
- [KEP-5304: DRA Device Attributes Downward API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5304-dra-attributes-downward-api)
- [dra-driver-cpu#114: NIC/CPU alignment](https://github.com/kubernetes-sigs/dra-driver-cpu/issues/114)
