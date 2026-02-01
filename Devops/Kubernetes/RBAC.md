В Kubernetes пользователей как таковых нет. Но можно эмулировать доступ пользователя используя ssl сертификаты.

- создадим очередной namespace

```shell
kubectl create ns lesson8
```

### Service Account

ServiveAccount используют для доступа к API Kubernetes из приложений.  
Когда в системе создаётся pod, ему по умолчанию присваивается default ServiceAccount текущего namespace.

- при создании namespace создается SA по умолчанию, при этом к подам k8s по умолчанию будет подключать данный SA

```shell
kubectl get sa -n lesson8
```

- создадим собственный SA, убедимся, что у него ничего нет и создадим и токен

```shell
kubectl create sa netology-sa -n lesson8
kubectl describe sa -n lesson8 netology-sa
```

- сгенерируем временный токен для нашего SA

```shell
kubectl create token netology-sa -n lesson8
```

- этот токен можно использовать для подключения к кластеру. Также нам потребуется ca.crt, который можно скачать с кластера (в microk8s сертификаты лежат в пути `/var/snap/microk8s/current/certs`)

```shell
export TOKEN=<token>
curl --cacert 08-rbac/ca.crt --header "Authorization: Bearer ${TOKEN}" -X GET https://<IP_address>:16443/api 
```

- то, что данный токен временный можно убедиться расшифровав его на сайте [jwt.io](https://jwt.io/). Видим, что expiration time через 1 час
- для того, чтобы иметь постоянный токен, необходимо сделать объект secret

```shell
kubectl apply -f 08-rbac/80_secret.yaml
kubectl describe sa netology-sa -n lesson8
```

- теперь можно создать под с использованием этого SA и обратиться к API кластера из пода

```shell
kubectl apply -f 08-rbac/81_pod_multitool_sa.yaml
kubectl exec -n lesson8 pod-multitool-default -- ls -la /var/run/secrets/kubernetes.io/serviceaccount
```

- теперь можно использовать этот токен для запроса к API.

```shell
kubectl exec -n lesson8 pod-multitool-default -it -- bash
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    -X GET https://<IP_address>:16443/api
```

Для определения ServiceAccount, при описании пода используйте serviceAccountName.

При запуске приложения ему автоматически подключается директория /var/run/secrets/kubernetes.io/serviceaccount в которой находятся файлы:

- ca.crt – сертификат CA кластера.
- Namespace – содержит имя namespace, в котором находится pod.
- token – содержит токен, используемый для доступа к API кластера.

---

### User Account


- создадим сертификат вручную с именем `test` и группой `ops` и подпишем CA

```shell
openssl genrsa -out test.key 2048
openssl req -new -key test.key -out test.csr -subj "/CN=test/O=ops"
openssl x509 -req -in test.csr -CA 08-rbac/ca.crt -CAkey 08-rbac/ca.key -CAcreateserial -out test.crt -days 10
cat test.crt
```

- получим сертификат, который можно использовать для подключения к кластеру

```shell
kubectl config set-credentials test_user --client-certificate test.crt --client-key test.key --embed-certs=true
kubectl config view
```

- создадим новый контекст к текущему кластеру с новыми кредами

```shell
kubectl config set-context test_connection --cluster=microk8s-cluster --user=test_user
kubectl config use-context test_connection
kubectl get nodes
```

- то же самое можно делать с помощью API k8s и объекта типа `CertificateSigningRequest`
- для этого надо сгенерировать CSR и отправить в кластер запрос

```shell
openssl genrsa -out test2.key 2048
openssl req -new -key test2.key -out test2.csr -subj "/CN=test2/O=ops"
cat test2.csr | base64
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: super-csr
spec:
  groups:
    - system:authenticated
  request: { BASE64_csr }
  expirationSeconds: 86400  # one day
  usages:
    - client auth
  signerName: kubernetes.io/kube-apiserver-client
```

```shell
kubectl apply -f 08-rbac/82_csr.yaml
kubectl get csr
```

- аппрувнем запрос

```shell
kubectl certificate approve super-csr
kubectl get csr
```

- заберем сертификат

```shell
kubectl get csr super-csr -o jsonpath={.status.certificate} | base64 --decode
```

- теперь можно создать конфигурацию для подключения к кластеру с полученным сертификатом

```shell
kubectl get csr super-csr -o jsonpath={.status.certificate} | base64 --decode
```

### Role, RoleBinding


- дадим пользователю права на просмотр подов и деплойментов

```shell
kubectl apply -f 08-rbac/83_role_rolebind.yaml
```

- список api ресурсов

```shell
kubectl api-resources
```

- дадим группе права описанные ранее

```shell
kubectl apply -f 08-rbac/84_rolebind.yaml
```

- переключимся на контекст пользователя и убедимся, что права есть


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
name: pod-reader
namespace: my-ns
rules:
- apiGroups: [""]
resources: ["pods", "pods/log"]
verbs: ["get", "watch", "list"]
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
name: pod-reader
namespace: my-ns
subjects:
- kind: User
name: dev
apiGroup: rbac.authorization.k8s.io
roleRef:
kind: Role
name: pod-reader
apiGroup: rbac.authorization.k8s.io
```