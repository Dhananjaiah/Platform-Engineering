# Lab 04-06 - Observability: Prometheus, Grafana, OpenTelemetry, Loki, SLOs

## Goal
Stand up the platform observability stack, instrument a service with OpenTelemetry, define SLOs with burn-rate alerts, and give developers self-service dashboards (linked from Backstage in Module 01).

## Prereqs
`hub` cluster, `helm`.

## Step 1 - Metrics: kube-prometheus-stack
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
cat > kps-values.yaml <<'EOF'
prometheus:
  prometheusSpec:
    retention: 7d
    serviceMonitorSelectorNilUsesHelmValues: false   # discover ServiceMonitors in all namespaces
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    resources: { requests: { cpu: 200m, memory: 1Gi } }
grafana:
  adminPassword: change-me-lab-only
  ingress: { enabled: true, ingressClassName: nginx, hosts: [grafana.127.0.0.1.nip.io] }
  sidecar: { dashboards: { enabled: true, searchNamespace: ALL }, datasources: { enabled: true } }
EOF
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f kps-values.yaml
kubectl -n monitoring get pods
```
Verify: open `http://grafana.127.0.0.1.nip.io:8080` (user `admin`). Long-term storage at scale: Thanos/Mimir/Azure Managed Prometheus; Grafana via Azure Managed Grafana. Mention cardinality control (`metric_relabel_configs`, drop high-cardinality labels) and per-tenant limits.

## Step 2 - Logs: Loki + Promtail/Alloy
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki -n monitoring --set deploymentMode=SingleBinary --set loki.auth_enabled=false \
  --set loki.commonConfig.replication_factor=1 --set singleBinary.replicas=1 \
  --set loki.storage.type=filesystem --set loki.schemaConfig.configs[0].from=2024-04-01 \
  --set loki.schemaConfig.configs[0].store=tsdb --set loki.schemaConfig.configs[0].object_store=filesystem \
  --set loki.schemaConfig.configs[0].schema=v13 --set loki.schemaConfig.configs[0].index.prefix=index_ \
  --set loki.schemaConfig.configs[0].index.period=24h
helm install alloy grafana/alloy -n monitoring
```
(Check chart notes for current value names; production uses object storage and microservices/simple-scalable mode.) Add Loki as a Grafana datasource, then in Explore: `{namespace="payments-dev"} |= "error"`. Standards: structured JSON logs, correlation (trace ID) in each line, retention per environment, PII scrubbing.

## Step 3 - Traces and the OTel Collector
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install otel-collector open-telemetry/opentelemetry-collector -n monitoring \
  --set mode=deployment --set image.repository=otel/opentelemetry-collector-contrib \
  --set config.exporters.debug.verbosity=basic \
  --set config.service.pipelines.traces.exporters={debug}
helm install tempo grafana/tempo -n monitoring
```
Wire the collector to export traces to Tempo (`otlp` exporter to `tempo.monitoring:4317`, `tls.insecure: true` in lab) and metrics to Prometheus remote-write or a scrape endpoint. Instrument the sample app with auto-instrumentation via the OTel Operator:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml 2>/dev/null
helm install otel-operator open-telemetry/opentelemetry-operator -n monitoring --set admissionWebhooks.certManager.enabled=true
cat <<'EOF' | kubectl apply -f -
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata: { name: default, namespace: payments-dev }
spec:
  exporter: { endpoint: http://otel-collector-opentelemetry-collector.monitoring:4318 }
  propagators: [tracecontext, baggage]
  sampler: { type: parentbased_traceidratio, argument: "0.1" }
EOF
kubectl -n payments-dev patch deploy payments --type merge -p '{"spec":{"template":{"metadata":{"annotations":{"instrumentation.opentelemetry.io/inject-nodejs":"true"}}}}}'
```
Standardise: one collector gateway per cluster, agents per node for logs, tail-based sampling for errors/latency, cost control by sampling, semantic conventions (`service.name`, `deployment.environment`, `team`).

## Step 4 - SLOs and multi-window burn-rate alerts
Define the SLO: 99.9 % of `payments` requests succeed over 30 days (error budget 0.1 %).
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata: { name: payments-slo, namespace: monitoring }
spec:
  groups:
    - name: payments-slo
      rules:
        - record: slo:payments:error_ratio_5m
          expr: sum(rate(http_requests_total{service="payments",status=~"5.."}[5m])) / sum(rate(http_requests_total{service="payments"}[5m]))
        - record: slo:payments:error_ratio_1h
          expr: sum(rate(http_requests_total{service="payments",status=~"5.."}[1h])) / sum(rate(http_requests_total{service="payments"}[1h]))
        - alert: PaymentsErrorBudgetFastBurn
          expr: slo:payments:error_ratio_5m > (14.4 * 0.001) and slo:payments:error_ratio_1h > (14.4 * 0.001)
          for: 2m
          labels: { severity: page, team: team-payments }
          annotations: { summary: "Burning 2% of monthly budget in 1h", runbook_url: "https://backstage.example/docs/payments/runbooks/errors" }
        - alert: PaymentsErrorBudgetSlowBurn
          expr: slo:payments:error_ratio_1h > (6 * 0.001)
          for: 15m
          labels: { severity: ticket, team: team-payments }
EOF
```
Why multi-window: fast burn pages for outages quickly with low false-positive; slow burn tickets chronic degradation. Page on symptoms (user-visible SLIs) not causes (CPU). Every alert has an owner label (routes in Alertmanager), a runbook link, and is reviewed quarterly for noise.

## Step 5 - Alert routing by team label
```bash
cat > am-route.yaml <<'EOF'
route:
  receiver: default
  group_by: [alertname, team]
  routes:
    - matchers: [severity="page", team="team-payments"]
      receiver: payments-pager
receivers:
  - name: default
  - name: payments-pager
    webhook_configs: [{ url: http://host.docker.internal:9999/pager }]
EOF
```
Production: PagerDuty/Opsgenie receivers, routing from the same `owner` metadata as the catalog; the PagerDuty service ID lives in `catalog-info.yaml` annotations (Module 01).

## Step 6 - Developer-facing observability (self-service)
Ship a **golden dashboard** per service type as a ConfigMap labelled `grafana_dashboard: "1"` (RED metrics: rate, errors, duration + saturation) templated by namespace/service variables; the Backstage template creates links. Verify the sidecar loads it:
```bash
kubectl -n monitoring create configmap dash-golden --from-file=golden.json && kubectl -n monitoring label cm dash-golden grafana_dashboard=1
```
Provide log and trace deep links in Backstage entity annotations (`grafana/dashboard-selector`, `grafana/alert-label-selector`). Measure usage.

## Break it on purpose
1. Add a high-cardinality label (`user_id`) in a test app, watch Prometheus memory/series growth (`prometheus_tsdb_head_series`), then drop it with relabeling.
2. Break a ServiceMonitor selector: target missing in `/targets`; fix.
3. Silence a page alert wrongly; show the policy: silences require expiry and owner, and auditing.

## Cleanup
```bash
helm uninstall kps loki alloy tempo otel-collector otel-operator -n monitoring
```

## Talking points
Metrics/logs/traces and their jobs, OTel as the vendor-neutral pipeline, cardinality and cost control, SLO burn-rate alerting by owner, golden dashboards as a platform product.
