
## Ad-hoc команды

```bash
ansible <host-pattern> -m <module-name> -a “<module-arguments>”
```

`ansible -i ~/myinventory all -m ping` — **Проверка связи**: пингует хосты и проверяет доступность через модуль ping.

`ansible -i ~/myinventory all -m setup` — **Сбор фактов**: собирает подробную системную информацию о ваших хостах (процессор, память, ОС) через модуль setup.

`ansible -i ~/myinventory all -m shell -a "uptime"` — **Время работы**: выполняет команду `uptime` на всех хостах через модуль shell, чтобы узнать, как долго они работают.


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

## Molecule

Molecule — это основной фреймворк для автоматизированного тестирования ролей, плейбуков и коллекций Ansible. Он позволяет разработчикам инфраструктуры (IaC) проверять корректность своего кода в изолированных средах перед развертыванием в продакшене.

Основные этапы цикла тестирования (`molecule test`)

1. **Dependency:** Установка зависимых ролей из Ansible Galaxy.
2. **Lint:** Проверка кода на соответствие стандартам.
3. **Create:** Поднятие тестовых инстансов (например, контейнеров Docker).
4. **Converge:** Запуск тестируемой роли на созданных инстансах.
5. **Idempotence:** Проверка, что повторный запуск не меняет состояние системы.
6. **Verify:** Выполнение проверочных скриптов (Unit/Integration тесты).
7. **Destroy:** Удаление тестовых инстансов после завершения

Одно окружение — много проектов

Тебе не нужно создавать новое окружение для _каждой_ роли. Создай одно мощное окружение под названием, например, `ansible-molecule` и используй его для всех своих плейбуков и ролей.

```
# Создаем один раз
pyenv virtualenv 3.12.3 ansible-molecule

# Устанавливаем всё необходимое один раз
pyenv activate ansible-molecule
pip install molecule "molecule-plugins[docker]" ansible-lint docker
```

В папке создаем окружение

```
cd ~/my-ansible-project 
pyenv local ansible-molecule
```

Чтобы запустился `ansible-lint` **надо** чтобы в корне лежал `.git`

### Шаг 1: Инициализация Molecule

Сначала вам нужно установить Molecule и его зависимости, если вы еще этого не сделали. Убедитесь, что у вас установлен pip (менеджер пакетов Python).

```shell
pip install "molecule[docker]" ansible
```

Затем перейдите в корневой каталог вашей Ansible-роли docker:

```shell
cd docker/
```

Теперь инициализируйте Molecule для создания нового сценария с использованием драйвера Docker. Обычно для начала используется сценарий default.

```shell
molecule init scenario -d docker
```
Эта команда создаст следующую структуру каталогов внутри вашей роли docker:

```
docker/
├── defaults/
├── handlers/
├── meta/
├── molecule/
│   └── default/
│       ├── converge.yml
│       ├── create.yml
│       ├── destroy.yml
│       ├── molecule.yml
│       └── verify.yml
├── tasks/
├── templates/
├── tests/
└── vars/
```

Прописываем мета информацию в роли иначе тесты не запустятся
```yaml
galaxy_info:
  author: pavelk
  description: your description
  company: none
  license: MIT
  min_ansible_version: "2.1"

  # ДОБАВЬ ЭТИ ДВЕ СТРОЧКИ:
  role_name: <role_name>
  namespace: pavelk
```

### Шаг 2: Настройка molecule.yml

Основной файл конфигурации для вашего Molecule-сценария находится по адресу `docker/molecule/default/molecule.yml`. Вам нужно настроить его, чтобы использовать драйвер Docker и определить платформы для тестирования.

Вот минимальный и рекомендуемый конфиг для тестирования с Docker, который включает настройки для systemd в контейнере:

*molecule/default/molecule.yml*

```yaml
---
driver:
  name: docker
platforms:
  - name: instance # Имя вашего тестового экземпляра (контейнера)
    image: geerlingguy/docker-ubuntu2204-ansible:latest # Образ Docker для использования
    pre_build_image: true # Не собирать образ, а загружать его, если его нет
    privileged: true # Предоставляет расширенные привилегии для systemd
    command: "" # Переопределяет команду контейнера по умолчанию, чтобы он работал
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw # Необходимо для работы systemd в Docker
    cgroupns_mode: host # Необходимо для работы systemd в Docker
    tmpfs:
      - /tmp
      - /run # Необходимо для работы systemd в Docker
provisioner:
  name: ansible
  inventory:
    host_vars:
      instance:
        ansible_connection: community.docker.docker # Использует docker в качестве соединения
verifier:
  name: ansible
```

Пояснения к molecule.yml:

 - **`driver.name: docker`**
  Указывает Molecule использовать Docker для создания и управления тестовыми экземплярами.
 - **`platforms`**
  Определяет один или несколько тестовых хостов.
- **`name: instance`**
  Уникальное имя для вашего контейнера.
- **`image: geerlingguy/docker-ubuntu2204-ansible:latest`**
  Рекомендуемый образ Docker, который уже содержит Ansible и настроен для тестирования с systemd.
- **`pre_build_image: true`**
Molecule будет использовать существующий образ и не пытаться его пересобрать.
- **`privileged: true, volumes, cgroupns_mode, tmpfs`**
Эти параметры критически важны, если ваша роль Ansible взаимодействует с системными службами, использующими systemd внутри контейнера Docker. Без них systemd может работать некорректно.

- **`provisioner.name: ansible`**
Указывает, что Ansible будет использоваться для применения вашей роли.
- **`inventory.host_vars.instance.ansible_connection: community.docker.docker`**
Настраивает Ansible для подключения к контейнеру Docker.
- **`verifier.name: ansible`**
Указывает, что Ansible будет использоваться для верификации.



### Шаг 3: Настройка converge.yml

Файл `docker/molecule/default/converge.yml` — это плейбук, который Molecule использует для применения вашей Ansible-роли к созданному контейнеру.

Его содержимое должно быть примерно таким:

*molecule/default/converge.yml*

```yaml
---
- name: Converge
  hosts: all
  become: true # Если вашей роли нужны привилегии root
  tasks:
    - name: "Include docker role"
      ansible.builtin.include_role:
        name: "../../.." # Относительный путь к вашей роли
```
Пояснения к converge.yml:

- `hosts: all`
  Применяет плейбук ко всем хостам, определенным в Molecule (в нашем случае, к instance).
- `become: true`
  Если ваша роль требует привилегий root (например, для установки пакетов или настройки служб), установите это значение в true.
- `ansible.builtin.include_role: name`
  "../../..": Это критически важно. Поскольку файл converge.yml находится глубоко в структуре Molecule (docker/molecule/default/converge.yml), вам нужно указать относительный путь к корневому каталогу вашей роли docker. ../../.. означает "выйти на три уровня вверх по каталогу", что приведет к каталогу docker/.

### Шаг 4: Настройка create.yml


```yaml
---
- name: Create
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Create docker container
      community.docker.docker_container:
        name: instance
        image: geerlingguy/docker-ubuntu2204-ansible:latest
        state: started
        privileged: true
        command: sleep infinity
        volumes:
          - /sys/fs/cgroup:/sys/fs/cgroup:rw
        cgroupns_mode: host
        tmpfs:
          - /tmp
          - /run
      register: result
    - name: Print container info
      ansible.builtin.debug:
        var: result
    - name: Add container to inventory
      ansible.builtin.add_host:
        name: instance
        ansible_connection: community.docker.docker
        groups:
          - molecule
```

### Шаг 5: Настройка destroy.yml

```yaml
---
- name: Destroy
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Destroy docker container
      community.docker.docker_container:
        name: instance
        state: absent
        # Удаление связанных томов, если они были созданы
        cleanup: true
```
### Шаг 6: Запуск тестов

После настройки этих файлов вы можете запустить тесты Molecule, находясь в корневом каталоге вашей роли docker:

```shell
molecule test -s default
```

Эта команда выполнит весь жизненный цикл тестирования Molecule: destroy, create, prepare, converge, idempotence (проверяет, что повторное выполнение роли не приводит к изменениям), side_effect, verify и destroy снова.


Это основные шаги и минимальные конфигурации, необходимые для начала тестирования вашей роли Ansible с Molecule и драйвером Docker.