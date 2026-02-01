
**Мониторинг** говорит вам, _что_ сломалось, а **обзервабилити** (наблюдаемость) помогает понять, _почему_ это произошло.

- **Мониторинг (Monitoring):** Это пассивный сбор данных на основе заранее известных метрик. Вы настраиваете дашборды для отслеживания «известных неизвестных» (тех проблем, которые вы ожидаете, например: загрузка CPU, остаток места на диске, доступность порта 9100).
- **Обзервабилити (Observability):** Это свойство системы, позволяющее понять её внутреннее состояние на основе внешних данных. Она нацелена на «неизвестные неизвестные» — непредсказуемые аномалии в сложных распределенных системах (микросервисах).

**Prometheus** - метрики
**Promtail** - это агент, ответственный за сбор логов и отправку их в Loki
**Loki** - это основной сервис, ответственный за хранение логов и обработку запросов
**Grafana** - инструмент визуализации

---

Установочный скрипт для Prometheus Server на Ubuntu

```shell
#!/bin/bash
PROMETHEUS_VERSION="2.51.1"
PROMETHEUS_FOLDER_CONFIG="/etc/prometheus"
PROMETHEUS_FOLDER_TSDATA="/etc/prometheus/data"

cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v$PROMETHEUS_VERSION/prometheus-$PROMETHEUS_VERSION.linux-amd64.tar.gz
tar xvfz prometheus-$PROMETHEUS_VERSION.linux-amd64.tar.gz
cd prometheus-$PROMETHEUS_VERSION.linux-amd64

mv prometheus /usr/bin/
rm -rf /tmp/prometheus*

mkdir -p $PROMETHEUS_FOLDER_CONFIG
mkdir -p $PROMETHEUS_FOLDER_TSDATA


cat <<EOF> $PROMETHEUS_FOLDER_CONFIG/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name      : "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
EOF

useradd -rs /bin/false prometheus
chown prometheus:prometheus /usr/bin/prometheus
chown prometheus:prometheus $PROMETHEUS_FOLDER_CONFIG
chown prometheus:prometheus $PROMETHEUS_FOLDER_CONFIG/prometheus.yml
chown prometheus:prometheus $PROMETHEUS_FOLDER_TSDATA


cat <<EOF> /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
ExecStart=/usr/bin/prometheus \
  --config.file       ${PROMETHEUS_FOLDER_CONFIG}/prometheus.yml \
  --storage.tsdb.path ${PROMETHEUS_FOLDER_TSDATA}

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
systemctl status prometheus --no-pager
prometheus --version
```

Установочный скрипт для Node Exporter

```shell
#!/bin/bash
# https://github.com/prometheus/node_exporter/releases
NODE_EXPORTER_VERSION="1.7.0"

cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v$NODE_EXPORTER_VERSION/node_exporter-$NODE_EXPORTER_VERSION.linux-amd64.tar.gz
tar xvfz node_exporter-$NODE_EXPORTER_VERSION.linux-amd64.tar.gz
cd node_exporter-$NODE_EXPORTER_VERSION.linux-amd64

mv node_exporter /usr/bin/
rm -rf /tmp/node_exporter*

useradd -rs /bin/false node_exporter
chown node_exporter:node_exporter /usr/bin/node_exporter

cat <<EOF> /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network.target
 
[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/bin/node_exporter
 
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
systemctl status node_exporter
node_exporter --version
```

Добавить node server в конфиг:

```shell
  - job_name      : "node_server"
    static_configs:
      - targets: ["localhost:9100"]
```


Установочный скрипт для установки Grafana Server

```shell
#!/bin/bash
# https://grafana.com/grafana/download
GRAFANA_VERSION="10.4.2"
PROMETHEUS_URL="http://localhost:9090"

apt-get install -y apt-transport-https software-properties-common wget
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
apt-get update
apt-get install -y adduser libfontconfig1 musl

wget https://dl.grafana.com/oss/release/grafana_${GRAFANA_VERSION}_amd64.deb
dpkg -i grafana_${GRAFANA_VERSION}_amd64.deb

echo "export PATH=/usr/share/grafana/bin:$PATH" >> /etc/profile

cat <<EOF> /etc/grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    url: ${PROMETHEUS_URL}
EOF

systemctl daemon-reload
systemctl enable grafana-server
systemctl start grafana-server
systemctl status grafana-server
```

Пример нагрузочного теста на CPU
```
docker run -it --name cpustress --rm containerstack/cpustress --cpu 4 --timeout 30s --metrics-brief
```

Пример *prometheus.yml* с доступом на EC2 сервера:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name      : "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name      : "ubuntu-servers"
    static_configs:
      - targets:
          - 10.0.0.1:9100
          - 10:0.0.2:9100

  - job_name      : "windows-servers"
    static_configs:
      - targets:
          - 10.0.0.3:9182
          - 10:0.0.4:9182

  - job_name      : "prod-servers"
    ec2_sd_configs:
    - port: 9100
      filters:
        - name  : tag:Environment
          values: ["prod"]

  - job_name      : "dev-servers"
    ec2_sd_configs:
    - port: 9100
      filters:
        - name  : tag:Environment
          values: ["dev"]
```

### Алерты

Правила алертов определяются в специальных YAML-файлах, которые Prometheus периодически проверяет.

Коллекция написанных алертов:
https://samber.github.io/awesome-prometheus-alerts/

Создай новый файл правил (например, `alert_rules.yml`) в каталоге конфигурации Prometheus (обычно `/etc/prometheus/`):

```yaml
# /etc/prometheus/alert_rules.yml
groups:
- name: GeneralAlerts
  rules:
  - alert: InstanceDown
    expr: up{job="node"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Инстанс {{ $labels.instance }} недоступен"
      description: "{{ $labels.instance }} перестал отвечать более 1 минуты назад."

  - alert: HighCPUUsage
    expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Высокая загрузка CPU на {{ $labels.instance }}"
      description: "Загрузка CPU превышает 80% в течение 5 минут."
```

Теперь надо подключить файл правил к Prometheus

Отредактируй основной файл конфигурации Prometheus (`/etc/prometheus/prometheus.yml`):

```yaml
# /etc/prometheus/prometheus.yml

global:
  # ... глобальные настройки ...

# Добавь эту строку для загрузки ваших правил
rule_files:
  - "alert_rules.yml" 

# Добавь этот блок для связи с Alertmanager (если он установлен)
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # Укажи адрес вашего Alertmanager (по умолчанию порт 9093)
      - localhost:9093 
```

Перезагружаем Prometheus
```
sudo systemctl restart prometheus
```

### Настройка Alertmanager

Alertmanager отвечает за дедупликацию, группировку и отправку уведомлений.

Скачать, распаковать и запустить Alertmanager.
Скрипт для установки:

```shell
#!/bin/bash
# https://github.com/prometheus/alertmanager/releases
ALERTMANAGER_VERSION="0.27.0"
ALERTMANAGER_FOLDER_CONFIG="/etc/alertmanager"

cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v${ALERTMANAGER_VERSION}/alertmanager-${ALERTMANAGER_VERSION}.linux-amd64.tar.gz
tar xvfz alertmanager-${ALERTMANAGER_VERSION}.linux-amd64.tar.gz
cd alertmanager-${ALERTMANAGER_VERSION}.linux-amd64

# Копируем бинарники
mv alertmanager /usr/bin/
mv amtool /usr/bin/
rm -rf /tmp/alertmanager*

# Создаем директорию для конфига
mkdir -p $ALERTMANAGER_FOLDER_CONFIG

# Создаем базовый конфиг
cat <<EOF> $ALERTMANAGER_FOLDER_CONFIG/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'

receivers:
  - name: 'default'
    # Можно настроить email, slack, telegram и т.д.
    # Пример с email (закомментирован):
    # email_configs:
    #   - to: 'your-email@example.com'
    #     from: 'alertmanager@example.com'
    #     smarthost: 'smtp.gmail.com:587'
    #     auth_username: 'your-email@gmail.com'
    #     auth_password: 'your-app-password'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
EOF

# Создаем пользователя
useradd -rs /bin/false alertmanager
chown -R alertmanager:alertmanager /usr/bin/alertmanager
chown -R alertmanager:alertmanager /usr/bin/amtool
chown -R alertmanager:alertmanager $ALERTMANAGER_FOLDER_CONFIG

# Создаем systemd service
cat <<EOF> /etc/systemd/system/alertmanager.service
[Unit]
Description=Prometheus Alertmanager
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
Restart=on-failure
ExecStart=/usr/bin/alertmanager \\
  --config.file=${ALERTMANAGER_FOLDER_CONFIG}/alertmanager.yml \\
  --storage.path=${ALERTMANAGER_FOLDER_CONFIG}/data

[Install]
WantedBy=multi-user.target
EOF

# Запускаем
systemctl daemon-reload
systemctl start alertmanager
systemctl enable alertmanager
systemctl status alertmanager --no-pager
alertmanager --version

echo "Alertmanager установлен!"
echo "Web UI доступен на: http://localhost:9093"
echo "Конфиг: ${ALERTMANAGER_FOLDER_CONFIG}/alertmanager.yml"
```


Создай или отредактируй файл конфигурации (`/etc/alertmanager/alertmanager.yml`).

Вот пример настройки для отправки уведомлений в **Slack** или на **Email**:

```yaml
# /etc/alertmanager/alertmanager.yml

global:
  # Настройки SMTP для отправки email (если вы используете email)
  # smtp_smarthost: 'smtp.gmail.com:587'
  # smtp_auth_username: 'ваша_почта@gmail.com'
  # smtp_auth_password: 'ваш_пароль'
  # smtp_from: 'prometheus@example.com'

# --- Маршрутизация алертов ---
route:
  # Все алерты сначала идут сюда
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h

receivers:
# --- Настройка Email получателя ---
- name: 'default-receiver'
  # email_configs:
  # - to: 'ваш_email@example.com'
  #   send_resolved: true

# --- Настройка Slack получателя ---
# - name: 'default-receiver'
  slack_configs:
  - channel: '#alerts-channel'
    api_url: 'hooks.slack.com' # Ваш Slack Webhook URL
    send_resolved: true
    text: '{{ .CommonAnnotations.description }}'
    title: '{{ .CommonAnnotations.summary }}'
```

Запусти сервис Alertmanager 
```
sudo systemctl start alertmanager
```

Веб-интерфейс Alertmanager доступен по адресу `http://<ваш_ip>:9093`.
