---
title: "Ignition"
weight: 43
labfoldernumber: "04"
description: >
  Use Ignition to initialize your VM
---

In the previous section we created a VM using a cloud-init configuration secret. This time we will do something similar but with Ignition and Linux Fedora CoreOS.

Known users of Ignition are:

* Fedora CoreOS
* Red Hat Enterprise Linux CoreOS
* Flatcar
* openSUSE MicroOS
* SUSE Linux Enterprise Micro


## Supported datasources

To provide Ignition data we have to use the ConfigDrive datasource.  


### ConfigDrive datasource

The relevant configuration of a ConfigDrive datasource in a VM looks like this:

```yaml
volumes:
  - name: ignitiondisk
    cloudInitNoCloud:
      userData: |
        {
          "ignition": {
            "version": "3.2.0"
          },
          [...]
        }
```

Similar to cloud-init, this volume must be referenced after the vm disk in the `spec.template.spec.domain.devices.disks` section:
```yaml
- name: ignitiondisk
  disk:
    bus: virtio
```

Using the `cloudInitConfigDrive` attribute gives us the following possibilities to provide our Ignition configuration:

* `userData`: inline Ignition configuration
* `userDataBase64`: Ignition configuration as base64 string.
* `secretRef`: reference to a k8s secret containing Ignition configuration.

The data format of an Ignition configuration is always JSON. There is a transpiler available to convert a YAML-formatted Butane config to a
JSON config if needed. As our config for the Lab is simple enough we directly write the JSON config.
You may find more details about Butane here: [Producing an Ignition Config](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/#_configuration_process)

{{% alert title="Important" color="warning" %}}
Make sure you use `secretRef` whenever you provide sensitive data like credentials, certificates and so on.
{{% /alert %}}


## {{% task %}} Creating an Ignition config secret

This time we are going to use a Fedora CoreOS VM and provide an Ignition configuration to initialize our VM.

You can find more information and examples in the official documentation under `Reference` at the end of this lab.

First we define our configuration. Create a file `ignition-data.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:
```yaml
{
  "ignition": {
    "version": "3.4.0"
  },
  "passwd": {
    "users": [
      {
        "name": "core",
        "passwordHash": "[...]"
      }
    ]
  }
}
```

This will set the password of the default user (`core` for Fedora CoreOS) according to the provided hash. You may wonder what the content
of `passwordHash` is. Actually it needs to be a hash understandable by the Linux system. There are several approaches to
generate a password hash to be used for this lab. Two of them are:

OpenSSL
```shell
openssl passwd -salt xyz <PASSWORD>
```
```
$1$xyz$I30aA[...]
```

Pythons Crypt Module
```shell
python -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
```
```
$6$vdwmUilVEr7j7j.T$.2wFftwtDSxK[...]
```

Generate a hash for `kubevirt` and add the generated `passwordHash` to the Ignition configuration.

After that don't forget to create the kubernetes secret containing the ignition configuration, similar to the previous lab.

{{% details title="Solution" %}}
To create the kubernetes secret run the following command:

```shell
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/ignition-data.yaml --namespace=$USER
```

{{% /details %}}


## {{% task %}} Creating a VirtualMachine using Ignition

Create a file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` and start with the
following VM configuration:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
          interfaces:
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 2Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: {{% param "fedoraCoreOSCDI" %}}
```

Similar to the cloud-init section alter the configuration above to include our Ignition config.

{{% details title="Task Hint" %}}
Your VirtualMachine configuration should look like this:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: ignitiondisk
              disk:
                bus: virtio
          interfaces:
            - name: default
              masquerade: {}
        resources:
          requests:
            memory: 2Gi
      networks:
        - name: default
          pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: {{% param "fedoraCoreOSCDI" %}}
        - name: ignitiondisk
          cloudInitConfigDrive:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
```
{{% /details %}}

Create your VM with:
```shell
kubectl create -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition.yaml --namespace=$USER
```

Start the VM and verify whether logging in with the defined user and password works as expected.

{{% details title="Solution" %}}
Start the newly created VM, this might take a while (a couple of minutes), due to the lab environment
```shell
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```
Connect to the console and login as soon as the prompt shows up with the defined credentials.

```shell
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```

{{% alert title="Note" color="info" %}}
Hit the `Enter` key if the login prompt doesn't show automatically.
{{% /alert %}}

{{% /details %}}


## {{% task %}} Enhance your startup script

With the help of the Documentation and Examples (See Section References below), try to enhance your Ignition configuration to include the following configurations:

* Create a group `ssh-users`
* Add user `core` to the group `docker` and `ssh-users`
* Add a `sshAuthorizedKeys` for the user `core`. See instructions below to generate your ssh key.
* Set the Hostname to `lab05-ignition-<user>` where `<user>` is your username
* Configure the SSH Daemon with:
  * Disable root login
  * Allow only group `ssh-users` to login through ssh.

Generate your ssh key with the following command:
```shell
ssh-keygen
```

Your ssh key to be used in the `sshAuthorizedKeys` is located in `/home/theia/.ssh/id_rsa.pub`. Get your key with:
```shell
cat /home/theia/.ssh/id_rsa.pub
```

Your output should be similar to:
```
ssh-rsa AAAAB3NzaC[...] theia@$USER-webshell-554b45d885-b79ks
```

Make sure the key starts with ssh-rsa and copy the key to the `sshAuthorizedKeys` attribute.

{{% details title="Task Hint" %}}
Make sure you replace the `<user>` and the `passwordHash` and `sshAuthorizedKeys` hashes. Your Ignition configuration will look like this:
```yaml
{
  "ignition": {
    "config": {},
    "version": "{{% param "ignitionVersion" %}}"
  },
  "passwd": {
    "groups": [
      {
        "name": "ssh-users"
      }
    ],
    "users": [
      {
        "name": "core",
        "groups": [
          "docker",
          "ssh-users"
        ],
        "passwordHash": "[...]",
        "sshAuthorizedKeys": [
          "ssh-rsa AAAAB3NzaC[...] theia@<user>-webshell-554b45d885-b79ks"
        ]
      }
    ]
  },
  "storage": {
    "files": [
      {
        "path": "/etc/hostname",
        "mode": 420,
        "overwrite": true,
        "contents": { "source": "data:,{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition-<user>" }
      },
      {
        "path": "/etc/ssh/sshd_config.d/30-disable-rootlogin.conf",
        "mode": 644,
        "overwrite": true,
        "contents": { "source": "data:,PermitRootLogin%20no" }
      },
      {
        "path": "/etc/ssh/sshd_config.d/30-allow-groups.conf",
        "mode": 644,
        "overwrite": true,
        "contents": { "source": "data:,AllowGroups%20ssh-users" }
      }
    ]
  }
}
```
{{% /details %}}

You need to recreate your secret:

```shell
kubectl delete secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/ignition-data.yaml --namespace=$USER
```

Next we need to restart our vm to pick up the changes in the ignition configuration.

```shell
virtctl restart {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```

{{% alert title="Note" color="info" %}}
It may take some minutes until your server is fully provisioned.
{{% /alert %}}


## {{% task %}} Testing your webserver on your Virtual Machine

To access our VM from the webshell we need to create a kubernetes service. Create a file
`service-ignition.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
spec:
  ports:
  - port: 22
    protocol: TCP
    targetPort: 22
  selector:
    kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition
  type: ClusterIP
```

And create it:

```shell
kubectl apply -f  {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/service-ignition.yaml --namespace=$USER
```

You may now be able to login with SSH from your webshell to your VM:
```shell
ssh core@{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition.$USER.svc.cluster.local
```

```
Fedora CoreOS 40.20240728.3.0
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

Last login: Fri Aug 23 12:21:09 2024
```

{{% alert title="Note" color="info" %}}
Our SSH Deamon is configured to only allow logins:

* Not from root (`PermitRootLogin no` in `30-disable-rootlogin.conf`)
* Only from users which are a member of the `ssh-users` group (`AllowGroups ssh-users` in `30-allow-groups.conf`)
* Using keys and not username/password (`PasswordAuthentication no` in default config).

You may verify the presence of the two configurations with:
```shell
ls /etc/ssh/sshd_config.d/*.conf
```

Hint: `sudo su - root`

Which should list the two files `30-disable-rootlogin` and `30-allow-groups.conf` created in the Ignition config.
{{% /alert %}}


Verify your assigned groups with:
```shell
groups
```
You should see the assigned groups docker and ssh-users:
```
core adm wheel sudo systemd-journal docker ssh-users
```

The default hostname would be the VM name `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition`.
Show the configured hostname:
```shell
hostname
```
We should see our postfix `-<user>` added to our hostname:
```
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition-<user>
```


## {{% task %}} (Optional) Expose ssh as NodePort to the Internet

In this optional lab we expose the ssh service as node port to the external world.
Similar to what we did in lab 2.4.

* create a NodePort Service
* Find out the Port
* Get the IP Address of one of the Kubernetes Nodes
* ssh to the service.

{{% details title="Solution" %}}

Create the NodePort
```shell
virtctl expose vmi {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --name={{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition-ssh-np --port=22 --type=NodePort --namespace=$USER
```

Find out the Port
```shell
kubectl get service --namespace=$USER
```

Get the IP Address of one of the Kubernetes Nodes
```shell
kubectl get nodes --selector=node-role.kubernetes.io/master!=true -o jsonpath={.items[*].status.addresses[?\(@.type==\"ExternalIP\"\)].address} --namespace=$USER
```
or
```shell
kubectl get nodes -o wide
```

ssh to the service.
```shell
ssh core@188.245.73.202 -p <port>
```


{{% /details %}}


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Stop your running VM with
```shell
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```
{{% /alert %}}


## Reference

You may find additional Information about Ignition here:

* [Ignition Documentation](https://coreos.github.io/ignition/)
* [Ignition Examples](https://coreos.github.io/ignition/examples/)
* [Supported Platforms](https://coreos.github.io/ignition/supported-platforms/)

