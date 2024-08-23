---
title: "5.2 Cloud-Init"
weight: 520
labfoldernumber: "05"
sectionnumber: 5.2
description: >
  Use cloud-init to initialize your VM
---

In this section we will use cloud-init to initialize a Fedora Cloud[^1] VM. Cloud-init is the defacto standard for providing
startup scripts to VMs.

Cloud-init is widely adopted. Some of the known users of cloud-init are:
- Ubuntu
- Arch Linux
- CentOS
- Red Hat
- FreeBSD
- Fedora
- Gentoo Linux
- openSUSE


## Supported datasources

KubeVirt supports the NoCloud and ConfigDrive datasource methods.

{{% alert title="Note" color="info" %}}
As it is the simplest datasource you should stick to `NoCloud` as the go to datasource. Only if `NoCloud` is not supported
by the cloud-init implementation you should switch to `ConfigDrive`. For example the implementation of coreos-cloudinit was known to
require the ConfigDrive datasource. However, as CoreOS has built Ignition this implementation is superseded but there
may be more implementations.
{{% /alert %}}


### NoCloud datasource

The NoCloud is a flexible datasource to configure an instance locally. It can work without network access but can also
fetch configuration from a remote server. The relevant configuration of a NoCloud datasource in a VM looks like this:

```yaml
volumes:
  - name: cloudinitdisk
    cloudInitNoCloud:
      userData: "#cloud-config"
[...]
```

This volume must be referenced after the vm disk in the `spec.template.spec.domain.devices.disks` section:
```yaml
- name: cloudinitdisk
  disk:
    bus: virtio
```


Using the `cloudInitNoCloud` attribute give us the following possibilities to provide our configuration:

* `userData`: inline NoCloud configuration in the user data format.
* `userDataBase64`: NoCloud configuration in the user data format as base64 string.
* `secretRef`: reference to a k8s secret containing NoCloud userdata.
* `networkData`: inline NoCloud networkdata.
* `networkDataBase64`: NoCloud networkdata as base64 string.
* `networkDataSecretRef`: reference to a k8s secret containing NoCloud networkdata.

The user data format recognized the following headers. Depending on the header the content is interpreted and executed
differently. For example if you use the `#!/bin/sh` header the content is treated as an executable shell script.

| User data format     | Content Header                  | Expected Content-Type     |
|----------------------|---------------------------------|---------------------------|
| Cloud config data    | `#cloud-config`                 | text/cloud-config         |
| User data script     | `#!`                            | text/x-shellscript        |
| Cloud boothook       | `#cloud-boothook`               | text/cloud-boothook       |
| MIME multi-part      | `Content-Type: multipart/mixed` | multipart/mixed           |
| Cloud config archive | `#cloud-config-archive`         | text/cloud-config-archive |
| Jinja template       | `## template: jinja`            | text/jinja                |
| Include file         | `#include`                      | text/x-include-url        |
| Part handler         | `#part-handler`                 | text/part-handler         |

The most convenient for the lab is to use the NoCloud userdata method.

Check [Cloud-inits Network configuration sources](https://cloudinit.readthedocs.io/en/latest/reference/network-config.html) for more Information about the format
of the network data. Be aware that there is a different format used whenever you use `NoCloud` or `ConfigDrive`.

{{% alert title="Important" color="warning" %}}
Make sure you use `secretRef` or `networkDataSecretRef` whenever you provide sensitive production information.
{{% /alert %}}


### ConfigDrive datasource

The ConfigDrive datasource works identical to the `NoCloud` datasource. However, the volume type changed from `cloudInitNoCloud` to `cloudInitConfigDrive`:

```yaml
volumes:
- name: cloudinitdisk
  cloudInitConfigDrive:
    userData: "#cloud-config"
[...]
```

This volume must be referenced after the vm disk in the `spec.template.spec.domain.devices.disks` section:
```yaml
- name: cloudinitdisk
  disk:
    bus: virtio
```

When using ConfigDrive the networkdata has to be in the [OpenStack Metadata Service Network](https://specs.openstack.org/openstack/nova-specs/specs/liberty/implemented/metadata-service-network-info.html) format


## {{% task %}} Creating a cloud-init config secret

We are now going to create a Fedora Cloud VM and provide a cloud-init userdata configuration to initialize our vm.

First we are going to define our configuration. Create a file `cloudinit-userdata.yaml` with the following content:
```yaml
#cloud-config
password: kubevirt
chpasswd: { expire: False }
```

This will set the password of the default user (`fedora` for Fedora Core) to `kubevirt` and configure the password
to never expire.

From this configuration we need to create the secret. You can use the following command to create the secret in the kubernetes cluster.

```shell
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit --from-file=userdata=cloudinit-userdata.yaml
```

The output should be:
```
secret/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit created
```

You can inspect the secret with:
```shell
kubectl get secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit -o yaml
```

```
apiVersion: v1
data:
  userdata: I2Nsb3VkLWNvbmZpZw[...]
type: Opaque
kind: Secret
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
[...]
```

It would also be possible to create the secret as resource or use a secret management. But for the lab it is more
convenient to create it from the raw configuration using the `kubectl` tool.


## {{% task %}} Creating a VirtualMachine using cloud-init

Create a file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit.yaml` and start with the
following VM configuration:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
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
            image: {{% param "fedoraCloudCDI" %}}
```

Extend the VM configuration to include our secret `{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit` we created above.

{{% details title="Task Hint" %}}
Your VirtualMachine configuration should look like this:
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
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
            memory: 2Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: {{% param "fedoraCloudCDI" %}}
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
```
{{% /details %}}


## {{% task %}} Enhance your startup script

In the previous section we have created a VM using a cloud-init script. Enhance the startup script with the following functionality:

- Set the Timezone to `Europe/Zurich`
- Install nginx package
- Write a custom nginx.conf to `/etc/nginx/nginx.conf`
- Start the nginx service

For the custom nginx configuration you can use the following content:
```text
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
  worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log  /var/log/nginx/access.log  main;
    
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;
    
    include             /etc/nginx/mime.types;
    default_type        text/plain;
    
    server {
        listen       80;
        server_name  _;
        root         /usr/share/nginx/html;
        
        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
        
        location /health {
            return 200 'ok';
        }
        
        location / {
            set $response 'Hello from ${hostname}\n';
            set $response '${response}GMT time:   $date_gmt\n';
            set $response '${response}Local time: $date_local\n';
        
            return 200 '${response}';
        }
    }
}
```

{{% details title="Task Hint" %}}
Your cloud-init configuration will look like this:
```yaml
#cloud-config
password: kubevirt
chpasswd: { expire: False }
packages:
  - nginx
timezone: Europe/Zurich
write_files:
  - content: |
      user nginx;
      worker_processes auto;
      error_log /var/log/nginx/error.log;
      pid /run/nginx.pid;

      events {
        worker_connections 1024;
      }

      http {
          log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
          '$status $body_bytes_sent "$http_referer" '
          '"$http_user_agent" "$http_x_forwarded_for"';

          access_log  /var/log/nginx/access.log  main;

          sendfile            on;
          tcp_nopush          on;
          tcp_nodelay         on;
          keepalive_timeout   65;
          types_hash_max_size 4096;

          include             /etc/nginx/mime.types;
          default_type        text/plain;

          server {
              listen       80;
              server_name  _;
              root         /usr/share/nginx/html;

              # Load configuration files for the default server block.
              include /etc/nginx/default.d/*.conf;

              location /health {
                return 200 'ok';
              }

              location / {
                set $response 'Hello from ${hostname}\n';
                set $response '${response}GMT time:   $date_gmt\n';
                set $response '${response}Local time: $date_local\n';

                return 200 '${response}';
              }
          }
      }
    path: /etc/nginx/nginx.conf
runcmd:
  - systemctl enable nginx
  - systemctl start nginx
```
{{% /details %}}

You need to recreate your secret:

```shell
kubectl delete secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit 
kubectl create secret generic {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit --from-file=userdata=cloudinit-userdata.yaml
```

Next we need to restart our vm to pick up the changes in the cloud-init configuration.

```shell
virtctl restart {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit 
```


{{% alert title="Note" color="info" %}}
It may take some minutes until your server is fully provisioned. While booting you may watch out for the message `Reached target cloud-init.target` in your VMs console.

```shell
virtctl console {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit 
```
{{% /alert %}}

## {{% task %}} Testing your webserver on your Virtual Machine

We have spawn a Virtual Machine which uses cloud-init and installs a simple nginx webserver. Let us test the webserver:

Create the following kubernetes service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
  type: ClusterIP
```

Test your working webserver from your webshell.
```shell
curl -s {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit.$USER.svc.cluster.local
```

```
Hello from {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit
GMT time:   Thursday, 22-Aug-2024 14:13:17 GMT
Local time: Thursday, 22-Aug-2024 16:13:17 CEST
```

## Reference

You may find additional Information about cloud-init here:

* [Cloud-Init Module Reference](https://cloudinit.readthedocs.io/en/latest/reference/modules.html#modules)
* [Cloud-Init Examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html)


[^1]: [Fedora Cloud](https://fedoraproject.org/cloud/)