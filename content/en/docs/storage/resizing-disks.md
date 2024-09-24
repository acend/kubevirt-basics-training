---
title: "Resizing a Disk"
weight: 65
labfoldernumber: "06"
description: >
  Resizing virtual machine disks
---

In this section we will resize the root disk of our virtual machine.


## Requirements

Resizing depends on the Kubernetes Storage provider. The CSI driver must support resizing volumes as well as it must be configured to `AllowVolumeExpansion`.

Further it may depend on the operating system you use. Whenever the volume is resized your VM might see the change
in disk size immediately. But there might still be the need to resize the partition and filesystem. For example Fedora Cloud
uses has the package `cloud-utils-growpart` installed. This rewrites the partition table so that partition take up all
the space it is available. This makes it very handy choice for virtual machines resizing disk images.


## {{% task %}} Create a volume and a virtual machine

In a first step have to create a fedora disk. Create the file `dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk.yaml` with the following content:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk
spec:
  source:
    registry:
      url: "docker://{{% param "fedoraCloudCDI" %}}"
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 6Gi
```

Create the data volume in the cluster:
```shell
kubectl create -f dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk.yaml --namespace=$USER
```

Create the file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand.yaml` and use the following yaml specification:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand
    spec:
      domain:
        cpu:
          cores: 1
        devices:
          disks:
            - name: datavolumedisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 2Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: datavolumedisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
```

Create the virtual machine with:
```shell
kubectl create -f vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand.yaml --namespace=$USER
```

Start the virtual machine with:
```shell
virtclt start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand --namespace=$USER
```


### Check the disk size

Enter your virtual machine with:
```shell
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand --namespace=$USER
```

Check your block devices with:
```shell
lsblk
```
```s
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
zram0  251:0    0  1.9G  0 disk [SWAP]
vda    252:0    0  5.7G  0 disk 
├─vda1 252:1    0    2M  0 part 
├─vda2 252:2    0  100M  0 part /boot/efi
├─vda3 252:3    0 1000M  0 part /boot
└─vda4 252:4    0  4.6G  0 part /var
                                /home
                                /
vdb    252:16   0    1M  0 disk
```


## {{% task %}} Resize the disk

Triggering a resize of a pvc in kubernetes can be done with editing the pvc size request. Get the PersistentVolumeClaim manifest with:
```shell
kubectl get pvc {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk -o yaml
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk
  [...]
spec:
  resources:
    requests:
      storage: 6Gi
[...]
```

Now patch the pvc to increase the disk size to `8Gi`.
```shell
kubectl patch pvc {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk --type='json' -p='[{"op": "replace", "path": "/spec/resources/requests/storage", "value":"8Gi"}]'
```
```
persistentvolumeclaim/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk patched
```

It might take some time for the storage provider to resize the persistent volume. You can see details about the process
in the events section when describing the PersistentVolumeClaim:
```shell
kubectl describe pvc {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk
```
```
Events:
  Type     Reason                       Age                 From                                                                                      Message
  ----     ------                       ----                ----                                                                                      -------
  Warning  ExternalExpanding            2m56s               volume_expand                                                                             waiting for an external controller to expand this PVC
  Normal   Resizing                     2m56s               external-resizer driver.longhorn.io                                                       External resizer is resizing volume pvc-bc3e89f3-1372-4e10-852a-0c94ea343ab4
  Normal   FileSystemResizeRequired     2m46s               external-resizer driver.longhorn.io                                                       Require file system resize of volume on node
  Normal   FileSystemResizeSuccessful   2m8s                kubelet                                                                                   MountVolume.NodeExpandVolume succeeded for volume "pvc-bc3e89f3-1372-4e10-852a-0c94ea343ab4" training-worker-2
```

If you still have a console open in your virtual machine you see that there was a message about the capacity change:
```shell
[  896.201742] virtio_blk virtio3: [vda] new size: 15853568 512-byte logical blocks (8.12 GB/7.56 GiB)
[  896.202409] vda: detected capacity change from 11890688 to 15853568
```

Recheck your block devices:
```shell
lsblk
```
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
zram0  251:0    0  1.9G  0 disk [SWAP]
vda    252:0    0  7.6G  0 disk 
├─vda1 252:1    0    2M  0 part 
├─vda2 252:2    0  100M  0 part /boot/efi
├─vda3 252:3    0 1000M  0 part /boot
└─vda4 252:4    0  4.6G  0 part /var
                                /home
                                /
vdb    252:16   0    1M  0 disk
```

You will see that the capacity change is visible from within the virtual machine. But at this time our partitions still
have the same size and do not use all available diskspace.

Issue a reboot to let the system expand the partitions:
```shell
sudo reboot
```

After the reboot you have to login again and check `lsblk` again:
```shell
lsblk
```

You can see that for example `vda4` has been resized from `5.7G` to `7.6G`.


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Delete your VirtualMachines:
```shell
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand
```

Delete your disks:
```shell
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fs-disk
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-expand-disk
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-hotplug-disk
```

Delete your VirtualMachineSnapshots:
```shell
kubectl delete vmsnapshot {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap
```

{{% /alert %}}
