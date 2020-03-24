---
title: automatic-detection-of-disks-in-Local-Storage-Operator
authors:
  - "@ashishranjan738"
  - "@rohantmp"
  - "@jarrpa"
reviewers:
  - "@jsafrane"
  - "@hekumar"
  - "@chuffman"
  - "@sapillai"
  - "@leseb"
  - "@travisn"
  - "@rohantmp"
  - "@jarrpa"
approvers:
  - "@jsafrane"
  - "@hekumar"
  - "@chuffman"
creation-date: 2020-01-21
last-updated: 2020-03-25
status: implementable
---

# Automatic detection of Disks in the Local-Storage-Operator

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

This is a proposal to enable
- automated discovery of local disks via the `LocalVolumeDiscovery` CR and exposing the result via `LocalVolumeDiscoveryResult` CR.
- automated provisioning of LocalPVs sorting them into StorageClasses based on their characteristics via the `LocalVolumeSet` CRD.

This CRD will be implemented in the LocalStorageOperator(LSO)

## Motivation

The existing method of manually managing disks on nodes does not scale, and cannot be consumed by another operator.
Choosing devices to be used should be handled on the platform so that multiple consumers of local storage don't conflict.
To enable layering, we need information about potential devices, so that we have information about availability,
and we can use that information to sort devices into storage classes based on their capabilities.

### Goals

- Discovery of potentially usable local disks on chosen nodes.
- Automatic sorting of available disks into StorageClasses based on their characteristics.
- Detect newly attached disks

### Non-Goals

- Inherently protect against confilict with provisioners that own local devices on the same nodes automatic detection is configured to run.

### Risks and Mitigations
- There is a risk that LSO detects a volume that contains data and we mitigate this risk by ensuring that the device:
  - can be openened exclusively.
  - is not read-only.
  - is not removable.
  - has no child partitions.
  - has no FS signature.
  - state (as outputted by `lsblk`) is not `suspended`
- There is a risk that LSO will detect just attached device as empty, before kubelet formats it, we mitigate this risk
    - by re-checking the device after ~1 minute.
- There is a risk that the path of the disk changes after reboot and disk can be re-detected as new, we mitigate this risk by
    - Using disk id for symlink.
- There is a risk that LSO will match all local disks that already have PVs, but these PVs are not bound / used yet.`
    - We leave it up to the cluster admin for selecting the inclusion spec.
- There is a risk that multiple `LocalVolumeSet` and `LocalVolume` CRs will use the same device.
    - We leave it up to the cluster admin to adjust the CR.
- There is a risk that LSO may create PV for a device/disk that's already used as another PV.
    - We can solve this by updating local-storage-static-provisioner to hash localPVs using hostname and disk name and remove the storageclass.

## Proposal

The `local-storage-operator` is already capable of consuming local disks from the nodes and provisioning PVs out of them, but the disk/device paths needs to explicitly specified in the `LocalVolume` CR.
The idea is to introduce two new features in LSO
  - Auto discovery of local devices
    - This will introduce two new CRs `LocalVolumeDiscovery` and `LocalVolumeDiscoveryResult`.
    - The purpose of this will be to expose local devices available in a node via the `LocalVolumeDiscoveryResult` CR.
    - The device discovery will be done once via a daemonSet pod.
  - Auto provisioning of localDevices
    - This will introduce a new CR called `LocalVolumeSet`.
    - The purpose of this will be to auto discover and provision PVs on devices which match the inclusion filter present in the CR.
    - This will involve a continous process of discovery of devices via the diskmaker daemons. Any discovered devices which matches the inclusion filter will be considered for provisioning of PVs.

Once we have the detected devices the administrator can create the localPVs by explicitly selecting the disks. This can be done by the `localVolume` CR. The other option would be to create localPVs via the `LocalVolumeSet` CR and passing the inclusion filters in it.

#### Workflow of LocalPV creation via LocalVolume CR

1. Discovery: The user can choose to run discovery if they want to understand what devices are available across the nodes in the cluster.
    - ##### NOTE: This step is optional for users directly creating CRs. They might already know what CRs they need for step 2.
2. PV Creation: After the user has decided how to created PVs, they can choose to create PVs with either the `LocalVolume` CR or the `LocalVolumeSet` CR.

## Design Details for `Auto discovery of local devices`

API scheme for `LocalVolumeDiscovery` and `LocalVolumeDiscoveryResult` CR:

```go
type LocalVolumeDiscoveryList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`
	Items           []LocalVolumeDiscovery `json:"items"`
}

type LocalVolumeDiscovery struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`
	Spec              LocalVolumeDiscoverySpec   `json:"spec"`
	Status            LocalVolumeDiscoveryStatus `json:"status,omitempty"`
}

type LocalVolumeDiscoverySpec struct {
	// Nodes on which the automatic detection policies must run.
	// +optional
	NodeSelector *corev1.NodeSelector `json:"nodeSelector,omitempty"`
	// If specified tolerations is the list of toleration that is passed to the
	// LocalVolumeDiscovery Daemon
	// +optional
	Tolerations []corev1.Toleration `json:"tolerations,omitempty"`
}

type LocalVolumeDiscoveryStatus struct {
	// Conditions is a list of conditions and their status.
	Conditions []operatorv1.OperatorCondition `json:"conditions,omitempty"`
	// observedGeneration is the last generation change the operator has dealt with
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}

type LocalVolumeDiscoveryResultList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`
	Items           []LocalVolumeDiscoveryResult `json:"items"`
}

type LocalVolumeDiscoveryResult struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`
	Spec              LocalVolumeDiscoveryResultSpec   `json:"spec"`
	Status            LocalVolumeDiscoveryResultStatus `json:"status,omitempty"`
}

type LocalVolumeDiscoveryResultSpec struct {
	NodeName string `json:"nodeName"`
}

type LocalVolumeDiscoveryResultStatus struct {
	// Conditions is a list of conditions and their status.
	Conditions          []operatorv1.OperatorCondition `json:"conditions,omitempty"`
	DiscoveredTimeStamp string                         `json:"discoveredTimeStamp,omitempty"`
	// Devices contains the list of devices on which LSO
	// is capable of creating LocalPVs
	// The devices in this list qualify these following conditions.
	// - it should be a non-removable device.
	// - it should not be a read-only device.
	// - it should not be mounted anywhere
	// - it should not be a boot device
	// - it should not have child partition in lsblk output
	Devices []Device `json:"devices"`
}

type Device struct {
	// Name of the detected device
	Name string `json:"name"`
	// Model of the detected device
	Model string `json:"model,omitempty"`
	// Type of the detected device
	Type string `json:"type,omitempty"`
	// Vendor of the detected device
	Vendor string `json:"vendor,omitempty"`
	// Size of the detected device
	Size resource.Quantity `json:"size,omitempty"`
	// Rotational represents whether the device type is rotational or not
	Rotational bool `json:"rotational,omitempty"`
    // FSType represents the filesystem available on the device
	FSType string `json:"fstype",omitempty`
}
```

### Workflow of `Auto Device Discovery`
The discovery of local devices will be done by the `LocalVolumeDiscovery` and `LocalVolumeDiscoveryResult` CR.
#### LocalVolumeDiscovery CR
- Once the `LocalVolumeDiscovery` CR is deployed, its controller will:
  - Create a daemonSet to run on each of the valid nodes.(Information about the valid list of nodes would be available in the `LocalVolumeDiscovery` CR as `nodeSelector`)
  - Remember the current timestamp when the discovery has been triggered.
- The DaemonSet pod running on each valid node will:
  - Identify the list of devices available on that node.
  - Create`LocalVolumeDiscoveryResult` CR on that node with the list of devices identified above.
- The operator would delete the daemonSet if:
  - `LocalVolumeDiscoveryResult` is created on the node where daemonSet pod is running.
  - The last updated timestamp on the `LocalVolumeDiscoveryResult` CR is greater than discovery start timestamp.
- The operator would delete the daemonSet in a retry logic that runs after every fixed interval of time.

Example of `LocalVolumeDiscovery` CR:
```yaml
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeDiscovery
metadata:
  name: example-autodetect
  namespace: localStorage
spec:
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - worker-0
              - worker-1
status:
  conditions:
  - lastTransitionTime: "2020-03-17T09:33:43Z"
    status: "True"
    type: Available
```

#### LocalVolumeDiscoveryResult CR
- The `LocalVolumeDiscoveryResult` CR will be exposing the available local devices of the node in its status.
- These information will mostly be the output of  `lsblk -o name,model,vendor,kname,pkname,fstype,type,uuid,tran,ro,rm,parttype,serial,size,rota,mountpoint`.

Example of `LocalVolumeDiscoveryResult` CR after detection:
```yaml
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeDiscoveryResult
metadata:
  name: example-volume-detect-1
  namespace: localStorage
spec:
  nodeName: worker-0
status:
  conditions:
  - lastTransitionTime: "2020-03-17T09:33:43Z"
    status: "True"
    type: Available
  discoveredTimeStamp: '2020-03-09T08:37:19Z'
  devices:
    - name: sda
      model: SAMSUNG MZ7LN512
      vendor: ATA
      type: disk
      size: 477G
      rotational: true
```
### Cadence for `Auto Device Discovery`
- Running device discovery continuously will be expensive. It should be a one time thing, that is, don't allow this operation to be performed multiple times even from the UI.
- If the user has selected wrong set of nodes or wants to change the selected nodes, then they can cancel out of the operation. In this case the device Discovery CRs will be deleted and we can start the operation again with the new device discovery CR.
- This can be achieved by making an `update` event on the `LocalVolumeDiscovery` CR as a `no-op`

## Design Details for `Auto provisioning of local devices`
API scheme for `LocalVolumeSets`:

```go
type LocalVolumeSetList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`
	Items           []LocalVolumeSet `json:"items"`
}

type LocalVolumeSet struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`
	Spec              LocalVolumeSetSpec   `json:"spec"`
	Status            LocalVolumeSetStatus `json:"status,omitempty"`
}

// PersistentVolumeMode describes how a volume is intended to be consumed, either Block or Filesystem.
type PersistentVolumeMode string

const (
	// PersistentVolumeBlock means the volume will not be formatted with a filesystem and will remain a raw block device.
	PersistentVolumeBlock PersistentVolumeMode = "Block"
	// PersistentVolumeFilesystem means the volume will be or is formatted with a filesystem.
	PersistentVolumeFilesystem PersistentVolumeMode = "Filesystem"
)

type LocalVolumeSetSpec struct {
	// Nodes on which the automatic detection policies must run.
	// +optional
	NodeSelector *corev1.NodeSelector `json:"nodeSelector,omitempty"`
	// StorageClassName to use for set of matched devices
	StorageClassName string `json:"storageClassName"`
	// MinDeviceCount is the minumum number of devices that needs to be detected per node.
	// If the match devices are less than the minCount specified then no PVs will be created.
	// +optional
	MinDeviceCount *int32 `json:"minDeviceCount"`
	// Maximum number of Devices that needs to be detected per node.
	// +optional
	MaxDeviceCount *int32 `json:"maxDeviceCount"`
	// VolumeMode determines whether the PV created is Block or Filesystem. By default it will
	// be block
	// + optional
	VolumeMode PersistentVolumeMode `json:"volumeMode,omitempty"`
	// FSType type to create when volumeMode is Filesystem
	// +optional
	FSType string `json:"fsType,omitempty"`
	// If specified, a list of tolerations to pass to the discovery daemons.
	// +optional
	Tolerations []corev1.Toleration `json:"tolerations,omitempty"`
	// DeviceInclusionSpec is the filtration rule for including a device in the device discovery
	// +optional
	DeviceInclusionSpec *DeviceInclusionSpec `json:"deviceInclusionSpec"`
}

// DeviceMechanicalProperty holds the device's mechanical spec. It can be rotational or nonRotational
type DeviceMechanicalProperty string

const (
	// The mechanical properties of the devices
	// Rotational refers to magnetic disks
	Rotational DeviceMechanicalProperty = "Rotational"
	// NonRotational refers to ssds
	NonRotational DeviceMechanicalProperty = "NonRotational"
)

type DeviceType string

const (
	// The DeviceTypes that will be supported by the LSO.
	// These Discovery policies will based on lsblk's type output
	// RawDisk represents a device-type of block disk
	RawDisk DeviceType = "RawDisk"
	// Part represents a device-type of partion
	Partition DeviceType = "Partition"
)

type DeviceInclusionSpec struct {
	// Devices is the list of devices that should be used for automatic detection.
	// This would be one of the types supported by the local-storage operator. Currently,
	// the supported types are: disk, part. If the list is empty no devices will be selected.
	Devices []DeviceType `json:"devices"`

	// DeviceMechanicalProperty denotes whether Rotational or NonRotational disks should be used.
	// by default, it selects both
	// +optional
	DeviceMechanicalProperties []DeviceMechanicalProperty `json:"deviceMechanicalProperties"`

	// MinSize is the minimum size of the device which needs to be included
	// +optional
	MinSize *resource.Quantity `json:"minSize"`

	// MaxSize is the maximum size of the device which needs to be included
	// +optional
	MaxSize *resource.Quantity `json:"maxSize"`

	// Models is a list of device models. If not empty, the device's model as outputted by lsblk needs
	// to contain at least one of these strings.
	// +optional
	Models []string `json:"models"`

	// Vendors is a list of device vendors. If not empty, the device's model as outputted by lsblk needs
	// to contain at least one of these strings.
	// +optional
	Vendors []string `json:"vendors"`
}

type LocalVolumeSetStatus struct {
	// Conditions is a list of conditions and their status.
	Conditions []operatorv1.OperatorCondition `json:"conditions,omitempty"`

	// TotalProvisionedDeviceCount is the count of the total devices over which the PVs has been provisioned
	TotalProvisionedDeviceCount *int32 `json:"totalProvisionedDeviceCount,omitempty"`

	// observedGeneration is the last generation change the operator has dealt with
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}
```
### Workflow of `LocalPV` creation via `LocalVolumeSet CR`
- Once the `LocalVolumeSet` is created, the controller will:
  - configure the local-static-provisioner to make a new StorageClass based on certain directories on the selected nodes.
  - assign diskmaker daemonset to the selected nodes.
  - there will be one daemonset for each `LocalVolumeSet` CR.
- The diskmaker daemon will find devices that match the disovery policy and symlink them into the directory that the local-static-provisioner is watching.

#### Note: There is a chance of race condition and duplicate creation of PVs
- If two `LocalVolumeSet` CR targets same nodes with overlapping inclusion filter.
- If `LocalVolumeSet` CR and `LocalVolume` CR targets the same node with overlapping devices.

Example of `LocalVolumeSet` CR:

```yaml
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeSet
metadata:
  name: example-autodetect
spec:
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - worker-0
              - worker-1
  storageClassName: example-storageclass
  volumeMode: Block
  minDeviceCount: 5
  maxDeviceCount: 10
  deviceInclusionSpec:
    deviceTypes:
      - RawDisk
    deviceMechanicalProperty:
      - Rotational
      - NonRotational
    minSize: 10G
    maxSize: 100G
status:
  conditions:
  - lastTransitionTime: "2020-03-17T09:33:43Z"
    status: "True"
    type: Available
  totalProvisionedDeviceCount: 5
```

```yaml
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeSet
metadata:
  name: example-autodetect
spec:
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - worker-0
              - worker-1
  storageClassName: example-storageclass
  volumeMode: filesystem
  fstype: ext4
  minDeviceCount: 5
  maxDeviceCount: 10
  deviceInclusionSpec:
    deviceTypes:
      - RawDisk
      - Partition
    deviceMechanicalProperty:
      - NonRotational
    minSize: 10G
    maxSize: 100G
    models:
      - SAMSUNG
      - Crucial_CT525MX3
    vendors:
      - ATA
      - ST2000LM
status:
  conditions:
  - lastTransitionTime: "2020-03-17T09:33:43Z"
    status: "True"
    type: Available
  totalProvisionedDeviceCount: 5
```

### Test Plan

- The integration tests for the LSO already exist. These tests will need to be updated to test this feature.
- The tests must ensure that detection of devices are working/updating correctly.
- The tests must ensure that data corruption are not happening during auto detection of devices.

### Graduation Criteria

- Documentation exists for the behaviour of each configuration item.
- Unit and End to End tests coverage is sufficient.

##### Removing a deprecated feature

- None of the features are getting deprecated

### Upgrade / Downgrade Strategy

Since this requires a new implementation no new upgrade strategy will be required.

### Version Skew Strategy

N/A

## Implementation History

N/A

## Drawbacks

N/A

## Alternatives

- Existing manual creation of LocalVolume CRs. With the node selector on the LocalVolume, a single CR can apply to an entire class of nodes (i.e., a machineset or a physical rack of homogeneous hardware). When a machineset is defined, a corresponding LocalVolume can also be created.
- Directly enhancing the LocalVolume CR to allow for auto discovery

The first approach requires some manual work and knowledge of underlying nodes, this makes it inefficient for large clusters. The second approach can introduce breaking change to the existing GA API. Therefore this approach makes sense.
