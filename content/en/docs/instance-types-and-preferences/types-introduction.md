---
title: "Introduction"
weight: 31
labfoldernumber: "03"
description: >
  Create your own Instancetype
---

Even if KubeVirt provides advanced options and a lot of configuration options for different VMs we usually have a common
set of VM specifications which we will use for most of our VMs. Therefore, it may make sense of defining such
specifications as Instancetypes and / or Preferences.

To achieve this, KubeVirt provides multiple Custom Resource
Definitions like `VirtualMachineInstancetype`, `VirtualMachineClusterInstancetype` or `VirtualMachinePreference`.


## VM VirtualMachineInstancetype

For an Instancetype we have the option of using the cluster wide `VirtualMachineClusterInstancetype` or the namespaced
`VirtualMachineInstancetype`.

With Instancetypes we can define the following resource related characteristics:

* **CPU** - Required number of vCPUs presented to the guest
* **Memory** - Required amount of memory presented to the guest
* **GPUs** - Optional list of vGPUs to passthrough
* **HostDevices** - Optional list of HostDevices to passthrough
* **IOThreadsPolicy** - Optional IOThreadsPolicy to be used
* **LaunchSecurity** - Optional LaunchSecurity to be used

{{% alert title="Important" color="warning" %}}
Any provided Instancetype characteristic can not be overridden from within the VirtualMachine. Be aware that `CPU` and
`Memory` both are required for an Instancetype. Therefore, any different request of `CPU` or `Memory` on a VirtualMachine
will conflict and the request will be rejected.
{{% /alert %}}


## VM Preference

KubeVirt also provides a CRD `VirtualMachineClusterPreference` for cluster wide preferences as well as a namespaced
version `VirtualMachinePreference`. A preference specification encapsulates every value of the remaining attributes of a VirtualMachine.

{{% alert title="Note" color="info" %}}
Not like the characteristics from an Instancetype the preferences only defined the preferred values. They can be overridden
in the VirtualMachine specification. The specification from the VirtualMachine has priority.
{{% /alert %}}


## Using Instancetype or Preference in a virtual machine

A sample virtual machine referencing an Instancetype and Preference can look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-examplevm
spec:
  instancetype:
    kind: VirtualMachineInstancetype
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-example-instancetype
  preference:
    kind: VirtualMachinePreference
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-example-preference
```
