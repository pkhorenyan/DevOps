
1. KVM (Kernel-based Virtual Machine) — «Двигатель»

Это **модуль ядра** Linux. Он превращает ядро в гипервизор.

- **Что делает:** Позволяет программе (QEMU) использовать мощь вашего реального процессора напрямую.

2. QEMU — «Эмулятор железа»

Это программа, которая создает «виртуальное железо».

- **Что делает:** Рисует для виртуалки виртуальную материнскую плату, видеокарту, сетевой адаптер и жесткий диск.
- **Связка:** QEMU может работать сам по себе (медленно), но в паре с **KVM** он работает почти со скоростью реального железа.

3. Libvirt — «Диспетчер»

Это библиотека и сервис (`libvirtd`).

- **Что делает:** Избавляет вас от необходимости писать километровые команды запуска QEMU. Он хранит настройки каждой виртуалки в XML-файлах и управляет их состоянием (старт, стоп, пауза, бэкап).
- **Софт:** Именно в него входят утилиты `virsh` и `virt-install`.

4. Интерфейсы управления (фронтенды)

То, через что вы даете команды Либвирту:

- **virt-install / virsh** — консольные утилиты.
- **Virt-manager** — графическое приложение (окна, кнопки, как в VirtualBox).
- **Cockpit** — веб-интерфейс для управления сервером и виртуалками через браузер.

---

Итого, цепочка выглядит так:

Вы вводите команду в **virt-install** - она идет в **libvirt**, тот настраивает **QEMU**, который через  **KVM** запускает код на вашем **процессоре**.

---

Перед установкой убедитесь, что процессор поддерживает аппаратную виртуализацию и она включена в BIOS/UEFI.
```bash
grep -Eoc '(vmx|svm)' /proc/cpuinfo
```

**VirtualBox**: Выключите ВМ. В настройках: _Система -> Процессор -> Включить Nested VT-x/AMD-V_. Если галка серая, выполните в терминале хоста (Windows/Mac)

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "<vm-name>" --nested-hw-virt on
```

Установка
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager virtinst -y
```

- **qemu-kvm**: сам гипервизор.
- **libvirt-daemon-system**: системная служба для управления виртуализацией.
- **virt-manager**: графический интерфейс для управления виртуалками.
- **bridge-utils**: инструменты для настройки сетевых мостов.

Чтобы управлять виртуальными машинами без использования `sudo`, добавьте своего пользователя в группы `libvirt` и `kvm`:
```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

Убедитесь, что демон `libvirtd` запущен и настроен на автозагрузку:
```bash
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
```

По умолчанию `libvirt` создает сеть NAT. Чтобы активировать её:
```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

#### Пример команды для создания ВМ

Для работы с виртуальными машинами из командной строки в Debian используется мощная утилита **`virt-install`**.

Вот типовой шаблон для создания виртуалки с Debian (или любым другим Linux):

```bash
virt-install \
  --connect qemu:///system \
  --name my-vm-01 \
  --ram 1048 \
  --vcpus 1 \
  --disk size=7 \
  --os-variant debian10 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --location 'http://archive.debian.org/debian/dists/buster/main/installer-amd64/' \
  --extra-args 'console=ttyS0,115200n8 serial'
```

**Разбор параметров:**

- `--name`: название вашей ВМ.
- `--ram` и `--vcpus`: ресурсы (ОЗУ в МБ и количество ядер).
- `--disk size=20`: создаст диск на 20 ГБ в стандартном хранилище (`/var/lib/libvirt/images`).
- `--graphics none`: отключает графику (удобно для SSH-сессий).
- `--location`: указывает, откуда брать установщик (можно заменить на `--cdrom /путь/к/iso`).
- `--extra-args`: пробрасывает вывод установщика прямо в вашу консоль.


```bash
sudo apt update && sudo apt install libosinfo-bin -y

# Найдите точное название для вашей системы:
osinfo-query os | grep debian
```

#### Управление через `virsh`

После создания ВМ основным инструментом управления становится **`virsh`**. Вот список самых нужных команд:

```bash
#Список всех ВМ
virsh list --all

#Запуск ВМ
virsh start my-vm-01

# Выключение
virsh shutdown my-vm-01

# Принудительнео выключение
virsh destroy my-vm-01

# Подключиться к консоли запущенной ВМ
virsh console my-vm-01

# Удаление ВМ (вместе с диском)
virsh undefine my-vm-01 --remove-all-storage
```

   
