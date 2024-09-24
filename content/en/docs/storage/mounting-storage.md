---
title: "Mounting Storage"
weight: 61
labfoldernumber: "06"
description: >
  Mounting storage as disks and filesystems.
---

There are multiple ways of mounting a disk to a virtual machine. In this section we will mount various disks to our vm.


## Kubernetes Storage

There are multiple methods to provide storage for your virtual machine. For a storage to be attached you have to specify
a volume in the `spec.templates.spec.volumes` block of your virtual machine. This volume must then be referenced as a
device in `spec.templates.spec.domain.devices.disks` or `spec.templates.spec.domain.devices.filesystems`.

Sample configuration of a PersistentVolumeClaim mounted as a disk:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
spec:
  template:
    spec:
      domain:
        devices:
          disks:
            - name: harddisk
              disk: {}
      [...]
      volumes:
        - name: harddisk
          persistentVolumeClaim:
            claimName: harddisk-pvc
```

The following types are the most important volume type which KubeVirt supports:

* cloudInitNoCloud: Attach a cloud-init data source. Requires a proper setup of cloud-init.
* cloudInitConfigDrive: Attach a cloud-init data source. Similar to cloudInitNoCloud this requires a proper setup of cloud-init. The config-drive can be used for Ignition.
* persistentVolumeClaim: Provide persistent storage using a PersistentVolumeClaim.
* dataVolume: Simplifies the process for creating virtual machine disks. Without using DataVolumes the creation of a PersistentVolumeClaim is on behalf of the user.
* ephemeral: Local Copy-On-Write image where the original data is never mutated. KubeVirt stores writes in an ephemeral image on local storage.
* containerDisk: Disk images are pulled and backed by a local store on the node.
* emptyDisk: Attach an empty disk to the virtual machine. An empty disk survives a guest restart but not a virtual machine recreation.
* hostDisk: Allows to attach a disk residing somewhere on the local node.
* configMap: Allows to mount a ConfigMap as a disk or a filesystem.
* secret: Allows to mount a Secret as a disk or a filesystem.
* serviceAccount: Allows to mount a ServiceAccount as a disk or a filesystem.
* downwardMetrics: Expose a limited set of VM and host metrics in a `vhostmd` compatible format to the guest.

You can find more Information about the configuration options of Volumes in the [Volume API Reference](https://kubevirt.io/api-reference/master/definitions.html#_v1_volume). You can find volume examples in the [KubeVirt Storage Documentation](https://kubevirt.io/user-guide/storage/disks_and_volumes/#volumes)


## KubeVirt Disks

Beside other options disk referenced in the `spec.templates.spec.domain.devices` section can be:

* lun: The disk is attached as a LUN device allowing to execute iSCSI command passthrough.
* disk: Expose the volume as a regular disk.
* cdrom: Expose the volume as a cdrom drive (read-only by defaul).
* fileystems: Expose the volume as a filesystem to the VM using virtiofs.

Sample configuration:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-vm
spec:
  template:
    spec:
      domain:
        devices:
          filesystems:
            - name: myconfigmap
              virtiofs: {}
          disks:
            - name: harddisk
              disk: 
                bus: virtio
            - name: lundisk
              lun:
                reservation: true
            - name: cdrom
              cdrom: 
                writable: false
                bus: sata
      [...]
      volumes:
        - name: harddisk
          persistentVolumeClaim:
            claimName: harddisk-pvc
        - name: lundisk
          persistentVolumeClaim:
            claimName: lun-pvc
        - name: cdrom
          persistentVolumeClaim:
            claimName: cdrom-pvc
```

You can find more Information about the configuration options of Disks in the [Disk API Reference](https://kubevirt.io/api-reference/master/definitions.html#_v1_disk). You can find disk examples in the [KubeVirt Storage Documentation](https://kubevirt.io/user-guide/storage/disks_and_volumes/#disks)

{{% alert title="Note" color="info" %}}
In contrast to `disks` a `filesystem` reflects changes in the source to the volume inside the VM. However, it is important
to know that a volume mounted as a filesystem does not allow Live Migration.
{{% /alert %}}


## PersistentVolumeClaims as block or filesystem devices

It might be a confusing as the name filesystem is overloaded. In this section we are talking about the storage mode of a
PersistentVolumeClaim and not the mounting inside our guest. Kubernetes supports the `volumeMode` type `block` and
`filesystem` (default). Whether you can use `block` devices or not depends on your CSI driver supporting block volumes.

* Filesystem: A volume is mounted into Pods into a directory. If the volume is backed by a block device and the device is empty, Kubernetes creates a filesystem on the device before mounting it for the first time.
* Block: Specifies that the volume is a raw block device. Such volume is presented into a Pod as a block device, without any filesystem on it. This mode is useful to provide a Pod the fastest possible way to access a volume, without any filesystem layer between the Pod and the volume.

When creating a DataVolume we can specify whether the disk should be of type `Filesystem` or `Block` device. Requesting a
blank volume will look like this:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk
spec:
  source:
    blank: {}
  storage:
    volumeMode: Block
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 128Mi
```

This specification translate to the PersistentVolumeClaim as following:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk
  [...]
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: "134217728"
  volumeMode: Block
  [...]
```

Clearly seeing that the volumeMode `Block` is requested.


## Mounting Storage

Let us create some storage and mount it to a virtual machine using different options.

{{% alert title="Note" color="info" %}}
Attached storage must be mounted in the virtual machine. The most convenient option is to use a system initialization
tool like cloud-init.
{{% /alert %}}


### {{% task %}} Prepare and create disks and configmaps

First we create a disk using the default `Filesystem` volumeMode. Create the file `dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fs-disk.yaml` with the following content:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fs-disk
spec:
  source:
    blank: {}
  storage:
    volumeMode: Filesystem
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 128Mi
```
Create the filesystem backed disk in the kubernetes cluster:
```shell
kubectl create -f dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fs-disk.yaml --namespace=$USER
```

Second we create a disk using the `Block` volumeMode. Create the file `dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk.yaml` with the following content:
```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk
spec:
  source:
    blank: {}
  storage:
    volumeMode: Block
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 128Mi
```
Create the block storage disk in the kubernetes cluster:
```shell
kubectl create -f dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk.yaml --namespace=$USER
```

Next we create a cloud-init configuration secret. Create a file `cloudinit-userdata.yaml` with the following content:
```yaml
#cloud-config
password: kubevirt
chpasswd: { expire: False }
timezone: Europe/Zurich
```
Create the secret in the kubernetes cluster:
```shell
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit --from-file=userdata=cloudinit-userdata.yaml --namespace=$USER
```

Last we create a ConfigMap with some values for an application. Create a file `cm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-application.yaml` with the following content:
```yaml
kind: ConfigMap 
apiVersion: v1 
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-application
data:
  singlevalue: hello
  application.properties: | 
    my.property=true
    another.property=42
```

Now create the ConfigMap in the kubernetes cluster:
```shell
kubectl create -f cm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-application.yaml --namespace=$USER
```


### {{% task %}} Create a VirtualMachine mounting the storage

We create a virtual machine mounting the following storage:

* Initialization using cloud-init
  * Create mount points for disks, secrets, serviceaccounts and configmaps.
  * Format pvc disks
* Filesystem backed disk to /disks/fs
* Block backed disk to /disks/block
* Secret to /secrets/cloudinit
* ServiceAccount to /serviceaccounts/default
* ConfigMap to /configmaps/application

The VirtualManifest for this looks like the following snipped. Create a file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage.yaml` with the following content:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage
    spec:
      domain:
        devices:
          filesystems:
            - name: serviceaccount-fs
              virtiofs: {}
            - name: cloudinit-fs
              virtiofs: {}
            - name: configmap-fs
              virtiofs: {}
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
            - name: blockdisk
              disk:
                bus: virtio
            - name: fsdisk
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
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:40
        - name: blockdisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk
        - name: fsdisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fs-disk
        - name: serviceaccount-fs
          serviceAccount:
            serviceAccountName: default
        - name: cloudinit-fs
          secret:
            secretName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
        - name: configmap-fs
          configMap:
            name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-application
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              password: kubevirt
              chpasswd: { expire: False }
              bootcmd: 
                - sudo test -z "$(sudo blkid /dev/vdc)" && sudo mkfs -t ext4 -L blockdisk /dev/vdc
                - sudo test -z "$(sudo blkid /dev/vdd)" && sudo mkfs -t ext4 -L fsdisk /dev/vdd
                - sudo mkdir -p /disks/block
                - sudo mkdir -p /disks/fs
                - sudo mkdir -p /serviceaccounts/default
                - sudo mkdir -p /secrets/cloudinit
                - sudo mkdir -p /configmaps/application
                - sudo mount -t virtiofs serviceaccount-fs /serviceaccounts/default
                - sudo mount -t virtiofs cloudinit-fs /secrets/cloudinit
                - sudo mount -t virtiofs configmap-fs /configmaps/application
              mounts:
                - ["/dev/vdc", "/disks/block", "ext4", "defaults,nofail", "0", "2" ]
                - ["/dev/vdd", "/disks/fs", "ext4", "defaults,nofail", "0", "2" ]
```

Create the virtual machine in the kubernetes cluster:
```shell
kubectl create -f vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage.yaml --namespace=$USER
```

Start the virtual machine with:
```shell
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage
```

Open a console to the virtual machine:
```shell
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage
```

After logging in you can examine the virtual machine. First you may check the block device output with `lsblk`:
```shell
lsblk
```
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
zram0  251:0    0  1.9G  0 disk [SWAP]
vda    252:0    0    5G  0 disk 
├─vda1 252:1    0    2M  0 part 
├─vda2 252:2    0  100M  0 part /boot/efi
├─vda3 252:3    0 1000M  0 part /boot
└─vda4 252:4    0  3.9G  0 part /var
                                /home
                                /
vdb    252:16   0    1M  0 disk 
vdc    252:32   0  252M  0 disk /disks/block
vdd    252:48   0  257M  0 disk /disks/fs
```

Another variant to also include the other mounts like is for example listing device usage:
```shell
df -h
```
```
Filesystem         Size  Used Avail Use% Mounted on
/dev/vda4          4.0G  418M  3.1G  12% /
devtmpfs           4.0M     0  4.0M   0% /dev
tmpfs              981M     0  981M   0% /dev/shm
tmpfs              393M  780K  392M   1% /run
tmpfs              981M     0  981M   0% /tmp
/dev/vda3          966M   74M  827M   9% /boot
/dev/vda2          100M   17M   84M  17% /boot/efi
/dev/vda4          4.0G  418M  3.1G  12% /home
/dev/vda4          4.0G  418M  3.1G  12% /var
serviceaccount-fs   16G   12K   16G   1% /serviceaccounts/default
cloudinit-fs        16G  4.0K   16G   1% /secrets/cloudinit
/dev/vdc           119M   15K  110M   1% /disks/block
/dev/vdd           115M   46K  106M   1% /disks/fs
tmpfs              197M  4.0K  197M   1% /run/user/1000
configmap-fs       226G  123G   94G  57% /configmaps/application
```

Next try to edit the config map within kubernetes. You can open a new terminal in your webshell or leave the console
and head back later. Issue the following command to alter the ConfigMap:
```shell
kubectl patch cm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-application --type json --patch '[{ "op": "replace", "path": "/data/singlevalue", "value": "kubevirt-training" }]'
```

After some time the change should be seamlessly be propagated to your vm. Head back to the console and check the value
of your mounted config map:
```shell
cat /configmaps/application/singlevalue
```
```
kubevirt-training
```


### Analyze mounting behaviour

Another thing to note is how our virtual machine pods is set up. Leave the console and describe the `virt-launcher` pod
responsible for your vm:
```shell
kubectl describe pod virt-launcher-{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage-qfmlk --namespace=$USER
```
```
Name:             virt-launcher-{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage-qfmlk
[...]
Containers:
  [...]
  compute:
    [...]
    Mounts:
      /var/run/kubevirt-private/vmi-disks/fsdisk from fsdisk (rw)
      /var/run/kubevirt/virtiofs-containers from virtiofs-containers (rw)
      [...]
    Devices:
      /dev/blockdisk from blockdisk
  virtiofs-serviceaccount-fs:
    [...]
    Command:
      /usr/libexec/virtiofsd
    Args:
      --socket-path=/var/run/kubevirt/virtiofs-containers/serviceaccount-fs.sock
      --shared-dir=/var/run/secrets/kubernetes.io/serviceaccount/
      --cache=auto
      --sandbox=none
    Mounts:
      /var/run/kubevirt/virtiofs-containers from virtiofs-containers (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mmf6t (ro)
  virtiofs-cloudinit-fs:
    [...]
    Mounts:
      /var/run/kubevirt-private/secret/cloudinit-fs from cloudinit-fs (rw)
      /var/run/kubevirt/virtiofs-containers from virtiofs-containers (rw)
      [...]
  virtiofs-configmap-fs:
    [...]
    Mounts:
      /var/run/kubevirt-private/config-map/configmap-fs from configmap-fs (rw)
      /var/run/kubevirt/virtiofs-containers from virtiofs-containers (rw)
      [...]
Volumes:
  [...]
  cloudinit-fs:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
  configmap-fs:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-application
  blockdisk:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-block-disk
  fsdisk:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fs-disk
  virtiofs-containers:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
  kube-api-access-mmf6t:
    Type:                    Projected (a volume that contains injected data from multiple sources)
````

This is a highly shortened output of the pod description showing the relevant sections. Our virt-launcher pod has multiple
helper containers managing the mounts. Every storage is shown in the volumes block. The ServiceAccount is actually in
the `kube-api-access` volume. The `compute` container is the container running our virtual machine where our storage
should be available.


#### Disk Mounts: Block storage and Filesystem storage

Our two PersistentVolumeClaims - one backed by Filesystem and one beeing a raw block device - are mounted differently.

* **fsdisk**: Is in the Mount block of the `compute` container.
* **blockdisk**: Our block storage disk is not show in the Mount block of the `compute` container. However, we see the `blockdisk` as `/dev/blockdisk` in the Devices section in the `compute` container. This means that our block storage is indeed passed as a device.


#### Filesystem Mounts: Virtiofs volumes

Our Secret, ConfigMap, ServiceAccount are mounted to the guest as `filesystem`. They are using the shared filesystem [Virtiofs](https://virtio-fs.gitlab.io/).
For each mounted filesystem there is a supporting container `virtiofs-<volume>`. In our case the supporting containers are
`virtiofs-serviceaccount-fs`, `virtiofs-cloudinit-fs`, `virtiofs-configmap-fs`.

Let's analyze the `virtiofs-serviceaccount-fs` and `compute` container.

```
Containers:
  compute:
    [...]
    Mounts:
      /var/run/kubevirt/virtiofs-containers from virtiofs-containers (rw)
      [...]
  virtiofs-serviceaccount-fs:
    [...]
    Command:
      /usr/libexec/virtiofsd
    Args:
      --socket-path=/var/run/kubevirt/virtiofs-containers/serviceaccount-fs.sock
      --shared-dir=/var/run/secrets/kubernetes.io/serviceaccount/
      --cache=auto
      --sandbox=none
    Mounts:
      /var/run/kubevirt/virtiofs-containers from virtiofs-containers (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mmf6t (ro)
```

* The container is running the `virtiofsd` file system daemon (See Command)
* The `virtiofsd` has access to the underlying data (in our case ServiceAccount) as it mounts the data (See Mounts in `virtiofs-serviceaccount-fs`).
* The `virtiofsd` is creating/using the socket `/var/run/kubevirt/virtiofs-containers/serviceaccount-fs.sock`
* This socket is on the `virtiofs-containers` volume (See Args and Mount in `virtiofs-serviceaccount-fs`)
* This volume is also mounted by the `compute` container (see Mount section in `compute`).

This establishes a communication channel using sockets. This channel is used by QEMU from the `compute` container to
communicate with `virtiofsd` in `virtiofs-serviceaccount-fs`.

![Virtiofs Design](../virtiofs.png)


## Stop your VM

As we are at the end of this section it is time to stop your running virtual machine with:
```yaml
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-storage
```
