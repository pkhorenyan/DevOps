Библиотеки подключаются в Manage > System > Global Trusted Pipeline Libraries
### Настройка докера в Jenkins

**запускаем контейнер с подключением волюмов и правильным доступом к сокетам**

```
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

**входим под рутом в конетйнер**
`docker exec -it -u 0 9bc0b320e2a5 bash`

**устанавливаем Докер внутри контейнера**
`apt update`
`apt install docker.io`

**фиксим права доступа**
`chmod 666 /var/run/docker.sock`

**добавляем юзера jenkins в группу docker**
`usermod -aG docker jenkins`

**выход и перезапуск**
`exit`
`docker restart 9bc0b320e2a5`

### Вебхук

В Jenkins поставить плагин Scan Multibranch Pipeline Triggers
Ставим галку *Scan by webhook*

**GitHub webhook:**

`https://{URL}/multibranch-webhook-trigger/invoke?token={githubtoken}`
application/json

**утилита localtunnel для хоста 0.0.0.0**

`lt --port 8080 --local-host localhost`

### SSH

**Ставим плагин SSH agent**

Добавляем новые credentials:
SSH Username with private key
username: ec2-user
privatekey: ключ из .pem 

**Jenkinsfile синтакс:**

```bash
sshagent(['ec2-server-key']) { 
	sh "ssh -o StrictHostKeyChecking=no ec2-user@16.171.227.148"
}
```

`-o StrictHostKeyChecking=no` означает, что ssh пропустит верификацию (y/n)

В AWS не забыть открыть 22-й порт для доступа ip сервера на котором работает Jenkins