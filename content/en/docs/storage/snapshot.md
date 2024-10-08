---
title: "VM snapshot and restore"
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

You can snapshot virtual machines in running or stopped state. Using the QEMU guest agent the snapshot can
temporarily freeze your VM to get a consistent backup.


## {{% task %}} Prepare a virtual machine with a persistent disk

We want to snapshot a virtual machine with a persistent disk. We need to prepare our disk and virtual machine to be
snapshotted.


### {{% task %}} Prepare persistent disk

First, we need to create a persistent disk. We use a DataVolume which imports a container disk and saves it to a persistent
volume. Create a file `dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following specification:

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

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/dv_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk.yaml --namespace=$USER
```


### {{% task %}} Create a virtual machine using the provisioned disk

Create a file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` for your virtual machine and add the following content:

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
      {{< onlyWhen tolerations >}}tolerations:
        - effect: NoSchedule
          key: baremetal
          operator: Equal
          value: "true"
      {{< /onlyWhen >}}volumes:
        - name: harddisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-disk
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
```

Create the virtual machine with:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot.yaml --namespace=$USER
```

Start your virtual machine with:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```


### {{% task %}} Edit a file in your virtual machine

We now make a file change and validate if the change is persistent.

Enter the virtual machine with:

```yaml
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

CirrOS' login prompt always show the user and default password:

```bash
  ____               ____  ____
 / __/ __ ____ ____ / __ \/ __/
/ /__ / // __// __// /_/ /\ \ 
\___//_//_/  /_/   \____/___/ 
   http://cirros-cloud.net


login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
```

Let's get rid of this message and replace it with our own. Log in with the credentials and change our `/etc/issue` file:

```bash
sudo cp /etc/issue /etc/issue.orig
echo "Greetings from the KubeVirt Training. This is a CirrOS virtual machine." | sudo tee /etc/issue
```

Check that the greeting is printed correctly by logging out:

```bash
exit
```

```
Greetings from the KubeVirt Training. This is a CirrOS virtual machine.
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot login:
```

Log in again and restart the virtual machine to verify the change was persistent:

```bash
sudo reboot
```

After the restart completed, you should see your new message:

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

Create a file `vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

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

Start the snapshot process by creating the VirtualMachineSnapshot resource:

```yaml
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap.yaml
```

Make sure you wait until the snapshot is ready. You can issue the following command to wait for that to happen:

```bash
kubectl wait vmsnapshot {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap --for condition=Ready
```

It should complete with:

```
virtualmachinesnapshot.snapshot.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap condition met
```

You can list your snapshots with:

```bash
kubectl get virtualmachinesnapshot --namespace=$USER
```

The output should be similar to:

```
NAME                  SOURCEKIND       SOURCENAME       PHASE       READYTOUSE   CREATIONTIME   ERROR
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-snap   VirtualMachine   {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot   Succeeded   true         102s 
```

You can describe the resource and have a look at the status of the `VirtualMachineSnapshot` and its
subresource `VirtualMachineSnapshotContent`.

```bash
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

In the status of the VirtualMachineSnapshot description, you may find information what volumes reside in the snapshot.

* `status.indications`: Information how the snapshot was made
  * `Online`: Indicates that the VM was running during snapshot creation
  * `GuestAgent` Indicates that the QEMU guest agent was running during snapshot creation
  * `NoGuestAgent` Indicates that the QEMU guest agent was not running during snapshot creation or the QEMU guest agent could not be used due to an error
* `status.snapshotVolumes`: Information of which volumes are included

Snapshots also include your virtual machine metadata `spec.template.metadata` and the specification `spec.template.spec`.


## {{% task %}} Changing our greeting message again

Enter the virtual machine with:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

Change the greeting message again:

```bash
sudo cp /etc/issue /etc/issue.bak
echo "Hello" | sudo tee /etc/issue
```

Now restart the virtual machine and verify the change was persistent:

```bash
sudo reboot
```

After the restart completed, you should see your new `Hello` message.

In addition to the changed file containing the greeting message, add a label `acend.ch/training: kubevirt` to the VirtualMachine's metadata `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot`. This will allow us to see what happens to the labels once we restore the previous snapshot.

You can do this by patching your virtual machine with:

```bash
kubectl patch virtualmachine {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --type='json' -p='[{"op": "add", "path": "/spec/template/metadata/labels/acend.ch~1training", "value":"kubevirt"}]' --namespace=$USER
```

```
virtualmachine.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot patched
```

Describe the virtual machine to check if the label is present:

```bash
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


## {{% task %}} Restoring a virtual machine

Before we restore the virtual machine from the snapshot, let us do a quick recap:

1. We provisioned a CirrOS VM with an attached persistent volume
1. We then changed the greeting Message to `Greetings from the KubeVirt Training. This is a CirrOS virtual machine.`
1. We created a snapshot from that volume
1. We change the greeting Message to `Hello` and added a label `acend.ch/training: kubevirt`

Now we want to restore the snapshot from step 3. Make sure your virtual machine is stopped:

```bash
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

Create the file `vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

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

Start the restore process by creating a VirtualMachineRestore resource:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vmsnapshot_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore.yaml --namespace=$USER
```

Make sure you wait until the restore is done. You can use the following command to wait until the restore is finished:

```bash
kubectl wait vmrestore {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore --for condition=Ready --namespace=$USER
```

It should complete with:

```
virtualmachinerestore.snapshot.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot-restore condition met
```


## {{% task %}} Check the restored virtual machine

Start the virtual machine:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

If the restore was successful the `Hello` greeting should be gone and we should see below message again.
Open the console to check the greeting message:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-snapshot --namespace=$USER
```

```
  ____               ____  ____
 / __/ __ ____ ____ / __ \/ __/
/ /__ / // __// __// /_/ /\ \ 
\___//_//_/  /_/   \____/___/ 
   http://cirros-cloud.net


Greetings from the KubeVirt Training. This is a CirrOS virtual machine.
```

What about the label on the virtual machine manifest? Describe the virtual machine and validate that it has been removed as well:

```bash
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
