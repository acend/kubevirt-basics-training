---
title: "Create a DataVolume"
weight: 22
labfoldernumber: "02"
description: >
  Create a DataVolume to import data into a PVC
---

## Provisioning a Virtual Machine Disk

In the following lab we will provision a disk from a given url. We will use a `DataVolume` to create the Disk. We then
create a VM with the disk attached.

We want to provision a VM running an Alpine Cloud Image[^1].


## {{% task %}} Write your own DataVolume manifest

Create a file `dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}`and start with the
yaml content from the previous section:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-blankdv
spec:
  source:
    blank: {}
  contentType: "kubevirt"
  storage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 64Mi
```

Now adapt your DataVolume to fulfill the following specifications:

1. Download a Disk Image from a given URL
    * Use `{{% param alpineContainerDiskDownload %}}` as URL for the disk image.
2. Create a KubeVirt VM Disk
3. Requested storage should be 256Mi
4. Name the volume `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk`
5. Set the `accessMode` to `ReadWriteOnce`

{{% details title="Task Hint" %}}
Your yaml should look like this:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk
spec:
  source:
    http:
      url: "{{% param alpineContainerDiskDownload %}}"
  contentType: "kubevirt"
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 256Mi
```
{{% /details %}}

Before you apply your DataVolume to the cluster check the currently available pvcs.

```shell
kubectl get pvc --namespace=$USER
```

The output should be similar to:
```shell
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
$USER-webshell          Bound    pvc-594cf281-4c34-4e7c-b345-2bf2692bbb78   1Gi        RWO            longhorn       <unset>                 1d
$USER-webshell-docker   Bound    pvc-86e4bc75-396f-4630-940e-b0a4b0cf23fa   10Gi       RWO            longhorn       <unset>                 1d
```

Now create the DataVolume in the kubernetes cluster with:
```shell
kubectl create -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk.yaml --namespace=$USER
```

The output should be:
```shell
datavolume.cdi.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk created
```

This will trigger the CDI operator which will start an importer pod to provision your pvc. If you are fast enough you may see the pod with:
```shell
kubectl get pods --namespace=$USER
```

The output should be similar to:
```shell
NAME                                                  READY   STATUS              RESTARTS   AGE
importer-prime-36720196-c64a-42d8-8db5-af31b75de034   0/1     ContainerCreating   0          9s
$USER-webshell-885dbc579-lwhtd                        2/2     Running             0          1d
```

After some time the pod will complete and your pvc should be provisioned. Let's check for the existence of the pvc.

```shell
kubectl get pvc --namespace=$USER
```

The output should be similar to:

```shell
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk        Bound    pvc-fe1faa27-048b-4270-a3ac-c2abf4a24aca   272Mi      RWO            longhorn       <unset>                 72s
user4-webshell          Bound    pvc-594cf281-4c34-4e7c-b345-2bf2692bbb78   1Gi        RWO            longhorn       <unset>                 1d
user4-webshell-docker   Bound    pvc-86e4bc75-396f-4630-940e-b0a4b0cf23fa   10Gi       RWO            longhorn       <unset>                 1d
```


## {{% task %}} Recap the provisioning process

You have successfully provisioned a disk image which is ready to be used within a virtual machine.

We have used the alpine image as it is reasonable small. Would there have been another way to provide the Disk Image for
every participant without the need to download the image multiple times?

{{% details title="Task Hint" %}}
We could have provided a provisioned pvc in a central namespace and create our DataVolume with a `pvc` source and let
the CDI operator clone the central pvc to a pvc your namespace.
{{% /details %}}

[^1]: [Alpine Cloud Image](https://alpinelinux.org/cloud/)
