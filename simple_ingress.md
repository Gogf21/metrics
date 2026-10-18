# Настройка Ingress-контроллера и доступ к сервисам мониторинга

Этот документ описывает настройку Ingress-контроллера Nginx в bare-metal кластере Kubernetes (kubeadm) для доступа к веб-интерфейсам стека мониторинга (Grafana, VictoriaMetrics, VMAgent, VMAlert, Alertmanager).

---

## 1. Почему выбран NodePort, а не LoadBalancer?

В облачных провайдерах (AWS, GCP, Yandex) тип сервиса `LoadBalancer` автоматически выделяет внешний IP-адрес и настраивает балансировщик. 

В bare-metal кластере (kubeadm) такого провайдера нет. Если создать сервис типа `LoadBalancer`, он навсегда останется в статусе `<pending>`, и трафик снаружи не пойдет.

**Решение:** Использовать тип `NodePort` для Ingress-контроллера. Это открывает фиксированный порт на всех нодах кластера, через который Ingress-контроллер принимает трафик.

---

## 2. Установка Ingress-контроллера (Nginx)

Установка выполняется через Helm с явным указанием типа `NodePort` и фиксированных портов (чтобы не искать их потом в случайном диапазоне).

```bash
# Добавляем репозиторий
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Устанавливаем контроллер
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30475 \
  --set controller.service.nodePorts.https=31040
```

**Что это дает:**
- Ingress-контроллер будет доступен по HTTP на порту **30475** любой ноды кластера.
- По HTTPS — на порту **31040**.

Проверка:
```bash
kubectl get svc -n ingress-nginx
```
В колонке `PORT(S)` должно быть `80:30475/TCP,443:31040/TCP`.

---

## 3. Настройка правил Ingress для сервисов мониторинга

Мы не создаем отдельные YAML-файлы для Ingress, а передаем правила прямо в `values.yaml` при установке стека `victoria-metrics-k8s-stack`. Это позволяет управлять всем стеком из одного файла.

В `my-stack-values.yaml` добавлены блоки `ingress` для каждого компонента:

```yaml
# Grafana
grafana:
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hosts:
      - grafana.local

# VictoriaMetrics (VMSingle)
vmsingle:
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hosts:
      - vmsingle.local
    path: /

# VMAgent
vmagent:
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hosts:
      - vmagent.local

# VMAlert
vmalert:
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hosts:
      - vmalert.local

# Alertmanager
alertmanager:
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hosts:
      - alertmanager.local
```

**Важно:** `ingressClassName: "nginx"` должен совпадать с именем класса, который создал Ingress-контроллер (по умолчанию — `nginx`).

---

## 4. Настройка локального DNS (файл hosts)

Чтобы браузер на твоем компьютере знал, куда отправлять запросы с доменом `grafana.local`, нужно прописать сопоставление IP-адреса и домена в файле `hosts` **на твоей локальной машине** (не на нодах кластера).

### Для Linux/Mac:
```bash
sudo nano /etc/hosts
```

### Для Windows:
Откройте `C:\Windows\System32\drivers\etc\hosts` от имени администратора.

**Добавьте строки (замените `192.168.0.101` на IP любой ноды кластера):**
```text
192.168.0.101 grafana.local
192.168.0.101 vmsingle.local
192.168.0.101 vmagent.local
192.168.0.101 vmalert.local
192.168.0.101 alertmanager.local
```

После сохранения файла браузер будет отправлять запросы с нужным заголовком `Host`.

---

## 5. Как заходить в сервисы

После настройки Ingress и файла `hosts`, доступ к интерфейсам осуществляется по доменным именам и порту Ingress-контроллера (`30475`):

| Сервис | URL |
| :--- | :--- |
| **Grafana** | `http://grafana.local:30475` |
| **VictoriaMetrics** | `http://vmsingle.local:30475` |
| **VMAgent (Targets)** | `http://vmagent.local:30475/targets` |
| **VMAlert** | `http://vmalert.local:30475` |
| **Alertmanager** | `http://alertmanager.local:30475` |

---

## 6. Альтернатива: доступ по IP без правки hosts

Если нет возможности или желания править файл `hosts`, можно использовать `curl` с подменой заголовка `Host` для проверки:

```bash
curl -H "Host: grafana.local" http://192.168.0.101:30475/
```

Для полноценной работы в браузере без правки `hosts` используйте `kubectl port-forward`:
```bash
kubectl -n monitoring port-forward svc/vmks-grafana 8080:80
```
И заходите на `http://localhost:8080`.

---

## 7. Устранение неполадок

1.  **Ingress не создался:** Проверьте, что `ingressClassName` в `values.yaml` совпадает с тем, что создал контроллер:
    ```bash
    kubectl get ingressclass
    ```
2.  **404 Not Found:** Проверьте, что в `/etc/hosts` правильно прописан IP (он должен быть доступен с твоей машины).
3.  **502 Bad Gateway:** Проверьте, что сервис, на который ссылается Ingress, запущен:
    ```bash
    kubectl -n monitoring get svc
    ```
