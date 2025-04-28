# Руководство по настройке соединения Home Server - Keenetic - VPS через WireGuard

Цель проекта - создать систему, позволяющую обслуживать веб-сервер, работающий на домашнем компьютере с "серым" IP-адресом (без прямого доступа из интернета), через публичный домен с SSL-сертификатом.

## Общая архитектура решения:
- **Домашний сервер**: Веб-сервер на порту 8080 + UFW
- **Keenetic роутер**: Клиент WireGuard + перенаправление портов
- **VPS**: WireGuard сервер (wg-easy) + Nginx-LE в режиме host network

## 1. Настройка VPS

### 1.1. Установка Docker и Docker Compose

```bash
# Обновление системы
apt update && apt upgrade -y

# Установка необходимых пакетов
apt install -y apt-transport-https ca-certificates curl software-properties-common

# Установка Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

### 1.2. Установка WireGuard-Easy

```bash
# Создать директорию для WireGuard
mkdir -p ~/wg-easy

# Создать docker-compose.yml
cat > ~/wg-easy/docker-compose.yml << 'EOF'
version: "3.8"
services:
  wg-easy:
    image: weejewel/wg-easy
    container_name: wg-easy
    environment:
      - WG_HOST=IP_ВАШЕГО_VPS
      - PASSWORD=ПАРОЛЬ_ДЛЯ_ИНТЕРФЕЙСА
      - WG_PORT=51820
      - WG_DEFAULT_DNS=1.1.1.1
      # Опционально, если нужно изменить диапазон IP в WireGuard:
      # - WG_ALLOWED_IPS=10.8.0.0/24
    volumes:
      - ./config:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
networks:
  default:
    name: wireguard-net
EOF

# Заменить IP_ВАШЕГО_VPS на реальный
sed -i "s/IP_ВАШЕГО_VPS/$(curl -s ifconfig.me)/g" ~/wg-easy/docker-compose.yml

# Запустить WireGuard
cd ~/wg-easy && docker compose up -d
```

### 1.3. Установка Nginx-LE в режиме host network

```bash
# Клонировать nginx-le
git clone https://github.com/nginx-le/nginx-le.git

# Удалить лишнее
rm ~/nginx-le/docker-compose.yml
rm ~/nginx-le/etc/service-example.conf

# Создать docker-compose.yml
cat > ~/nginx-le/docker-compose.yml << 'EOF'
version: '3'
services:
  nginx:
    image: umputun/nginx-le:latest
    container_name: nginx
    hostname: nginx
    restart: always
    volumes:
      - ./etc/ssl:/etc/nginx/ssl
      - ./etc/service.conf:/etc/nginx/service.conf
      # - ./etc/service-example-2.conf:/etc/nginx/service2.conf # дополнительные конфигурации
      # - ./etc/stream-example-2.conf:/etc/nginx/stream2.conf # дополнительные потоковые конфигурации
      # - ./etc/conf.d:/etc/nginx/conf.d-le # папка конфигурации
      # - ./etc/stream.conf:/etc/nginx/stream.conf.d-le # конфигурация потоков
    network_mode: host # !!
    # ports: # секция удалена
    #     - "80:80"
    #     - "443:443"
    environment:
      - TZ=Europe/Moscow
      - LETSENCRYPT=true
      - LE_EMAIL=your_email@example.com
      - LE_FQDN=www.your_domain1.com,your_domain1.com,www.your_domain2.com,your_domain2.com
      # - SSL_CERT=le-crt.pem
      # - SSL_KEY=le-key.pem
      # - SSL_CHAIN_CERT=le-chain-crt.pem
      # - LE_ADDITIONAL_OPTIONS='--preferred-chain "ISRG Root X1"'
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
EOF

# Создать конфигурацию nginx
cat > ~/nginx-le/etc/service.conf << 'EOF'
server {
    listen 443 ssl http2;
    server_name LE_FQDN;

    ssl_certificate SSL_CERT;
    ssl_certificate_key SSL_KEY;
    ssl_trusted_certificate SSL_CHAIN_CERT;

    location / {
        proxy_pass http://10.8.0.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    location /.well-known/ {
       root /usr/share/nginx/html;
    }
}
EOF

# Создать стартовый скрипт
cat > ~/nginx-le/start-all.sh << 'EOF'
#!/bin/bash

# Остановить текущие контейнеры
echo "Stopping running containers..."
docker stop nginx || true
docker rm nginx || true

# Запустить контейнеры
echo "Starting containers..."
docker compose -f docker-compose.yml up -d

# Проверяем статус
echo "Checking container status..."
docker ps | grep nginx

echo "Done!"
EOF

# Сделать скрипт исполняемым
chmod +x ~/nginx-le/start-all.sh

# Запустить сервисы
cd ~/nginx-le && ./start-all.sh
```

### 1.4. Диагностические команды для VPS

```bash
# Проверить список Docker сетей
docker network ls

# Проверить IP-адреса контейнеров
docker inspect -f '{{range $key, $value := .NetworkSettings.Networks}}{{$key}}: {{$value.IPAddress}}{{printf "\n"}}{{end}}' wg-easy

# Проверить таблицу маршрутизации
ip route

# Проверить состояние WireGuard
docker exec -it wg-easy wg show

# Проверить доступность домашнего сервера через WireGuard
curl -v http://10.8.0.2:8080

# Проверить логи nginx
docker logs -f nginx

# Проверить конфигурацию nginx
docker exec -it nginx nginx -t

# Проверить сертификаты Certbot
docker exec -it nginx certbot certificates
```

## 2. Настройка роутера Keenetic

### 2.1. Включение и настройка клиента WireGuard на Keenetic

```
1. Зайти в веб-интерфейс Keenetic (обычно 192.168.1.1 или my.keenetic.net)
2. Перейти в раздел "Системные настройки" -> "Дополнения"
3. Найти и установить дополнение "WireGuard"
4. После установки перейти в раздел "VPN" -> "WireGuard"
5. Нажать "Добавить клиент"
6. Настроить клиент:
   - Имя: VPS-WireGuard
   - Тип подключения: Клиент
   - Private Key: [оставить пустым для автогенерации]
   - Туннель: Включено
   - MTU: 1420 (стандартное значение)
   - Public Key: [скопировать из конфигурации на VPS]
   - Предварительный ключ: [скопировать из конфигурации на VPS, если используется]
   - IP-адрес сервера: [IP вашего VPS]
   - Порт сервера: 51820
   - Разрешенные IP-адреса: 0.0.0.0/0 (для всего трафика) или 10.8.0.0/24 (только для трафика WireGuard)
   - Локальный IP-адрес: 10.8.0.2/24 (должен соответствовать адресу в настройках WireGuard на VPS)
```

### 2.2. Фиксация IP для домашнего сервера на Keenetic

```
1. Перейти в раздел "Список клиентов"
2. В списке "Зарегистрированные клиенты" выбрать редактирование для домашнего сервера
3. Отменить галочку "Постоянный IP-адрес" 
```

### 2.3. Настройка переадресации портов на Keenetic

```
1. Перейти в раздел "Домашняя сеть" -> "Серверы"
2. Нажать "Добавить" для создания нового правила
3. Настроить правило:
   - Описание: Web сервер
   - Протокол: TCP
   - Внешний порт: 8080
   - Внутренний порт: 8080 (или порт вашего web-сервера)
   - IP-адрес: [IP-адрес вашего домашнего сервера в локальной сети]
4. Сохранить правило
```

### 2.4. Диагностические команды для Keenetic

```
1. В веб-интерфейсе Keenetic перейти в "Диагностика" -> "Системный журнал" для проверки статуса WireGuard
2. Для проверки соединения перейти в "Диагностика" -> "Реальный IP"
3. Статус WireGuard можно проверить в разделе "VPN" -> "WireGuard" -> "Состояние"
4. Для проверки переадресации портов можно использовать раздел "Домашняя сеть" -> "Серверы" -> "Статистика"
```

## 3. Настройка домашнего сервера

### 3.1. Настройка веб-сервера

```bash
# Это пример для простого веб-сервера на Node.js
# Создайте директорию для вашего приложения
mkdir -p ~/web-server

# Создайте простой веб-сервер
cat > ~/web-server/server.js << 'EOF'
const http = require('http');

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello from Home Server!\n');
});

const port = 8080;
server.listen(port, '0.0.0.0', () => {
  console.log(`Server running at http://0.0.0.0:${port}/`);
});
EOF

# Установите Node.js, если его нет
# sudo apt update && sudo apt install -y nodejs npm

# Запустите сервер
# node ~/web-server/server.js
```

### 3.2. Настройка UFW

```bash
# Установите UFW, если его нет
sudo apt update && sudo apt install -y ufw

# Настройка базовых правил
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Разрешите SSH (важно сделать до включения UFW)
sudo ufw allow ssh

# Разрешите веб-сервер
sudo ufw allow 8080/tcp

# Разрешите трафик WireGuard
sudo ufw allow proto udp from 10.8.0.0/24 to any

# Включите пересылку пакетов
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/ufw/sysctl.conf
sudo sysctl -p /etc/ufw/sysctl.conf

# Разрешите трафик между интерфейсами
# Замените eth0 на ваш внешний интерфейс (проверьте через ip a)
# Замените wg0 на интерфейс WireGuard, если он есть на домашнем сервере
IFACE=$(ip route get 1.1.1.1 | grep -oP 'dev \K\S+')
sudo ufw route allow in on $IFACE out on $IFACE

# Включите UFW
sudo ufw enable

# Проверьте статус
sudo ufw status verbose
```

### 3.3. Диагностические команды для домашнего сервера

```bash
# Проверить запущенные сервисы
sudo ss -tuln | grep 8080

# Проверить соединения
sudo netstat -antp | grep 8080

# Проверить доступность веб-сервера локально
curl -v http://localhost:8080

# Проверить журналы UFW
sudo grep UFW /var/log/syslog

# Проверить правила iptables
sudo iptables -L -v -n
```

## 4. Проверка и диагностика всей системы

### 4.1. Проверка соединения Keenetic с WireGuard на VPS

```bash
# На VPS
docker exec -it wg-easy wg show
# В выводе должен быть виден пир с IP 10.8.0.2 и recent handshake
```

### 4.2. Проверка доступности домашнего сервера через WireGuard

```bash
# На VPS
# Пинг через WireGuard
ping -c 4 10.8.0.2

# Проверка доступности веб-сервера напрямую
curl -v http://10.8.0.2:8080

# Проверка доступность веб-сервера внутри wg-easy
docker exec -it wg-easy wget --spider http://10.8.0.2:8080
```

### 4.3. Проверка работы Nginx

```bash
# На VPS
# Проверка конфигурации Nginx
docker exec -it nginx nginx -t

# Проверка сертификатов
docker exec -it nginx certbot certificates

# Проверка доступности сайта через HTTPS
curl -v -k https://localhost
```

### 4.4. Проверка внешнего доступа

```bash
# С любого компьютера
# Замените домен на ваш
curl -v https://your_domain.com
```

## 5. Решение проблем и отладка

### 5.1. Проблемы с маршрутизацией на VPS

```bash
# Проверить маршруты
ip route

# Проверить IP-адреса контейнеров
docker inspect -f '{{range $key, $value := .NetworkSettings.Networks}}{{$key}}: {{$value.IPAddress}}{{printf "\n"}}{{end}}' wg-easy

# Проверить доступность до домашнего сервера
ping -c 4 10.8.0.2
curl -v http://10.8.0.2:8080
```

### 5.2. Проблемы с WireGuard

```bash
# Проверить статус WireGuard
docker exec -it wg-easy wg show

# Перезапустить WireGuard
cd ~/wg-easy && docker compose restart

# Проверить логи
docker logs -f wg-easy
```

### 5.3. Проблемы с Nginx и сетевыми настройками

```bash
# Убедиться, что nginx работает в режиме host
docker inspect nginx | grep -A 10 NetworkMode

# Проверить, видит ли nginx домашний сервер
docker exec -it nginx wget --spider http://10.8.0.2:8080

# Проверить конфигурацию
docker exec -it nginx nginx -t

# Проверить логи Nginx
docker exec -it nginx cat /var/log/nginx/error.log
```

### 5.4. Проблемы с Nginx и сертификатами

```bash
# Проверить конфигурацию
docker exec -it nginx nginx -t

# Проверить сертификаты
docker exec -it nginx certbot certificates

# Проверить доступность сертификатов
docker exec -it nginx ls -la /etc/letsencrypt/live/

# Проверить логи Nginx
docker exec -it nginx cat /var/log/nginx/error.log
```

## 6. Обслуживание системы

### 6.1. Обновление компонентов

```bash
# Обновление WireGuard
cd ~/wg-easy && docker compose pull && docker compose up -d

# Обновление Nginx
cd ~/nginx-le && docker compose pull && docker compose up -d
```

### 6.2. Резервное копирование

```bash
# Создать директорию для резервных копий
mkdir -p ~/backups

# Резервное копирование конфигурации WireGuard
tar -czf ~/backups/wg-easy-config-$(date +%Y%m%d).tar.gz -C ~/wg-easy config

# Резервное копирование Nginx и сертификатов
tar -czf ~/backups/nginx-le-$(date +%Y%m%d).tar.gz -C ~/nginx-le etc
```

### 6.3. Автоматический запуск после перезагрузки

```bash
# Создать скрипт для запуска всех сервисов
cat > ~/start-services.sh << 'EOF'
#!/bin/bash

# Запуск WireGuard
cd ~/wg-easy && docker compose up -d

# Подождать 10 секунд для инициализации WireGuard
sleep 10

# Запуск Nginx
cd ~/nginx-le && ./start-all.sh
EOF

chmod +x ~/start-services.sh

# Добавить в crontab для запуска при перезагрузке
(crontab -l 2>/dev/null; echo "@reboot ~/start-services.sh") | crontab -
```

## 7. Дополнительные рекомендации

### 7.1. Мониторинг

```bash
# Установка простого мониторинга
apt install -y htop iftop iotop

# Установка Glances для общего мониторинга
apt install -y glances

# Для запуска мониторинга:
glances
```

### 7.2. Защита VPS

```bash
# Усиление SSH
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# Установка Fail2ban
apt install -y fail2ban

# Базовая конфигурация Fail2ban
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
EOF

# Перезапуск Fail2ban
systemctl restart fail2ban
```

### 7.3. Улучшение производительности WireGuard

```bash
# Оптимизация настроек TCP для WireGuard
cat >> /etc/sysctl.conf << 'EOF'
# TCP BBR congestion control
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

# Увеличение буферов сети
net.core.rmem_max=26214400
net.core.wmem_max=26214400
net.core.rmem_default=1048576
net.core.wmem_default=1048576
net.ipv4.tcp_rmem=4096 1048576 26214400
net.ipv4.tcp_wmem=4096 1048576 26214400
EOF

# Применить изменения
sysctl -p
```

Это полное руководство по настройке и эксплуатации системы без использования socat. Основные изменения:
1. Используется `network_mode: host` для контейнера nginx, что позволяет ему напрямую обращаться к сети WireGuard
2. Удален контейнер socat, так как в нем больше нет необходимости
3. Обновлены команды диагностики и проверки, убраны упоминания socat
4. Упрощен процесс обслуживания, так как теперь нужно управлять меньшим количеством контейнеров
