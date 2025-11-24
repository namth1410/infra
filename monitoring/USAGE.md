# Hướng Dẫn Sử Dụng Monitoring Stack

## Tổng Quan

Monitoring stack đã được triển khai với các thành phần:
- **Prometheus**: Thu thập và lưu trữ metrics
- **Grafana**: Visualization và dashboards
- **AlertManager**: Quản lý alerts
- **Node Exporter**: Metrics từ nodes
- **Kube State Metrics**: Metrics từ Kubernetes objects

## Access Grafana

### Qua Cloudflare Tunnel (Khuyến nghị)
```bash
# Truy cập qua domain
https://grafana.namth.online

# Username: admin
# Password: admin123456
```

> **Lưu ý**: Nên đổi password ngay sau lần đăng nhập đầu tiên!

### Qua Port-Forward (Development)
```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Truy cập: http://localhost:3000
# Username: admin
# Password: admin123456
```

## Access Prometheus

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Truy cập: http://localhost:9090
```

## Access AlertManager

```bash
# Port-forward AlertManager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093

# Truy cập: http://localhost:9093
```

## Kiểm Tra Trạng Thái

### Verify Pods
```bash
# Kiểm tra tất cả pods trong monitoring namespace
kubectl get pods -n monitoring

# Expected output:
# - alertmanager-kube-prometheus-stack-alertmanager-0
# - kube-prometheus-stack-grafana-xxx
# - kube-prometheus-stack-kube-state-metrics-xxx
# - kube-prometheus-stack-operator-xxx
# - kube-prometheus-stack-prometheus-node-exporter-xxx
# - prometheus-kube-prometheus-stack-prometheus-0
```

### Verify ArgoCD Application
```bash
# Kiểm tra ArgoCD application status
kubectl get application -n argocd kube-prometheus-stack

# Xem chi tiết
kubectl describe application -n argocd kube-prometheus-stack
```

### Verify ServiceMonitors
```bash
# List tất cả ServiceMonitors
kubectl get servicemonitor -A

# Expected ServiceMonitors:
# - monitoring/kube-prometheus-stack-* (nhiều default monitors)
# - kube-system/sealed-secrets
# - ingress-nginx/ingress-nginx
```

### Verify Prometheus Targets
```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Truy cập: http://localhost:9090/targets
# Kiểm tra tất cả targets đều "UP"
```

## Default Dashboards

Grafana đã được cài đặt với các dashboards mặc định:

1. **Kubernetes / Compute Resources / Cluster**
   - Tổng quan về cluster resources
   - CPU, Memory, Network usage

2. **Kubernetes / Compute Resources / Namespace (Pods)**
   - Resource usage theo namespace
   - Pod metrics

3. **Kubernetes / Compute Resources / Node (Pods)**
   - Resource usage theo node
   - Node metrics

4. **Kubernetes / Networking / Cluster**
   - Network traffic
   - Bandwidth usage

5. **NGINX Ingress Controller**
   - Request rate, latency
   - Error rates

## Queries Hữu Ích

### Prometheus Queries

```promql
# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod, namespace)

# Memory usage by pod
sum(container_memory_working_set_bytes) by (pod, namespace)

# Pod restart count
kube_pod_container_status_restarts_total

# Ingress request rate
sum(rate(nginx_ingress_controller_requests[5m])) by (ingress)

# Sealed Secrets controller metrics
sealed_secrets_controller_unseal_requests_total
```

### Alert Examples

Một số alerts mặc định đã được cấu hình:
- `KubePodCrashLooping`: Pod đang crash loop
- `KubeDeploymentReplicasMismatch`: Deployment replicas không khớp
- `KubePersistentVolumeFillingUp`: PV sắp đầy
- `NodeMemoryPressure`: Node thiếu memory
- `NodeDiskPressure`: Node thiếu disk

## Troubleshooting

### Prometheus không scrape được targets

```bash
# Kiểm tra ServiceMonitor
kubectl get servicemonitor -A

# Kiểm tra Service có label selector đúng không
kubectl get svc -n <namespace> --show-labels

# Xem logs của Prometheus Operator
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator
```

### Grafana không hiển thị data

```bash
# Kiểm tra Grafana datasource
# Login vào Grafana -> Configuration -> Data Sources
# Verify Prometheus datasource đang hoạt động

# Kiểm tra Grafana logs
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana
```

### Storage đầy

```bash
# Kiểm tra PVC usage
kubectl get pvc -n monitoring

# Tăng size nếu cần (edit ArgoCD application)
# Hoặc giảm retention period
```

## Cấu Hình Nâng Cao

### Thêm Custom Dashboard

1. Tạo dashboard trong Grafana UI
2. Export JSON
3. Tạo ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  custom-dashboard.json: |
    {
      "dashboard": {...}
    }
```

### Thêm Alert Rules

Tạo PrometheusRule:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
spec:
  groups:
    - name: custom
      rules:
        - alert: HighPodMemory
          expr: container_memory_usage_bytes > 1e9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage"
```

### Configure AlertManager Receivers

Edit kube-prometheus-stack application để thêm receivers (Slack, Email, etc):

```yaml
alertmanager:
  config:
    receivers:
      - name: 'slack'
        slack_configs:
          - api_url: 'YOUR_SLACK_WEBHOOK'
            channel: '#alerts'
```

## Maintenance

### Backup Grafana Dashboards

```bash
# Export tất cả dashboards
kubectl exec -n monitoring <grafana-pod> -- grafana-cli admin export-dashboards
```

### Update Stack

```bash
# Update version trong ArgoCD application
# ArgoCD sẽ tự động sync
```

### Clean Up Old Metrics

Prometheus sẽ tự động xóa metrics cũ theo retention period (15 ngày).
Nếu cần xóa thủ công:

```bash
# Delete Prometheus data
kubectl delete pvc -n monitoring prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0
```

## Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [kube-prometheus-stack Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
