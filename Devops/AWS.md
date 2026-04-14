
### Создаем и запускаем EC2 инстанс (Launch an instance)

Firewall -> allow ssh traffic from my ip
если пул адресов, то вводим маску (CIDR - IP-адрес/число):
45.85.105.32/28
- `/28` означает, что первые 28 бит IP-адреса фиксированы (определяют сеть)
- Остальные 32 - 28 = 4 бита доступны для хостов
- 2⁴ = 16 возможных адресов в этом диапазоне

создаем ключи и копируем их в папку ~/.ssh/

**меняем права доступа (чтение только владельцем)**
```
chmod 400 .ssh/docker-server.pem
```

**можем подключаться к инстансу, но не как рут, а как обычный юзер ec2-user**
```
ssh -i .ssh/docker-server.pem ec2-user@16.171.227.148
```

**обновляем пакетный менеджер (на Amazon Linux серверах это yum)**
```
sudo yum update
```

**ставим докер**
```
sudo yum install docker
```

**запускаем докер**
```
sudo service docker start
```

**теперь можно запускать команды из под sudo, поэтому добавляем юзера в группу docker**
```
sudo usermod -aG docker $USER
```

**смотрим группы**
```
> groups
ec2-user adm wheel systemd-journal
```

пока группы docker нет, потомучто нужно перелогиниться

**Установка docker compose**
```
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
```

**Меняем права доступа**
```
sudo chmod +x /usr/local/bin/docker-compose
```

# AWS CLI

установка
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
задаем юзера
`aws configure`
в output format пишем json

пример типичного запроса
`aws <command> <subcommand> parameters`

### EC2

информация про vpc и названия групп
`aws ec2 describe-security-groups`
`aws ec2 describe-vpcs`

создаем новую security group
`aws ec2 create-security-group --group-name my-sg --description "My SG" --vpc-id vpc-0b1a35bbde94ad826`

создаем правила доступа
```
aws ec2 authorize-security-group-ingress \
--group-id sg-0a9c27ac252b7fdb5 \
--protocol tcp \
--port 22 \
--cidr 45.85.105.32/28
```

создаем ключи
```
aws ec2 create-key-pair \
--key-name MyKpCli \
--query 'KeyMaterial' \
--output text > MyKpCli.pem
```

subnets
`aws ec2 describe-subnets`


Запускаем EC2 instance
```
aws ec2 run-instances
	--image-id ami-055e4d03ab1de5def
	--count 1
	--instance-type t3.micro
	--key-name MyKpCli
	--security-group-ids sg-0a9c27ac252b7fdb5
	--subnet-id subnet-0dc4909410db8563d
```

query и filter
`aws ec2 describe-instances --filters "Name=instance-type, Values=t3.micro" --query "Reservations[].Instances[].InstanceId"`

### IAM

создание группы
`aws iam create-group --group-name MyGroupCli`

создание юзера
`aws iam create-user --user-name MyUserCli`

добавление юзера в группу
`aws iam add-user-to-group --user-name MyUserCli --group-name MyGroupCli`

информация о полиси
``aws iam list-policies --query 'Policies[?PolicyName==`AmazonEC2FullAccess`].Arn' --output text``

ответ
`arn:aws:iam::aws:policy/AmazonEC2FullAccess`

добавление полиси группе
`aws iam attach-group-policy --group-name MyGroupCli --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess`

посмотреть полиси конкретной группы
`aws iam list-attached-group-policies --group-name MyGroupCli`

задать пароль юзера
`aws iam create-login-profile --user-name MyUserCli --password MyPassword! --password-reset-required`

информация про юзера
`aws iam get-user --user-name MyuserCli`

пример создания полиси, нужен json файл
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:ChangePassword"
            ],
            "Resource": [
	            "arn:aws:iam:911675024381:user/${aws:username}"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
	            "iam:GetAccountPasswordPolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

создаем файл changePwdPolicy.json

теперь создаем полиси
`aws iam create-policy --policy-name changePwd --policy-document file://changePwdPolicy.json`

прикрепляем созданный полиси к группе
`aws iam attach-group-policy --group-name MyGroupCli --policy-arn {arn из ответа}`

теперь создаем access key
`aws iam create-access-key --user-name MyUserCli`

поменять юзера для aws cli
`aws configure set aws_access_key_id {AccessKeyId}`

если нужно из под админа поменять какие-то настройки
 `export AWS_ACCESS_KEY_ID={key}`
 `export AWS_SECRET_ACCESS_KEY={key}`
 

# Сети

### 🏙️ VPC — Virtual Private Cloud

**VPC = твой собственный изолированный кусок сети в AWS.**

Как личный мини-интернет внутри AWS, где:

- у тебя свои IP-адреса (например 10.0.0.0/16)    
- свои правила безопасности    
- свои маршруты    
- свои подсети    

Ты полностью управляешь:

✔️ адресным пространством  
✔️ подсетями  
✔️ маршрутизацией  
✔️ доступом внутрь и наружу

То есть, когда ты создаёшь EC2, RDS, ECS — они живут внутри VPC.

Есть два типа подсетей:
### 🌐 Public subnet

Подсеть с выходом в интернет через Internet Gateway (IGW).  
Сюда обычно ставят:

- EC2 (веб-серверы)    
- Application Load Balancer    
- NAT Gateway    

### 🔒 Private subnet

Подсеть без прямого выхода в интернет.  
Здесь живут:

- Базы данных (RDS)
- backend-сервисы    
- внутренние микросервисы

### 🛣️ Route Table — таблица маршрутизации

**Route Table = карта дорог**, которая говорит куда какой трафик направлять.

Например:

|Destination|Target|
|---|---|
|10.0.0.0/16|local|
|0.0.0.0/0|igw-123456|

Это означает:

- весь внутренний трафик остаётся в VPC    
- весь внешний трафик идёт через Internet Gateway

У каждой subnet есть привязанная route table.
Route Table управляет только исходящим трафиком (outbound) из подсети.  
Входящий (ingress) трафик НЕ проходит через Route Table.

### 🚧 Network ACL (NACL) — сетевой забор

**NACL = грубый, статичный firewall на уровне подсети.**

Особенности:

- действует на всю подсеть    
- stateless (правила для inbound ≠ outbound)    
- порядок правил важен    
- по умолчанию разрешено всё    

Это обычный сетевой фильтр на уровне IP/портов, но гораздо менее удобный, чем Security Groups.

### 🔐 Security Groups — персональный охранник

Они не спрашивали, но тебе пригодится.

**SG = firewall на уровне машины (EC2, RDS, ECS).**

Особенности:

- stateful (если вход разрешён, выход автоматически разрешён)    
- привязываются к ресурсам, а не к подсетям    
- проще в управлении    
- предпочтительнее, чем NACL (в 99% случаев)

### 🌐 Почему private subnet должна идти через NAT, а не IGW?

Потому что:

- для private subnet **нельзя** давать публичные IP
- она не должна принимать входящие соединения от интернета    

Но приложения внутри private subnet (например, backend, RDS proxy) могут:

✔️ ходить в интернет обновлять пакеты  
✔️ обращаться в API внешних сервисов

Для этого используется **NAT Gateway**

### Маршрутизация.

**Таблицы маршрутов определяют маршрутизацию для трафика, ИСХОДЯЩЕГО из их подсети.**
   
Они **не управляют входящим трафиком** на уровне маршрутизации. Контроль входящего трафика осуществляется с помощью **Групп безопасности (Security Groups)** и **Сетевых ACL (Network Access Control Lists)**.
   
Трафик в пределах одной подсети **не проходит** через таблицу маршрутизации.