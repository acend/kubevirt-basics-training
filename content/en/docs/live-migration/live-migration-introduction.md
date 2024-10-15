---
title: "Introduction"
weight: 70
labfoldernumber: "07"
description: >
  Introduction to the concept of live migration
---

The live migration feature (LiveMigration) is enabled by default in recent KubeVirt versions. Previously it needed to be enabled in the KubeVirt configuration running on the cluster (`kubectl get kubevirt kubevirt -n kubevirt -o yaml`):

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
      - Sidecar
      - CommonInstancetypesDeploymentGate
      - ExperimentalIgnitionSupport
      - HotplugVolumes
      - ExpandDisks
      - Snapshot
      - VMExport
      - BlockVolume
```

Additionally, the live migration feature can be configured under `spec.configuration.migrations` with a set of configuration parameters. The full list can be found in the [API reference](https://kubevirt.io/api-reference/v1.3.0/definitions.html#_v1_migrationconfiguration).

* `bandwidthPerMigration` - BandwidthPerMigration limits the amount of network bandwidth live migrations are allowed to use. The value is in quantity per second.
* `parallelMigrationsPerCluster` - ParallelMigrationsPerCluster is the total number of concurrent live migrations allowed cluster-wide.
* `parallelOutboundMigrationsPerNode` - ParallelOutboundMigrationsPerNode is the maximum number of concurrent outgoing live migrations allowed per node.
* `completionTimeoutPerGiB` - CompletionTimeoutPerGiB is the maximum number of seconds per GiB a migration is allowed to take. If a live-migration takes longer to migrate than this value multiplied by the size of the VMI, the migration will be cancelled, unless AllowPostCopy is true.
* `progressTimeout` - ProgressTimeout is the maximum number of seconds a live migration is allowed to make no progress. Hitting this timeout means a migration transferred 0 data for that many seconds. The migration is then considered stuck and therefore cancelled.

{{% alert title="Limitations" color="warning" %}}

* Virtual machines using PVC must have a `RWX` access mode to be live-migrated
* Additionally, pod network binding of the bridge interface is not allowed. 
* Volumes mounted as filesystems prevent virtual machines from being live migrated
* Directly attached devices
* Live migration requires ports `49152`, `49153` to be available in the `virt-launcher` pod; if these ports are explicitly specified in the masquarade interface, live migration will not work

{{% /alert %}}


## Migration strategies

Live migration is a complex process. During migration, the entire state of the source VM (primarily its memory) must be transferred to the target VM. When sufficient resources, such as network bandwidth and CPU power, are available, the migration should proceed smoothly. However, if resources are lacking, the migration may stall without making further progress.

A key factor influencing migration from the guest VM's perspective is its _dirty rate_: The rate at which the VM modifies memory. A high dirty rate creates a challenge, as memory is continuously transferred to the target while the guest simultaneously modifies it. In such cases, more advanced migration strategies may be needed.

The following 3 migration strategies are currently supported, and can be configured in the KubeVirt configuration:


### Pre-copy

Pre-copy is the default strategy and works like this:

1. The target VM is created, but the guest keeps running on the source VM
1. The source starts sending chunks of VM state (mostly memory) to the target and continues until all of the state has been transferred
1. The guest starts executing on the target VM
1. The source VM is removed

Pre-copy is generally the safest and fastest migration strategy in most scenarios. It supports multi-threading, can be easily canceled, and works efficiently. If there's no strong reason to choose a different approach, pre-copy is typically the best option.

However, in some cases, migrations may struggle to converge. This means that by the time a portion of the source VM’s state reaches the target VM, it may have already been modified by the source VM (the one running the guest). Factors such as a high dirty rate or limited resources, like insufficient network bandwidth or CPU power, can cause this lack of convergence.


### Post-copy

Post-copy is the default strategy and works like this:

1. The target VM is created
1. The guest is run on the target VM
1. The source starts sending chunks of VM state (mostly memory) to the target
1. When the guest, running on the target VM, would access memory:
   * The guest can access it if the memory exists on the target VM
   * Otherwise, the target VM asks for a chunk of memory from the source VM
1. Once all of the memory state is updated on the target VM, the source VM is removed

Post-copy's main idea and advantage is that the VM starts immediately. There are, however, more advantages and disadvantages to this strategy:

Advantages:

* The same memory chunk is never transferred more than once because the guest is already running on the target VM, making it irrelevant if a page is modified.
* As a result, a high dirty rate has significantly less impact.
* It also uses less network bandwidth.

Disadvantages:

* In post-copy migration, the VM state has not one source of truth. While the guest running on the target VM writes to memory, some parts of the state may still reside on the source VM. This creates a risky situation, as the VM state may become unrecoverable if either the target or source VM crash.
* Post-copy also has a slow warm-up phase as no memory is initially available on the target VM. The guest must wait for a significant amount of memory to be transferred in a short time.
* Post-copy is generally slower than pre-copy in most cases.
* It is more difficult to cancel once the migration has started.


### Auto-converge

Auto-converge is a technique designed to accelerate the convergence of pre-copy migrations without altering the fundamental migration process.

Since a high dirty rate is often the primary reason migrations fail to converge, auto-converge addresses this by throttling the guest’s CPU. If the migration is progressing quickly enough, the CPU is either not throttled or only minimally. However, if the migration lags, the CPU throttling increases over time.

This approach significantly boosts the likelihood that the migration will eventually succeed.


## Migration networks

It is possible and also reasonable for production setups to use separate networks for:

* Workload network traffic
* Migration network traffic

This will reduce the risk of unwanted side effects during live migrations.

A dedicated physical network is necessary, meaning each node in the cluster must have at least two NICs. The NICs designated for migrations need to be interconnected, they must all be connected to the same switch. In the examples below, eth1 is assumed to handle migrations.

Additionally, the Kubernetes cluster must have Multus (a CNI meta-plugin)[^1] installed.

The following NetworkAttachmentDefinition resource configures the `migration-network` and needs to be created in the `kubevirt` namespace:

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: migration-network
  namespace: kubevirt
spec:
  config: '{
      "cniVersion": "0.3.1",
      "name": "migration-bridge",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "10.1.1.0/24"
      }
    }'
```

To configure KubeVirt to migrate the VirtualMachineInstance resources over that network, we need to adopt the migrations configuration in the `kubevirt` manifest:

```yaml
apiVersion: kubevirt.io/v1
kind: Kubevirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
      [...]
      - LiveMigration
    migrations:
      network: migration-network
[...]
```

[^1]: [Multus-CNI](https://github.com/k8snetworkplumbingwg/multus-cni)
