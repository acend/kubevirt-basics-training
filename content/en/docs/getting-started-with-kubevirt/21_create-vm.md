---
title: "2.1 Create a VM"
weight: 210
labfoldernumber: "02"
sectionnumber: 2.1
description: >
  Create your first VirtualMachine
---


## Creating a virtual machine

To create a virtual machine in our Kubernetes cluster we have to create and apply a `VirtualMachine` manifest.

This is a very basic example of a bootable virtual machine manifest:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm
    spec:
      domain:
        devices: {}
        resources:
          requests:
            memory: 64M
```


{{% alert title="Note" color="info" %}}
A VirtualMachine manifest requires a memory resource specification. Therefore, we always have `spec.domain.resources.requests.memory` set.
You may also use `spec.domain.memory.guest` or `spec.domain.memory.hugepages.size` as a resource specification.
{{% /alert %}}


### {{% task %}} Review the VirtualMachine manifest

Do you see any problems with the specification above? Try to answer the following questions:

* What happens when you run this VM?
* What is required to successfully boot a machine?

{{% details title="Task Hint" %}}
Our created manifest does not contain any bootable devices. Our VM is able to start, but it will just hang as there are
no bootable devices available.

![No bootable device](../no-bootable-device.png)

Having a bootable disk within the specification is all you need to start the VM. However, as there is no network
interface specified which is connected to the underlying network, our system would not be capable of interacting with the
outside world.
{{% /details %}}


### {{% task %}} Write your own VirtualMachine manifest

Create a new file `firstvm.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/` and copy the `VirtualMachine` manifest from above as a starting point. You then need to add a bootable disk for your VM and a network and interface specification.
To achieve this you need to specify the following parts in the `VirtualMachine` manifest:

* `spec.template.spec.domain.devices`
* `spec.template.spec.networks`
* `spec.template.spec.volumes`

The easiest way is to use an ephemeral `containerDisk` mountable as a volume. Regarding the network, we simply connect our VM to the underlying Kubernetes default network:

```yaml
spec:
  template:
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk: 
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: {{% param "cirrosCDI" %}}
```

Make sure you implement the required parts for a container disk and the network interface specification in you VM manifest.

{{% details title="Task hint: Resulting yaml" %}}
Your VirtualMachine definition should look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
        resources:
          requests:
            memory: 64M
      networks:
        - name: default
          pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: {{% param "cirrosCDI" %}}
```
{{% /details %}}


### {{% task %}} Create the VirtualMachine

It is now time to create the VM on the Kubernetes cluster using the definition you just created:

```shell
kubectl create -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/firstvm.yaml --namespace=$USER
```

The output should be:

```shell
virtualmachine.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm created
```
