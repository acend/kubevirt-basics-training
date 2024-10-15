---
title: "Sample Solution"
weight: 92
labfoldernumber: "09"
description: >
  Sample solution for the super lab.
---

As we did not provide a step-by-step guide for the super lab, you can find a sample solution for the lab as kubernetes resources. This is a sample solution and your outcome may vary.
The following code blocks contain multiple manifests separated with `---`. You can write them to one file (e.g. `multiple-manifests.yaml`) and apply them all together.

Make sure all occurrences of

* `<user>` - your username (eg. `user4`)
* `<appdomain>` - ask the trainer

are replaced accordingly, before creating the manifests with:

```shell
kubectl create -f multiple-manifests.yaml
```

If you want to inspect the used cloud-init scripts you can decode the base64 string from the secrets in the code blocks below. Copy the base64 string and use the following command:
```shell
echo -n "<base64string>" | base64 -d
kubectl get secrets lab09-cloudinit-mariadb-provisioner -o jsonpath={'.data.userdata'} | base64 -d
```


## Provision the Fedora MariaDB database disk

To create a template disk for spinning up MariaDB database instances we used the following kubernetes manifests.

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-base
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
---
apiVersion: v1
kind: Secret
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-mariadb-provisioner
data:
  userdata: I2Nsb3VkLWNvbmZpZy1hcmNoaXZlCi0gdHlwZTogInRleHQvY2xvdWQtY29uZmlnIgogIGNvbnRlbnQ6IHwKICAgIHBhY2thZ2VzOgogICAgICAtIG1hcmlhZGItc2VydmVyCiAgICB1c2VyczoKICAgICAgLSBuYW1lOiBub2RlX2V4cG9ydGVyCiAgICAgICAgZ2Vjb3M6IE5vZGUgRXhwb3J0ZXIgVXNlcgogICAgICAgIHByaW1hcnlfZ3JvdXA6IG5vZGVfZXhwb3J0ZXIKICAgICAgICBncm91cHM6IG5vZGVfZXhwb3J0ZXIKICAgICAgICBzaGVsbDogL2Jpbi9ub2xvZ2luCiAgICAgICAgc3lzdGVtOiB0cnVlCiAgICB3cml0ZV9maWxlczoKICAgICAgLSBjb250ZW50OiB8CiAgICAgICAgICBbVW5pdF0KICAgICAgICAgIERlc2NyaXB0aW9uPU5vZGUgRXhwb3J0ZXIKICAgICAgICAgIEFmdGVyPW5ldHdvcmsudGFyZ2V0CiAgICAgICAgICAKICAgICAgICAgIFtTZXJ2aWNlXQogICAgICAgICAgVXNlcj1ub2RlX2V4cG9ydGVyCiAgICAgICAgICBHcm91cD1ub2RlX2V4cG9ydGVyCiAgICAgICAgICAjIEZhbGxiYWNrIHdoZW4gZW52aXJvbm1lbnQgZmlsZSBkb2VzIG5vdCBleGlzdAogICAgICAgICAgRW52aXJvbm1lbnQ9T1BUSU9OUz0KICAgICAgICAgIEVudmlyb25tZW50RmlsZT0tL2V0Yy9zeXNjb25maWcvbm9kZV9leHBvcnRlcgogICAgICAgICAgRXhlY1N0YXJ0PS91c3IvbG9jYWwvYmluL25vZGVfZXhwb3J0ZXIgJE9QVElPTlMKICAgICAgICAgICAgICAgIAogICAgICAgICAgW0luc3RhbGxdCiAgICAgICAgICBXYW50ZWRCeT1tdWx0aS11c2VyLnRhcmdldAogICAgICAgIHBhdGg6IC9ldGMvc3lzdGVtZC9zeXN0ZW0vbm9kZV9leHBvcnRlci5zZXJ2aWNlCiAgICAgIC0gY29udGVudDogfAogICAgICAgICAgT1BUSU9OUz0iLS13ZWIubGlzdGVuLWFkZHJlc3M9MC4wLjAuMDo5MTAwIgogICAgICAgIHBhdGg6IC9ldGMvc3lzY29uZmlnL25vZGVfZXhwb3J0ZXIKLSB0eXBlOiAidGV4dC94LXNoZWxsc2NyaXB0IiAgICAKICBjb250ZW50OiB8ICAgIAogICAgIyEvYmluL3NoICAgIAogICAgIyBpbnN0YWxsIG5vZGVfZXhwb3J0ZXIKICAgIGN1cmwgLWZzU0wgaHR0cHM6Ly9naXRodWIuY29tL3Byb21ldGhldXMvbm9kZV9leHBvcnRlci9yZWxlYXNlcy9kb3dubG9hZC92MS44LjIvbm9kZV9leHBvcnRlci0xLjguMi5saW51eC1hbWQ2NC50YXIuZ3ogfCBzdWRvIHRhciAtenh2ZiAtIC1DIC91c3IvbG9jYWwvYmluIC0tc3RyaXAtY29tcG9uZW50cz0xIG5vZGVfZXhwb3J0ZXItMS44LjIubGludXgtYW1kNjQvbm9kZV9leHBvcnRlciAmJiBzdWRvIGNob3duIG5vZGVfZXhwb3J0ZXI6bm9kZV9leHBvcnRlciAvdXNyL2xvY2FsL2Jpbi9ub2RlX2V4cG9ydGVyCiAgICAjIGNsZWFudXAKICAgIHJtIC1yZiAvZXRjL3NzaC9zc2hfaG9zdF8qCiAgICAjIHJlbW92aW5nIGluc3RhbmNlcyBlbnN1cmVzIGNsb3VkIGluaXQgd2lsbCBleGVjdXRlIGFnYWluIGFmdGVyIHJlYm9vdCAgICAKICAgIHJtIC1yZiAvdmFyL2xpYi9jbG91ZC9pbnN0YW5jZXMgICAgCiAgICBzaHV0ZG93biBub3cgCg==
type: Opaque
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-provisioner
spec:
  runStrategy: "RerunOnFailure"
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-provisioner
    spec:
      domain:
        devices:
          disks:
            - name: fedora-mariadb
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
      {{< onlyWhen tolerations >}}tolerations:
      - effect: NoSchedule
        key: baremetal
        operator: Equal
        value: "true"
      {{< /onlyWhen >}}networks:
        - name: default
          pod: {}
      volumes:
        - name: fedora-mariadb
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-base
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-mariadb-provisioner
```


## Creating the Fedora MariaDB virtual machine

To create a MariaDB database form the provisioned template disk we used the following kubernetes manifests.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
spec:
  ports:
  - port: 3306
  selector:
    kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
  clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-node-exporter
spec:
  ports:
  - port: 9100
    protocol: TCP
    targetPort: 9100
  selector:
    kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
  type: ClusterIP
---
apiVersion: v1
kind: Secret
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-mariadb
data:
  userdata: I2Nsb3VkLWNvbmZpZwpwYXNzd29yZDoga3ViZXZpcnQKY2hwYXNzd2Q6IHsgZXhwaXJlOiBGYWxzZSB9CnRpbWV6b25lOiBFdXJvcGUvWnVyaWNoCmJvb3RjbWQ6CiAgLSBzdWRvIHRlc3QgLXogIiQoc3VkbyBibGtpZCAvZGV2L3ZkYykiICYmIHN1ZG8gbWtmcyAtdCBleHQ0IC1MIGJsb2NrZGlzayAvZGV2L3ZkYyAKICAtIHN1ZG8gbWtkaXIgLXAgL3NlY3JldHMvbWFyaWFkYgogIC0gc3VkbyBtb3VudCAtdCB2aXJ0aW9mcyBtYXJpYWRiIC9zZWNyZXRzL21hcmlhZGIKbW91bnRzOgogIC0gWyIvZGV2L3ZkYyIsICIvdmFyL2xpYi9teXNxbCIsICJleHQ0IiwgImRlZmF1bHRzLG5vZmFpbCIsICIwIiwgIjIiIF0KcnVuY21kOgogICAtIHN1ZG8gY2hvd24gbXlzcWw6bXlzcWwgL3Zhci9saWIvbXlzcWwKICAgLSBzdWRvIGNobW9kIDA3NTEgL3Zhci9saWIvbXlzcWwKICAgLSBzdWRvIHN5c3RlbWN0bCBlbmFibGUgLS1ub3cgbWFyaWFkYgogICAtIHN1ZG8gc3lzdGVtY3RsIGVuYWJsZSAtLW5vdyBub2RlX2V4cG9ydGVyCiAgIC0gc3VkbyBteXNxbCAtdXJvb3QgPCAvc2VjcmV0cy9tYXJpYWRiL2luaXQuc3FsCgo=
type: Opaque
---
apiVersion: v1
kind: Secret
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
data:
  database-name: YWNlbmRfZXhhbXBsZWRi
  database-password: bXlzcWxwYXNzd29yZA==
  database-root-password: bXlzcWxyb290cGFzc3dvcmQ=
  database-user: YWNlbmRfdXNlcg==
  init.sql: QUxURVIgVVNFUiAncm9vdCdAJ2xvY2FsaG9zdCcgSURFTlRJRklFRCBCWSAnbXlzcWxyb290cGFzc3dvcmQnOwpDUkVBVEUgREFUQUJBU0UgYWNlbmRfZXhhbXBsZWRiOwpDUkVBVEUgVVNFUiAnYWNlbmRfdXNlcidAJyUnIGlkZW50aWZpZWQgYnkgJ215c3FscGFzc3dvcmQnOwpHUkFOVCBBTEwgUFJJVklMRUdFUyBPTiBhY2VuZF9leGFtcGxlZGIuKiBUTyAnYWNlbmRfdXNlcidAJyUnOwpGTFVTSCBQUklWSUxFR0VTOwo=
type: Opaque
---
apiVersion: cdi.kubevirt.io/v1beta1    
kind: DataVolume    
metadata:    
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
spec:    
  storage:    
    accessModes:    
    - ReadWriteOnce    
  source:    
    pvc:    
      name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-base
      namespace: <user>
---
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-data
spec:
  storage:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
    volumeMode: Filesystem
  source:
    blank: {}
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachineInstancetype
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-mariadb
  labels:
    instancetype.kubevirt.io/class: general.purpose
    instancetype.kubevirt.io/cpu: "1"
    instancetype.kubevirt.io/icon-pf: pficon-server-group
    instancetype.kubevirt.io/memory: 2Gi
    instancetype.kubevirt.io/vendor: kubevirt-basics-training
    instancetype.kubevirt.io/version: "1"
spec:
  cpu:
    guest: 1
  memory:
    guest: 2Gi
---
apiVersion: instancetype.kubevirt.io/v1beta1
kind: VirtualMachinePreference
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-mariadb
  annotations:
    iconClass: icon-fedora
    openshift.io/display-name: Fedora MariaDB
    tags: hidden,kubevirt,fedora,mariadb
  labels:
    app.kubernetes.io/component: kubevirt
    instancetype.kubevirt.io/os-type: linux
spec:
  devices:
    preferredDiskBus: virtio
    preferredInterfaceModel: virtio
  requirements:
    cpu:
      guest: 1
    memory:
      guest: 2Gi
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
spec:
  runStrategy: RerunOnFailure
  instancetype:
    kind: VirtualMachineInstancetype
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-mariadb
  preference:
    kind: VirtualMachinePreference
    name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-fedora-mariadb
  template:
    metadata:
      labels:
        kubevirt.io/domain: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
    spec:
      domain:
        devices:
          filesystems:
            - name: mariadb
              virtiofs: {}
          disks:
            - name: fedora-disk
              disk: {}
            - name: cloudinitdisk
              disk: {}
            - name: mariadb-data
              disk: {}
          interfaces:
          - name: default
            masquerade: {}
      readinessProbe:
        initialDelaySeconds: 90
        tcpSocket:
          port: 3306 
      livenessProbe:
        initialDelaySeconds: 90
        tcpSocket:
          port: 3306 
      networks:
      - name: default
        pod: {}
      {{< onlyWhen tolerations >}}tolerations:
      - effect: NoSchedule
        key: baremetal
        operator: Equal
        value: "true"
      {{< /onlyWhen >}}volumes:
        - name: fedora-disk
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
        - name: cloudinitdisk
          cloudInitNoCloud:
            secretRef:
              name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-mariadb
        - name: mariadb-data
          persistentVolumeClaim:
            claimName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-data
        - name: mariadb
          secret:
            secretName: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb
```


## Creating the web application container

With the following kubernetes manifests we start a web application pod connecting to our MariaDB database.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
spec:
  ports:
  - port: 5000
    protocol: TCP
    targetPort: 5000
  selector:
    app: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
spec:
  rules:
    - host: webapp-<user>.<appdomain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: 
                name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
                port: 
                  number: 5000
  tls:
  - hosts:
    - webapp-<user>.<appdomain>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
  name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
    spec:
      containers:
        - image: {{% param "exampleWebAppImage" %}}
          name: {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp
          resources:
            limits:
              cpu: 100m
              memory: 128Mi
            requests:
              cpu: 50m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 3
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


## End of lab

{{% alert title="Cleanup resources" color="warning" %}}  {{% param "end-of-lab-text" %}}

If you applied the manifests above, delete the resources with:

```bash
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb --namespace=$USER
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-base --namespace=$USER
kubectl delete dv {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-data --namespace=$USER
kubectl delete secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb --namespace=$USER
kubectl delete secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-mariadb-provisioner --namespace=$USER
kubectl delete secret {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-cloudinit-mariadb --namespace=$USER
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb-provisioner --namespace=$USER
kubectl delete vm {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb --namespace=$USER
kubectl delete svc {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-mariadb --namespace=$USER
kubectl delete svc {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp --namespace=$USER
kubectl delete svc {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-node-exporter --namespace=$USER
kubectl delete ingress {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp --namespace=$USER
kubectl delete deployment {{% param "labsubfolderprefix" %}}{{% param "labfoldernumber" %}}-webapp --namespace=$USER
```

{{% /alert %}}
