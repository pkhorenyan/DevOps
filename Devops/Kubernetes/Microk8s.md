удаляем, если есть в системе
```
sudo snap remove microk8s --purge
```

ставим

```
sudo apt update
sudo apt install snapd
sudo snap install microk8s --classic
```

поправить права
```
sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
chmod 0700 ~/.kube
newgr microk8s
```

проверяем версию, на всех нодах должна быть одна и та же версия
```
snap list microk8s
```

ждем запуска
```
microk8s status --wait-ready
```

Если установка не в VirtualBox, а на удаленной виртуалке.
Обновляем /var/snap/microk8s/current/certs/csr.conf.template. Добавляем внешний ip адрес сервера в блок alt_names

```
vim /var/snap/microk8s/current/certs/csr.conf.template
```

```
...
IP.2 = 10.152.183.1
IP.3 = 158.160.133.134
...
```

Генерируем сертификат для подключения к внешнему ip адресу

```
sudo microk8s refresh-certs --cert front-proxy-client.crt
```

Устанавливаю kubectl на локальную машину

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

Записываем данные ноды
```
microk8s config > ~/.kube/microk8s-config
```

Копируем к себе на тачку

```
scp pavelk@192.168.0.110:~/microk8s-config ~/microk8s-config-remote
```

Записываем данные кластера у себя в конфиге
```
KUBECONFIG=./config:./microk8s-config-remote kubectl config view --flatten > ./config
```
Теперь можно менять контекст, смотреть get nodes и добавлять worker ноды

Заходим на мастер ноду и смотрим как добавить еще одну ноду в кластер:
```
microk8s add-node
```
