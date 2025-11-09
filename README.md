# Prometheus + Grafana on Kubernetes (Local)

This project deploys kube-prometheus-stack (Prometheus, Grafana, Alertmanager, exporters) on a local Kubernetes cluster (Docker Desktop K8s or minikube) using Helm.

## Prerequisites
- kubectl
- Helm 3+
- A local Kubernetes cluster
  - Option A: Docker Desktop with Kubernetes enabled
  - Option B: minikube (`brew install minikube`)

## Quickstart

1) Create or select a Kubernetes cluster
- Docker Desktop: enable Kubernetes in Preferences and wait until it's running.
- minikube:
```bash
minikube start
```

2) Add Helm repo and install kube-prometheus-stack
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace (Helm can also do this with --create-namespace)
kubectl create namespace monitoring || true

# Install with defaulted values (Grafana admin password set to 'admin')
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/values.yaml
```

3) Access Grafana and Prometheus (port-forward)
```bash
# Grafana UI
kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80
# Open http://localhost:3000
# Login: admin / admin

# Prometheus UI
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

4) Explore dashboards
- Grafana comes preloaded with Kubernetes and Prometheus dashboards.
- Data source Prometheus is pre-configured by the chart.

## Access via Ingress (no port-forward)
This repo enables Ingress for Grafana and Prometheus with hosts `grafana.local` and `prometheus.local`.

1) Ensure Ingress controller exists (minikube):
```bash
minikube addons enable ingress
```

2) Upgrade Helm release to apply Ingress/persistence settings:
```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/values.yaml
```

3) Map hosts to minikube IP (on your machine):
```bash
minikube ip
# Suppose it prints 192.168.49.2, then add to /etc/hosts:
# 192.168.49.2 grafana.local prometheus.local
```

4) Open:
- http://grafana.local
- http://prometheus.local

Login Grafana: `admin` / `admin` (change in values for non-dev).

## Full Sequence

### Install minikube and start cluster
```bash
brew install minikube
minikube start
```

### Install Helm and add repo
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Create namespace and install kube-prometheus-stack
```bash
kubectl create namespace monitoring || true
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/values.yaml
```

### Enable Ingress and persistence
```bash
minikube addons enable ingress
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/values.yaml
```

### Create TLS secrets
```bash
# Generate self-signed certs (dev only)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/grafana.local.key -out /tmp/grafana.local.crt \
  -subj "/CN=grafana.local/O=Local"
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/prometheus.local.key -out /tmp/prometheus.local.crt \
  -subj "/CN=prometheus.local/O=Local"

# Create secrets
kubectl -n monitoring create secret tls grafana-tls \
  --cert=/tmp/grafana.local.crt --key=/tmp/grafana.local.key --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring create secret tls prometheus-tls \
  --cert=/tmp/prometheus.local.crt --key=/tmp/prometheus.local.key --dry-run=client -o yaml | kubectl apply -f -
```

### Upgrade Helm release with TLS and persistence
```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm/values.yaml
```

### Access via Ingress with tunnel
```bash
# Make ingress controller a LoadBalancer service
kubectl -n ingress-nginx patch svc ingress-nginx-controller -p '{"spec":{"type":"LoadBalancer"}}'

# Run tunnel (long-running, needs sudo)
minikube tunnel

# Point hosts to 127.0.0.1 so the tunnel terminates TLS locally
echo "127.0.0.1 grafana.local prometheus.local" | sudo tee -a /etc/hosts

# Test (self-signed -> use -k)
curl -I -k https://grafana.local
```

### Troubleshooting
- DNS/Hosts
  - Ensure `/etc/hosts` contains either `$(minikube ip)` or `127.0.0.1` entries for `grafana.local prometheus.local`.
- Ingress controller
  - `kubectl -n ingress-nginx get pods` should show controller `Running`.
  - `kubectl -n ingress-nginx get svc` should show controller as `LoadBalancer` when using tunnel.
- Grafana service endpoints
  - `kubectl -n monitoring get endpoints monitoring-grafana` should list a pod IP and port 3000.
- Local conflicts
  - If browser reports timeout to localhost, verify no firewall/VPN is blocking `localhost:3000` and try incognito.
- Quick verification
  - `kubectl -n monitoring port-forward svc/monitoring-grafana 3000:80` then open http://localhost:3000

## Uninstall
```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

## Notes
- The stack includes Prometheus Operator, Alertmanager, node/kube-state exporters, and curated dashboards.
- For production, change `grafana.adminPassword` and consider an Ingress/LoadBalancer rather than port-forward.
- Persistence is enabled for Grafana and Prometheus (PVCs). Adjust sizes in `helm/values.yaml`.
- Ingress is enabled (nginx class). Ensure an ingress controller is installed (minikube addon or nginx ingress on other envs).

## ServiceMonitor example
To have Prometheus scrape your own app (with `/metrics`), label its Service with `app: sample-app` and port name `http`, then apply:
```bash
kubectl apply -f k8s/servicemonitor-sample.yaml
```

## Persistence Notes
Persistence is enabled for Grafana and Prometheus (PVCs). Adjust sizes in `helm/values.yaml`.

## HTTPS via Ingress (TLS)
- This repo configures TLS for `grafana.local` and `prometheus.local` in `helm/values.yaml` (self-signed, dev only).
- TLS secrets must exist in namespace `monitoring`:
  - `grafana-tls` (CN=grafana.local)
  - `prometheus-tls` (CN=prometheus.local)

## Ingress on minikube: LoadBalancer + tunnel
On minikube (Docker driver), to serve HTTPS on 443 you typically need:
```bash
# Make ingress controller a LoadBalancer service
kubectl -n ingress-nginx patch svc ingress-nginx-controller -p '{"spec":{"type":"LoadBalancer"}}'

# Run tunnel (long-running, needs sudo)
minikube tunnel

# Point hosts to 127.0.0.1 so the tunnel terminates TLS locally
echo "127.0.0.1 grafana.local prometheus.local" | sudo tee -a /etc/hosts

# Test (self-signed -> use -k)
curl -I -k https://grafana.local
```

If you prefer not to run a tunnel, use port-forward (HTTP) as in Quickstart.

## Project Structure
```
.
├─ helm/
│  └─ values.yaml              # Chart values (Ingress, TLS, persistence, retention)
├─ k8s/
│  └─ servicemonitor-sample.yaml
└─ README.md
```

## Configuration Reference (values.yaml)
- **grafana.adminPassword**: default admin password (dev only). Change for prod.
- **grafana.ingress**: enabled + `ingressClassName: nginx`; TLS hosts and secret.
- **grafana.persistence.enabled/size**: enable PVC and set size.
- **prometheus.prometheusSpec.retention**: data retention (e.g., 15d).
- **prometheus.prometheusSpec.storageSpec**: PVC template for Prometheus TSDB.
- **prometheus.ingress**: enabled + TLS.

If your cluster requires a specific StorageClass, add:
```yaml
grafana:
  persistence:
    enabled: true
    size: 5Gi
    storageClassName: standard
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
```

## Grafana Credentials
- Default user: `admin`
- Default password: from values.yaml or query secret:
```bash
kubectl -n monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Change admin password (options):
- Update `grafana.adminPassword` in values and `helm upgrade`.
- Via UI: Configuration → Users → admin.

## Prometheus Quick Queries
- `up` — healthy targets.
- `node_cpu_seconds_total` — raw node CPU metrics.
- `rate(container_cpu_usage_seconds_total[5m])` — CPU usage rate (if cAdvisor/metrics present).
- `kube_pod_container_status_restarts_total` — restarts per container.

## Add Your App: Deployment & Service compatible with ServiceMonitor
Example app exposing `/metrics` on port 8080 with port name `http` and label `app: sample-app`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
        - name: app
          image: your-registry/your-app:latest
          ports:
            - name: http
              containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: default
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
    - name: http
      port: 8080
      targetPort: http
```
Apply the provided `k8s/servicemonitor-sample.yaml` to start scraping.

## Security Hardening (Prod)
- Change `grafana.adminPassword`, enable Grafana OAuth/SSO if needed.
- Restrict Ingress by IP allowlist or auth at reverse proxy.
- Use real TLS certs (e.g., cert-manager + Let’s Encrypt) instead of self-signed.
- Set resource requests/limits for components.
- Consider separate namespaces and RBAC.

## Cleanup (including PVCs)
Uninstall release and namespace:
```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```
If you enabled persistence and want to delete PV/PVCs created by the release, you might need to remove leftover PVCs (careful: data loss):
```bash
kubectl get pvc -n monitoring
kubectl delete pvc -n monitoring <pvc-name>
```

## Tips
- Stop port-forward: Ctrl+C di terminal yang menjalankan port-forward.
- `minikube tunnel` berjalan terus; hentikan dengan Ctrl+C pada terminalnya.
- Jika menggunakan VPN/Firewall, pastikan localhost dan 127.0.0.1 tidak diblokir.

## MQTT Integration (Telegraf → Prometheus → Grafana)

This repository includes a ready-to-use MQTT pipeline using Telegraf to subscribe to topics and expose Prometheus metrics, plus a prebuilt Grafana dashboard.

### What gets deployed
- Telegraf ConfigMap: `k8s/telegraf-configmap.yaml`
- Telegraf Deployment + Service (port 9273): `k8s/telegraf-deployment.yaml`, `k8s/telegraf-service.yaml`
- ServiceMonitor for Prometheus scrape: `k8s/telegraf-servicemonitor.yaml`
- Grafana Dashboard (auto-provisioned): `k8s/grafana-dashboard-mqtt.yaml`

Telegraf subscribes to these topics on `tcp://202.74.74.42:1883`:
- `PDM1106J/voltage`
- `PDM1106J/power`
- `PDM1106J/current`
- `PDM1106J/energy`
- `PDM1106J/pf`
- `PDM1106J/fr`

Assumption: each payload is a plain numeric value (float). Metrics are exported as `pdm_metric_value{topic="..."}`.

### Deploy
```bash
kubectl apply -f k8s/telegraf-configmap.yaml \
  -f k8s/telegraf-deployment.yaml \
  -f k8s/telegraf-service.yaml \
  -f k8s/telegraf-servicemonitor.yaml

# Apply Grafana dashboard
kubectl apply -f k8s/grafana-dashboard-mqtt.yaml

# Check Telegraf pod
kubectl -n monitoring get pods -l app=telegraf-mqtt -o wide

# Check metrics endpoint (optional, via port-forward)
kubectl -n monitoring port-forward svc/telegraf-mqtt 9273:9273
curl http://localhost:9273/metrics | head
```

Prometheus target should appear as `job="telegraf-mqtt"`:
```bash
# If Prometheus is port-forwarded to 9090
curl -sG --data-urlencode 'query=up{job="telegraf-mqtt"}' http://localhost:9090/api/v1/query
```

### View in Grafana
- Open Grafana and find dashboard: "MQTT - PDM1106J Metrics"
- Panels use queries like:
  - `pdm_metric_value{topic="PDM1106J/voltage"}` (Volt)
  - `pdm_metric_value{topic="PDM1106J/power"}` (Watt)
  - `pdm_metric_value{topic="PDM1106J/current"}` (Ampere)
  - `pdm_metric_value{topic="PDM1106J/energy"}` (kWh)
  - `pdm_metric_value{topic="PDM1106J/pf"}` (0..1)
  - `pdm_metric_value{topic="PDM1106J/fr"}` (Hz)

### Customize broker, topics, or auth
- Edit `k8s/telegraf-configmap.yaml` under `[[inputs.mqtt_consumer]]`:
  - `servers = ["tcp://<host>:<port>"]`
  - `topics = ["<topic1>", "<topic2>", ...]`
  - Add credentials if required:
    ```toml
    username = "<user>"
    password = "<pass>"
    ```
  - For JSON payloads, switch parser:
    ```toml
    data_format = "json"
    json_string_fields = ["field1","field2"]
    name_override = "pdm_metric"
    # Use processors to map JSON fields into separate series if needed
    ```
- Apply changes:
```bash
kubectl apply -f k8s/telegraf-configmap.yaml
kubectl -n monitoring rollout restart deploy/telegraf-mqtt
```

### Troubleshooting MQTT path
- Telegraf pod logs:
```bash
kubectl -n monitoring logs deploy/telegraf-mqtt --tail=200 -f
```
- Prometheus target health:
```bash
# Requires Prometheus port-forward on 9090
curl -s http://localhost:9090/api/v1/targets | jq -r '.data.activeTargets[] | select(.labels.job=="telegraf-mqtt")'
```
- No data points:
  - Ensure broker publishes messages on the specified topics.
  - Confirm payloads are numeric; otherwise switch to JSON parsing.
  - Firewall/NAT may block cluster egress to the broker IP/port.

### Remove MQTT components
```bash
kubectl delete -f k8s/grafana-dashboard-mqtt.yaml || true
kubectl delete -f k8s/telegraf-servicemonitor.yaml || true
kubectl delete -f k8s/telegraf-service.yaml || true
kubectl delete -f k8s/telegraf-deployment.yaml || true
kubectl delete -f k8s/telegraf-configmap.yaml || true
