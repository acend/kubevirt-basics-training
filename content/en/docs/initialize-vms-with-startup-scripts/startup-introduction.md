---
title: "Introduction"
weight: 41
labfoldernumber: "04"
description: >
  Introduction to VM provisioning with startup scripts. 
---

When we create a virtual Machine we often want to configure the virtual Machine to fit in our environment. To achieve this, KubeVirt
supports to assign startup scripts which are executed automatically when the VM initializes. They are typically used to
provide SSH Keys, required configuration to run further configuration with ansible, deploy network configuration and so on.

The startup script methods are supported.

* Cloud-Init[^1] and Ignition[^2] which are targeting Linux and Unix systems
* Sysprep[^3] to initialize Windows based VMs.

Cloud-Init is the most widely adopted method, and you will find great support on cloud providers such as AWS, GCP and Azure.


## Can I run these tools with every disk image?

No, the Disk/System must include the software to run cloud-init, Ignition or Sysprep. This means that you have to use special images/disks.
They are usually called **Cloud Images** and are pre-installed disk images that can be customized using startup scripts.
Some variants are:

* Fedora Cloud: https://fedoraproject.org/de/cloud/
* Ubuntu Cloud: https://cloud-images.ubuntu.com/
* AlmaLinux: https://almalinux.org/get-almalinux/

Most of them are also directly available on your cloud provider.


[^1]: [cloud-init](https://cloud-init.io/)
[^2]: [Ignition](https://coreos.github.io/ignition/)
[^3]: [Sysprep](https://learn.microsoft.com/de-de/windows-hardware/manufacture/desktop/sysprep--system-preparation--overview?view=windows-11)
