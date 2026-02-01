
## Ad-hoc команды

```bash
ansible <host-pattern> -m <module-name> -a “<module-arguments>”
```

`ansible -i ~/myinventory all -m ping` — **Проверка связи**: пингует хосты и проверяет доступность через модуль ping.

`ansible -i ~/myinventory all -m setup` — **Сбор фактов**: собирает подробную системную информацию о ваших хостах (процессор, память, ОС) через модуль setup.

`ansible -i ~/myinventory all -m shell -a "uptime"` — **Время работы**: выполняет команду `uptime` на всех хостах через модуль shell, чтобы узнать, как долго они работают.

`ansible -i ~/myinventory all -m shell -a "df -h"` — **Место на диске**: проверяет свободное пространство на всех хостах в удобном для чтения формате.

`ansible -i ~/myinventory all -m command -a "cat /etc/passwd"` — **Список пользователей**: просматривает содержимое файла `/etc/passwd` на всех хостах, используя модуль command.

`ansible -i ~/myinventory all -m command -a "free -m"` — **Память**: получает информацию о состоянии оперативной памяти (в мегабайтах) на всех хостах.


- **To ensure a service is running**: Use `state: started` or `state: present` with a service module.
- **To ensure a file exists**: Use `state: touch` or `state: present` with the `file` module.
- **To ensure a user is removed**: Use `state: absent` with the `user` module

В Ansible «пинг» — это первая команда, которую стоит выполнить, чтобы проверить связь с вашими виртуалками.

```
ansible all -m ping
```

Перед выполнением задач Ansible заходит на сервер и собирает о нем подробное «досье». Этот процесс называется **Gathering Facts**.
Когда вы запускаете плейбук, самая первая строчка, которую вы видите в консоли — `TASK [Gathering Facts]`. В этот момент Ansible запускает на целевом сервере скрипт, который вытягивает всю системную информацию:

- Версию ОС и ядра.
- Количество процессоров и объем ОЗУ.
- **Размер свопа** (именно он записывается в `ansible_swaptotal_mb`).
- IP-адреса, диски и многое другое.

```
ansible all -m setup
```

## Ansible inventory

Файл inventory содержит хосты и опционально юзера, через которого Ansible подключается по SSH

```bash
[web]
webserver1 ansible_host=192.168.11.7 ansible_user=root 
webserver2 ansible_host=192.168.11.8 ansible_user=root 

[db]
dbserver1 ansible_host=192.168.12.10 ansible_user=dbadmin
```
## Конфиг файл

*ansible.cfg*

```
[defaults]

inventory = hosts.txt
private_key_file = ~/.ssh/id_ed25519
interpreter_python = /usr/bin/python3
host_key_checking = false
remote_user = pavelk
```

*hosts.txt*
```
[web_servers]
192.168.0.110

[db_servers]
192.168.0.111
```
## Ansible плейбуки

Структура типичного плейбука:

```yaml
- hosts: all
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
```

Запуск плэйбука
```bash
ansible-playbook <file.yaml>
```

Запуск плэйбука с указанием пароля для выполнения определенных команд
```bash
ansible-playbook <file.yaml> -K
```


## Ansible роли

Роль - законченный функциональный блок.
Роль не знает, **где** она запускается, но знает, **что нужно сделать**.

```bash
ansible-galaxy role init <my_new_role>
```

Стандартная структура роли

```text
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    │   └── index.html
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    └── meta/
        └── main.yml
```

### `tasks/main.yml`

➡️ **Что делать**

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
```

Это **обязательная часть** роли. 
Это стартовая точка роли, из нее можно подцеплять другие файлы.

---

### `handlers/main.yml`

➡️ **Реакция на изменения**

```yaml
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

Вызывается так:

```yaml
notify: restart nginx
```

Используется для:

- перезапуска сервисов;
- reload конфигов.

---

### `templates/`

➡️ **Jinja2-шаблоны**

```yaml
server {
  listen {{ nginx_port }};
}
```

Применение:

```yaml
template:
  src: nginx.conf.j2
  dest: /etc/nginx/nginx.conf
```
---

### `files/`

➡️ **Статические файлы**

```yaml
copy:
  src: index.html
  dest: /var/www/html/index.html
```

Без шаблонизации.

---

### `defaults/main.yml`

➡️ **Переменные по умолчанию (слабый приоритет)**

```yaml
nginx_user: www-data
```

Можно переопределять где угодно.

✔️ **Рекомендуется всё выносить сюда**

---

### `vars/main.yml`

➡️ **Жёсткие переменные (высокий приоритет)**

```yaml
nginx_user: www-data
```
❗ Используй **редко** — их сложно переопределять.

---

### `meta/main.yml`

➡️ **Зависимости роли**

```yaml
dependencies:
  - role: common
```

## Структура проекта для dev/prod

```text
ansible/
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── prod/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
│
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── postgres/
│   └── app/
│
├── playbooks/
│   ├── site.yml
│   ├── web.yml
│   └── db.yml
│
├── ansible.cfg
└── requirements.yml
```

Одна и та же роль — разные переменные.

Playbook — связывает роли и хосты. Идеальный плейбук должен содержать только роли и минимум тасок.
### `playbooks/web.yml`

```yaml
- hosts: web
  become: true
  roles:
    - common
    - nginx
    - app
```


Запуск

```bash
ansible-playbook -i inventories/dev web.yml
ansible-playbook -i inventories/prod web.yml
```

#### Requirements

`requirements.yml` — это **декларация зависимостей Ansible**.

Он говорит: «Вот какие роли (и/или коллекции) нужны проекту и откуда их взять».
Используется командой:

```bash
ansible-galaxy install -r requirements.yml
```

```yaml
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"

  - name: common
    src: git@gitlab.example.com:infra/ansible-role-common.git
    scm: git
    version: v1.4.1

  - name: docker
    src: https://github.com/geerlingguy/ansible-role-docker.git
    version: master
```

Всегда фиксируй `version` (tag или commit), не `master`
