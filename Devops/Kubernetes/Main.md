Мы используем minikube чтобы развернуть кластер кубернетес локально на машине. Control plane и worker node будут крутиться внутри одной ноды.
Для работы нужен запущенный Docker.

В *.kube/config* записываются данные кластера с которым мы сецчас работаем

Мы взаимодействуем с кластером через консольную утилиту kubectl.

Показать все ноды в кластере:
```
kubectl get nodes
```

Посмотреть и изменить используемый контекст:
```
kubectl config get-contexts
kubectl config use-context <context-name>
```

#### Структура

В кубернетес мы не работает с подами напрямую. Над подами есть абстракция под названием Deployment.
Когда мы создаем deployment у него внутри есть план для создания подов.

Иерархия абстракций:
Deployment руководит
Replicaset, который руководит репликами пода
Pod это абстракция контейнера

1. Pod - объект в котором работают один или больше контейнеров
2. Deployment - сэт одинаковых подов
3. Service - предоставляет доступ к Deployment через ClusterIP, NodePort, LoadBalancer, ExternalName
4. Nodes - сервера где все это работает

создаем под на основе nginx image
```
kubectl create deployment nginx-depl --image=nginx
kubectl get deployment
kubectl get pod
kubectl edit deployment <name>
```

следим за отступами при редактировании(!)

#### Дебаг подов

```
kubectl logs <name>
kubectl logs -l app=<app_name>
kubectl describe pod <name>
kubectl exec -it <name> -- bin/bash
```

Чтобы сгенерировать конфиг
```
kubectl create deployment nginx --image=nginx:1.25 --dry-run=client -o yaml > nginx-deployment.yaml

kubectl create service clusterip nginx --tcp=80:80 --dry-run=client -o yaml > nginx-service.yaml
```

Применяем конфиг файлы
```
kubectl apply -f nginx-deployment.yaml
```
 
## Пример деплоя MongoDB

*mongo.yaml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mongodb
  name: mongodb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - image: mongo
        name: mongodb
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
  selector:
    app: mongodb
```

#### Секреты

шифруем логин и пароль
```
echo -n 'username' | base64
echo -n 'password' | base64
```
 
*mongo-secret.yaml*
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: dXNlcm5hbWU=
  mongo-root-password: cGFzc3dvcmQ=
```

Сначало деплоим секреты, а потом все остальное
```
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
```

Теперь приступаем к Mongo Express, чтобы коннектиться к БД через веб-интерфейс

Сначала создаем ConfigMap, чтобы указать урл куда подключаться к БД

*mongo-configmap.yaml*

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service
```

#### Service

1. Поды имеют внутренние ip-адреса и часто умирают; чтобы была возможность к ним подключиться, перед ними ставят сервисы со стабильными ip.
2. Load balancing, распределяют нагрузку

**ClusterIP Services**
Это абстракция со своим ip-адресом, Internal Service внутри кластера

*mongo-express.yaml*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mongo-express
  name: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - image: mongo-express
        name: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
	nodePort: 32766
  selector:
    app: mongo-express
```

Нам нужен External service, чтобы открыть доступ к Mongo Express из браузера.

**`type: LoadBalancer`** означает что это External service
**`nodePort`** это порт который доступен снаружи, он в диапазоне 30000-32767

порядок деплоя тоже важен, сначала конфиг, потом все остальное
```bash
kubectl apply -f mongo-configmap.yaml
kubectl apply -f mongo-express.yaml
```

```bash
❯ kubectl get service                                       
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes              ClusterIP      10.96.0.1        <none>        443/TCP          168m
mongo-express-service   LoadBalancer   10.109.209.124   <pending>     8081:32766/TCP   8m41s
mongodb-service         ClusterIP      10.100.102.159   <none>        27017/TCP        20m
```

В minikube надо ввести команду, которая выделит external service айпи адрес
` minikube service mongo-express-service `

#### Namespace

Устанавливаем утилиту kubens

```
sudo apt install kubectx
```


команда kubens дает список всех namespace
можно задать свой
`kubens <my-namespace>`


#### Ingress

Сетевые балансировщики у облачных провайдеров стоят дорого и, в основном, предоставляют простые правила доступа на уровне протоколов TCP или UDP. Для приложений часто требуется поддержка TLS (HTTPS), маршрутизация http-запросов на основе доменных имён или путей в URL, поэтому в Kubernetes этим занимается специальный API-ресурс `Ingress`.

Основное назначение `Ingress` состоит в обеспечении внешнего доступа к HTTP/HTTPS-приложениям, в то время как другой тип трафика лучше прогонять через сервисы типа `NodePort` или `LoadBalancer`.

**LoadBalancer** для каждого микросервиса - это дорогое удовольствие. Лучше поставить Ingress Controller, который включает правила маршрутизации трафика до сервисов.

myweb1,myweb2,...,mywebn <-> ingress controller <-> Service ingress loadbalancer

```
[ Интернет ]
      |
[ Cloud LoadBalancer ] (Создан для сервиса типа LoadBalancer)
      | (Внешний IP)
[ Ingress Controller Pod ] (Запущен в кластере, управляет правилами)
      |
[----------------------- Правила Ingress -----------------------]
      |                           |                            |
[ Сервис: frontend ]    [ Сервис: backend-api ]    [ Сервис: auth-service ]
      |                           |                            |
[ Поды frontend ]      [ Поды backend-api ]      [ Поды auth-service ]
```

**Ingress Controller** — это **приложение** (например, Nginx, Traefik, HAProxy), которое следит за объектами Ingress в кластере и динамически перенастраивает свою конфигурацию на основе этих правил.

1. В кластере установлен Ingress Controller. Он сам обычно exposed наружу через сервис типа `LoadBalancer` (реже `NodePort`). То есть он получает свой внешний IP.    
2. Вы создаете ресурс `Ingress`, в котором прописываете правила.
3. Ingress Controller видит этот манифест и настраивает себя согласно правилам.
4. Весь внешний HTTP/HTTPS трафик приходит на один IP-адрес Ingress Controller'а, который уже, основываясь на правилах, решает, куда его направить.

Самый популярный IngressController - это Nginx Ingress
В YAML файл надо добавить `ingressClassName: nginx` или `kubernetes.io/ingress.class: nginx` в annotation.

Ingress можно поставить двумя способами

1) Через Minikube

```
minikube addons enable ingress
```

2) Через Helm Chart

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

в etc/hosts добавляем 127.0.0.1 (или тот адрес что показал `minikube ip`) my-java-app.com
```
minikube tunnel
curl my-java-app.com
```

Всегда при Ingess используем `minikube tunnel`

3) Идем на сайт приложения и устанавливаем, например:

`kubectl apply -f https://projectcontour.io/quickstart/contour.yaml`

Сервисы будут видны только в конкретном нэймспейсе
`kubectl get services -n projectcontour`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flask-hello-ingress
spec:
  ingressClassName: contour
  rules:
    - host: pavelk1.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: flask-hello
                port:
                  number: 80
    - host: pavelk2.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: flask-hello
                port:
                  number: 80
```

Если нужен TLS сертификат

```yaml
....
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-secret-tls
  rules:
    - host: myapp.com
      http:
	      paths:
....
```

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: myapp-secret-tls
	namespace: default
    type: Opaque
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

## DaemonSet

DaemonSet гарантирует, что на всех (или нужных) хостах будет запущена копия Pod'а. Когда в кластер добавляется ещё один хост, DaemonSet controller автоматически запускает копию Pod'а на всех хостах кластера. При удалении ноды из кластера DaemonSet удаляет все созданные им Pod'ы.

Если же нужно обновить Pod, есть 2 стратегии OnDelete и RollingUpdate:

1. **OnDelete**. После обновления манифеста DaemonSet новые Pod'ы создадутся только после ручного удаления старых Pod'ов DaemonSet.
2. **RollingUpdate** (дефолтная). После обновления манифеста старые Pod'ы будут уничтожены, а новые создадутся автоматически. Гарантируется, что в течение всего процесса обновления на каждой ноде будет работать не более одного Pod'а DaemonSet.

Описание манифеста DaemonSet'а почти такое же, как у Deployment'а: `kind: DaemonSet`. Дальше по аналогии с Deployment: `selector`, `template`, `Pod template`, но нет раздела `replicas` — мы не указываем количество реплик, равное количеству нод в кластере. Однако можно использовать `nodeSelector`, чтобы ограничивать ноды для развёртывания.

### Pod Affinity (Сродство подов)

**Pod Affinity** — это правила, которые говорят планировщику (scheduler): «Размести этот Pod **рядом** с другими конкретными подами».

- **Зачем это нужно?** Для уменьшения сетевых задержек. Например, вы хотите, чтобы ваш кэш (Redis) и веб-приложение (App) находились на одном узле (Node), чтобы они общались максимально быстро.
- **Как это работает?** Вы указываете метки (labels) подов, рядом с которыми хотите оказаться, и `topologyKey` (обычно это имя хоста `kubernetes.io/hostname`).

**Существует два типа:**

1. **Required (Hard):** «Запускай только если рядом есть нужный под. Если нет — под будет висеть в статусе Pending».
2. **Preferred (Soft):** «Постарайся запустить рядом, но если не получится — запускай где угодно».

**Есть и обратное понятие — Pod Anti-Affinity:**  
Оно говорит: «Не запускай этот под рядом с такими же». Это критично для High Availability (HA) — например, чтобы две копии одной базы данных не оказались на одном сервере, который может выйти из строя.

### Quality Of Service

**QoS (Качество обслуживания)** — это механизм, по которому Kubernetes определяет «важность» пода и решает, кого первым «убить» (Eviction), если на сервере закончится оперативная память (RAM).

Это база стабильности системы. Вам не нужно указывать QoS вручную — Kubernetes сам присваивает один из трех классов на основе ваших настроек `requests` и `limits`:

**1. Guaranteed (Самый надежный)**

- **Условие:** Вы указали `requests` и `limits` для CPU и RAM, и они **строго равны** друг другу.
- **Приоритет:** Самый высокий. Kubernetes пообещал эти ресурсы и убьет такой под самым последним.

```yaml
containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"  # Равно request
        memory: "256Mi" # Равно request
```

**2. Burstable (Средний)**

- **Условие:** Вы указали и `requests`, и `limits`, но они **не равны** (лимит больше запроса) ИЛИ указали только один из них.
- **Приоритет:** Средний. Позволяет поду потреблять больше ресурсов, если они есть, но если серверу станет плохо — этот под будет кандидатом на вылет после "BestEffort".

**3. BestEffort (Самый рискованный)**

- **Условие:** Вы вообще **не указали** ни `requests`, ни `limits`.
- **Приоритет:** Никакой. Под живет на «честном слове». Как только любому другому поду (Guaranteed или Burstable) понадобится память, Kubernetes пристрелит BestEffort под первым.



### Taint

Если **Node Affinity** притягивает под к узлу, то **Taint** — наоборот, отталкивает поды от узла.

- **Taint** вешается на **узел (Node)**. Он говорит: «Я плохой узел (или специфичный), не пускай сюда никого, кто ко мне не привык».
- **Toleration** вешается на **под (Pod)**. Он говорит: «Я знаю про этот недостаток узла и готов с ним мириться (терпеть его)».

**Зачем это нужно?**

1. **Выделение узлов под GPU:** Чтобы обычные легкие поды не занимали дорогие сервера с видеокартами.
2. **Выделение Master-узлов:** По умолчанию на мастер-ноды вешается taint, чтобы там не запускались обычные приложения и не мешали работе кластера.
3. **Инциденты:** Если узел потерял связь с сетью, K8s вешает на него временный taint `node.kubernetes.io/unreachable`.

Сначала добавим Taint на узел (через терминал):

```bash
kubectl taint nodes node1 hardware=gpu:NoSchedule
```

Эффект: Теперь ни один под не попадет на node1.

Чтобы наш под смог там запуститься, добавим ему **Toleration**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:12.0-base
  tolerations:
  - key: "hardware"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule" # Должен совпадать с эффектом на узле
```

Коротко разница эффектов Taint:

1. **NoSchedule:** Новые поды не придут, но старые (уже запущенные) останутся.
2. **PreferNoSchedule:** Мягкий запрет (система постарается не пускать, но если мест нет — пустит).
3. **NoExecute:** Самый жесткий. Выгонит (evict) даже те поды, которые уже там работают, если у них нет нужного Toleration.
## LivenessProbe и ReadinessProbe

**livenessProbe** должны быть быстрыми и довольно примитивными. Не стоит сходить с ума по мегасложным проверкам с тоннами детализации, ведь проверка лишь говорит, жив сервис или нет: жив — идём дальше; нет — рестарт Pod'а. Например, если сервис внутри Pod'а зависит от другого внешнего сервиса, который недоступен, то рестарт не спасёт — наше приложение стартануло, а обслуживать клиентов всё равно не может, но это уже совсем другая история.

**livenessProbe** не дружит с авторизацией, поэтому endpoint с healthcheck'ом должен быть без авторизации, иначе проверка не сработает.

Для того чтобы проверить, готово ли приложение к работе с клиентами в Кубере, есть второй набор проверок — **readinessProbe**.

Синтаксис у них точно такой же, как и у **livenessProbe**. Главное отличие в том, что **livenessProbe** смотрит за тем, поднялся ли контейнер, в то время как **readinessProbe** — готово ли приложение принимать и обрабатывать запросы.

Представим что, наше приложение зависит от других сервисов и перед обслуживанием клиентов хочется проверить, что все внешние сервисы доступны и работают. Тогда проверки внешних сервисов можно написать в **readinessProbe**.

В двух словах о том, что проверяют пробы:
- **startupProbe** — запущено ли приложение внутри контейнера?
- **livenessProbe** — запущен ли контейнер?
- **readinessProbe** — приложение готово работать с пользователями и можно ли пускать трафик?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-probes
spec:
  containers:
  - name: app-container
    image: your-app-image:latest
    ports:
    - containerPort: 8080

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
      timeoutSeconds: 5
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 2

```


```yaml
# ПРИМЕР полной настройки
readinessProbe:
  httpGet:
    path: /health/readiness
    port: 8080
  
  # 1. INITIAL DELAY - ожидание перед первой проверкой
  initialDelaySeconds: 30  # Ждем 30 сек после старта контейнера
  
  # 2. ПЕРИОДИЧНОСТЬ проверок
  periodSeconds: 10        # Проверяем каждые 10 секунд
  
  # 3. ТАЙМАУТ на выполнение проверки
  timeoutSeconds: 3        # Ждем ответа максимум 3 секунды
  
  # 4. ПОРОГ УСПЕХА
  successThreshold: 1      # Нужно 1 успешная проверка для перехода в Ready/Healthy
  
  # 5. ПОРОГ ОШИБОК
  failureThreshold: 3      # После 3 неудачных проверок подряд -> проба падает
```

#### HPA

HPA настраивает само количество Pod'ов для конкретной сущности (Deployment, ReplicaSet или StatefulSet), с указанием диапазона и параметров CPU, при которых надо добавлять новый Pod
```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-report-hpa
  labels:
    app: backend-report-hpa
spec:
  minReplicas: 3
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-report
  metrics:
    — type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80 
```

```
kubectl get horizontalpodautoscaler
```

Перед созданием `HPA` убедитесь, что для подов сервиса `backend-report` в манифесте `Deployment` указаны `requests` по CPU. `Horizontal Pod Autoscaler` использует эти значения для определения, когда необходимо масштабировать количество подов в зависимости от загрузки CPU. Отсутствие этих параметров может привести к некорректной работе или неактивности HPA.

пример:

```yaml
...
spec:
  template:
    spec:
      containers:
        — name: backend-report
          image: sausage-report:latest
          resources:
            requests:
              memory: "512Mi"
              cpu: 0.1
            limits:
              memory: "1025Mi"
              cpu: 0.2 
```

#### Private Repo

Если пуллим образы из приватной репы и используем minikube нужно потанцевать с бубном.

`minikube ssh`
`docker login --username <username> -p <password>`

если это AWS
`docker login --username <username> AWS -p <password> <aws-repo-url>`

Создается *.docker/config.json*

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-registry-key
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64>
```

чтобы сработал этот файл выполняем команду
`minikube cp minikube:/home/docker/.docker/config.json /users/pavelk/.docker/config.json`

шифруем файл через base64:
`cat .docker/config.json | base64`

и вставляем результат в значение .dockerconfigjson в вышеприведенном yaml файле



