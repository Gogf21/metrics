# Настройка динамического хранилища (NFS + StorageClass) для Kubernetes

Этот гайд описывает процесс настройки NFS-сервера и создания динамического StorageClass для кластера Kubernetes (bare-metal / kubeadm). После выполнения этих шагов тебе больше не придется вручную создавать `PersistentVolume` (PV) и привязывать их через `claimRef`. Достаточно будет создать `PersistentVolumeClaim` (PVC), и диск появится сам.

---

## 1. Настройка NFS-сервера

Для примера используется сервер с IP `192.168.0.106` и директорией `/srv/nfs/kubedata`.

### 1.1. Установка пакетов NFS (на сервере)
Выполните на машине, которая будет выступать в роли NFS-сервера:
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

### 1.2. Создание директории и прав
```bash
sudo mkdir -p /srv/nfs/kubedata
sudo chown nobody:nogroup /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata
```
*(Права `777` даны для простоты. В продакшене лучше настроить доступ по UID/GID).*

### 1.3. Экспорт директории
Откройте файл `/etc/exports` и добавьте строку, разрешающую доступ всем нодам кластера (или всей подсети):
```text
/srv/nfs/kubedata 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```
*Флаг `no_root_squash` позволяет клиентам (нодам K8s) создавать файлы от имени root, что упрощает работу с подами.*

Примените изменения:
```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

---

## 2. Установка NFS Subdir External Provisioner

Чтобы Kubernetes мог автоматически создавать папки на NFS-сервере, используется специальный провижионер.

### 2.1. Добавление Helm-репозитория
Выполните на машине, где настроен `kubectl` (мастер-нода):
```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm repo update
```

### 2.2. Установка провижионера и создание StorageClass
```bash
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner --create-namespace \
  --set nfs.server=192.168.0.106 \
  --set nfs.path=/srv/nfs/kubedata \
  --set storageClass.name=nfs-storage
```

**Что делает эта команда:**
1.  Разворачивает сам провижионер (под, который слушает API Kubernetes).
2.  Создает `StorageClass` с именем `nfs-storage`.
3.  Настраивает его на использование NFS-сервера `192.168.0.106` и папки `/srv/nfs/kubedata`.

---

## 3. Проверка работоспособности

### 3.1. Проверка StorageClass
Убедитесь, что класс создан:
```bash
kubectl get storageclass
```
В выводе должен быть `nfs-storage` с провижионером `cluster.local/nfs-provisioner-nfs-subdir-external-provisioner`.

### 3.2. Тестовый PVC
Создайте файл `test-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage # <-- Указываем наш класс
  resources:
    requests:
      storage: 1Gi
```

Примените его:
```bash
kubectl apply -f test-pvc.yaml
```

Проверьте статус:
```bash
kubectl get pvc my-app-pvc
```
Если статус **`Bound`**, значит всё работает. Проверить, что папка появилась на NFS-сервере, можно командой `ls /srv/nfs/kubedata`. Там появится директория вида `default-my-app-pvc-pvc-xxxxx`.

---

## 4. Использование в рабочих нагрузках

Теперь в любых манифестах (для Grafana, VictoriaMetrics и т.д.) можно просто указывать `storageClassName: nfs-storage`.

Пример `values.yaml` для Grafana:
```yaml
grafana:
  persistence:
    enabled: true
    storageClassName: "nfs-storage"
    size: 1Gi
```
Kubernetes сам создаст PV и папку на NFS. Тебе больше не нужно вручную создавать PV и прописывать `nodeSelector` для привязки к нодам.

---

## 5. Устранение неполадок (Troubleshooting)

Если PVC висит в статусе `Pending`, проверьте логи провижионера:
```bash
kubectl -n nfs-provisioner logs -l app=nfs-subdir-external-provisioner
```

**Типичные ошибки:**
*   **`mount: wrong fs type`**: На нодах не установлен пакет `nfs-common` (`sudo apt install nfs-common`).
*   **`permission denied`**: Проблема с правами на папку NFS (проверьте `chmod 777` и флаг `no_root_squash` в `/etc/exports`).
*   **`connection refused`**: NFS-сервер недоступен по сети (проверьте фаервол и `ping`).
