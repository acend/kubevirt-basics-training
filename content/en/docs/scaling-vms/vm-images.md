---
title: "VM disk images"
weight: 51
labfoldernumber: "05"
description: >
  Creating disk images for scaling virtual machines
---

If we want to run VMs at scale it makes sense to manage a set of base images to use for these VMs. It is not very convenient
to spin up and install some requirements for each single VM oneself. There are several ways we can distribute VM images in our cluster.

* Distribute images as ephemeral container disks using a container registry
  * Be aware of the non-persistent root disk
  * Depending on the disk size, this approach may not be the best choice
* Create a namespace (e.g., `vm-images`) with pre-provisioned PVCs containing base disk images
  * Each VM would then use CDI to clone the PVC from the `vm-images` namespace to the local namespace

At the end of this section, we will have two PVCs containing base disks in our namespace:

* `fedora-cloud-base`: Original Fedora Cloud
* `fedora-cloud-nginx-base`: Fedora Cloud with nginx installed


## Creating a Fedora Cloud image with nginx

In the previous section, we created a VM using cloud-init to install nginx and start the webserver. If we created a pool based on this, each VM would go through the same initialization process and install nginx.
This is obviously not very efficient. Additionally, there would eventually be different versions of nginx in the VM pool depending on when the VM was started and the installation took place.

In order to optimize this, let's create a base image which has nginx already installed instead of installing it during the first boot.

{{% alert title="Note" color="info" %}}
Normally we would to this in a central namespace like `vm-images`. In this lab you will use your own namespace `<user>`.
{{% /alert %}}


### {{% task %}} Create the fedora-cloud-base disk

First we need to create our base disk for Fedora Cloud 40. We will use a `DataVolume` and CDI to provision a PVC
containing a disk base on the container disk `{{% param "fedoraCloudCDI" %}}`.

Create the following file `dv_fedora-cloud-base.yaml` in folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}`:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: fedora-cloud-base
spec:
  source:
    registry:
      url: "docker://{{% param "fedoraCloudCDI" %}}"
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 6Gi
```

Create the DataVolume with the following command:

```bash
kubectl apply -f  {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/dv_fedora-cloud-base.yaml --namespace=$USER
```

This will download the container disk `{{% param "fedoraCloudCDI" %}}` and store it in a PVC named `fedora-cloud-base`:

```bash
kubectl get datavolume --namespace=$USER
```

This will result in something like this:

```bash
NAME                PHASE             PROGRESS   RESTARTS   AGE
fedora-cloud-base   ImportScheduled   N/A
```

Note the `Succeeded` phase when the import process is completed:

```bash
NAME                PHASE       PROGRESS   RESTARTS   AGE
fedora-cloud-base   Succeeded   100.0%                105s
```

With the following command, you can verify the existence of the PVC which contains the imported images:

```bash
kubectl get pvc --namespace=$USER
```

```bash
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
fedora-cloud-base       Bound    pvc-4c617a10-24f5-427c-8d11-da45723593e9   6Gi        RWO            longhorn       <unset>                 2m56s
[...]
```


### {{% task %}} Create the provisioner VM

Next we will create a VM which installs our packages and creates the final provisioned PVC. This VM will:

* Clone the Fedora base disk `fedora-cloud-base` to our provisioned disk `fedora-cloud-nginx-base`
* Start a VM and install nginx using cloud-init
* Remove the cloud-init configuration to make it possible for further VMs cloning this disk to rerun cloud-init
* Shutdown the VM

Create the file `vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-nginx-provisioner.yaml` in the folder `{{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}` with the following content:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-nginx-provisioner
spec:
  runStrategy: "RerunOnFailure"
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-nginx-provisioner
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk
          - disk:
              bus: virtio
            name: cloudinitdisk
        machine:
          type: ""
        resources:
          requests:
            memory: 1Gi
      terminationGracePeriodSeconds: 0 
      {{< onlyWhen tolerations >}}tolerations:
        - effect: NoSchedule
          key: baremetal
          operator: Equal
          value: "true"
      {{< /onlyWhen >}}volumes:
      - name: datavolumedisk
        dataVolume:
          name: fedora-cloud-nginx-base
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config-archive    
            - type: "text/cloud-config"    
              content: |    
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
            - type: "text/x-shellscript"
              content: |
                #!/bin/sh
                yum install -y nginx
                systemctl enable nginx
                # removing instances ensures cloud init will execute again after reboot
                rm -rf /var/lib/cloud/instances
                shutdown now
  dataVolumeTemplates:
  - metadata:
      name: fedora-cloud-nginx-base
    spec:
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 6Gi
      source:
        pvc:
          name: fedora-cloud-base
```

Create the VM with the following command:

```bash
kubectl apply -f  {{% param "labsfoldername" %}}/{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}/vm_{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-nginx-provisioner.yaml --namespace=$USER
```

There are the following important details in this VM manifest:

* `runStrategy: "RerunOnFailure"`: This tells KubeVirt to run the VM like a Kubernetes Job. The VM will retry as long as the guest is not shut down gracefully.
* `cloudInitNoCloud`: These are the instructions to provision our disk. Please note the deletion of the cloud-init data to ensure it is rerun whenever we start a VM based on this disk. Further we shutdown the VM gracefully at the end of the script.
* `dataVolumeTemplate`: This creates a new PVC for the provisioned disk containing nginx.

As mentioned, the VM has been scheduled due to the `runStrategy: "RerunOnFailure"`, therefore the VMI should be running. Use the following command to verify that with:

```bash
kubectl get vmi --namespace=$USER
```

or:

```bash
kubectl get pod --namespace=$USER
```

{{% alert title="Note" color="info" %}}
Take your time to closely inspect the cloud-init provisioning. This is a more complex version of a `cloudInitNoCloud`
configuration combining two available `userData` formats. To achieve this, we used the `#cloud-config-archive` as the
parent type. This allowed us to use multiple items with different types. The first type is the regular `#cloud-config`
format. For the second item we used a shell script `#!/bin/sh`.

As specified above, we delete the data in `/var/lib/cloud/instances`. As this is a base image, we want to run cloud-init again.
{{% /alert %}}

After the provisioning was successful, the VM will terminate itself due to the `shutdown now` statement.

```bash
kubectl get vm --namespace=$USER
```

```
NAME                             AGE     STATUS    READY
[...]
{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-nginx-provisioner   8m52s   Stopped   False
```

After the VM has been shut down, we will see a `fedora-cloud-nginx-base` PVC in our namespace:

```bash
kubectl get pvc
```

```
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
fedora-cloud-base         Bound    pvc-c1541b25-2414-41b2-84a6-99872a19d7c4   6Gi        RWO            longhorn       <unset>                 26m
fedora-cloud-nginx-base   Bound    pvc-27ba0e54-ff7d-4782-bd23-0823f5f3010f   6Gi        RWO            longhorn       <unset>                 9m59s
```

The VM is still present in the namespace. As we do not need it anymore, we can delete the VM. In this case, we have to be
careful as the fedora-cloud-nginx-base belongs to the VM and would be removed when we just delete the VM. We have to use
`--cascade=orphan` to not delete our provisioned disk.

Delete the VM without deleting the newly created PVC:

```bash
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-nginx-provisioner --cascade=orphan
```

Now we have our immutable custom VM image based on Fedora Cloud with nginx installed. We can now create as many VMs as we
want using that custom image.
