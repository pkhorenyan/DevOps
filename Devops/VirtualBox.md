
```
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```

Settings -> Network

Adapter 1
Brudged Adapter
Adapter 2
NAT

Меняем имя хоста
```
sudo hostnamectl set-hostname <hostname>
```

### Входить как root по SSH

По умолчанию **Ubuntu Server не позволяет входить по SSH как root**, это нормально и сделано специально.
Скопируй ключ в root:

```
sudo mkdir -p /root/.ssh
sudo cp /home/pavelk/.ssh/authorized_keys /root/.ssh/
sudo chown -R root:root /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
```

Разреши root по ключу (в sshd_config)
```
sudo vim /etc/ssh/sshd_config
```

В sshd_config:

`PermitRootLogin prohibit-password`
`PasswordAuthentication no`

И перезапуск:

```
sudo systemctl restart ssh
```

Теперь вход:

```
ssh -i ~/.ssh/id_ed25519 root@<ip-address>
```

