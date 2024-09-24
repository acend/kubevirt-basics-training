---
title: "Changing files"
weight: 15
labfoldernumber: "01"
description: >
  Changing files in your running VM
---

In the previous chapter, we gained access to the VM's console. In this chapter, we will use this access to make a change
to our running VM and observe what happens if we restart the VM.


### {{% task %}} Create a file

Head over to your vm console with:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --namespace=$USER
```

Login with the specified credentials and create a file:

```bash
touch myfile
```

Verify that the file really is present. Check it with:

```bash
stat myfile
```

Exit the console and restart your VM with:

```bash
virtctl restart {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --namespace=$USER
```

The output should be:

```bash
VM {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm was scheduled to restart
```

Ask yourself:

* After restarting, is the created file still present?
* And why do you think so?

{{% details title="Task Hint" %}}

Check yourself with entering the console again with:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --namespace=$USER
```

Check the file status with:

```bash
stat myfile
```

The output fill be:

```bash
stat: can't stat 'myfile': No such file or directory
```

This means that the file is gone. But why?

Do you remember that we added a container disk to our VM manifest in the first section? A container disk is always
ephemeral and is therefore an easy way to provide a disk wich does not have to be persistent. We will have a closer
look about storage in a later section.
{{% /details %}}


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Stop your running VM with:

```bash
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-firstvm --namespace=$USER
```
{{% /alert %}}
