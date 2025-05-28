---
title: "Requirements"
weight: 91
labfoldernumber: "09"
description: >
  Requirements for the demo application.
---

For this lab we do not provide a step-by-step guide for the implementation. The goal is that you implement the required
manifests yourself. However, we provide a sample solution in the next chapter. Your solution may vary from ours.


## Basic Information

The goal of this super lab is to deploy a database and a web application connecting to this database. The database should
be run within a KubeVirt virtual machine. We do provide the web application as container image.

The setup should fulfill the following requirements:

Namespace:

* Use the same namespace as for your previous labs (`<user>`)

Virtual machine:

* Operating System: Fedora Cloud 40
  * Recommended image: `{{% param "fedoraCloudCDI" %}}`
* Database: MariaDB
* Metrics Exporter: Node Exporter

Web application:

* Application: Python Example Web Application from acend.
  * Image: `{{% param "exampleWebAppImage" %}}`
* The webapp will listen on port `5000`
* The connection to the database can be configured with the environment variable `MYSQL_URI=mysql://user:password@hostname/database-name`


### Disk provisioning

To reduce the startup time of the database, we recommend to use a pre provisioned persistent disk. Pre-provision this generic
database disk with the mariadb package and the setup for the node exporter. Use a provisioning virtual machine with cloud-init
scripts to prepare the disk.


### Runtime instances

The virtual machine running the mariadb instance should clone the pre-provisioned vm disk. Additionally, this instance
should mount another empty `1Gi` disk as data folder of the database. Use cloud-init to create and configure the database
users, create the database, start the required services and do the basic virtual machine configuration. You are encouraged
to define and use VirtualMachineInstanceType and VirtualMachinePreference to manage resource usage and preferences.

You should use a secret to store the database details. All components (database vm and webapp) should have health-checks
configured.  


## Required details

Database users and password

* database-name: `acend_exampledb`
* database-user: `acend_user`
* database-password: `mysqlpassword`
* database-root-password: `mysqlrootpassword`


## Advanced information

{{% alert title="Note" color="info" %}}
Here you'll find more useful information and hints for the setup. If you like you can skip this section and start
implementing on your own.
{{% /alert %}}

{{% details title="Show overview graphic" %}}
![Superlab Overview](../superlab-overview.png)
{{% /details %}}


### Disk provisioning

* Use a DataVolume to download the fedora image `{{% param "fedoraCloudCDI" %}}` to a vm disk.
  * Use this disk as the disk for the provisioning virtual machine
* Your virtual machine should use the `runStrategy: RerunOnFailure`
* Cloud-Init script should:
  * Install `mariadb-server` package
  * Create a system user `node_exporter` with a group `node_exporter` and the login shell `/bin/nologin`
  * Download node-exporter from `{{% param "nodeExporter" %}}`
    * Create a systemd service file to easily start the node exporter. You can find some details [in this gist](https://gist.github.com/jarek-przygodzki/735e15337a3502fea40beba27e193b04).
    * Configure the node exporter to run on port `9100`.
  * Do some cleanup
    * Remove the ssh host files `/etc/ssh/ssh_host_*`
    * Remove the `/var/lib/cloud/instances` directory to re-run cloud-init
  * Shutdown the virtual machine

You most likely need a cloud-init script like:
```yaml
#cloud-config-archive
- type: "text/cloud-config"
  content: |
    packages:
        [...]
    users:
        [...]
    write_files:
        [...]
- type: "text/x-shellscript"    
  content: |
    #!/bin/sh
    # install node_exporter
    # [...]
    # cleanup
    # [...]
    shutdown now
```

{{% details title="Hint for node exporter" %}}
Installation of node exporter:
```shell
curl -fsSL {{% param "nodeExporter" %}} | \
  sudo tar -zxvf - -C /usr/local/bin --strip-components=1 node_exporter-{{% param "nodeExporterVersion" %}}.linux-amd64/node_exporter && \
  sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Systemd service file `/etc/systemd/system/node_exporter.service`.
```shell
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
# Fallback when environment file does not exist
Environment=OPTIONS=
EnvironmentFile=-/etc/sysconfig/node_exporter
ExecStart=/usr/local/bin/node_exporter $OPTIONS
    
[Install]
WantedBy=multi-user.target
```
{{% /details %}}


### Database virtual machine

* Use a DataVolume to clone the provisioned disk and use another DataVolume to create an empty disk.
  * Use the cloned disk as the root disk for the database virtual machine
  * Mount the additional disk on `/var/lib/mysql`
* Your virtual machine should use the `runStrategy: RerunOnFailure`
* Create a secret containing
  * The database details
  * An init script for your mariadb database
* Cloud-Init script should:
  * Set password for fedora user
  * Mount secret with mysql details using virtiofs
  * Mount additional disk for the database data
  * Enable and start the mariadb and node exporter service
  * Load the database init script
  * Create a kubernetes Service for the node exporter and mariadb
* The easiest health checks are tcp probes against the mariadb port

You most likely need a cloud-init script like:
```yaml
#cloud-config
password: 
chpasswd: 
bootcmd:
  [...]
mounts:
  [...]
runcmd:
  [...]
```

Be aware of the runtime order of the cloud-init modules and place the commands accordingly. By default, the mounted disks are accessible by root. You may fix the permissions for the mysql user:

```shell
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 0751 /var/lib/mysql
```

{{% details title="Hint for secret creation and database init script" %}}
Basic `init.sql` script to setup the database:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'mysqlrootpassword';
CREATE DATABASE acend_exampledb;
CREATE USER 'acend_user'@'%' identified by 'mysqlpassword';
GRANT ALL PRIVILEGES ON acend_exampledb.* TO 'acend_user'@'%';
FLUSH PRIVILEGES;
```

Creating a secret containing the values for the database as well as the init script can be done with:
```shell
kubectl create secret generic lab09-mariadb \
  --from-literal=database-name=acend_exampledb \
  --from-literal=database-password=mysqlpassword \
  --from-literal=database-root-password=mysqlrootpassword \
  --from-literal=database-user=acend_user \
  --from-file=init.sql=init.sql
```
{{% /details %}}


### Web application

* Create a deployment for the web application
* In the deployment you need to set environment variable `MYSQL_URI`
  * Use the values (database-user, database-password, database-name) from the secret created above.
* The webapp has a `/health` rest endpoint to be used for health checks
* Create a service targeting port `5000` of the deployed webapp.
* Create an ingress pointing to the service of your webapp

Your ingress will be similar to:

```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  [...]
spec:
  rules:
    - host: webapp-<user>.<appdomain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: 
                [...]
  tls:
  - hosts:
    - webapp-<user>.<appdomain>
```

Make sure you replace the occurrences of

* `<user>` - your username (eg. `user4`)
* `<appdomain>` - ask the trainer

with the appropriate values.


{{% details title="Configuration Hint" %}}
The configuration of the webapp is done with environment variables. Your config will look similar to this one:
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  [...]
spec:
  [...]
  template:
    [...]
    spec:
      containers:
        - image: quay.io/acend/example-web-python:latest
          name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
          [...]
          env:
          - name: MYSQL_DATABASE_NAME
            valueFrom:
              secretKeyRef:
                key: database-name
                name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
          - name: MYSQL_DATABASE_PASSWORD
            valueFrom:
              secretKeyRef:
                key: database-password
                name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
          - name: MYSQL_DATABASE_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                key: database-root-password
                name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
          - name: MYSQL_DATABASE_USER
            valueFrom:
              secretKeyRef:
                key: database-user
                name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
          - name: MYSQL_URI
            value: mysql://$(MYSQL_DATABASE_USER):$(MYSQL_DATABASE_PASSWORD)@{{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb/$(MYSQL_DATABASE_NAME)
```
{{% /details %}}

