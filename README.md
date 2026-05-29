# monitoring-application

Minikube monitoring stack for the demo platform:

- Prometheus scrapes Kubernetes and microservice metrics.
- Grafana reads Prometheus, Loki, and Tempo datasources.
- Grafana evaluates alert rules and sends email notifications.
- kube-state-metrics exposes Kubernetes object state metrics.
- node-exporter exposes node CPU, memory, disk, and network metrics.
- kubelet cAdvisor metrics expose container CPU, memory, and filesystem metrics.
- Loki stores logs.
- Tempo stores traces.
- Alloy collects logs/traces and forwards them to Loki/Tempo.

## Grafana Email Alerts

Update the Grafana SMTP secret before applying the monitoring kustomization:

```yaml
monitoring-application/k8s/grafana/secret.yaml
```

Replace these placeholder values:

```text
GF_SMTP_HOST
GF_SMTP_USER
GF_SMTP_PASSWORD
GF_SMTP_FROM_ADDRESS
ALERT_EMAIL_TO
```

For Gmail, use an app password for `GF_SMTP_PASSWORD`.

Grafana alerting is provisioned from:

```text
monitoring-application/k8s/grafana/alerting.yaml
```

Provisioned Grafana resources:

- Contact point: `email-alerts`
- Notification policy: sends alerts to `email-alerts`
- Alert folder: `Microservices`

Current Grafana-managed alerts:

- `MicroserviceDown`
- `MicroserviceHighErrorRate`
- `MicroserviceNoTraffic`

To test a 5xx alert after the apps are deployed:

```bash
for i in $(seq 1 30); do curl -s http://microservices.local/orders/fail >/dev/null; done
```

To test a down alert:

```bash
kubectl scale deployment/order-service -n microservices --replicas=0
```

Scale it back after the alert fires:

```bash
kubectl scale deployment/order-service -n microservices --replicas=1
```

In Grafana, open:

```text
Alerts & IRM -> Alerting -> Alert rules
```

The provisioned rules appear under the `Microservices` folder. Provisioned rules are managed from Git, so edit `k8s/grafana/alerting.yaml` and restart Grafana to change them.

Minikube-friendly Kubernetes monitoring stack with:

- Grafana for dashboards
- Prometheus for metrics
- Loki for logs
- Tempo for distributed traces
- Grafana Alloy for Kubernetes pod log collection
  and OpenTelemetry trace forwarding

All resources deploy into the `monitoring` namespace.

Pinned image versions:

- Grafana: `grafana/grafana:13.0.1-security-01`
- Prometheus: `prom/prometheus:v3.11.3`
- Loki: `grafana/loki:3.7.1`
- Tempo: `grafana/tempo:2.8.2`
- Alloy: `grafana/alloy:v1.16.0`
- kube-state-metrics: `registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.15.0`
- node-exporter: `quay.io/prometheus/node-exporter:v1.9.1`

## Repository Layout

```text
k8s/      Kustomize Kubernetes manifests
```

## Prerequisites

- Minikube is running.
- `kubectl` is pointed at the Minikube cluster.
- Internet access is available from Minikube nodes to pull public images.
- For GitOps deployment, the Argo CD `Application` is managed from the
  `gitops-platform-minikube` repository.

## Configure Grafana Admin Password

The GitOps manifests create the Grafana admin secret:

```text
username: admin
password: admin123
```

For manual deploys, the equivalent command is:

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic grafana-admin \
  -n monitoring \
  --from-literal=admin-password='admin123' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Grafana username is `admin`.

## Deploy With kubectl

```bash
kubectl apply -k k8s
kubectl rollout status deployment/prometheus -n monitoring
kubectl rollout status deployment/loki -n monitoring
kubectl rollout status deployment/grafana -n monitoring
kubectl rollout status daemonset/alloy -n monitoring
```

## Deploy With GitOps

Argo CD deployment configuration is owned by the `gitops-platform-minikube`
repository. That repo defines the Argo CD `Application` which syncs the `k8s/`
kustomization from this repository into the `monitoring` namespace.

Keep this repository focused on monitoring manifests. Do not add Argo CD
`Application` or `AppProject` manifests here.

## Access Grafana

Using the NodePort service:

```bash
minikube service grafana -n monitoring
```

Or port-forward:

```bash
kubectl port-forward -n monitoring svc/grafana 3000:3000
```

Then open:

```text
http://localhost:3000
```

Grafana is provisioned with:

- Prometheus: `http://prometheus.monitoring.svc.cluster.local:9090`
- Loki: `http://loki.monitoring.svc.cluster.local:3100`
- Tempo: `http://tempo.monitoring.svc.cluster.local:3200`

## Access Prometheus and Loki Directly

```bash
kubectl port-forward -n monitoring svc/prometheus 9090:9090
kubectl port-forward -n monitoring svc/loki 3100:3100
kubectl port-forward -n monitoring svc/tempo 3200:3200
```

Prometheus:

```text
http://localhost:9090
```

Loki readiness:

```text
http://localhost:3100/ready
```

Tempo readiness:

```text
http://localhost:3200/ready
```

## Scrape Application Metrics

Prometheus discovers pods with these annotations:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
```

Add those annotations to any application pod that exposes Prometheus metrics.

For the `microservices-deployment` repo, the Express services expose `/metrics` with `prom-client`, and the service Deployments already include these annotations.

Example pod template annotation block:

```yaml
template:
  metadata:
    labels:
      app: user-service
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "3000"
      prometheus.io/path: "/metrics"
```

## Kubernetes Dashboards

Imported Kubernetes dashboards usually need several metric families:

- `kube_*` metrics from kube-state-metrics
- `node_*` metrics from node-exporter
- `container_*` metrics from kubelet cAdvisor
- app metrics such as `http_requests_total`

If a dashboard shows `N/A`, open a Grafana panel, inspect the PromQL query, and check whether that metric exists in Prometheus.

Useful Prometheus checks:

```promql
up
kube_pod_info
kube_deployment_status_replicas
node_cpu_seconds_total
node_memory_MemAvailable_bytes
container_cpu_usage_seconds_total
http_requests_total
```

After changing Prometheus scrape config, reload/restart Prometheus:

```bash
kubectl rollout restart deployment/prometheus -n monitoring
kubectl rollout status deployment/prometheus -n monitoring
```

## Logs

Alloy runs as a DaemonSet and reads pod logs from `/var/log/pods` on each Minikube node. It forwards logs to Loki with labels such as:

- `namespace`
- `pod`
- `container`
- `app`

In Grafana Explore, select the Loki datasource and try:

```logql
{namespace="microservices"}
```

## Traces With Tempo

Tempo stores distributed traces. A trace follows one request as it moves between services.

Example request path:

```text
frontend -> user-service -> order-service -> payment-service -> notification-service
```

The monitoring path is:

```text
OpenTelemetry SDK in app -> Alloy OTLP receiver -> Tempo -> Grafana Explore
```

Alloy exposes OTLP endpoints inside the cluster:

```text
http://alloy.monitoring.svc.cluster.local:4318/v1/traces
alloy.monitoring.svc.cluster.local:4317
```

Use the HTTP endpoint for Node.js services configured with:

```text
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://alloy.monitoring.svc.cluster.local:4318/v1/traces
```

In Grafana Explore, select the Tempo datasource. Traces appear after instrumented applications send requests.

## What OpenTelemetry Does Here

OpenTelemetry is the instrumentation standard used by applications to create telemetry.

In this setup:

- Prometheus metrics answer: "How many requests, errors, CPU, memory?"
- Loki logs answer: "What did the container print?"
- Tempo traces answer: "Where did this exact request spend time?"
- Alloy receives telemetry and forwards it to the right backend.
- Grafana connects the backends so you can inspect metrics, logs, and traces from one UI.

The `microservices-deployment` services use OpenTelemetry Node.js auto-instrumentation. That instrumentation creates spans for inbound HTTP requests and sends them to Alloy over OTLP HTTP.

## Storage Notes

This stack uses `emptyDir` volumes for Prometheus, Loki, Grafana, and Alloy data to keep local Minikube cleanup simple. Data is lost when pods are recreated.

For longer-lived data, replace `emptyDir` with `PersistentVolumeClaim` resources for:

- Prometheus `/prometheus`
- Loki `/loki`
- Grafana `/var/lib/grafana`
- Tempo `/var/tempo`

## Cleanup

```bash
kubectl delete -k k8s --ignore-not-found
kubectl delete namespace monitoring --ignore-not-found
```
