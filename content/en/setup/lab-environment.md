---
title: "Lab environment"
weight: 10
type: docs
description: >
  Getting to know the lab environment.
---


## Login

Authentication depends on the specific Kubernetes cluster environment. The **URL** and **credentials** to access the lab environment will be provided by your trainer. Use Chrome for the best experience.


## Webshell

The first thing we are going to do is to explore our lab environment and and the different components.

The provided lab environment contains an _Eclipse Theia IDE_[^1]. Your IDE will look something like this:

![Eclipse Theia IDE](../theia.png)

* On the left side you can open the **Explorer** which you can use for managing files and directories
* The terminal is accessible using the menubar on top, clicking **Terminal**, then **New Terminal** or using `Ctrl+Shit+^`

The available environment in the webshell contains all the needed tools like `kubectl` and `virtctl` as well as
the configuration needed to access the Kubernetes cluster.

Let's verify that by executing the following command in a freshly created terminal window:

```bash
kubectl version
```

The files in the home directory under `{{% param "projecthome" %}}` are stored in a persistent volume, so please make sure to store all your data in this directory and use it as starting point for all our lab files.
You can create files within your webshell or using the explorer.

If you have a terminal running you can interact with the Kubernetes cluster. For example, you can list your context or get the Pods of the current Namespace:

```bash
kubectl config get-contexts
```

This should result in something like this:

```bash
CURRENT   NAME    CLUSTER   AUTHINFO   NAMESPACE
          local   local     local      <user>
```

With the following command will show that the current webshell is also running as a Pod within your Namespace:

```bash
kubectl get pods --namespace=$USER
```

So the expected outcome is a webshell Pod:

```bash
NAME                             READY   STATUS    RESTARTS   AGE
<user>-webshell-885dbc579-lwhtd   2/2     Running   0          11d
```


## Namespace

We are going to use the following Namespace for the labs

* `<user>` where we deploy our Virtual Machines.

Alternatively there is a Namespace `<user>-dev` available.


## General lab notes


### Placeholders

In this lab we will use the following placeholders or naming conventions:

* `<user>` your username (for example `user4`)
* `$USER` the environment variable containing your username. Execute `echo $USER` to verify that in your terminal session
* `[...]` means that some lines in the listing or command have been omitted for readability


### Exiting a console of a virtual machine

In various labs we will connect to the console of a virtual machine using the `virtctl console`. Your terminal will look like this:

```bash
virtctl console kubevirtvm
```

When the terminal is connected to the virtual machine the following line appears:
```bash
Successfully connected to kubevirtvm console. The escape sequence is ^
```

This simple escape sequence `^` does not work within the webshell terminal. You have the following options to exit the console:

* Press `Ctrl+AltGr+]]` (yes, press `]` twice). This might not work in all browsers. Use Chrome for the best experience.
* Close the webshell terminal and open a new one with `Ctrl+Shift+^`


### Hints

We usually provide help for a task you have to complete. For example, if you have to implement a method you most likely find the solution in a _Task hint_ like this one:

{{% details title="Task hint" %}}
Your yaml should look like this:

```yaml
kind: VirtualMachine
metadata:
  name: kubevirtvm
spec:
  running: false
  template:
    spec:
      domain:
        devices: {}
        memory:
          guest: 64Mi
        resources: {}
status: {}
```
{{% /details %}}


### Using Kubernetes context

In our labs we specify the Namespace for the commands by explicitly defining the `--namespace` parameter.
We deem this best practice as it makes it harder to execute commands in the wrong Namespace.

However, some people prefer to use the context by, e.g., using the following commands:

```bash
kubectl config set-context $(kubectl config current-context) --namespace $USER
```

If you get the following error from the command above you first have to initially set your current context:

```bash
error: current-context is not set
error: you must specify a non-empty context name or --current
```

Set the current context with:

```bash
kubectl config use-context local
```

And repeat the following command:

```bash
kubectl config set-context $(kubectl config current-context) --namespace $USER
```


### Highlighting important things

We will use the following styling to highlight various things.

{{% alert title="Note" color="info" %}}This is an information providing some additional help or information.{{% /alert %}}

{{% alert title="Important" color="warning" %}}This is an important note which you should read and follow.{{% /alert %}}

{{% alert title="Alert" color="danger" %}}This is an alert or important warning. You should carefully read and follow its details.{{% /alert %}}

[^1]: [Eclipse Theia IDE Project](https://theia-ide.org/)
