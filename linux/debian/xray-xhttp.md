# Установка и настройка Xray (XHTTP) на Debian GNU/Linux
**Важно:** В данной инструкции для наглядности в примерах и конфигурационных файлах **используется домен `r2bny.com`**. Перед применением обязательно **замените его на ваш собственный домен** во всех упоминаниях в конфигурациях сервисов, путях к файлам и директориях, ссылках и сертификатах.

## 1. Установка Xray
Выполните официальный скрипт установки Xray:
```bash
sudo apt install -y curl
bash <(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh) install
```
- Бинарный файл: `/usr/local/bin/xray`
- Конфигурация: `/usr/local/etc/xray/config.json`

## 2. Установка дополнительных пакетов
Установите необходимые пакеты:
```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

## 3. Настройка DNS
Домен третьего уровня `app.r2bny.com` должен указывать на IP сервера (A-запись).

## 4. Выпуск SSL-сертификата
Получите сертификат Let's Encrypt:
```bash
sudo certbot --nginx -d app.r2bny.com
```
Certbot автоматически пропишет SSL в nginx.

## 5. Настройка nginx
Создайте конфигурационный файл:
```bash
sudo nano /etc/nginx/sites-available/app.r2bny.com
```
Конфигурация:
```conf
server {
    listen 443 ssl http2;
    server_name app.r2bny.com;

    ssl_certificate /etc/letsencrypt/live/app.r2bny.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.r2bny.com/privkey.pem;

    location / {
        root /var/www/html;
        index index.html;
    }

    location /api/v1/status {
        default_type application/json;
        return 200 '{"status":"ok"}';
    }

    location /api/v1/connect {
        proxy_pass http://127.0.0.1:10085;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Connection "";

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
  }
}
```
Активируйте конфигурационный файл Nginx:
``` bash
sudo ln -s /etc/nginx/sites-available/app.r2bny.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 6. Конфигурация Xray (XHTTP)
Сгенерируйте UUID который будет использоваться для аутентификации клиента:
``` bash
cat /proc/sys/kernel/random/uuid
```
Откройте в редакторе конфигурационный файл:
``` bash
sudo nano /usr/local/etc/xray/config.json
```
Замените <UUID> на сгенерированный UUID и укажите свои параметры при необходимости:
``` json
{
  "inbounds": [
    {
      "port": 10085,
      "listen": "127.0.0.1",
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<UUID>",
            "level": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "none",
        "xhttpSettings": {
          "path": "/api/v1/connect",
          "host": "app.r2bny.com",
          "mode": "auto"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```
Если необходимо добавить несколько пользователей, используйте расширенный вариант конфигурации ниже. Для каждого пользователя укажите  отдельный UUID:
``` json
{
  "log": {
    "loglevel": "warning"
  },

  "inbounds": [
    {
      "port": 10085,
      "listen": "127.0.0.1",
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<UUID-1>",
            "level": 0,
            "email": "user1"
          },
          {
            "id": "<UUID-2>",
            "level": 0,
            "email": "user2"
          },
          {
            "id": "<UUID-3>",
            "level": 0,
            "email": "user3"
          },
          {
            "id": "<UUID-4>",
            "level": 0,
            "email": "user4"
          },
          {
            "id": "<UUID-5>",
            "level": 0,
            "email": "user5"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "none",
        "xhttpSettings": {
          "path": "/api/v1/connect",
          "host": "app.r2bny.com",
          "mode": "auto"
        }
      }
    }
  ],

  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```

## 9. Перезапуск и проверка
Выполните перезапуск сервисов:
``` bash
sudo systemctl restart xray
sudo systemctl restart nginx
```
Просмотр логов Xray в реальном времени:
``` bash
journalctl -u xray -f
```

## 8. Клиентская ссылка
Замените <UUID> на UUID пользователя:
```vless
vless://<UUID>@app.r2bny.com:443?encryption=none&security=tls&type=xhttp&path=/api/v1/connect#XRAY XHTTP
```

## 10. Проверка работоспособности
Если конфигурация настроена корректно, должна отображаться стандартная страница nginx. Откройте в браузере: https://app.r2bny.com/
