---
title: "Using StorageProfiles"
weight: 62
labfoldernumber: "06"
description: >
  Setting defaults for storage provisioning using StorageProfiles
---

When working with storage and the containerized data importer one usually wants to have meaningful defaults. Let us have a look
how we can configure storage classes to be used with KubeVirt.


## What are StorageProfiles

For each available StorageClass KubeVirt creates a StorageProfile. StorageProfiles serve as a source of information about
the recommended parameters for a pvc. They are used when provisioning a PVC using a DataVolume. Having recommended parameters
defined centrally in a StorageProfile reduces the complexity of your DataVolume definition.

You can check the StorageProfiles with:
```yaml
kubectl get storageprofiles
```
```
NAME             AGE
hcloud-volumes   38d
longhorn         38d
```

You may check the configuration of the StorageProfile with:
```yaml
kubectl describe storageprofile longhorn
```
```
Name:         longhorn
Namespace:    
Labels:       app=containerized-data-importer
              app.kubernetes.io/component=storage
              app.kubernetes.io/managed-by=cdi-controller
              cdi.kubevirt.io=
Annotations:  <none>
API Version:  cdi.kubevirt.io/v1beta1
Kind:         StorageProfile
Metadata:
  Creation Timestamp:  2024-07-27T07:54:21Z
  Generation:          2
  Owner References:
    API Version:           cdi.kubevirt.io/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  CDI
    Name:                  cdi
    UID:                   a9f271d9-e57a-4199-b61d-28dd6925587c
  Resource Version:        16172528
  UID:                     5ebfeeb1-c45d-44b6-8d48-d31cfc501110
Spec:
Status:
  Clone Strategy:                  snapshot
  Data Import Cron Source Format:  pvc
  Provisioner:                     driver.longhorn.io
  Snapshot Class:                  longhorn-snapshot-vsc
  Storage Class:                   longhorn
Events:                            <none>
```


## Create a DataVolume

Let's have a look how this works. Assume we create a DataVolume `my-dv` with the following specification and apply it to the cluster:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: my-dv
spec:
  source:
    blank: {}
  storage:
    resources:
      requests:
        storage: 128Mi
```

The DataVolume will not get created. Whenever we describe the DataVolume with:
```shell
kubectl describe datavolume my-dv
```

We see that CDI is lacking some information to create the PVC and return with an error.
```
Status:
  Conditions:
    Message:               no accessMode defined on StorageProfile for longhorn StorageClass
    Reason:                ErrClaimNotValid
    Status:                Unknown
```

This means that the controller did not know which accessMode to use for the PVC.


### Define StorageProfiles

Beside others a storage profile `spec` block can take the following parameters:

* `claimPropertySets`
  * `accessMode` - contains the desired access modes the volume should have
  * `volumeMode` - defines what type of volume is required by the claim
* `cloneStrategy` - defines the preferred method for performing a CDI clone
  * `copy` - copy blocks of data over the network
  * `snapshot` - clones the volume by creating a temporary VolumeSnapshot and restoring it to a new PVC
  * `csi-clone` - clones the volume using a CSI clone

If you want to read more about parameters, defaults and how storage profiles are used, check the [StorageProfiles](https://github.com/kubevirt/containerized-data-importer/blob/main/doc/storageprofile.md#parameters) documentation.


### Setting AccessMode and VolumeMode

To fix the issue above and provide default values for the storage profile `longhorn` we can set defaults in the `spec`
block of the storage profile.

Let's assume we add the `accessMode` and `volumeMode` to the `longhorn` storage profile like this:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: StorageProfile
metadata:
  name: longhorn
[...]
spec:
  claimPropertySets:
    - accessModes:
      - ReadWriteOnce
      volumeMode: Filesystem
[...]
```

When we have the storage profile configured and in place, we can re-apply our DataVolume:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: my-dv
spec:
  source:
    blank: {}
  storage:
    resources:
      requests:
        storage: 128Mi
```

It is now successfully provisioned using the defaults from the storage profile.
```shell
kubectl describe datavolume my-dv
```
```
Status:
  Conditions:
    Message:               PVC my-dv Bound
    Reason:                Bound
    Status:                True
```
