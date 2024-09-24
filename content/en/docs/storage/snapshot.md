---
title: "VM Snapshot and Restore"
weight: 64
labfoldernumber: "06"
description: >
  Snapshot and restore a virtual machine
---

KubeVirt provides a snapshot and restore functionality. This feature is only available if your storage driver supports `VolumeSnapshots` and a `VolumeSnapshotClass` is configured.

You can list the available `VolumeSnapshotClass` with:
```yaml
kubectl get volumesnapshotclass --namespace=$USER
```
```
NAME                    DRIVER               DELETIONPOLICY   AGE
longhorn-snapshot-vsc   driver.longhorn.io   Delete           21d
```

You can snapshot virtual machines in a running state or in the stopped state. Using the QEMU guest agent the snapshot can
temporarily freeze your VM to get a consistent backup.


## {{% task %}} Prepare a virtual machine with a persistent disk

We want to snapshot a virtual machine with a persistent disk. We need to prepare our disk and virtual machine to be
snapshotted.


### {{% task %}} Prepare persistent disk

First, we need to create a persistent disk. We use a DataVolume which imports a container disk and saves it to a persistent
volume. Create a file `dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros.yaml` with the
following specification:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk
spec:
  source:
    registry:
      url: "docker://{{% param "cirrosCDI" %}}"
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 256Mi
```

Create the data volume with:
```shell
kubectl create -f dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk.yaml --namespace=$USER
```


### {{% task %}} Create a virtual machine using the provisioned disk

Create a file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot.yaml` for your virtual
machine and add the following content:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
    spec:
      domain:
        devices:
          disks:
            - name: harddisk
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
            memory: 64M
      networks:
        - name: default
          pod: {}
      volumes:
        - name: harddisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
```

Create the virtual machine with:
```shell
kubectl create -f vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot.yaml --namespace=$USER
```

Start your virtual machine with:
```shell
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```


### {{% task %}} Edit a file in your virtual machine

Now we make a file change and validate if the change is persistent.

Enter the virtual machine with:
```yaml
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

Whenever you see the login prompt CirrOS shows the user and the default password.
```shell
  ____               ____  ____
 / __/ __ ____ ____ / __ \/ __/
/ /__ / // __// __// /_/ /\ \ 
\___//_//_/  /_/   \____/___/ 
   http://cirros-cloud.net


login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
```

Let's get rid of this message and replace it with our own. Login with the credentials and change our `/etc/issue` file.
```shell
sudo cp /etc/issue /etc/issue.orig
echo "Greetings from the KubeVirt Training. This is a CirrOS virtual machine." | sudo tee /etc/issue
```

Check that the greeting is printed correctly by logging out:
```shell
exit
```
```
Greetings from the KubeVirt Training. This is a CirrOS virtual machine.
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot login:
```

Now restart the virtual machine to verify the change was persistent.
```shell
sudo reboot
```
After the restart completed you should see your new Greeting message.
```
  ____               ____  ____
 / __/ __ ____ ____ / __ \/ __/
/ /__ / // __// __// /_/ /\ \ 
\___//_//_/  /_/   \____/___/ 
   http://cirros-cloud.net


Greetings from the KubeVirt Training. This is a CirrOS virtual machine.
```


## {{% task %}} Create a snapshot of the virtual machine

This is our configuration we want to save. We now create a snapshot of the virtual machine at this time.

Create a file `vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap.yaml` with the following content:

```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
```

Start the snapshot process by creating the VirtualMachineSnapshot:
```yaml
kubectl create -f vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap.yaml
```

Make sure you wait until the snapshot is ready. You can issue the following command to wait until the snapshot is ready:
```shell
kubectl wait vmsnapshot {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap --for condition=Ready
```
It should complete with:
```
virtualmachinesnapshot.snapshot.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap condition met
```

You can list your snapshots with:
```shell
kubectl get virtualmachinesnapshot --namespace=$USER
```
The output should be similar to:
```
NAME                  SOURCEKIND       SOURCENAME       PHASE       READYTOUSE   CREATIONTIME   ERROR
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap   VirtualMachine   {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot   Succeeded   true         102s 
```

You can describe the resource and have a look at the status of the `VirtualMachineSnapshot` and its
subresource `VirtualMachineSnapshotContent`.
```shell
kubectl describe virtualmachinesnapshot --namespace=$USER
kubectl describe virtualmachinesnapshotcontent --namespace=$USER
```

```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  [...]
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap
status:
  [...]
  creationTime: "2024-09-05T08:34:31Z"
  indications:
    - Online
    - NoGuestAgent
  phase: Succeeded
  readyToUse: true
  snapshotVolumes:
    excludedVolumes:
      - cloudinitdisk
    includedVolumes:
      - harddisk
  [...]
```

For example in the status of the VirtualMachineSnapshot description you may find information what volumes are in the snapshot.

* `status.indications`: Information how the snapshot was made.
  * `Online`: The VM was running during snapshot creation.
  * `GuestAgent` QEMU guest agent was running during snapshot creation.
  * `NoGuestAgent` QEMU guest agent was not running during snapshot creation or the QEMU guest agent could not be used due to an error.
* `status.snapshotVolumes`: Information of which volumes are included

Snapshots also include your virtual machine metadata `spec.template.metadata` and the specification `spec.template.spec`.


## {{% task %}} Changing our Greeting message again

Enter the virtual machine with:
```yaml
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

Change the Greeting message again:
```shell
sudo cp /etc/issue /etc/issue.bak
echo "Hello" | sudo tee /etc/issue
```

Now restart the virtual machine again and verify the change was persistent.
```shell
sudo reboot
```
After the restart completed you should see your new Hello message.

Beside changing a file we add a label `acend.ch/training: kubevirt` to our VirtualMachine metadata `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot`.

You can do this by patching your virtual machine with:
```shell
kubectl patch virtualmachine {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --type='json' -p='[{"op": "add", "path": "/spec/template/metadata/labels/acend.ch~1training", "value":"kubevirt"}]' --namespace=$USER
```
```
virtualmachine.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot patched
```

Describe the virtual machine to check if the label is present:
```shell
kubectl describe virtualmachine {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```
```
API Version:  kubevirt.io/v1
Kind:         VirtualMachine
Name:         {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
[...]
Spec:
  Running:  true
  Template:
    Metadata:
      Labels:
        acend.ch/training:   kubevirt
        kubevirt.io/domain:  {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
[...]
```


## {{% task %}} Restoring a Virtual Machine

Now you decide to restore your snapshot. Make sure your virtual machine is stopped.

```shell
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

Create the file `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore.yaml` with the following content:
```yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
  virtualMachineSnapshotName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap
```

Start the restore process by creating the VirtualMachineRestore:
```shell
kubectl create -f vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore.yaml --namespace=$USER
```

Make sure you wait until the restore is done. You can use the following command to wait until the restore is finished:
```shell
kubectl wait vmrestore {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore --for condition=Ready --namespace=$USER
```
It should complete with:
```
virtualmachinerestore.snapshot.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore condition met
```


## {{% task %}} Check the restored virtual machine

Start the virtual machine again with:
```shell
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

Whenever the restore was successful out `Hello` greeting should be gone and we should see the following Greeting again.
Open the console to check the greeting:
```
  ____               ____  ____
 / __/ __ ____ ____ / __ \/ __/
/ /__ / // __// __// /_/ /\ \ 
\___//_//_/  /_/   \____/___/ 
   http://cirros-cloud.net


Greetings from the KubeVirt Training. This is a CirrOS virtual machine.
```

What about the label on the virtual machine manifest? Describe the virtual machine with and validate that it has been removed as well:
```shell
kubectl describe virtualmachine {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```
```
API Version:  kubevirt.io/v1
Kind:         VirtualMachine
Name:         {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
[...]
Spec:
  Running:  true
  Template:
    Metadata:
      Labels:
        kubevirt.io/domain:  {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot
[...]
```
