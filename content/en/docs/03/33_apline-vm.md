---
title: "3.3 Using the Alpine Disk"
weight: 330
labfoldernumber: "03"
sectionnumber: 3.3
description: >
  Create a VirtualMachine using the provisioned Alpine Cloud Disk Image.
---

In the previous section we have provisioned a custom alpine cloud disk image. We will now create a VM which uses this
disk image.


## {{% task %}} Write a VM attaching our disk

With your knowledge write a new VirtualMachine manifest named `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine.yaml`.

{{% alert title="Note" color="info" %}}  
This Alpine Cloud Image has cloud-init included. We will need another disk (cloudinitdisk) which configures our environment.
We will look into cloud-init in a later section. For now, just use the cloudinitdisk specification from the snippet below.
{{% /alert %}}

Block needed to reference the pvc as a disk and the `cloudinitdisk`:
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

You will have to reference the `cloudinitdisk` as a second disk in `spec.template.spec.domain.devices.disks`

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
{{% /details %}}


## {{% task %}} Create and start the VM

Create the vm in the kubernetes cluster:
```shell
kubectl create -f {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine.yaml
```

Start your vm with:
```shell
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine
```


## {{% task %}} Testing our VM

Open the console of your VM:
```shell
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine
```

After some time your VM should be successfully be provisioned with the alpine cloud disk image.
You should be able to successfully login with user `alpine` and the configured password.


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Stop your running VM with
```shell
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpine
```

Delete your DataVolume which will delete the PVC and free the diskspace.
```shell
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-alpinedisk
```
{{% /alert %}}