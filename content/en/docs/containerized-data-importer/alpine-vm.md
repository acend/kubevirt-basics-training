---
title: "Using the Alpine disk"
weight: 23
labfoldernumber: "02"
description: >
  Create a virtual machine using the provisioned Alpine Cloud Disk Image
---

In the previous section we have provisioned a custom Alpine cloud disk image. We will now create a VM which uses this
disk image.


## {{% task %}} Attach the disk

Write a new VirtualMachine manifest named `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine.yaml` in the directory `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}`.

{{% alert title="Note" color="info" %}}  
This Alpine Cloud Image has cloud-init included. We will need another disk (cloudinitdisk) which configures our environment.
We will look into cloud-init in a later section. For now, just use the cloudinitdisk specification from the snippet below.
{{% /alert %}}

Block needed to reference the PVC as a disk and the `cloudinitdisk`:

```yaml
      volumes:
        - name: alpinedisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk
        - name: cloudinitdisk
          cloudInitNoCloud:
             userData: |-
                #cloud-config
                password: {{% param "dummypwd" %}}
                chpasswd: { expire: False }
```

You will have to reference the `cloudinitdisk` as a second disk in `spec.template.spec.domain.devices.disks`.

{{% onlyWhen tolerations %}}

{{% alert title="Tolerations" color="warning" %}}
Don't forget the `tolerations` from the setup chapter to make sure the VM will be scheduled on one of the baremetal nodes.
{{% /alert %}}

{{% /onlyWhen %}}

{{% details title="Task Hint" %}}
Your yaml should look like this:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine
spec:
  running: false
  template:
    spec:
      domain:
        devices:
          disks:
            - name: alpinedisk
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
            memory: 256M
      networks:
        - name: default
          pod: {}
      {{< onlyWhen tolerations >}}tolerations:
        - effect: NoSchedule
          key: baremetal
          operator: Equal
          value: "true"
      {{< /onlyWhen >}}volumes:
        - name: alpinedisk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |-
              #cloud-config
              password: {{% param "dummypwd" %}}
              chpasswd: { expire: False }
```
{{% /details %}}


## {{% task %}} Create and start the VM

Create the VM on the Kubernetes cluster:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine.yaml --namespace=$USER
```

Start your VM with:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine --namespace=$USER
```


## {{% task %}} Testing the VM

Open the VM's console:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine --namespace=$USER
```

After some time your VM should be successfully provisioned with the Alpine cloud disk image.
You should be able to successfully log in with user `alpine` and the configured password.


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Stop your running VM with:

```bash
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine --namespace=$USER
```

Delete your DataVolume which will delete the PVC and free the diskspace:

```bash
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk --namespace=$USER
```
{{% /alert %}}
