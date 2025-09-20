# Настройка безопасности VPN-сервера

## 1. Установка UFW
```bash
# Установка UFW
sudo apt install ufw

# Базовые правила
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 443/tcp  # HTTPS/VPN
sudo ufw limit 22/tcp   # Защита SSH

# Политики по умолчанию
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Включить UFW
sudo ufw enable
```

## 2. Настройка Fail2ban
```bash
# Установка
sudo apt install fail2ban

# Конфигурация (/etc/fail2ban/jail.local)
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true

[ocserv]
enabled = true
port = 443
filter = ocserv
```

## 3. Ограничение соединений Iptables
```bash
# Правило для порта 443
sudo iptables -A INPUT -p tcp --dport 443 -m connlimit --connlimit-above 10 -j REJECT

# Сохранение правил
sudo iptables-save > /etc/iptables/rules.v4

# Создание службы восстановления (/etc/systemd/system/iptables-restore.service)
[Unit]
Description=Restore iptables rules
Before=network.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/iptables/rules.v4

[Install]
WantedBy=multi-user.target
```

## 4. Проверка настроек после перезагрузки (reboot)
```bash
# Статус UFW
sudo ufw status verbose
sudo ufw show added

# Статус Fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client status ocserv

# Правила Iptables
sudo iptables -L INPUT -v -n | grep 443
```
