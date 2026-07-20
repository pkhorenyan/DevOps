
В современных дистрибутивах Linux параллельно сосуществуют две архитектуры логирования: классическая текстовая система (`syslog`/`rsyslog`) и современная бинарная система (`systemd-journald`).

1. **`systemd-journald` (Современный стандарт)**
    - **Как работает:** Этот демон первым перехватывает все сообщения от ядра, системных служб (юнитов systemd) и стандартных потоков вывода (stdout/stderr).
    - **Где хранит:** В структурированном **бинарном виде** по пути `/var/log/journal/`. Они не занимают лишнего места, индексируются и защищены от ручного редактирования.
    - **Как читать:** Только через специальную утилиту `journalctl`.
2. **`rsyslogd` (Классическая текстовая система)**
    - **Как работает:** Служба `rsyslog` забирает часть логов у `journald`, фильтрует их по типу/важности согласно правилам в файле `/etc/rsyslog.conf` и распределяет по текстовым файлам.
    - **Где хранит:** В виде обычного **открытого текста** в каталоге `/var/log/`.
    - **Как читать:** Любыми текстовыми утилитами (cat, tail, grep, less).


Главные текстовые логи в `/var/log/`

Большинство стандартных логов распределяется по конкретным файлам в зависимости от их назначения: 
- **`/var/log/syslog`** (или `/var/log/messages`) — глобальный системный журнал, куда пишется всё, кроме сообщений аутентификации.
- **`/var/log/auth.log`** (или `/var/log/secure`) — безопасность и авторизация (успешные и проваленные попытки входа, использование `sudo`).
- **`/var/log/kern.log`** — сообщения от ядра Linux, драйверов устройств и аппаратные сбои.
- **`/var/log/dmesg`** — логи этапа начальной загрузки системы до старта основных служб.

Как читать логи через `journalctl` (Примеры команд)

Для чтения бинарного журнала используйте следующие конструкции (часто требуются права `sudo`): 
- `sudo journalctl` — открыть весь журнал с самого начала (с возможностью прокрутки).
- `sudo journalctl -e` — сразу перейти в самый конец списка к свежим записям.
- `sudo journalctl -f` — режим «живого» мониторинга (выводит новые строки на экран по мере их появления).
- `sudo journalctl -u nginx` — показать логи конкретной службы (например, веб-сервера Nginx).
- `sudo journalctl -b` — показать логи только текущей загрузки ОС.
- `sudo journalctl -p err` — отфильтровать логи и показать только ошибки (уровень важности error и выше).

Механизм ротации логов (`logrotate`)

Если бы логи писались бесконечно в один файл, жесткий диск сервера быстро бы переполнился. Чтобы этого не произошло, используется утилита **`logrotate`**.

Она работает по расписанию (обычно раз в день/неделю) и выполняет следующие действия:
1. Переименовывает текущий файл лога (например, `syslog` становится `syslog.1`).
2. Создает новый пустой файл `syslog` для свежих записей.
3. Старые файлы архивирует для экономии места (`syslog.2.gz`, `syslog.3.gz`).
4. Удаляет самые старые архивы (например, хранящиеся больше 4 недель) согласно конфигурации в файле `/etc/logrotate.conf`.

---
# Практический пример

Настраиваем центральный сервер для сбора логов в вагранте поднимаем 2 машины web и log
на web поднимаем nginx
на log настраиваем центральный лог сервер на любой системе на выбор
- journald
- rsyslog
- elk
настраиваем аудит следящий за изменением конфигов нжинкса

все критичные логи с web должны собираться и локально и удаленно
все логи с nginx должны уходить на удаленный сервер (локально только критичные)
логи аудита должны также уходить на удаленную систему

---

# Централизованный сбор логов (Vagrant + rsyslog + nginx + auditd)

  
Ниже — полностью воспроизводимая инструкция для двух VM в Vagrant:

- `web` — 192.168.56.98 (nginx, auditd, rsyslog)
- `log` — 192.168.56.99 (центральный rsyslog)

Цели:

- Все логи nginx уходят на удалённый `log`.
- Локально на `web` остаются только критичные логи nginx.
- Критичные системные логи `web` сохраняются локально и уходят на `log`.
- Audit-логи изменений конфигов nginx уходят на `log`.

## 1. Vagrantfile

Создайте `Vagrantfile` в директории проекта:

```ruby

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.define "web" do |web|
    web.vm.hostname = "web"
    web.vm.network "private_network", ip: "192.168.56.98"
  end

  
  config.vm.define "log" do |log|
    log.vm.hostname = "log"
    log.vm.network "private_network", ip: "192.168.56.99"
  end
end

```

Запуск и доступ:

```bash
vagrant up
vagrant ssh web
vagrant ssh log
```

## 2. Настройка log (центральный сервер)  

### 2.1 Установка
  
```bash
sudo apt update
sudo apt install -y rsyslog
```

### 2.2 Включаем приём по TCP/514
  

Создайте файл `/etc/rsyslog.d/10-listen-tcp.conf`:  

```conf
# Приём логов по TCP/514
module(load="imtcp")
input(type="imtcp" port="514")
```
  
### 2.3 Разделение логов по хостам

  
Создайте файл `/etc/rsyslog.d/20-remote-templates.conf`:
  
```conf
# Шаблон пути для удалённых логов
$template RemoteHostLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"

# Все удалённые логи пишем по шаблону
*.* ?RemoteHostLogs
& stop
```

Создаём директорию и права:

  
```bash
sudo mkdir -p /var/log/remote
sudo chown syslog:adm /var/log/remote
```

Перезапуск:
 
```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog --no-pager
```

### 2.4 Ротация логов

По умолчанию rsyslog использует logrotate. При необходимости можно добавить отдельный файл, например `/etc/logrotate.d/remote-logs`. Базовый logrotate уже есть в системе и покрывает стандартные логи.

## 3. Настройка web (клиент)

### 3.1 Установка
 

```bash
sudo apt update
sudo apt install -y nginx rsyslog auditd
```
  
### 3.2 Nginx: локально только критичные логи, все логи — в syslog

В `/etc/nginx/nginx.conf` задайте (пример в http{}):

```nginx
# Локально оставляем только критичные ошибки
error_log /var/log/nginx/error.log crit; 

# Все ошибки отправляем в syslog (уйдут на центральный сервер)
error_log syslog:server=unix:/dev/log,tag=nginx_error,severity=info;

# Все access логи отправляем в syslog
access_log syslog:server=unix:/dev/log,tag=nginx_access,facility=local6;
```

Перезапуск nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

### 3.3 Rsyslog: отправка nginx и audit логов на log

#### 3.3.1 Читаем audit.log

  
Создайте `/etc/rsyslog.d/11-imfile-audit.conf`:

```conf
module(load="imfile")
input(type="imfile"
      File="/var/log/audit/audit.log"
      Tag="auditd"
      Severity="info"
      Facility="local5")
```

#### 3.3.2 Отправка на центральный сервер (ранние правила)
 

Важно: чтобы **nginx access/error не писались локально**, правила должны отрабатывать **до стандартных локальных правил rsyslog**. Поэтому добавьте следующий блок в начало `/etc/rsyslog.conf` (перед секцией `#### RULES ####`):
  

```conf
# Отправляем все nginx логи и не пишем локально
if ($programname == "nginx_access") then {
  action(type="omfwd" target="192.168.56.99" port="514" protocol="tcp")
  stop
}

if ($programname == "nginx_error") then {
  action(type="omfwd" target="192.168.56.99" port="514" protocol="tcp")
  stop
} 

# Отправляем audit логи (локально можно хранить)
if ($programname == "auditd") then {
  action(type="omfwd" target="192.168.56.99" port="514" protocol="tcp")
}

# Все критичные системные логи — локально + удалённо
if ($syslogseverity-text == "crit" or $syslogseverity-text == "alert" or $syslogseverity-text == "emerg") then {
  action(type="omfwd" target="192.168.56.99" port="514" protocol="tcp")
}
```

Перезапуск rsyslog:

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog --no-pager
```

### 3.4 Auditd: слежение за конфигами nginx

Создайте файл `/etc/audit/rules.d/nginx.rules`:

```conf
-w /etc/nginx/ -p wa -k nginx_conf
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
```

Применяем:

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

## 4. Проверка работы

### 4.1 На web

1. Критичный системный лог:

```bash
logger -p user.crit "test critical"
```

Проверка:

```bash
sudo tail -n 5 /var/log/syslog
```

2. Логи nginx (access должны быть только на log):  

```bash
curl -s http://localhost/ >/dev/null
```  

3. Ошибка nginx (критичный лог локально + удалённо):

```bash
sudo sed -i 's/worker_processes/worker_processess/' /etc/nginx/nginx.conf
sudo nginx -t
sudo sed -i 's/worker_processess/worker_processes/' /etc/nginx/nginx.conf
sudo nginx -t
```

### 4.2 На log
 

Проверка наличия логов:

```bash
sudo ls -la /var/log/remote/web/
```
 

Ожидаемые файлы:

- `syslog.log`
- `nginx_access.log`
- `nginx_error.log`
- `auditd.log`

  

Проверка содержимого:

  

```bash
sudo tail -n 20 /var/log/remote/web/syslog.log
sudo tail -n 20 /var/log/remote/web/nginx_access.log
sudo tail -n 20 /var/log/remote/web/nginx_error.log
sudo tail -n 20 /var/log/remote/web/auditd.log
```

  

## 5. Итог

- Логи nginx всегда уходят на центральный сервер.
- Локально на web остаются только критичные ошибки nginx.
- Критичные системные логи web пишутся локально и отправляются на log.
- Изменения конфигов nginx фиксируются auditd и отправляются на log.