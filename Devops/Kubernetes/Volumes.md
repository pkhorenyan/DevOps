
Создается и удаляется вместе с подом, но при перезапуске данные из volume не удаляются.
Если под удален админом, то данные исчезнут.
Если нам нужно, чтобы в рамках пода контейнеры обменивались информацией, но не нужно чтобы данные сохарнялись потом.
```yaml
emptyDir: {}
```

Когда нужно предоставить доступ к локальной файловой системе ноды. При переезде на другую ноды данные не сохраняются.
В основном используется для сбора логов на конкретной ноде.
```yaml
hostPath:
  path: /var/data
```

#### PV и PVC

Обычно в кластере больше, чем одна Node, и при перезапуске Pod может запуститься на любой Node, необязательно там же, где был раньше. И в этом случае на новой Node может не оказаться нужных для работы файлов. Поэтому правильнее будет создать внешнее хранилище, которое не потеряется после остановки Pod или даже выключения всех Node. Для описания внешнего хранилища в Kubernetes используется сущность под названием **PersistentVolume**.

Это **статический метод**: вручную создаем `PersistentVolume` (PV), привязываем его к железке или облачному диску, а потом пишем `PersistentVolumeClaim` (PVC).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/nginx-html
    type: DirectoryOrCreate
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      initContainers:
      - name: copy-html
        image: busybox
        command: ['sh', '-c', 'if [ ! -f /usr/share/nginx/html/index.html ]; then cp /tmp/index.html /usr/share/nginx/html/; fi']
        volumeMounts:
        - name: nginx-persistent-storage
          mountPath: /usr/share/nginx/html
        - name: nginx-configmap
          mountPath: /tmp/index.html
          subPath: index.html
      containers:
      - image: nginx:1.25
        name: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-persistent-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: nginx-persistent-storage
        persistentVolumeClaim:
          claimName: nginx-pvc
      - name: nginx-configmap
        configMap:
          name: nginx-index-html-config
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector: 
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

`PersistentVolume`(PV) и `PersistentVolumeClaim` (PVC) — это два отдельных, но взаимосвязанных ресурса в Kubernetes, которые вместе управляют хранилищем в кластере.

```bash
kubectl get pv,pvc
```

Процесс взаимодействия

1. **Администратор** создает несколько `PersistentVolume` (PV) в кластере.
2. **Разработчик** создает `PersistentVolumeClaim` (PVC), запрашивающий, например, 5 ГиБ места.
3. **Kubernetes** находит подходящий PV, который соответствует требованиям PVC (размер, режим доступа) и **связывает (bind)** их вместе.
4. **Разработчик** монтирует этот PVC в свой под.

Таким образом, PV является **поставщиком (provider)** хранилища, а PVC — **потребителем (consumer)**, который абстрагирует детали базовой инфраструктуры хранения от разработчика приложения.

##### Нюанс

Когда `PVC` (1 ГБ) находит подходящий `PV` (2 ГБ), происходит процесс **Binding**. С этого момента этот PV помечается как `Bound`.

- К нему больше **никто не может подключиться**.
- Оставшийся 1 ГБ остается внутри этого PV, но Kubernetes не умеет «отрезать» его для другого подика.
    
**Результат:** 1 ГБ физического места просто простаивает. Ты его оплачиваешь (в облаке) или занимаешь им место на диске, но использовать не можешь.

#### StorageClass

Со `StorageClass` схема упрощается:

1. Ты создаешь **StorageClass** (один раз для всего кластера).
2. Ты создаешь **PVC**, где просто указываешь имя этого класса.
3. Kubernetes сам «идет» к облачному провайдеру (или хранилищу), создает диск и автоматически генерирует нужный **PV**.

Для локальных серверов или bare-metal (не облако) один из самых популярных и простых вариантов — это **Rancher Local Path Provisioner**. Он ведет себя как динамическое хранилище, но просто создает папки на дисках твоих нод.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path # Драйвер, который управляет созданием папок
reclaimPolicy: Delete              # Удалить данные при удалении PVC
volumeBindingMode: WaitForFirstConsumer # Ждать запуска Пода, чтобы знать, на какой ноде создать папку
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html # Куда в контейнере вешаем диск
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: nginx-pvc # Привязываем Deployment к нашему PVC
```
### Что произойдет после запуска:

1. **Deployment** попытается запуститься и увидит, что ему нужен `nginx-pvc`.
2. **StorageClass** (local-path) увидит запрос, выберет ноду и создаст там папку.
3. **Kubernetes** автоматически создаст объект **PV** (Persistent Volume) и «склеит» его с твоим **PVC**.
4. **Nginx** запустится и всё, что он запишет в `/usr/share/nginx/html`, физически сохранится на диске сервера.


Выбор `StorageClass` (SC) зависит от того, где живет твой кластер: в облаке, на своих серверах (bare-metal) или на домашнем ноутбуке для тестов.

Главное различие между ними — это **Provisioner** (движок, который «нарезает» диски).
### Популярные типы StorageClass

|**Среда**|**Provisioner (Драйвер)**|**Особенности**|
|---|---|---|
|**Google Cloud (GKE)**|`pd.csi.storage.gke.io`|Создает Google Persistent Disks (SSD/HDD).|
|**AWS (EKS)**|`ebs.csi.aws.com`|Создает диски EBS.|
|**Локально (MicroK8s/Minikube)**|`microk8s.io/hostpath`|Просто папка на диске твоего компа.|
|**Bare-metal (Свои серверы)**|`rancher.io/local-path`|Умное управление папками на разных нодах.|
|**Распределенные (On-premise)**|`ceph.com/rbd` или `longhorn.io`|Данные реплицируются между всеми серверами (отказоустойчиво).|

Откуда именно берется место, зависит от типа используемого хранилища:

- **Облачные диски (самый частый случай):** Если вы используете AWS, Google Cloud или Azure, место берется из облачного сервиса (например, AWS EBS или Azure Disk). Это внешние по отношению к ноде диски, которые «примонтируются» к ней, когда на ней запускается под.
- **Сетевые хранилища (NFS, Ceph, iSCSI):** Место находится на отдельном сервере или в кластере хранения. Нода просто подключается к этому ресурсу по сети.
- **Локальное место на ноде (Local PV):** В этом случае место **берется с диска конкретной ноды**. Однако это накладывает ограничение: под сможет работать только на этой конкретной ноде, так как его данные физически привязаны к её железу

В современных кластерах чаще всего используется **внешнее сетевое хранилище**, а не свободное место на системном диске ноды. Это позволяет поду «переезжать» с одной ноды на другую, не теряя доступ к своим данным.