---
title: "Getting started with KubeVirt"
weight: 1
labfoldernumber: "01"
description: >
  Create and run your first virtual machine
---


## Lab goals

* Get familiar with the lab environment
* Create your first VM using KubeVirt
* Start and stop a VM using `virtctl` or `kubectl`
* Connect to the VM's console
* Expose and access ports of your VM
* Change a file inside a VM and observe the behaviour


## Folder structure

This is your first lab where you will create and apply files to the Kubernetes cluster. It may make sense to structure
your files according to the labs. Feel free to create a folder structure something like this:

```text
{{% param "labsfoldername" %}}
|-- {{% param "labsubfolderprefix" %}}01
|-- {{% param "labsubfolderprefix" %}}02
|-- {{% param "labsubfolderprefix" %}}03
[...]
```

Make sure you're in the correct directory:

```bash
cd {{% param "projecthome" %}}
```

Initialize the directory structure by executing the following command:

```bash
mkdir -p {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{01..{{% param "maxlabnumber" %}}}/
```

Finally, verify the structure:

```bash
tree {{% param "labsfoldername" %}}
```
