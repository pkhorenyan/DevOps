

Полностью отключаем запуск через сокет 

```
systemctl stop ssh.socket 
systemctl disable ssh.socket 
systemctl mask ssh.socket
```
# Принудительно запускаем обычный сервис 

```
systemctl enable ssh.service
systemctl restart ssh.service
```

