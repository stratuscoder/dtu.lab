id: dt-k8s-otel-o11y-logs
summary: dynatrace log ingest for kubernetes using opentelemetry collector
author: Tony Pope-Cruz

# Kubernetes Log Ingest with OpenTelemetry & Dynatrace
<!-- ------------------------ -->
## Overview 
Total Duration: 20

### What You’ll Learn Today
In this lab we'll utilize the OpenTelemetry Collector deployed as a DaemonSet (Node Agent) to collect pod/container logs from a Kubernetes cluster and ship them to Dynatrace.

Lab tasks:
1. Create a Kubernetes cluster on Google GKE
2. Deploy OpenTelemetry's demo application, astronomy-shop
3. Deploy OpenTelemetry Collector as a DaemonSet
4. Configure OpenTelemetry Collector service pipeline for log enrichment
5. Query and visualize logs in Dynatrace using DQL

![astronomy-shop logs](img/astronomy-shop_logs.png)

<!-- -------------------------->
## Technical Specification 
Duration: 2

#### Technologies Used
- [Dynatrace](https://www.dynatrace.com/trial)
- [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine)
  - tested on GKE v1.29.4-gke.1043002
- [OpenTelemetry Demo astronomy-shop](https://opentelemetry.io/docs/demo/)
  - tested on release 1.10.0
- [Istio](https://istio.io/latest/docs/)
  - tested on v1.22.1
- [OpenTelemetry Collector - Dynatrace Distro](https://docs.dynatrace.com/docs/extend-dynatrace/opentelemetry/collector/deployment)
  - tested on v0.8.0
- [OpenTelemetry Collector - Contrib Distro](https://github.com/open-telemetry/opentelemetry-collector-contrib/releases/tag/v0.103.0)
  - tested on v0.103.0

#### Reference Architecture
TODO

#### Prerequisites
- Google Cloud Account
- Google Cloud Project
- Google Cloud Access to Create and Manage GKE Clusters
- Google CloudShell Access

<!-- -------------------------->
## Setup
Duration: 18

### Prerequisites

#### Deploy GKE cluster & demo assets
https://github.com/popecruzdt/dt-k8s-otel-o11y-cluster

#### Clone the repo to your home directory
Command:
```sh
git clone https://github.com/popecruzdt/dt-k8s-otel-o11y-logs.git
```
Sample output:
> Cloning into 'dt-k8s-otel-o11y-logs'...\
> ...\
> Receiving objects: 100% (12/12), 10.61 KiB | 1.77 MiB/s, done.

#### Move into repo base directory
Command:
```sh
cd dt-k8s-otel-o11y-logs
```

#### Generate Dynatrace Access Token
Generate a new API access token with the following scopes:
```
Ingest events
Ingest logs
Ingest metrics
Ingest OpenTelemetry traces
```
[See Related Dynatrace API Token Creation Documentation](https://docs.dynatrace.com/docs/dynatrace-api/basics/dynatrace-api-authentication#create-token)
![dt access token](img/dt_access_token.png)

#### (optional) Import Notebook into Dynatrace
[notebook](/k8s-otel-o11y-logs_dt_notebook.json)

#### Define workshop user variables
In your GCP CloudShell Terminal:
```
DT_ENDPOINT=https://{your-environment-id}.live.dynatrace.com/api/v2/otlp
DT_API_TOKEN={your-api-token}
```
### OpenTelemetry Collector - Dynatrace Distro
https://docs.dynatrace.com/docs/extend-dynatrace/opentelemetry/collector/deployment

#### Create `dynatrace` namespace
Command:
```sh
kubectl create namespace dynatrace
```
Sample output:
> namespace/dynatrace created

#### Create `dynatrace-otelcol-dt-api-credentials` secret
Command:
```sh
kubectl create secret generic dynatrace-otelcol-dt-api-credentials --from-literal=DT_ENDPOINT=$DT_ENDPOINT --from-literal=DT_API_TOKEN=$DT_API_TOKEN -n dynatrace
```
Sample output:
> secret/dynatrace-otelcol-dt-api-credentials created

#### Deploy `cert-manager`, pre-requisite for `opentelemetry-operator`
https://cert-manager.io/docs/installation/

Command:
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```
Sample output:
> namespace/cert-manager created\
> customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created\
> customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created\
> ...\
> validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created

#### Deploy `opentelemetry-operator`
Command:
```sh
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```
Sample output:
> namespace/opentelemetry-operator-system created\
> customresourcedefinition.apiextensions.k8s.io/instrumentations.opentelemetry.io created\
> customresourcedefinition.apiextensions.k8s.io/opampbridges.opentelemetry.io created\
> ...\
> validatingwebhookconfiguration.admissionregistration.k8s.io/opentelemetry-operator-validating-webhook-configuration configured

#### Deploy OpenTelemetry Collector - Dynatrace Distro - Daemonset (Node Agent)
https://docs.dynatrace.com/docs/extend-dynatrace/opentelemetry/collector/deployment#tabgroup--dynatrace-docs--agent
```yaml
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: dynatrace-logs
  namespace: dynatrace
spec:
  envFrom:
  - secretRef:
      name: dynatrace-otelcol-dt-api-credentials
  mode: "daemonset"
  image: "ghcr.io/dynatrace/dynatrace-otel-collector/dynatrace-otel-collector:latest"
```
Command:
```sh
kubectl apply -f opentelemetry/collector/logs/otel-collector-logs-crd-01.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-logs created

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-8q8tz   | 1/1   | Running | 0        | 1m  |

##### `filelog` receiver
https://opentelemetry.io/docs/kubernetes/collector/components/#filelog-receiver
```yaml
config: |
    receivers:
      filelog:
        ...
    service:
      pipelines:
        logs:
          receivers: [filelog]
          processors: [batch]
          exporters: [otlphttp/dynatrace]
```

##### Query logs in Dynatrace
DQL:
```sql
fetch logs
| filter isNotNull(log.file.path) and isNotNull(log)
| sort timestamp desc
| limit 100
| fields timestamp, loglevel, status, k8s.namespace.name, k8s.pod.name, k8s.container.name, content, log.file.path
```
Result:\
![dql_filelog_receiver](img/dql_filelog_receiver.png)

##### Create `clusterrole` with read access to Kubernetes objects
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-k8s-clusterrole-logs
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces", "nodes"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]
```
Command:
```sh
kubectl apply -f opentelemetry/rbac/otel-collector-k8s-clusterrole-logs.yaml
```
Sample output:
> clusterrole.rbac.authorization.k8s.io/otel-collector-k8s-clusterrole-logs created

##### Create `clusterrolebinding` for OpenTelemetry Collector service account
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-k8s-clusterrole-logs-crb
subjects:
- kind: ServiceAccount
  name: dynatrace-logs-collector
  namespace: dynatrace
roleRef:
  kind: ClusterRole
  name: otel-collector-k8s-clusterrole-logs
  apiGroup: rbac.authorization.k8s.io
```
Command:
```sh
kubectl apply -f opentelemetry/rbac/otel-collector-k8s-clusterrole-logs-crb.yaml
```
Sample output:
> clusterrolebinding.rbac.authorization.k8s.io/otel-collector-k8s-clusterrole-logs-crb created

##### Add `k8sattributes` processor
https://opentelemetry.io/docs/kubernetes/collector/components/#kubernetes-attributes-processor
```yaml
k8sattributes:
    auth_type: "serviceAccount"
    passthrough: false
        filter:
        node_from_env_var: KUBE_NODE_NAME
    extract:
        metadata:
            - k8s.namespace.name
            - k8s.deployment.name
            - k8s.daemonset.name
            - k8s.job.name
            - k8s.cronjob.name
            - k8s.replicaset.name
            - k8s.statefulset.name
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.node.name
            - k8s.container.name
            - container.id
            - container.image.name
            - container.image.tag
        labels:
        - tag_name: app.label.component
            key: app.kubernetes.io/component
            from: pod
    pod_association:
        - sources:
            - from: resource_attribute
              name: k8s.pod.uid
        - sources:
            - from: resource_attribute
              name: k8s.pod.name
        - sources:
            - from: resource_attribute
              name: k8s.pod.ip
        - sources:
            - from: connection
```
Command:
```sh
kubectl apply -f opentelemetry/collector/logs/otel-collector-logs-crd-02.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-logs configured

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-dns4x   | 1/1   | Running | 0        | 1m  |

##### Query logs in Dynatrace
DQL:
```sql
fetch logs
| filter k8s.namespace.name == "astronomy-shop" and isNotNull(k8s.deployment.name)
| sort timestamp desc
| limit 100
| fields timestamp, loglevel, status, k8s.namespace.name, k8s.deployment.name, k8s.pod.name, k8s.container.name, app.label.component, content
```
Result:\
![dql_k8sattributes_processor](img/dql_k8sattributes_processor.png)

##### Add `resourcedetection` processor (gcp)
https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/resourcedetectionprocessor/README.md#gcp-metadata
```yaml
processors:
  resourcedetection/gcp:
    detectors: [env, gcp]
    timeout: 2s
    override: false
```
Command:
```sh
kubectl apply -f opentelemetry/collector/logs/otel-collector-logs-crd-03.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-logs configured

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-fbtk5   | 1/1   | Running | 0        | 1m  |

##### Query logs in Dynatrace
DQL:
```sql
fetch logs
| filter isNotNull(cloud.account.id) and isNotNull(k8s.cluster.name)
| filter k8s.namespace.name == "astronomy-shop" and isNotNull(k8s.deployment.name)
| sort timestamp desc
| limit 100
| fields timestamp, loglevel, status, cloud.account.id, k8s.cluster.name, k8s.namespace.name, k8s.deployment.name, content
```
Result:\
![dql_resourcedetection_processor](img/dql_resourcedetection_processor.png)

##### Add `resource` processor (attributes)
https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourceprocessor
```yaml
processors:
    resource:
        attributes:
        - key: k8s.pod.ip
          action: delete
        - key: telemetry.sdk.name
          value: opentelemetry
          action: insert
        - key: dynatrace.otel.collector
          value: dynatrace-logs
          action: insert
        - key: dt.security_context
          from_attribute: k8s.cluster.name
          action: insert
```
Command:
```sh
kubectl apply -f opentelemetry/collector/logs/otel-collector-logs-crd-04.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-logs configured

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-logs-collector-xx6km   | 1/1   | Running | 0        | 1m  |

##### Query logs in Dynatrace
DQL:
```sql
fetch logs
| filter isNotNull(dt.security_context)
| filter isNotNull(cloud.account.id) and isNotNull(k8s.cluster.name)
| filter k8s.namespace.name == "astronomy-shop" and isNotNull(k8s.deployment.name)
| sort timestamp desc
| limit 100
| fields timestamp, loglevel, status, dt.security_context, dynatrace.otel.collector, cloud.account.id, k8s.cluster.name, k8s.namespace.name, k8s.deployment.name, content
```
Result:\
![dql_resource_processor](img/dql_resource_processor.png)

##### Add `k8sobjects` receiver to collect Kubernetes events as logs
https://opentelemetry.io/docs/kubernetes/collector/components/#kubernetes-objects-receiver
```yaml
receivers:
  k8sobjects/events:
    auth_type: serviceAccount
    objects:
      - name: events
        mode: watch
        namespaces: [astronomy-shop,dynatrace]
```

The `k8sobjects` receiver is only available on the Contrib Distro of the OpenTelemetry Collector.  Therefore we must deploy a new Collector using the `contrib` image.

#### Deploy OpenTelemetry Collector - Contrib Distro - Deployment (Gateway)
https://github.com/open-telemetry/opentelemetry-operator
```yaml
---
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: dynatrace-events
  namespace: dynatrace
spec:
  envFrom:
  - secretRef:
      name: dynatrace-otelcol-dt-api-credentials
  mode: "deployment"
  image: "otel/opentelemetry-collector-contrib:0.103.0"
```
Command:
```sh
kubectl apply -f opentelemetry/collector/events/otel-collector-events-crd-01.yaml
```
Sample output:
> opentelemetrycollector.opentelemetry.io/dynatrace-events created

##### Validate running pod(s)
Command:
```sh
kubectl get pods -n dynatrace
```
Sample output:
| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| dynatrace-events-collector-559d5b9d77-rb26d   | 1/1   | Running | 0        | 1m  |

##### Create `clusterrole` with read access to Kubernetes events
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-k8s-clusterrole-events
rules:
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "watch", "list"]
```
Command:
```sh
kubectl apply -f opentelemetry/rbac/otel-collector-k8s-clusterrole-events.yaml
```
Sample output:
> clusterrole.rbac.authorization.k8s.io/otel-collector-k8s-clusterrole-events created

##### Create `clusterrolebinding` for OpenTelemetry Collector service account
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-k8s-clusterrole-events-crb
subjects:
- kind: ServiceAccount
  name: dynatrace-events-collector
  namespace: dynatrace
roleRef:
  kind: ClusterRole
  name: otel-collector-k8s-clusterrole-events
  apiGroup: rbac.authorization.k8s.io
```
Command:
```sh
kubectl apply -f opentelemetry/rbac/otel-collector-k8s-clusterrole-events-crb.yaml
```
Sample output:
> clusterrolebinding.rbac.authorization.k8s.io/otel-collector-k8s-clusterrole-events-crb created

Command:
```sh
kubectl apply -f opentelemetry/collector/events/otel-collector-events-crd-01.yaml
```

##### Generate events using deployment scale command
https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/

Command:
```sh
kubectl scale deployment astronomy-shop-imageprovider -n astronomy-shop --replicas=2
```
Sample output:
> deployment.apps/astronomy-shop-imageprovider scaled

Command:
```sh
kubectl scale deployment astronomy-shop-imageprovider -n astronomy-shop --replicas=1
```
Sample output:
> deployment.apps/astronomy-shop-imageprovider scaled

##### Query logs in Dynatrace
DQL:
```sql
fetch logs
| filter dynatrace.otel.collector == "dynatrace-events" and event.domain == "k8s" and k8s.resource.name == "events"
| filter object.metadata.namespace == "astronomy-shop"
| sort timestamp desc
| limit 100
| fields timestamp, k8s.cluster.name, {object.metadata.namespace, alias: k8s.namespace.name}, object.message, object.reason, event.name
```
Result:\
![dql_k8s_events](img/dql_k8s_events.png)

<!-- ------------------------ -->
## Demo The New Functionality
TODO

<!-- -------------------------->
## Wrap Up

### What You Learned Today 
By completing this lab, you've successfully deployed the OpenTelemetry Collector to collect logs, enrich log attributes for better context, and ship those logs to Dynatrace for analysis.
- The OpenTelemetry Collector was deployed as a DaemonSet, behaving as an Agent running on each Node
- The Dynatrace Distro of OpenTelemetry Collector includes supported modules needed to ship logs to Dynatrace
  - The `filelog` receiver scrapes logs from the Node filesystem and parses the contents
  - The `k8sattributes` processor enriches the logs with Kubernetes attributes
  - The `resourcedetection` processor enriches the logs with cloud and cluster (GCP/GKE) attributes
  - The `resource` processor enriches the logs with custom (resource) attributes
- The Community Contrib Distro of OpenTelemetry Collector includes modules needed to ship events to Dynatrace
  - The `k8sobjects` receiver watches for Kubernetes events (and other resources) on the cluster
- Dynatrace DQL (via Notebooks) allows you to perform powerful queries and analysis of the log data

<!-- ------------------------ -->
### Supplemental Material
TODO
