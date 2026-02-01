
Helm - это менеджер пакетов для kuberntes.
Как любой менеджер пакетов, Helm упрощает задачу управления жизненным циклом приложений.

**Helm Charts**

Упаковка приложения называется helm chart.

создаем шаблоны
```
helm create App
```

делаем template
```
helm template App .
```

деплоим приложение App используя данные в папке App-HelmChart
```
helm install App App-HelmChart
```

информация про helm чарт
```
helm history App
```

передеплоим новую версию чарта через upgrade
```
helm upgrade App
```

Откат на предыдущую версию
```
helm rollback <имя_релиза> [номер_ревизии]
```

удалить чарт
```
helm uninstall App
```


---
деплоим через файл
```
helm install App -f <service.yaml>
```

добавляем репозиторий bitnami
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Можно локально загрузить
```
helm pull <service-name> <bitnami-chart-name>
```


Если ищем чарты через bitnami и нужно перезаписать какие-то данные (также можно использовать флаг `--values`)
```
helm install <service-name> <bitnami-chart-name> -f <values.yaml>
```

# Деплой в разные неймспейсы

Деплой в разные окружения (**Dev**, **Test**, **Prod**) с помощью Helm обычно строится на принципе **одного чарта и нескольких файлов конфигурации**.

Вместо того чтобы создавать отдельный чарт для каждой среды, вы создаете универсальный шаблон, а специфические настройки (количества реплик, доступы к БД, лимиты ресурсов) выносите в отдельные файлы `values.yaml`.

---
## 1. Структура проекта

Типичная структура репозитория выглядит так:

Plaintext

```
my-app/
├── charts/                # Зависимости
├── templates/             # Шаблоны манифестов (Deployment, Service и т.д.)
├── Chart.yaml             # Описание чарта
├── values.yaml            # Значения по умолчанию (обычно для Dev)
├── values-test.yaml       # Специфичные настройки для Test
└── values-prod.yaml       # Специфичные настройки для Prod
```

## 2. Различия в конфигурациях

В каждом файле `values-*.yaml` вы переопределяете только то, что меняется:

| **Параметр**      | **Dev**                                         | **Test**                                         | **Prod**         |
| ----------------- | ----------------------------------------------- | ------------------------------------------------ | ---------------- |
| **Replica Count** | 1                                               | 2                                                | 5+ (с HPA)       |
| **Resources**     | Minimal                                         | Medium                                           | Guaranteed (QoS) |
| **Ingress**       | https://www.google.com/search?q=dev.example.com | https://www.google.com/search?q=test.example.com | example.com      |
| **DB Config**     | Embedded/Shared                                 | Dedicated Test DB                                | Managed Cloud DB |

---

## 3. Процесс деплоя (CI/CD)

Обычно процесс автоматизируется через CI/CD (GitLab CI, GitHub Actions, ArgoCD).

### Способ А: Императивный (Helm CLI)

В конвейере CI/CD вызывается команда установки с указанием нужного файла:

В Dev:   
```
helm upgrade --install my-app ./my-app -f values.yaml --namespace dev
```
   
В Test:
   
```
helm upgrade --install my-app ./my-app -f values-test.yaml --namespace test
```
   
В Prod:
   
```
helm upgrade --install my-app ./my-app -f values-prod.yaml --namespace prod
```


## Helmfile

Устанавливаем helmfile и создаем helmfile.yaml
Структура папок та же, что была создана при помощи helm.

```yaml
releases:
  - name: rediscart
    chart: charts/redis
    values: 
      - values/redis-values.yaml
      - appReplicas: "1"
      - volumeName: "redis-cart-data"
  
  - name: emailservice
    chart: charts/microservice
    values:
      - values/email-service-values.yaml
```

Запуск:
```
helmfile sync
helmfile list
```

Uninstall
```
helmfile destroy
```

