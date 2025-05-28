---
title: "Ignition"
weight: 43
labfoldernumber: "04"
description: >
  Use Ignition to initialize your VM
---

In the previous section we created a VM using a cloud-init configuration Secret. This time we will do something similar but with Ignition and Linux Fedora CoreOS.

Known users of Ignition are:

* Fedora CoreOS
* Red Hat Enterprise Linux CoreOS
* Flatcar
* openSUSE MicroOS
* SUSE Linux Enterprise Micro


## Supported data sources

To provide Ignition data we have to use the `cloudInitNoCloud` data source.  


### `cloudInitNoCloud` data source

The relevant configuration of a `cloudInitNoCloud` data source in a VM looks like this:

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

Similar to cloud-init, this volume must be referenced after the VM disk in the `spec.template.spec.domain.devices.disks` section:

```yaml
- name: ignitiondisk
  disk:
    bus: virtio
```

Using the `cloudInitConfigDrive` attribute gives us the following possibilities to provide our Ignition configuration:

* `userData`: Inline Ignition configuration
* `userDataBase64`: Ignition configuration as a base64-encoded string
* `secretRef`: Reference to a K8s Secret containing Ignition configuration

The data format of an Ignition configuration is always JSON. There is a transpiler available to convert a YAML-formatted Butane config to a
JSON config if needed. As our config for the Lab is simple enough we directly write the JSON config.
You may find more details about Butane [in the official documentation](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/#_configuration_process).

{{% alert title="Important" color="warning" %}}
Make sure you use `secretRef` whenever you provide sensitive data like credentials, certificates and similar.
{{% /alert %}}


## {{% task %}} Creating an Ignition config Secret

This time we are going to use a Fedora CoreOS VM and provide an Ignition configuration to initialize our VM.

First, we define our configuration. Create a file `ignition-data.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

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

This will set the the default user's password (`core` for Fedora CoreOS) according to the provided hash. You may wonder what the content
of `passwordHash` is. Actually it needs to be a hash understandable by the Linux system. There are several approaches to
generate a password hash to be used for this lab. Two of them are:

Using OpenSSL:

```bash
openssl passwd -salt xyz <PASSWORD>
```

Using Python's crypt module:

```bash
python -c 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))'
```

Generate a hash for `kubevirt` and add the generated `passwordHash` to the Ignition configuration.

After that don't forget to create the Kubernetes Secret containing the Ignition configuration, similar to the previous lab.

{{% details title="Solution" %}}

If you generated the password hash using the OpenSSL command `openssl passwd -salt xyz kubevirt`, then the `ignition-data.yaml` should look like:


```yaml
{
  "ignition": {
    "version": "3.4.0"
  },
  "passwd": {
    "users": [
      {
        "name": "core",
        "passwordHash": "$1$xyz$I30aASnHH5bA2yVRoRlsI1"
      }
    ]
  }
}
```

To create the Kubernetes Secret, run the following command:

```bash
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/ignition-data.yaml --namespace=$USER
```

{{% /details %}}


## {{% task %}} Creating a VirtualMachine using Ignition

Create a file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition.yaml`
in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}`
and start with the following VM configuration:

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
      {{< onlyWhen tolerations >}}tolerations:
        - effect: NoSchedule
          key: baremetal
          operator: Equal
          value: "true"
      {{< /onlyWhen >}}volumes:
        - name: containerdisk
          containerDisk:
            image: {{% param "fedoraCoreOSCDI" %}}
```

Similar to the cloud-init section, alter the configuration above to include our Ignition config.

{{% details title="Task hint" %}}
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
      {{< onlyWhen tolerations >}}tolerations:
        - effect: NoSchedule
          key: baremetal
          operator: Equal
          value: "true"
      {{< /onlyWhen >}}volumes:
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

```bash
kubectl apply -f {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition.yaml --namespace=$USER
```

Start the VM and verify whether logging in with the defined user and password works as expected.

{{% details title="Solution" %}}
Start the newly created VM. This might take a couple of minutes:

```bash
virtctl start {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```

Connect to the console and log in as soon as the prompt shows up with the defined credentials:

```bash
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```

{{% alert title="Note" color="info" %}}
Hit the `Enter` key if the login prompt doesn't show automatically.
{{% /alert %}}

{{% /details %}}


## {{% task %}} Enhance your startup script

With the help of the [referenced documentation and examples](#references), try to enhance your Ignition configuration to include the following configuration:

* Create a group `ssh-users`
* Add user `core` to the group `docker` and `ssh-users`
* Add a `sshAuthorizedKeys` for the user `core` (see instructions below to generate your ssh key)
* Set the hostname to `lab{{% param "labfoldernumber" %}}-ignition-<user>` where `<user>` is your username
* Configure the SSH daemon with:
  * Disable root login
  * Only allow group `ssh-users` to login via ssh

Generate your ssh key with the following command:

```bash
ssh-keygen
```

Your ssh key to be used in the `sshAuthorizedKeys` is located in `/home/theia/.ssh/id_rsa.pub`. Get your key with:

```bash
cat /home/theia/.ssh/id_rsa.pub
```

Your output should be similar to:

```
ssh-rsa AAAAB3NzaC[...] theia@$USER-webshell-554b45d885-b79ks
```

Make sure the key starts with ssh-rsa and copy the key to the `sshAuthorizedKeys` attribute.

{{% details title="Task Hint" %}}
Make sure you replace `<user>` (line 32), `passwordHash` (line 19) and `sshAuthorizedKeys` (line 21). Your Ignition configuration will look like this:

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

You need to recreate your Secret:

```bash
kubectl delete secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --from-file=userdata={{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/ignition-data.yaml --namespace=$USER
```

Next, we need to restart our VM to pick up the changes in the Ignition configuration.

```bash
virtctl restart {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```

{{% alert title="Note" color="info" %}}
It may take some minutes until your server is fully provisioned.
{{% /alert %}}


## {{% task %}} Testing your ssh server on your virtual machine

To access our VM from the webshell, we need to create a Kubernetes Service. Create a file called
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

And create it with:

```bash
kubectl apply -f  {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/service-ignition.yaml --namespace=$USER
```

You may now be able to login with SSH from your webshell to your VM:

```bash
ssh core@{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition.$USER.svc.cluster.local
```

And hit `yes` to confirm the authenticity of host.

```
Fedora CoreOS 40.20240728.3.0
Tracker: https://github.com/coreos/fedora-coreos-tracker
Discuss: https://discussion.fedoraproject.org/tag/coreos

Last login: Fri Aug 23 12:21:09 2024
```

{{% alert title="Note" color="info" %}}
Our SSH daemon is configured to only allow logins:

* Not from root (`PermitRootLogin no` in `30-disable-rootlogin.conf`)
* Only from users which are a member of the `ssh-users` group (`AllowGroups ssh-users` in `30-allow-groups.conf`)
* Using keys, not passwords (`PasswordAuthentication no` in default config).

You may verify the presence of the two configuration files with:

```bash
ls /etc/ssh/sshd_config.d/*.conf
```

Hint: `sudo su - root`

Which should list the two files `30-disable-rootlogin` and `30-allow-groups.conf` created in the Ignition config.
{{% /alert %}}

Make sure to switch back to the user `core` and verify your assigned groups with:

```bash
groups
```

You should see the assigned groups `docker` and `ssh-users`:

```
core adm wheel sudo systemd-journal docker ssh-users
```

The default hostname would be the VM name `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition`.
Show the configured hostname:

```bash
hostname
```

We should see our postfix `-<user>` added to our hostname:

```
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition-<user>
```


## {{% task %}} (Optional) Expose ssh as NodePort to the internet

In this optional lab we will expose the ssh service as a node port to the external world, similar to what we did in {{< link "/content/en/docs/getting-started-with-kubevirt/ssh.md" >}}.

* Create a NodePort Service
* Find out the port number
* Get the IP address of one of the Kubernetes nodes
* Test it

{{% details title="Solution" %}}

Create the NodePort Service:

```bash
virtctl expose vmi {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --name={{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition-ssh-np --port=22 --type=NodePort --namespace=$USER
```

Find out the port number:

```bash
kubectl get service --namespace=$USER
```

Get the IP address of one of the Kubernetes nodes:

```bash
kubectl get nodes --selector=node-role.kubernetes.io/master!=true -o jsonpath={.items[*].status.addresses[?\(@.type==\"ExternalIP\"\)].address} --namespace=$USER
```

or

```bash
kubectl get nodes -o wide
```

Test it:

```bash
ssh core@<node-ip> -p <port>
```

{{% /details %}}


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

Stop your running VM with:

```bash
virtctl stop {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-ignition --namespace=$USER
```

{{% /alert %}}


## References

You may find additional information about Ignition here:

* [Ignition documentation](https://coreos.github.io/ignition/)
* [Ignition examples](https://coreos.github.io/ignition/examples/)
* [Supported platforms](https://coreos.github.io/ignition/supported-platforms/)
