
Создаем неймспейс
```
kubectl create namespace argocd
```

Инсталлируем последнюю стабильную версию ArgoCD

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

*Либо указываем точную версию, чтобы не было сюрпризов*
```
curl -o install.yaml https://raw.githubusercontent.com/argoproj/argo-cd/v2.0.1/manifests/install.yaml
```

Заходим на веб-интерфейс
```
kubectl get services -n argocd
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Логин: `admin`
Пароль для доступа
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Удаление ArgoCD
```
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd
```


пример Gitlab + Helm + Argo CD

https://www.youtube.com/watch?v=9URuLt21LvM

https://github.com/virtualelephant/python-pipeline/blob/main/.gitlab-ci.yml


----
# Install ArgoCD CLI / Login via CLI

```
brew install argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login 127.0.0.1:8080
```

# Creating an Application using ArgoCD CLI:

```
argocd app create webapp-kustom-prod \
--repo https://github.com/devopsjourney1/argo-examples.git \
--path kustom-webapp/overlays/prod --dest-server https://kubernetes.default.svc \
--dest-namespace prod
```

# Command Cheat sheet

```
argocd app create #Create a new Argo CD application.
argocd app list #List all applications in Argo CD.
argocd app logs <appname> #Get the application’s log output.
argocd app get <appname> #Get information about an Argo CD application.
argocd app diff <appname> #Compare the application’s configuration to its source repository.
argocd app sync <appname> #Synchronize the application with its source repository.
argocd app history <appname> #Get information about an Argo CD application.
argocd app rollback <appname> #Rollback to a previous version
argocd app set <appname> #Set the application’s configuration.
argocd app delete <appname> #Delete an Argo CD application.
```









---
## Install argo rollouts plugin for kubectl

https://github.com/devopsjourney1/argo-rollouts-101#install-argo-rollouts-plugin-for-kubectl

https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation

```
brew install argoproj/tap/kubectl-argo-rollouts
```

## Rollout Demo

https://github.com/devopsjourney1/argo-rollouts-101#rollout-demo

1. Create the rollout and service

```
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/service.yaml
```

2. Watching rollouts

Via CLI:

```
kubectl argo rollouts get rollout rollouts-demo --watch
```

Dashboard:

```
kubectl argo rollouts dashboard
```

3. Rollout a new image

Change the image of the docker image you are using. Experiment with `blue`, `yellow`, `green`, `red`, etc.

```
kubectl argo rollouts set image rollouts-demo \
  rollouts-demo=argoproj/rollouts-demo:yellow
```

Once you set the image, use the methods in step 2 to watch the rollout. If you want to connect to the application the below command works in minikube - otherwise you need to setup an ingress.

```
minikube service --all -n default
```

Canary деплой

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: canary-demo
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {}
      - setWeight: 20
      - pause: {duration: 10}
      - setWeight: 60
      - pause: {duration: 10}
      - setWeight: 80
      - pause: {duration: 10}
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: canary-demo
  template:
    metadata:
      labels:
        app: canary-demo
    spec:
      containers:
      - name: rollouts-demo
        image: nginx:1.23.0
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: canary-demo
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: canary-demo
```

Blue/Green деплой

```yaml
# This example demonstrates a Rollout using the blue-green update strategy, which contains a manual
# gate before promoting the new stack.
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollout-bluegreen
spec:
  replicas: 2
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: rollout-bluegreen
  template:
    metadata:
      labels:
        app: rollout-bluegreen
    spec:
      containers:
      - name: rollouts-demo
        image: argoproj/rollouts-demo:blue
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
  strategy:
    blueGreen: 
      # activeService specifies the service to update with the new template hash at time of promotion.
      # This field is mandatory for the blueGreen update strategy.
      activeService: rollout-bluegreen-active
      # previewService specifies the service to update with the new template hash before promotion.
      # This allows the preview stack to be reachable without serving production traffic.
      # This field is optional.
      previewService: rollout-bluegreen-preview
      # autoPromotionEnabled disables automated promotion of the new stack by pausing the rollout
      # immediately before the promotion. If omitted, the default behavior is to promote the new
      # stack as soon as the ReplicaSet are completely ready/available.
      # Rollouts can be resumed using: `kubectl argo rollouts promote ROLLOUT`
      autoPromotionEnabled: false
---
kind: Service
apiVersion: v1
metadata:
  name: rollout-bluegreen-active
spec:
  selector:
    app: rollout-bluegreen
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
---
kind: Service
apiVersion: v1
metadata:
  name: rollout-bluegreen-preview
spec:
  selector:
    app: rollout-bluegreen
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

Удаление
```bash
kubectl delete rollout bluegreen-demo canary-demo
kubectl delete svc bluegreen-active-demo bluegreen-preview-demo canary-demo
```