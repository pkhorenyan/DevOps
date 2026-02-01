Создать кластер
```
eksctl create cluster \
--name demo-cluster \
--version 1.27 \
--region eu-central-1 \
--nodegroup-name demo-nodes \
--node-type t2.micro
--nodes 2 \
--nodes-min 1 \
--nodes-max 3
```

Чтобы заставть Jenkins работать с кубером надо поставить в контейнер софт.

Заходим по ssh на сервер.
Заходим под рутом в докер контейнер
```
docker execute -it -u 0 <hash> bash
```
### Ставим кубер

1) первый вариант
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```
Права
```
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
```
2) второй вариант
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

```
 chmod +x ./kubectl
 sudo mv ./kubectl /usr/local/bin/kubectl
```

### Ставим aws-iam-authenticator

Для AWS ставим

```
curl -Lo aws-iam-authenticator https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.7.8/aws-iam-authenticator_0.7.8_linux_amd64
```
Права
```
chmod +x ./aws-iam-authenticator
sudo mv ./aws-iam-authenticator /usr/local/bin
sudo mv ./aws-iam-authenticator /usr/local/bin
```
### Kubeconfig

Создаем файл config

```
vim config
```

Содержание

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: /etc/kubernetes/pki/ca.crt
    server: <api-server-endpoint-url>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: aws
  name: aws
current-context: aws
users:
- name: aws
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: /usr/bin/aws-iam-authenticator
      args:
      - "token"
      - "-i"
      - "<cluster-name>"
```

cluster-name = demo-cluster
certificate-authority-data находится в ./kube/config

выходим из контейнера и заходим как обычный пользователь
```
docker execute -it <hash> bash
cd ~
pwd
mkdir .kube
exit
docker cp config <hash>:/var/jenkins_home/.kube/
```

### AWS User credentials

В Jenkins добавляем credentials -> secret text
два credentials:

jenkins_aws_access_key_id
jenkins_aws_secret_access_key

информацию берем отсюда

```
cat .aws/credentials
```

специально для AWS надо указывать переменные, в Linode или на сыром железе такого делать не надо
```groovy
pipeline {
    agent any
    stages {
        stage('build app') {
            steps {
               script {
                   echo "building the application..."
               }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                }
            }
        }
        stage('deploy') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
            }
            steps {
                script {
                   echo 'deploying docker image...'
                   sh 'kubectl create deployment nginx-deployment --image=nginx'
                }
            }
        }
    }
}
```

The most common way to use the command is by specifying the cluster name and region:
```
aws eks update-kubeconfig --name <cluster-name> --region <region-code>
```
- Replace `<cluster-name>` with the name of your Amazon EKS cluster.
- Replace `<region-code>` with the AWS Region where your cluster is located (e.g., `us-east-2`). 

By default, the command updates the default kubeconfig file located at `~/.kube/config`