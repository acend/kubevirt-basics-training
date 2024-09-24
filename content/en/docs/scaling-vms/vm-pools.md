---
title: "VirtualMachine Pools"
weight: 52
labfoldernumber: "05"
description: >
  Using VirtualMachine Pools
---

A VirtualMachinePool tries to ensure that a specified number of virtual machines are always in a ready state.

However, the VirtualMachinePool does not maintain any state or provide guarantees about the maximum number of VMs
running at any given time. For instance, the pool may initiate new replicas if it detects that some VMs have entered
an unknown state, even if those VMs might still be running.


## Using a VirtualMachinePool

Using the custom resource `VirtualMachinePool` we can specify a template for our VM. A VirtualMachinePool consists of a
vm specification just like a regular `VirtualMachine`. This specification resides in `spec.virtualMachineTemplate.spec`.
Beside the VM specification the pool requires some additional metadata like labels to keep track of the VMs in the pool.
This metadata resides in `spec.virtualMachineTemplate.metadata`.

The amount of VMs we want the pool to manage is specified as `spec.replicas`. This number defaults to `1` if it is left empty.
If you change the number of replicas in-flight, the controller will react to it and change the VMs running in the pool.

The pool controller needs to keep track of VMs running in this pool. This is done by specifying a `spec.selector`. This
selector must match the labels in `spec.virtualMachineTemplate.metadata.labels`.

A basic `VirtualMachinePool` template looks like this:
```yaml
apiVersion: pool.kubevirt.io/v1alpha1    
kind: VirtualMachinePool    
metadata:    
  name: virtualmachine-pool   
spec:    
  replicas: 2 # desired instanced in the pool
  selector:    
    # VirtualMachinePool selector
  virtualMachineTemplate:    
    metadata:    
      # VirtualMachine metadata
    spec:    
      # VirtualMachine Template
[...]
```

{{% alert title="Note" color="info" %}}
Be aware that if `spec.selector` does not match `spec.virtualMachineTemplate.metadata.labels` the controller will do nothing
except logging an error. Further, it is your responsibility to not create two `VirtualMachinePool`s conflicting with each other.
{{% /alert %}}

To avoid conflicts a common practice is to use the label `kubevirt.io/vmpool` and simply set it to the `metadata.name` of the `VirtualMachinePool`.

As an example this could look like this:
```yaml
apiVersion: pool.kubevirt.io/v1alpha1    
kind: VirtualMachinePool    
metadata:    
  name: my-webserver-pool    
spec:
  replicas: 2
  selector:    
    matchLabels:    
      kubevirt.io/vmpool: my-webserver-pool
  virtualMachineTemplate:    
    metadata:    
      labels:    
        kubevirt.io/vmpool: my-webserver-pool
    spec:    
      template:    
        metadata:    
          labels:    
            kubevirt.io/vmpool: my-webserver-pool
[...]
```


## {{% task %}} Preparation for our Virtual Machine

At the beginning of this lab we created a custom disk based on Fedora Cloud with nginx installed. We will use this image
for our VirtualMachinePool. We still have to use cloud-init to configure our login-credentials.

Since we have done this in a previous lab you can practice creating a cloud-init. The script should:

* Set a password and configure it to not expire
* Set the timezone to `Europe/Zurich`

{{% details title="Task Hint" %}}
Create a file `cloudinit-userdata.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:
```yaml
#cloud-config
password: kubevirt
chpasswd: { expire: False }
timezone: Europe/Zurich
```

Create the secret:
```shell
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/cloudinit-userdata.yaml --namespace=$USER
```
{{% /details %}}


### {{% task %}} Create a VirtualMachinePool

Now we have all our prerequisites in place and are ready to create our virtual machine pool.

Create a file `vmpool_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` and start with the following boilerplate config:
```yaml
apiVersion: pool.kubevirt.io/v1alpha1    
kind: VirtualMachinePool    
metadata:    
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
spec:
  replicas: 2
  selector:    
    matchLabels:    
      kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
  virtualMachineTemplate:    
    metadata:    
      labels:    
        kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
    spec:
      template:    
        metadata:    
          labels:    
            kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
[...]
```

Now edit the section `spec.virtualMachineTemplate.spec` to specify your virtual machine. You can have a look at the cloud-init
vm from the previous lab. Make sure the vm has the following characteristics:

* Use a dataVolumeTemplate to clone `fedora-cloud-nginx-base` pvc to the `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver-disk` pvc.
* Mount this pvc as the `datavolumedisk`
* Use a `cloudInitNoCloud` named `cloudinitdisk` and reference the created secret to initialize our credentials.

{{% details title="Task Hint" %}}
Your VirtualMachinePool should look like this (Make sure you replace `<user>` to your username):
```yaml
apiVersion: pool.kubevirt.io/v1alpha1
kind: VirtualMachinePool
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
spec:
  replicas: 2
  selector:
    matchLabels:
      kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
  virtualMachineTemplate:
    metadata:
      labels:
        kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
    spec:
      running: true
      template:
        metadata:
          labels:
            kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
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
                claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver-disk
            - name: cloudinitdisk
              cloudInitNoCloud:
                secretRef:
                  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
      dataVolumeTemplates:
      - metadata:
          name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver-disk
        spec:
          pvc:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 6Gi
          source:
            pvc:
              namespace: <user>
              name: fedora-cloud-nginx-base
```
{{% /details %}}

Create the VirtualMachinePool with:
```shell
kubectl create -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vmpool_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver.yaml --namespace=$USER
```
```
virtualmachinepool.pool.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver created
```

As we used `spec.virtualMachineTemplate.spec.dataVolumeTemplates` the VirtualMachinePool will create a disk for each
instance in the pool. As we have configured our replicas to be `2` there should be two disks created. Each disk as its
sequential id as postfix of the disk.

Investigate the availability of our pvc for the webserver instances
```shell
kubectl get pvc --namespace=$USER
```

We see the two disk images to be present in our namespace. This means that each our instance is a completely unique and
independent stateful instance using its own disk.
```
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver-disk-0    Bound    pvc-95931127-195a-4814-82d0-11d604cdceae   6Gi        RWO            longhorn       <unset>                 3m42s
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver-disk-1    Bound    pvc-4469db26-b820-4950-ab9f-7e6534ebfb5c   6Gi        RWO            longhorn       <unset>                 3m42s
```


### {{% task %}} Access the VirtualMachinePool

Create a service to access our webservers from within the webshell.

Create a file `svc_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    kubevirt.io/vmpool: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
  type: ClusterIP
```

Apply the service with:
```shell
kubectl create -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/svc_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver.yaml --namespace=$USER
```
```
service/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver created
```

From within your webshell try to access the service using (make sure you replace `$USER` with your username):
```shell
curl -s {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver.$USER.svc.cluster.local
```
```
Hello from {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver-0
GMT time:   Monday, 02-Sep-2024 14:05:04 GMT
Local time: Monday, 02-Sep-2024 14:05:04 UTC
```

If you issue the request multiple times and watch for the greeting webserver. Do you see that both webservers respond in a
loadbalanced way? This is the default behaviour of kubernetes service.


## Unique Secrets and ConfigMaps

We have seen that the VirtualMachinePool created unique disks for our webserver. However, the referenced secret in the
`cloudInitNoCloud` section is the same and all instances access und use the same secret. If we had used machine specific
settings in this config this would have been a problem.

This is the default behaviour, but it can be changed using `AppendPostfixToSecretReferences` and `AppendPostfixToConfigMapReferences`
in the VirtualMachinePool `spec` section. When these booleans are set to true, the VirtualMachinePool ensures that
references to Secrets or ConfigMaps have the sequential id as postfix. It is your responsibility to pre-generate the
secrets with the postfixes.  


## Scaling the VirtualMachinePool

As the VirtualMachinePool implements the kubernetes standard `scale` subresource you could scale the VirtualMachinePool using
the `kubectl scale` command.

```shell
kubectl scale vmpool {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver --replicas 1 --namespace=$USER
```


## Horizontal Pod Autoscaler

The Horizontal Pod Autoscaler (HPA)[^1] can be used to manage the replica count depending on resource usage.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
spec:
  maxReplicas: 5
  minReplicas: 2
  scaleTargetRef:
    apiVersion: pool.kubevirt.io/v1alpha1
    kind: VirtualMachinePool
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver
  targetCPUUtilizationPercentage: 50
```

This will ensure that the VirtualMachinePool is automatically scaled depending on the CPU utilization.


## {{% task %}} Scale down the VirtualMachinePool

Scale down the VM pool with:
```shell
kubectl scale vmpool {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver --replicas 0 --namespace=$USER
```

[^1]: [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
