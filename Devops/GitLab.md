Устанавливаем раннер

```
# Download the binary for your system
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# Install and run as a service
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

регистрируем

```shell
gitlab-runner register \
 --url https://gitlab.com \
 --token <TOKEN>
```

редактируем конфиг
```
sudo vim /etc/gitlab-runner/config.toml
```

Для Docker in Docker (DinD):

```toml
[runners.docker]
  privileged = true
  volumes = [
    "/cache",
    "/var/run/docker.sock:/var/run/docker.sock",  # Лучше для production
    # Или для DinD:
    # "/builds:/builds"
  ]
```

.gitlab-ci.yml

```yaml
...
docker-build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
...
```

Проблемы

```bash
# Проверить статус runner
sudo gitlab-runner status

# Проверить конфигурацию
sudo gitlab-runner verify

# Посмотреть список runner'ов
sudo gitlab-runner list

# Перезагрузить конфигурацию
sudo gitlab-runner restart

# Посмотреть логи
sudo journalctl -u gitlab-runner -f
```

Проблема с правами

```shell
sudo chmod 666 /var/run/docker.sock
# Или лучше
sudo usermod -aG docker gitlab-runner
```

## GitLab Runner в Docker

Этот конфиг идеально настроен для метода **Docker-in-Docker (DinD)**:
- `privileged = true` — обязательно для DinD.
- `docker.sock` — позволяет контейнерам «общаться» с докером на виртуалке.

*config/config.toml*
```toml
concurrent = 10
log_level = "warning"
log_format = "json"
check_interval = 5

[[runners]]
  name = "gitlab-runner-1"
  url = "gitlab.example.com"  # FIXME Change to your GitLab instance URL
  executor = "docker"
  token = ""  # FIXME Add your registration token here
  limit = 0
  # FIXME To increase rate limits, when pulling down images from the Docker Hub you might want to authenticate:
  # 1. Create a Docker Hub account and generate a personal access token
  # 2. Encode the username and token in base64
  #    Example: echo -n 'username:token' | base64
  # 3. Replace the <BASE64_ENCODED_AUTH> with the base64 encoded string
  environment = ["DOCKER_AUTH_CONFIG={\"auths\":{\"https://index.docker.io/v1/\":{\"auth\":\"<BASE64_ENCODED_AUTH>\"}}}"]
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    dns = ["8.8.8.8", "1.1.1.1"]
  [runners.cache]
    Insecure = false
```

*docker-compose.yaml*
```yaml
services:
  gitlab-runner:
    image: docker.io/gitlab/gitlab-runner:alpine-v17.9.1
    container_name: gitlab-runner-1
    volumes:
      - ./config/config.toml:/etc/gitlab-runner/config.toml:ro
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

# Kaniko

Kaniko — это инструмент от Google, предназначенный для сборки Docker-образов внутри контейнера (например, в Kubernetes или GitLab CI), не требующий привилегий root или запущенного Docker-демона (Docker-in-Docker).

Kaniko не обращается к демону Docker. Он распаковывает файловую систему базового образа в памяти, выполняет команды из `Dockerfile` в пользовательском пространстве и отправляет готовый результат (слои) напрямую в Registry (Docker Hub, AWS ECR, GitLab Registry).

Пример файла `.gitlab-ci.yml`, который собирает образ и пушит его в реестр вашего проекта

```yaml
stages:
  - build

build_image:
  stage: build
  # Используем debug-образ, так как в нем есть shell (необходим для работы GitLab Runner)
  image:
    name: gcr.io/kaniko-project/executor:v1.23.2-debug 
    entrypoint: [""]
  
  script:
    # 1. Подготовка конфигурации для авторизации в GitLab Registry
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"auth\":\"$(printf "%s:%s" "$CI_REGISTRY_USER" "$CI_REGISTRY_PASSWORD" | base64 | tr -d '\n')\"}}}" > /kaniko/.docker/config.json
    
    # 2. Запуск сборки
    - /kaniko/executor \
      --context "${CI_PROJECT_DIR}" \
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT_SHA}" \
      --destination "${CI_REGISTRY_IMAGE}:latest" \
      --cache=true \
      --cache-ttl=24h
  
  # Правила: запускать только при пуше в ветку main или при создании тегов
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_TAG
```

Разбор ключевых параметров:

 **Конфигурация `/kaniko/.docker/config.json`**:
    - Kaniko не делает `docker login`. Вы должны вручную создать файл с учетными данными.
    - Переменные `$CI_REGISTRY_USER` и `$CI_REGISTRY_PASSWORD` GitLab подставляет автоматически — это временные токены с правами на запись в реестр текущего проекта.
**`--context`**: Указывает корень проекта (где лежит ваш исходный код).
**`--dockerfile`**: Путь к вашему Dockerfile.
**`--destination`**: Куда пушить образ. В примере мы пушим сразу два тега: уникальный (SHA коммита) и `latest`.
**`--cache=true`**: Включает кэширование слоев. Kaniko будет сохранять слои в специальный репозиторий в вашем Registry, чтобы не пересобирать их каждый раз.

## Типичный пайплайн в GitLab

1)

```yaml
stages:
  - test
  - build
  - update

# 1. Тесты и линтеры
linting:
  stage: test
  image: python:3.13-slim
  tags:
    - ubuntu-2
  script:
    - pip install flake8
    - flake8 . --exclude=venv
  allow_failure: true

# 2. Сборка и пуш в Registry
build_image:
  stage: build
  image: docker:latest  # Здесь нужен только клиент, сервис dind не пишем
  tags:
    - ubuntu-2
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG-$CI_COMMIT_SHORT_SHA
    LATEST_TAG: $CI_REGISTRY_IMAGE:latest
  before_script:
    # Используем встроенные переменные GitLab для логина
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Сборка будет происходить прямо на вашей виртуалке
    - docker build -t $IMAGE_TAG -t $LATEST_TAG .
    - docker push $IMAGE_TAG
    - docker push $LATEST_TAG
  only:
    - main


update_helm_values:
  stage: update
  image: alpine:3.18.2
  tags:
    - ubuntu-2
  variables:
    NEW_TAG: $CI_COMMIT_REF_SLUG-$CI_COMMIT_SHORT_SHA
  before_script:
    # 1. Устанавливаем необходимые инструменты
    - apk add --no-cache git yq
    # 2. Помечаем директорию как безопасную (защита от ошибок прав в git)
    - git config --global --add safe.directory $CI_PROJECT_DIR
  script:
    # 1. Глобальные настройки Git
    - git config --global user.email "gitlab-runner@example.com"
    - git config --global user.name "GitLab Runner"

    # 2. Переключаемся на актуальную ветку, чтобы избежать состояния 'detached HEAD'
    - git checkout $CI_COMMIT_REF_NAME
    - git pull origin $CI_COMMIT_REF_NAME

    # 3. Обновление значений через yq.
    # Используем конструкцию env(), чтобы передать переменные из CI в yq максимально безопасно.
    - yq -i ".image.tag = env(NEW_TAG)" k8s/my-flask-app/values.yaml
    - yq -i ".image.repository = env(CI_REGISTRY_IMAGE)" k8s/my-flask-app/values.yaml

    # 4. Проверяем, есть ли реальные изменения.
    # Если тег не изменился (например, перезапуск джобы), git commit упадет.
    # "git diff --quiet" вернет 0, если изменений нет.
    - |
      if git diff --quiet k8s/my-flask-app/values.yaml; then
        echo "No changes to commit"
      else
        git remote set-url origin https://oauth2:${GIT_PUSH_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git
        git add k8s/my-flask-app/values.yaml
        git commit -m "chore: update image to $NEW_TAG [skip ci]"
        git push origin HEAD:$CI_COMMIT_REF_NAME
      fi
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

2)

```yaml
stages:
  - lint
  - plan
  - apply
  - destroy

# Глобальные переменные
variables:
  TF_VAR_pm_api_url: ${pm_api_url}
  TF_VAR_pm_api_token_id: ${pm_api_token_id}
  TF_VAR_pm_api_token_secret: ${pm_api_token_secret}
  TF_VAR_template_name: ${template_name}
  TF_VAR_ssh_key: ${ssh_key}
  TF_VAR_target_node: "pve"
  STATE_URL: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/default"

# Описываем шаблон инициализации, чтобы не дублировать код
.terraform_init: &terraform_init
  image:
    name: hashicorp/terraform:1.6.0
    entrypoint: [""]
  variables:
    # Эти переменные автоматически подхватываются HTTP-бэкендом
    TF_HTTP_ADDRESS: ${STATE_URL}
    TF_HTTP_LOCK_ADDRESS: ${STATE_URL}/lock
    TF_HTTP_UNLOCK_ADDRESS: ${STATE_URL}/lock
    TF_HTTP_USERNAME: "gitlab-ci-token"
    TF_HTTP_PASSWORD: ${CI_JOB_TOKEN}
    TF_HTTP_LOCK_METHOD: "POST"
    TF_HTTP_UNLOCK_METHOD: "DELETE"
  before_script:
    # Копируем файл для настройки зеркала
    - cp .terraformrc ~/.terraformrc
    # Теперь init максимально простой, всё остальное в переменных
    - terraform init -reconfigure

# 1. Проверка синтаксиса (использует свой спец. образ)
tflint:
  stage: lint
  image:
    name: ghcr.io/terraform-linters/tflint:latest
    entrypoint: [""]
  script:
    - tflint --init
    - tflint
  allow_failure:
    exit_codes: [ 2, 3 ]
  tags:
    - ubuntu-2

# 2. Создание плана
plan:
  <<: *terraform_init
  stage: plan
  needs: ["tflint"]
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan
    expire_in: 1 hour
  tags:
    - ubuntu-2

# 3. Применение (ручной запуск)
apply:
  <<: *terraform_init
  stage: apply
  needs: ["plan"]
  script:
    - terraform apply -input=false tfplan
  when: manual
  tags:
    - ubuntu-2

# 4. Удаление (ручной запуск)
destroy:
  <<: *terraform_init
  stage: destroy
  script:
    - terraform destroy -auto-approve
  when: manual
  tags:
    - ubuntu-2
```