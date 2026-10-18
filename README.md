# ОТЧЕТ: РАЗВЕРТЫВАНИЕ СТЕКА МОНИТОРИНГА VICTORIAMETRICS

## 1. АРХИТЕКТУРА СТЕКА

Стек развернут в неймспейсе `monitoring` на bare-metal кластере (kubeadm). 

Поток данных:
`[Node Exporter] / [Kube-State-Metrics]` -> `[VMAgent]` -> `[VictoriaMetrics]` -> `[Grafana]` (визуализация)
                                                            |
                                                            -> `[VMAlert]` -> `[Alertmanager]` (алерты)

## 2. РЕПОЗИТОРИИ И УСТАНОВКА

Перед установкой были добавлены необходимые Helm-репозитории:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Команды установки компонентов:

```bash
helm install victoriametrics vm/victoria-metrics-single -n monitoring -f victoriametrics-values.yaml
helm install vmagent vm/victoria-metrics-agent -n monitoring -f vmagent-values.yaml
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter -n monitoring
helm install kube-state-metrics prometheus-community/kube-state-metrics -n monitoring
helm install grafana grafana/grafana -n monitoring -f grafana-values.yaml
helm install alertmanager prometheus-community/alertmanager -n monitoring -f alertmanager-values.yaml
helm install vmalert vm/victoria-metrics-alert -n monitoring -f vmalert-values.yaml
```

## 3. ОСОБЕННОСТИ РАЗВЕРТЫВАНИЯ (STORAGE & NETWORK)

Из-за отсутствия динамического StorageClass в bare-metal кластере, использованы локальные PV (`hostPath`) с привязкой к нодам:

*   **VictoriaMetrics** -> нода `k8s-1` (лейбл `vm-node=true`)
*   **Grafana** -> нода `k8s-2` (лейбл `grafana-node=true`)
*   **Alertmanager** -> нода `k8s-3` (лейбл `alertmanager-node=true`)

Сетевой доступ (NodePort):
*   **VictoriaMetrics**: `30090`
*   **Grafana**: `30080`
*   **Alertmanager**: `30093`

## 4. КОНФИГУРАЦИОННЫЕ ФАЙЛЫ (values.yaml)

### 4.1. VictoriaMetrics

Создание диска (`vm-pv.yaml`):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: victoria-metrics-pv
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes: [ReadWriteOnce]
  hostPath:
    path: "/mnt/data/victoria-metrics"
    type: DirectoryOrCreate
```

Установка (`victoriametrics-values.yaml`):
```yaml
rbac:
  create: false
server:
  retentionPeriod: 6h
  nodeSelector:
    vm-node: "true"
  tolerations: 
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
  persistentVolume:
    enabled: true
    storageClassName: "manual"
    size: "2Gi"
  service:
    type: NodePort
    port: 9090
    nodePort: 30090
```

### 4.2. VMAgent (`vmagent-values.yaml`)

```yaml
remoteWrite:
  - url: http://victoriametrics-victoria-metrics-single-server.monitoring.svc.cluster.local:8428/api/v1/write
persistence:
  enabled: false
extraScrapeConfigs:
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [ __meta_kubernetes_endpoints_name ]
        regex: 'node-exporter-prometheus-node-exporter'
        action: keep
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: [ 'kube-state-metrics.monitoring.svc.cluster.local:8080' ]
```

### 4.3. Grafana (`grafana-values.yaml`)

```yaml
nodeSelector:
  grafana-node: "true"
persistence:
  enabled: true
  existingClaim: "grafana-pvc"
service:
  type: NodePort
  port: 80
  nodePort: 30080
adminPassword: "admin"
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: VictoriaMetrics
        type: prometheus
        url: http://victoriametrics-victoria-metrics-single-server.monitoring.svc.cluster.local:8428
        isDefault: true
```

### 4.4. Alertmanager (`alertmanager-values.yaml`)

```yaml
nodeSelector:
  alertmanager-node: "true"
persistence:
  enabled: true
  storageClassName: "manual"
  size: "2Gi"
service:
  type: NodePort
  port: 9093
  nodePort: 30093
```

### 4.5. VMAlert (`vmalert-values.yaml`)

```yaml
alertmanager:
  enabled: false
server:
  datasource:
    url: http://victoriametrics-victoria-metrics-single-server.monitoring.svc.cluster.local:8428
  remote:
    write:
      url: http://victoriametrics-victoria-metrics-single-server.monitoring.svc.cluster.local:8428
  notifier:
    url: http://alertmanager.monitoring.svc.cluster.local:9093
  config:
    alerts:
      groups:
        - name: "vm-health"
          rules:
            - alert: "TooManyRestarts"
              expr: changes(process_start_time_seconds[15m]) > 2
```

## 5. ИНСТРУКЦИЯ ПО ПРОВЕРКЕ

1. Проверить статус подов:
   ```bash
   kubectl -n monitoring get pods
   ```

2. **VictoriaMetrics** (`http://<IP>:30090`):
   Ввести запрос `up`. Отобразятся все таргеты.

3. **Grafana** (`http://<IP>:30080`, admin/admin):
   Проверить `Connections -> Data Sources` (VictoriaMetrics должна быть зеленой).

4. **Alertmanager** (`http://<IP>:30093`):
   Проверить наличие алертов.
