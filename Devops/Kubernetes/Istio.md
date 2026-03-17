
Istio добавляет **свой слой управления трафиком поверх Kubernetes**.  
Обычный Kubernetes балансирует только между Pod’ами сервиса, а Istio позволяет:

- управлять **процентами трафика**
- делать **канареечные релизы**
- **A/B тесты**
- **retry / timeout**
- **mTLS**

#### Установка Istio

Самый простой способ — **istioctl**.

```shell
curl -L https://istio.io/downloadIstio | sh -  
cd istio-*
```

```shell
export PATH=$PWD/bin:$PATH
```
#### Установка в Kubernetes

Установка профиля `demo` (он включает все необходимые компоненты для тестов):

```shell
istioctl install --set profile=demo -y
```

```shell
kubectl get pods -n istio-system
```
#### Включить sidecar injection

Istio работает через **Envoy sidecar в каждом pod**.
Включаем для namespace:

```shell
kubectl label namespace default istio-injection=enabled
```

#### 1. Service

Обычный Kubernetes Service.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    name: http
```

#### 2. Gateway

Gateway = **точка входа в кластер** (L7 proxy на базе Envoy).

- принимает внешний трафик
- слушает порт
- передает его VirtualService

Пример:

```yaml
apiVersion: networking.istio.io/v1beta1  
kind: Gateway  
metadata:  
  name: nginx-gateway  
spec:  
  selector:  
    istio: ingressgateway  
  servers:  
  - port:  
      number: 80  
      name: http  
      protocol: HTTP  
    hosts:  
    - "*"
```

#### 3. VirtualService

**VirtualService управляет маршрутизацией трафика.**

Здесь делается:

- 80/20 балансировка
- канареечный релиз
- header routing

Пример 80/20:

```yaml
apiVersion: networking.istio.io/v1beta1  
kind: VirtualService  
metadata:  
  name: nginx  
spec:  
  hosts:  
  - "*"  
  gateways:  
  - nginx-gateway  
  http:  
  - route:  
    - destination:  
        host: nginx  
        subset: v1  
      weight: 80  
    - destination:  
        host: nginx  
        subset: v2  
      weight: 20
```

#### 4. DestinationRule

DestinationRule определяет **subsets (версии сервиса)**.

```yaml
apiVersion: networking.istio.io/v1beta1  
kind: DestinationRule  
metadata:  
  name: nginx  
spec:  
  host: nginx  
  subsets:  
  - name: v1  
    labels:  
      version: v1  
  - name: v2  
    labels:  
      version: v2
```
Теперь Istio знает:

```
subset v1 → pod с label version=v1  
subset v2 → pod с label version=v2
```

Деплоим две версии nginx
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: v2
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

#### Получаем внешний IP

```shell
kubectl get svc istio-ingressgateway -n istio-system
```
#### Проверяем балансировку

```shell
while true; do curl http://192.168.0.240; sleep 1; done
```

Как это работает:

1. **Gateway** принимает внешний HTTP-запрос на 80 порт.
2. **VirtualService** видит этот запрос и применяет правило весов: 80% отправляет на `v1`, 20% — на `v2`.
3. **DestinationRule** подсказывает Istio, какие именно поды относятся к `v1` и `v2` (фильтрация по `labels`).

#### Удаление

Чтобы **полностью удалить Istio** нужно убрать:

1. Istio control plane
2. Istio CRD
3. namespace `istio-system`
4. твои манифесты (gateway / virtualservice / destinationrule и т.д.)

**Удалить Istio через istioctl (правильный способ)**

```shell
istioctl uninstall --purge -y
```

Это удалит:
- `istiod`
- `istio-ingressgateway`
- все CRD Istio
- webhooks

**Удалить namespace Istio**

Иногда он остаётся:

```shell
kubectl delete namespace istio-system
```

**Убрать sidecar injection label**

```shell
kubectl label namespace default istio-injection-
```

(минус в конце удаляет label)

**Удалить свои манифесты**

```shell
kubectl delete -f .
```