---
title: "Virtual machine replica sets"
weight: 53
labfoldernumber: "05"
description: >
  Using virtual machine replica sets
---

Just like a VirtualMachinePool, a VirtualMachineInstanceReplicaSet resource tries to ensure that a specified number of virtual machines
are always in a ready state. The `VirtualMachineInstanceReplicaSet` is very similar to the Kubernetes ReplicaSet[^1].

However, the VirtualMachineInstanceReplicaSet does not maintain any state or provide guarantees about the maximum number of VMs
running at any given time. For instance, the VirtualMachineInstanceReplicaSet may initiate new replicas if it detects that some VMs have entered
an unknown state, even if those VMs might still be running.


## Using a VirtualMachineInstanceReplicaSet

Using the custom resource VirtualMachineInstanceReplicaSet, we can specify a template for our VM. A VirtualMachineInstanceReplicaSet consists of a
VM specification just like a regular VirtualMachine. This specification resides in `spec.template`.
Besides the VM specification, the replica set requires some additional metadata like labels to keep track of the VMs in the replica set.
This metadata resides in `spec.template.metadata`.

The amount of VMs we want the replica set to manage is specified as `spec.replicas`. This number defaults to 1 if it is left empty.
If you change the number of replicas in-flight, the controller will react to it and change the VMs running in the replica set.

The replica set controller needs to keep track of the VMs running in this replica set. This is done by specifying a `spec.selector`. This
selector must match the labels in `spec.template.metadata.labels`.

A basic `VirtualMachineInstanceReplicaSet` template looks like this:

```yaml
apiVersion: kubevirt.io/v1 
kind: VirtualMachineInstanceReplicaSet    
metadata:    
  name: vmi-replicaset   
spec:    
  replicas: 2 # desired instances in the replica set
  selector:    
    # VirtualMachineInstanceReplicaSet selector
  template:    
    metadata:    
      # VirtualMachineInstance metadata
    spec:    
      # VirtualMachineInstance Template
[...]
```

{{% alert title="Note" color="info" %}}
Be aware that if `spec.selector` does not match `spec.template.metadata.labels`, the controller will do nothing
except log an error. Further, it is your responsibility to not create two `VirtualMachineInstanceReplicaSet` conflicting with each other.
{{% /alert %}}

As an example, this could look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceReplicaSet
metadata:    
  name: vmi-cirros-replicaset
spec:
  replicas: 2
  selector:    
    matchLabels:
      kubevirt.io/domain: vmi-cirros
  template:    
    metadata:    
      labels:
        kubevirt.io/domain: vmi-cirros
    spec:
[...]
```


### When to use VirtualMachineInstanceReplicaSet

You should use VirtualMachineInstanceReplicaSet whenever you want multiple exactly identical instances not requiring
persistent disk state. In other words, you should only use replica sets if your VM is ephemeral and every used disk is
read-only. If the VM writes data this should only be allowed in a tmpfs.

{{% alert title="Warning" color="warning" %}}
You should expect data corruption if the VM writes data to a storage not being a tmpfs or an ephemeral type.
{{% /alert %}}

Volume types which can safely be used with replica sets are:

* `cloudInitNoCloud`
* `ephemeral`
* `containerDisk`
* `emptyDisk`
* `configMap`
* `secret`
* any other type, if the VM instance writes internally to a tmpfs

{{% alert title="Note" color="info" %}}
This is the most important difference to a VirtualMachinePool. If you want to manage multiple unique instances using
persistent storage, you have to use a VirtualMachinePool. If you want to manage identical ephemeral instances which do
not require persistent storage or different data sources (startup scripts, configmaps, secrets) you should use a
VirtualMachineInstanceReplicaSet.
{{% /alert %}}


## Using a VirtualMachineInstanceReplicaSet

We will create a VirtualMachineInstanceReplicaSet using a cirros container disk from a container registry. As we know,
container disks are ephemeral, this fits our use case very well.


### {{% task %}} Create a VirtualMachineInstanceReplicaSet

Create a file `vmirs_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` and start with the following boilerplate config:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceReplicaSet
metadata:    
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset
spec:
  replicas: 2
  selector:
    matchLabels:
      kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros
    spec:
[...]
```

Enhance the `spec.template.spec` block to start a VM matching these criteria:

* Use the container disk {{% param "cirrosCDI" %}}
* Use an empty `cloudInitNoCloud` block
* Use `1` replicas
* Configure guest to have `1` cores
* Resources
  * Request `265Mi` of memory
  * Request `100m` of cpu
  * Limit `300m` of cpu

Use this empty `cloudInitNoCloud` block to prevent cirros from trying to instantiate by using a remote url:

```yaml
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
```

{{% onlyWhen tolerations %}}

{{% alert title="Tolerations" color="warning" %}}
Don't forget the `tolerations` from the setup chapter to make sure the VM will be scheduled on one of the baremetal nodes.
{{% /alert %}}

{{% /onlyWhen %}}

{{% details title="Task Hint" %}}
Your VirtualMachineInstanceReplicaSet should look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstanceReplicaSet
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset
spec:
  replicas: 2
  selector:
    matchLabels:
      kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros
    spec:
      domain:
        cpu:
          cores: 1
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: default
            masquerade: {}
        resources:
          limits:
            cpu: 300m
          requests:
            cpu: 100m
            memory: 265Mi
      networks:
      - name: default
        pod: {}
      {{< onlyWhen tolerations >}}tolerations:
            - effect: NoSchedule
              key: baremetal
              operator: Equal
              value: "true"
      {{< /onlyWhen >}}volumes:
      - name: containerdisk
        containerDisk:
          image: {{% param "cirrosCDI" %}}
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
```
{{% /details %}}

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vmirs_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros.yaml --namespace=$USER
```

```
virtualmachineinstancereplicaset.kubevirt.io/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset created
```


### {{% task %}} Access the VirtualMachineInstanceReplicaSet

There is not much the cirros disk image provides besides entering the VMs using the console.

Check the availability of the `VirtualMachineInstanceReplicaSet` with:

```bash
kubectl get vmirs --namespace=$USER
```

```
NAME                      DESIRED   CURRENT   READY   AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset   2         2         2       1m
```

List the created VirtualMachineInstances using:

```bash
kubectl get vmi --namespace=$USER
```

```
NAME                           AGE    PHASE     IP             NODENAME            READY
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicasetnc5p5   11m    Running   10.244.3.96    training-worker-0   True
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicasetp25s2   11m    Running   10.244.3.149   training-worker-0   True
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicasetsn4f5   8m6s   Running   10.244.3.249   training-worker-0   True
```

You can access the console using the name of the VMI with `virtctl`:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicasetnc5p5 --namespace=$USER
```


## Scaling the VirtualMachineInstanceReplicaSet

As the VirtualMachineInstanceReplicaSet implements the Kubernetes standard `scale` sub-command, you could scale the VirtualMachineInstanceReplicaSet using:

```bash
kubectl scale vmirs {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset --replicas 1 --namespace=$USER
```

{{% alert title="Note" color="info" %}}
Scaling is currently not possible with regular user permissions. You can change the replica count by editing the
VirtualMachineInstanceReplicaSet.

```shell
kubectl edit vmirs {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset --namespace=$USER 
```
{{% /alert %}}


## Horizontal pod autoscaler

The HorizontalPodAutoscaler (HPA)[^1] resource can be used to manage the replica count depending on resource usage:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset
spec:
  maxReplicas: 2
  minReplicas: 1
  scaleTargetRef:
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstanceReplicaSet
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset
  targetCPUUtilizationPercentage: 75
```

This will ensure that the VirtualMachineInstanceReplicaSet is automatically scaled depending on the CPU utilization.

You can check the consumption of your pods with:

```bash
kubectl top pod --namespace=$USER
```

```
NAME                                               CPU(cores)   MEMORY(bytes)   
user2-webshell-f8b44dfdc-92qjj                     6m           188Mi           
virt-launcher-lab06-cirros-replicasetck6rw-9s8wd   3m           229Mi           
```


### {{% task %}} Enable the HorizontalPodAutoscaler

Create a file `hpa_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset
spec:
  maxReplicas: 2
  minReplicas: 1
  scaleTargetRef:
    apiVersion: kubevirt.io/v1
    kind: VirtualMachineInstanceReplicaSet
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset
  targetCPUUtilizationPercentage: 75
```

Create the HorizontalPodAutoscaler in the cluster:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/hpa_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros.yaml --namespace=$USER
```

```
horizontalpodautoscaler.autoscaling/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset created
```

Check the status of the HorizontalPodAutoscaler with:

```bash
kubectl get hpa --namespace=$USER
```

```
NAME                      REFERENCE                                                  TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset   VirtualMachineInstanceReplicaSet/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset   cpu: 2%/75%   1         2         1          7m44s
```

Open a second terminal in the webshell and connect to the console of one of your VM instances:

```bash
kubectl get vmi --namespace=$USER
```

```
NAME                           AGE     PHASE     IP             NODENAME            READY
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicasetck6rw   9m47s   Running   10.244.3.171   training-worker-0   True
```

Pick the VMI and open the console:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset<pod> --namespace=$USER
```

Start to generate some load. Issue the following command in your webshell:

```bash
load() { dd if=/dev/zero of=/dev/null & }; load; read; killall dd
```

In the other webshell check the following commands regularly:

```bash
kubectl top pod --namespace=$USER
```

```bash
kubectl get hpa --namespace=$USER
```

After a short delay the HorizontalPodAutoscaler kicks in and scales your replica set to `2`:

```
NAME                      REFERENCE                                                  TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset   VirtualMachineInstanceReplicaSet/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset   cpu: 283%/75%   1         2         2          11m
```

And you will see that a second VMI is started:

```bash
kubectl get vmi --namespace=$USER
```

After the horizontal pod autoscaler scaled up your instances, head over to the console where you generated the load.
Hit `enter` in the console to stop the load generation. By default, the horizontal pod autoscaler tries to stabilize
the replica set by using a `stabilizationWindowSeconds` of 300 seconds. This means that it will keep the replica set stable
for at least 300 seconds before issuing a scale down. For more information about the configuration, head over to
the [Horizontal pod autoscaler documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/).  


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Delete your `VirtualMachinePool`:

```bash
kubectl delete vmpool {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webserver --namespace=$USER
```

Delete your `VirtualMachineInstanceReplicaSet`:

```bash
kubectl delete vmirs {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset --namespace=$USER
```

Delete the horizontal pod autoscaler

```bash
kubectl delete hpa {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cirros-replicaset --namespace=$USER
```
{{% /alert %}}

[^1]: [Kubernetes ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
