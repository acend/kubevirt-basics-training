---
title: "Introduction"
weight: 21
labfoldernumber: "02"
description: >
  Introduction to the Containerized Data Importer
---

The Containerized Data Importer Project[^1] is an independent installed operator. It is primary used to provide a declarative way
to build Virtual Machine Disks on PVCs for KubeVirt VMs. Since the CDI Operator provides the custom resource `DataVolume`
which is an abstraction above a PVC.


## CDI Operator components

{{% alert title="Note" color="info" %}}
The operator has already been installed in the namespace `cdi`. If you want to deploy the operator check the
[Deploy](https://github.com/kubevirt/containerized-data-importer?tab=readme-ov-file#deploy-it) section of the CDI documentation.
{{% /alert %}}

You may see the cdi operator components with:
```shell
kubectl get pods --namespace cdi
```

The output should similar to:
```shell
NAME                               READY   STATUS    RESTARTS   AGE
cdi-apiserver-5d565ddb6-lrrrk      1/1     Running   0          1d
cdi-deployment-fb59bcc87-xm6dx     1/1     Running   0          1d
cdi-operator-595bfb44cd-j5s4h      1/1     Running   0          1d
cdi-uploadproxy-7657d8d89d-qth44   1/1     Running   0          1d
```

* **cdi-deployment** - Long-lived cdi controller pod managing the cdi operations.
* **cdi-operator** - the operator pod managing the cdi components.
* **cdi-apiserver** - Issues secure tokens and manages authorization to upload VM disks into pvcs.
* **cdi-uploadproxy** - Handles upload traffic and writes content to correct pvc.
* **cdi-importer** - Short-lived helper pod that imports a VM image to a pvc.


## DataVolume manifest

A DataVolume manifest usually has three important fields: `source`, `contentType` and a target which can be either `storage` or `pvc`.

The following DataVolume manifest specifies that the CDI operator create a blank KubeVirt VM disk in a pvc with the size of `64Mi`.

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


### Source

The source location of the data.

* `http` - Download data from a given URL.
* `registry` - Download a Container Disk from a given URL.
* `pvc` - Clone a given pvc referenced by name and namespace to a new pvc.
* `upload` - Provide an upload proxy to receive data from an authenticated client.
* `blank` - Create an empty VM disk.
* `imageio` - Import a VM disk from a running oVirt installation.
* `vddk` - Import a VMware VM disk using vCenter/ESX API.


### ContentType

Type of the source data. Either `kubevirt` (default) or `archive`.

* `kubevirt` defines that the source data should be treated as a virtual disk. CDI itself has the ability to
    convert the source disk as well as to resize it appropriately.
* `archive` defines that the source is a tar archive. CDI will extract the content into the DataVolume.


### Target (Storage/PVC)

You can define the target as `pvc` or `storage`. Both will result in a PVC but there are differences how they work.

* `pvc` defines that the following configuration is passed directly to the pvc.
* `storage` is similar to pvc but provides some logic to fill in fields for the pvc request. As an example we could omit the storageClassName parameter and CDI will detect the default virtualization storage class if there is one which is annotated by `storageclass.kubevirt.io/is-default-virt-class`.

This basic configuration provides the flexibility to use CDI as a provisioner for KubeVirt Virtual Machine Disk Images
as well as a provisioner for regular pvcs used with containers.


### Limitations

Not all combinations of `source` and `contentType` are valid.

CDI supports the following combinations:

| Source     | ContentType                                        |
|------------|----------------------------------------------------|
| `http`     | `kubevirt`, `archive`                              |
| `registry` | `kubevirt`                                         |
| `pvc`      | Not applicable - content is always cloned to a pvc |
| `upload`   | `kubevirt`, `archive`                              |
| `blank`    | `kubevirt`                                         |
| `imageio`  | `kubevirt`                                         |
| `vddk`     | `kubevirt`                                         |


[^1]: [Containerized Data Importer Project](https://github.com/kubevirt/containerized-data-importer)
