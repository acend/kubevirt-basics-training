---
title: "2.4 Exposing VM Ports"
weight: 240
labfoldernumber: "02"
sectionnumber: 2.4
description: >
  Accessing ports of the running VirtualMachine
---

In the previous chapter, we accessed our VM console using the `virtctl` tool. In this section we will expose the SSH port
of our VM and access it directly.

{{% alert title="Note" color="info" %}}
This can be done for any port you want to use. For example, if your virtual machine provides a webserver, you can expose
the webserver port.
{{% /alert %}}


## Checking available Services resources

As you see with the following command, creating the VM does not create any Kubernetes Service for it.

```bash
kubectl get service --namespace=$USER
```

In your Namespace you should only see the service of your webshell:

```bash
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
$USER-webshell   ClusterIP   10.43.248.212   <none>        3000/TCP   1d
```


## Exposing the SSH port inside the Kubernetes cluster

To access the SSH port from the Kubernetes default Pod network we have to create a simple Service resource.
For this, we use a Service of type `ClusterIP`.

Create a file `svc_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh.yaml` in the `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` directory and write the following Service configuration to it:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh
spec:
  ports:
  - port: 22
    protocol: TCP
    targetPort: 22
  selector:
    kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm
    kubevirt.io/size: small
  type: ClusterIP
```

Apply the Service with:

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/svc_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh.yaml --namespace=$USER
```

You may now log in from your webshell terminal to the ssh port of the virtual machine using the following command (password: `gocubsgo`):

```bash
ssh cirros@{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh.$USER.svc.cluster.local
```

{{% alert title="Note" color="info" %}}
We could also use the `virtctl` command to create a service for us. The command for the service above would be:

```bash
virtctl expose vmi {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --name={{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh --port=22 --namespace=$USER
```

We will use this approach in the next section.
{{% /alert %}}


## Exposing the SSH port for external use

Our exposed Service with type `ClusterIP` is only reachable from within the Kubernetes cluster. On our Kubernetes
cluster, we can expose ports such as the one for SSH as a `NodePort` Service to access it from the outside of the cluster.

This time we will use the `virtctl` command to expose the port as type `NodePort`. Us this command to create the Service:

```bash
virtctl expose vmi {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --name={{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh-np --port=22 --type=NodePort --namespace=$USER
```

You should now see both Services you just created for your VM:

```bash
kubectl get service --namespace=$USER
```

Which should produce a similar output:

```bash
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm-ssh   ClusterIP   10.43.89.29     <none>        22/TCP         17m
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ssh-np        NodePort    10.43.223.242   <none>        22:32664/TCP   49s
$USER-webshell      ClusterIP   10.43.248.212   <none>        3000/TCP       1d
```

With this, our service is reachable from every node on the indicated port. You may check the `PORT(S)` column for the
assigned port. In this example, our assigned NodePort is `32664/TCP`, which then targets port 22 on our VM.

To connect to the port indicated by the NodePort Service, we need to know the worker nodes' IP addresses. You can get them with:

```bash
kubectl get nodes --selector=node-role.kubernetes.io/master!=true -o jsonpath={.items[*].status.addresses[?\(@.type==\"ExternalIP\"\)].address} --namespace=$USER
```

Which will produce a similar output to this:

```bash
188.245.73.202 116.203.61.242 159.69.207.154
```

{{% alert title="Note" color="info" %}}
You could also see the nodes' IP addresses nodes using:

```bash
kubectl get nodes -o wide
```
{{% /alert %}}

Since the NodePort Service is accessible on any worker node, you can simply pick one IP address and issue the following command
from within your webshell (make sure you replace the IP and the assigned NodPort to match your details):

```bash
ssh cirros@188.245.73.202 -p 32664
```

You should be able to use the same command from outside your webshell, e.g., from your computer.
