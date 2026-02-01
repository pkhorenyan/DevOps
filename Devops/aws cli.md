
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
 