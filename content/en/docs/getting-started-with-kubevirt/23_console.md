---
title: "2.3 Accessing the Console"
weight: 230
labfoldernumber: "02"
sectionnumber: 2.3
description: >
  Opening a console to your VirtualMachine
---


## Console access

If your VM is running, you can open a console on the VM using the `virtctl` tool.

First, list your VMs to get the name of your VirtualMachine:

```bash
kubectl get vm --namespace=$USER
```

The output should be:

```bash
NAME            AGE   STATUS    READY
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm   10m   Running   True
```


### {{% task %}} Entering the console

Enter the console using:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --namespace=$USER
```

Congratulations, you successfully connected to your VMs console! The expected output is:

```bash
Successfully connected to {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm console. The escape sequence is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
training-worker-0 login:
```

{{% alert title="Note" color="info" %}}
You probably have to hit `Enter` to get to the login prompt.
{{% /alert %}}

You can now use the default credentials to log in:

* User: `cirros`
* Password: `gocubsgo`

Try executing a couple of commands, such as:

```bash
whoami
ping acend.ch
```

{{% alert title="Note" color="info" %}}
Remember that you have to press `Ctrl+AltGr+]]` (`]` twice) to exit the console.
{{% /alert %}}
