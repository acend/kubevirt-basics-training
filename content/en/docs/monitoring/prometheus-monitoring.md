---
title: "Prometheus Monitoring"
weight: 83
labfoldernumber: "08"
description: >
  Monitoring virtual machines with prometheus
---


Prometheus is an open-source monitoring and alerting toolkit designed for reliability and scalability. It collects real-time metrics from services and systems, stores them in a time-series database, and provides powerful querying capabilities. Prometheus operates with a pull-based model, scraping metrics from endpoints at regular intervals. It supports multi-dimensional data through labels, enabling flexible queries and insights. Paired with tools like Grafana for visualization, Prometheus is widely used for monitoring cloud-native applications, infrastructure, and system performance, with built-in alerting to notify users of potential issues.

It is the defacto standard tool set for monitoring workload on Kubernetes.

All KubeVirt components expose Prometheus metrics by default and are therefore easily integrable into an existing prometheus monitoring stack.


## {{% task %}} Raw KubeVirt Prometheus Metrics

Prometheus is highly integrated into the Kubernetes ecosystem. It uses a concept called service discovery to discover components within a Kubernetes cluster which expose metrics. All discovered components will then be scraped and the metrics end up in the above mentioned time-series database.

All Kubevirt Pods which expose metrics are labeled with `prometheus.kubevirt.io` and contain a port which is called `metrics`. In addition to that all the Pods are summarized in a Kubernetes Service `kubevirt-prometheus-metrics`.

Execute the following command to display the service.

```bash
kubectl describe service kubevirt-prometheus-metrics -n kubevirt
```
You can see a long list of endpoint addresses. This is the collection of KubeVirt components which expose prometheus metrics.

```bash
Name:                     kubevirt-prometheus-metrics
Namespace:                kubevirt
Labels:                   app.kubernetes.io/component=kubevirt
                          app.kubernetes.io/managed-by=virt-operator
                          kubevirt.io=
                          prometheus.kubevirt.io=true
Annotations:              kubevirt.io/customizer-identifier: bf21a9e8fbc5a3846fb05b4fa0859e0917b2202f
                          kubevirt.io/generation: 21
                          kubevirt.io/install-strategy-identifier: 96d0fd48fa88abe041085474347e87222b076258
                          kubevirt.io/install-strategy-registry: quay.io/kubevirt
                          kubevirt.io/install-strategy-version: v1.3.0
Selector:                 prometheus.kubevirt.io=true
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       None
IPs:                      None
Port:                     metrics  443/TCP
TargetPort:               metrics/TCP
Endpoints:                10.244.1.238:8443,10.244.5.229:8443,10.244.5.156:8443 + 10 more...
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

We can get the metrics provided by a pod, by simply sending an HTTP Get request to one of the endpoint addresses.

Execute the following command
```bash
kubectl describe endpoints -n kubevirt kubevirt-prometheus-metrics
```

and use the fist IP address in the Addresses list for the next command:

```bash
curl -k https://<Endpoint IP Address>:8443/metrics
```

The result will be a list of KubeVirt Metrics, this specific Pod exposes.


## Configure Prometheus to scrape KubeVirt Metrics

To integrate all those KubeVirt Components into a running Prometheus stack, the following configuration is required in the `KubeVirt` custom resource:

* monitorAccount: `<prometheus-serviceaccount>`
* monitorNamespace: `<prometheus-namesapce>`
* serviceMonitorNamespace: `<sm-namespace>`


```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
[...]
  monitorAccount: <prometheus-serviceaccount>`
  monitorNamespace: <namespace>
  serviceMonitorNamespace: <sm-namespace>
[...]
```

This will then have the effect, that KubeVirt itself will deploy the necessary resources to be integrated into the prometheus stack.

One of the most important components is the ServiceMonitor, which tells Prometheus where to scrape the KubeVirt Metrics from, as we have learned in the previous lab.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  annotations:
  name: prometheus-kubevirt-rules
  namespace: monitoring
spec:
  endpoints:
  - honorLabels: true
    port: metrics
    scheme: https
    tlsConfig:
      ca: {}
      cert: {}
      insecureSkipVerify: true
  namespaceSelector:
    matchNames:
    - kubevirt
  selector:
    matchLabels:
      prometheus.kubevirt.io: "true"
```

This integration has been done already on the lab cluster.

Alongside with the KubeVirt ServiceMonitor, KubeVirt also deployed a set of PrometheusRules (Alerts).

You can have a look at the Alerts by executing the following command:

```bash
kubectl get PrometheusRule prometheus-kubevirt-rules -n kubevirt -o yaml
```

Those Alerts are a very good monitoring foundation for our workload. Make sure in a production environment, that firing alerts are monitored and fixed.


## {{% task %}} Explore Prometheus UI

Ask the trainer for the correct prometheus URL and open the Prometheus UI in a separate browser tab.

First navigate to the Service Discovery page under Status --> Service Discovery. You will find the discovered `kubevirt-servicemonitor` ServiceMonitor.
The successful Service Discovery process will configure all the endpoints as seen above under Prometheus Targets, to scrape the metrics correctly. Check the Status --> Targets page to check on the KubeVirt Targets.

On the Alerts page, you see an overview of all configured Alerts, including KubeVirt Alerts.

Going back to the Main view, by clicking on the Prometheus Logo, start searching our data.

Execute the following queries:

**What KubeVirt verion is running?**
```promql
kubevirt_info
```

**How many VMs per Namespace exist?**
```promql
kubevirt_number_of_vms
```

**How much CPU time have the VMIs used?**
```promql
kubevirt_vmi_vcpu_seconds_total
```

You can also use the Graph tab to display the data over time.


## {{% task %}}  Explore the complete list of prometheus metrics in the documentation

The typeahead feature of the Prometheus UI allows you to search for metrics. All KubeVirt Metrics start with `kubevirt_`

You can find the complete list of KubeVirt metrics here: <https://github.com/kubevirt/monitoring/blob/main/docs/metrics.md>

Try to answer questions like:

* how much memory do my VMIs use?
* how many live migrations were successful?
* how much network traffic was received by VMI xy?