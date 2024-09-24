---
title: "KubeVirt configuration"
weight: 11
type: docs
description: >
  Getting to know the lab environment.
---


## KubeVirt installation

On the Kubernetes cluster used for this training, the following Operators have been installed:

| Operator name                    | Namespace  |
|----------------------------------|------------|
| KubeVirt                         | kubevirt   |
| Containerized Data Impoter (CDI) | cdi        |

As the configuration and the required feature flags are subject to change, we do not highlight them in every section.


### Emulation

This cluster is using emulation. This has an impact on the VM performance.

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      useEmulation: true
[...]
```


### Enabled Kubernetes feature flags

In the cluster, the following KubeVirt feature flags are enabled:

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
```
