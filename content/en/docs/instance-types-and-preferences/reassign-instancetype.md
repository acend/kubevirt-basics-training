---
title: "Re-assign an Instancetype"
weight: 34
labfoldernumber: "03"
description: >
  Re-assign an Instance type of your VM
---

In this section we will change the instancetype of our running VMs to our newly created instance type.


## Changing an Instancetype

Whenever a VM referencing an instancetype or preference is created, the definition at time of creation is stored in a `ControllerRevision`. This revision is then referenced in a new field `.spec.instancetype.revisionName` in our VM manifest.

This field ensures that our VirtualMachine knows the original specification even when the type or preference would be changed. This ensures that there are no accidental changes of the VM resources or preferences.

Example of a VM with a `revisionName`:
```yaml
[...]
spec:
  instancetype:
    kind: VirtualMachineClusterInstancetype
    name: u1.nano
    revisionName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros-u1.nano-v1beta1-e15b4047-3ff9-4308-9cd7-9f30b25336e0-1
[...]
```

Use the following command to list all available `controllerrevision`:
```shell
kubectl get controllerrevision --namespace=$USER
```

```
NAME                                                                                     CONTROLLER                                   REVISION   AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros-cirros-v1beta1-fa1da1cd-7e10-4e89-a8ac-5bded8f7129e-1                    virtualmachine.kubevirt.io/lab04-o1-cirros   0          2h
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros-cirros-v1beta1-fa1da1cd-7e10-4e89-a8ac-5bded8f7129e-1                    virtualmachine.kubevirt.io/lab04-u1-cirros   0          2h
[...]
```

If we want to explicitly change the Instancetype or preference we have to remove the `revisionName` attribute completely otherwise it will reject the change.

The easies way is to edit the resource directly is:
```shell
kubectl edit vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros --namespace=$USER
```

An alternative would be patching the resource directly with:
```shell
kubectl patch vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros --type merge --patch '{"spec":{"instancetype":{"kind":"<KIND>","name":"<NAME>","revisionName":null}}}' --namespace=$USER
```


## {{% task %}} Adapt your VMs to use the new instancetype

Edit the VM `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros` and reference the `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-pico` and do the same for the VM `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros` and reference `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-pico`

{{% alert title="Note" color="info" %}}
Since so far we were referencing a `VirtualMachineClusterPreference` also change the kind to `VirtualMachineInstancetype` for our new InstanceTypes.
And don't forget to remove the referenced `revisionName`.
{{% /alert %}}


{{% details title="Task Hint" %}}
The relevant section for the VM `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros` should look like this:

```yaml
spec:
  instancetype:
    kind: VirtualMachineInstancetype
    name: lab04-u1-pico
```

For `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros` it will be:
```yaml
spec:
  instancetype:
    kind: VirtualMachineInstancetype
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-pico
```
{{% /details %}}

Make sure you restart both VMs to reflect the change of their instancetype:
```shell
virtctl restart {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros --namespace=$USER
virtctl restart {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros --namespace=$USER
```

Verify whether the two VMIs are running again properly:
```shell
kubectl get vmi --namespace=$USER
```

Describe both VirtualMachine instances and observe the effect:
```shell
kubectl get vmi {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros -o yaml --namespace=$USER
```
```shell
kubectl get vmi {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros -o yaml --namespace=$USER
```

The `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros` instance:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros
spec:
  domain:
    resources:
      requests:
        memory: 256Mi
    memory:
      guest: 256Mi
[...]
```

`{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros` instance:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros
spec:
  domain:
    resources:
      requests:
        memory: 192Mi
    memory:
      guest: 256Mi
[...]
```


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Stop your running VM with
```shell
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-u1-cirros --namespace=$USER
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-o1-cirros --namespace=$USER
```
{{% /alert %}}
