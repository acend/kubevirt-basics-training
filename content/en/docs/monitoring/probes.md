---
title: "Readiness and liveness probes"
weight: 82
labfoldernumber: "08"
description: >
  Using readiness and liveness probes to ensure the health of virtual machines
---

Liveness and readiness probes can be configured for VirtualMachineInstances similarly to how they are defined for containers. You can find more information about these probes [in Kubernetes' documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

Liveness probes will stop the VirtualMachineInstance if they fail, allowing higher-level controllers, such as VirtualMachine or VirtualMachineInstanceReplicaSet, to create new instances that should be responsive.

Readiness probes signal to Services and Endpoints whether the VirtualMachineInstance is ready to handle traffic. If these probes fail, the VirtualMachineInstance will be removed from the list of Endpoints backing the Service until the probe recovers.

Watchdogs, on the other hand, monitor the operating system's responsiveness, complementing the workload-centric probes. They require kernel support from the guest OS and additional tools like the commonly used watchdog binary.

Exec probes are specific liveness or readiness probes for VMs. They execute commands inside the VM to assess its readiness or liveliness. The qemu-guest-agent package facilitates running these commands inside the VM. The command provided to an exec probe is wrapped by `virt-probe` in the operator and sent to the guest.


## {{% task %}} Define a HTTP liveness probe

First, we are going to define our `cloud-init` configuration. Create a file called `cloudinit-probe.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

```yaml
#cloud-config
password: kubevirt
chpasswd: { expire: False }
bootcmd:
  - ["sudo", "dnf", "install", "-y", "nmap-ncat"]
  - ["sudo", "systemd-run", "--unit=httpserver", "nc", "-klp", "8081", "-e", '/usr/bin/echo -e HTTP/1.1 200 OK\\nContent-Length: 12\\n\\nHello World!']
```

This will install a simple `httpserver` which will return `HTTP 200` and will be used as a health endpoint in the HTTP liveness probe.

Create a secret by executing the following command:

```bash
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-probe --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/cloudinit-probe.yaml --namespace=$USER
```

Create a virtual machine referencing the configuration from above by creating a new file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/` with the following content:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
    spec:
      domain:
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
          requests:
            memory: 1024M
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
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-probe
```
Now configure a HTTP `livenessProbe` with the following specification:

* initialDelaySeconds: `120`
* periodSeconds: `20`
* http probe on port: `8081`
* timeoutSeconds: `10`

{{% details title="Task hint: Solution" %}}

Your VirtualMachine configuration should look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
    spec:
      domain:
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
          requests:
            memory: 1024M
      livenessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        httpGet:
          port: 8081
        timeoutSeconds: 10
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
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-probe
```
{{% /details %}}

Make sure you create your VM with:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml --namespace=$USER
```

Start the newly-created VM. This might take a couple of minutes:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe --namespace=$USER
```


## {{% task %}} Add a HTTP readiness probe

In addition to the previously configured liveness probe, we will add a readiness probe in this lab. For convenience reasons, we will use the same httpserver and port. Those can be different depending on your needs.

Add a `readinessProbe` with the following specification to the virtual machine (`vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml`):

* initialDelaySeconds: `120`
* periodSeconds: `10`
* http probe on port: `8081`
* timeoutSeconds: `5`
* failureThreshold: `5`
* successThreshold: `5`

{{% details title="Task hint: Solution" %}}

Your VirtualMachine configuration should look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
    spec:
      domain:
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
          requests:
            memory: 1024M
      livenessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        httpGet:
          port: 8081
        timeoutSeconds: 10
      readinessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        timeoutSeconds: 10
        httpGet:
          port: 8081
        failureThreshold: 5
        successThreshold: 5
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
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-probe
```
{{% /details %}}

Apply and restart the virtual machine, either via `kubectl`:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml --namespace=$USER
```

Or via `virtctl`:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe --namespace=$USER
```

Make sure to observe the `READY` state of the VMI. It should take around 3 minutes to become `TRUE`:

```bash
kubectl get vmi --namespace=$USER
```

```bash
NAME              AGE     PHASE     IP              NODENAME               READY
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe       2m32s   Running   10.244.20.230   training-baremetal-0   False
```


## {{% task %}} Change the HTTP liveness probe to TCP

Instead of checking a HTTP endpoint in the liveness probe, we can also check a TCP socket.

Change the virtual machine's `livenessProbe` (`vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml`) from `HTTP` to `TCP`. Apply the changes and restart the virtual machine.

{{% details title="Task hint: Solution" %}}

Your VirtualMachine configuration should look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
    spec:
      domain:
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
          requests:
            memory: 1024M
      livenessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        tcpSocket:
          port: 8081
        timeoutSeconds: 10
      readinessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        timeoutSeconds: 10
        httpGet:
          port: 8081
        failureThreshold: 5
        successThreshold: 5
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
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-probe
```

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml --namespace=$USER
```

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe --namespace=$USER
```

{{% /details %}}


## {{% task %}} Test the behaviour when probes fail

Experiment with the probes from the last couple of labs.

* Change the port to something different and observe the behaviour
* Change the configuration of the probes and observe the behaviour
* Stop the running webserver process and observe the results


## {{% task %}} Guest agent liveness probe

It's also possible the use the guest agent, which we had a look at in the last lab, as indicator for probes.

Configure the virtual machine's `livenessProbe` (`vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml`) to use `guestAgentPing` instead of `tcpSocket`. Apply the changes and restart the virtual machine.

{{% details title="Task hint: Solution" %}}

Your VirtualMachine configuration should look like this:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe
    spec:
      domain:
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
          requests:
            memory: 1024M
      livenessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        guestAgentPing: {}
        timeoutSeconds: 10
      readinessProbe:
        initialDelaySeconds: 120
        periodSeconds: 20
        timeoutSeconds: 10
        httpGet:
          port: 8081
        failureThreshold: 5
        successThreshold: 5
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
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-probe
```

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe.yaml --namespace=$USER
```

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe --namespace=$USER
```

{{% /details %}}


{{% alert title="Note" color="info" %}}  
Additionally to the guest agent ping probe, `exec` probes can also be used. An `exec` probe executes a command to determine the status of the virtual machine.

As a precondition, the guest agent needs to be installed in the virtual machine for the probe to work.

{{% /alert %}}


## Watchdog example

{{% alert title="Warning" color="warning" %}}
This lab is not meant to be executed. There is an issue with the execution of the watchdog.
{{% /alert %}}


A watchdog offers a more VM-centric approach, meaning the OS monitors itself by sending heartbeats to a `i6300esb` device. When the heartbeat stops, the watchdog device executes an action. In our example the `poweroff` action. Other possible actions are `reset` and `shutdown`.

Inside the virtual machine, a component is required to send the heartbeat. In the following example we will use busybox, which sends a watchdog heartbeat to `/dev/watchdog`.

We achieve this by first going to define a new `cloud-init` configuration. Create a file called `cloudinit-watchdog.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

```yaml
#cloud-config
password: kubevirt
chpasswd: { expire: False }
bootcmd:
  - ["sudo", "dnf", "install", "-y", "busybox"]
  
```

Create a secret with above file's content by executing the following command:

```bash
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-watchdog --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/cloudinit-watchdog.yaml --namespace=$USER
```

Create the virtual machine referencing the configuration from above by creating a new file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/` with the following content:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog
    spec:
      domain:
        devices:
          watchdog:
            name: mywatchdog
            i6300esb:
              action: "poweroff"
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
          requests:
            memory: 1024M
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
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-watchdog
```

Make sure you create your VM with:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog.yaml --namespace=$USER
```

Start the VM. This might take a couple of minutes:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog --namespace=$USER
```

Connect to the console:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog --namespace=$USER
```

And execute the following command:

```bash
sudo busybox watchdog -t 2000ms -T 10000ms /dev/watchdog
```

This will send heartbeats every two seconds for ten seconds, after that the virtual machine should be powered off. In a non-demo setup, you would start the watchdog during startup and not turn it off on purpose afterwards.


## End of lab

{{% alert title="Cleanup resources" color="warning" %}} {{% param "end-of-lab-text" %}}

Stop the VirtualMachineInstance again:

```bash
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-probe  --namespace=$USER
```

```bash
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-watchdog  --namespace=$USER
```
{{% /alert %}}
