# Установка и настройка n8n на Debian GNU/Linux
**Важно:** В данной инструкции для наглядности в примерах и конфигурационных файлах **используется домен `r2bny.com`**. Перед применением обязательно **замените его на ваш собственный домен** во всех упоминаниях в конфигурациях сервисов, путях к файлам и директориях, ссылках и сертификатах.

## 1. Обновление системы
Перед установкой рекомендуется обновить все пакеты системы:
```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Установка необходимых зависимостей и n8n
Произведите установку ряда системных пакетов:
```bash
sudo apt install -y curl gnupg build-essential git nodejs npm nginx certbot python3-certbot-nginx postgresql postgresql-contrib
```
Проверка установки:
```bash
node -v
npm -v
```
Установка N8N глобально через npm:
```bash
sudo npm install -g n8n 
```
Проверьте версию n8n:
```bash
n8n --version
```

## 3. Настройка PostgreSQL
Запуcтите сервис PostgreSQL:
```
sudo systemctl start postgresql.service
```
Войдите под пользователем postgres, привязываемый к роли PostgreSQL:
```
sudo su - postgres
```
Войдите в интерактивный терминал PostgreSQL:
```
psql
```
Ввыполните команды в интерактивном терминале PostgreSQL:
```
CREATE DATABASE n8n ENCODING 'UTF8';
CREATE USER n8n WITH PASSWORD 'P@ssw0rd';
GRANT ALL PRIVILEGES ON DATABASE n8n TO n8n;
ALTER DATABASE n8n OWNER TO n8n;
\q
```

## 3. Настройка nginx
Создай новый конфигурационный файл:
```
sudo nano /etc/nginx/sites-available/n8n.r2bny.com.conf
```
Вставь следующий конфиг (замени n8n.r2bny.com на свой домен):
```
server {
    listen 80;
    server_name n8n.r2bny.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_http_version 1.1;

        client_max_body_size 20M;
    }
}
```
Активируем сайт и проверим конфигурацию:
```
sudo ln -s /etc/nginx/sites-available/n8n.r2bny.com.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
Получим сертификат HTTPS от Let's Encrypt:
```
sudo certbot --nginx -d n8n.r2bny.com
```

## 3. Настройка окружения
Для безопасного запуска сервиса n8n создайте отдельного системного пользователя без доступа к оболочке:
```bash
sudo useradd -r -M -d /opt/n8n -s /usr/sbin/nologin n8n
```
Чтобы убедиться, что пользователь создан, выполните следующую команду:
```bash
getent passwd n8n
```
Создайте каталоги для корректной работы n8n:
```bash
sudo mkdir -p /opt/n8n/
sudo mkdir -p /var/log/n8n
```
Создайте конфигурационный файл `.env`:
```bash
sudo nano /opt/n8n/.env
```
Пример содержимого:
```
N8N_HOST=n8n.r2bny.com
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.r2bny.com/

N8N_USER_FOLDER=/opt/n8n

DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=127.0.0.1
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=P@ssw0rd
DB_POSTGRESDB_SCHEMA=public

N8N_LOG_OUTPUT=/var/log/n8n/n8n.log
N8N_LOG_LEVEL=info
```
```
sudo chown -R n8n:n8n /opt/n8n
sudo chmod 600 /opt/n8n/.env
```

## 4. Создание systemd-сервиса
Создайте unit-файл и введите содержимое:
```bash
sudo nano /etc/systemd/system/n8n.service
```
Содержимое:
```ini
[Unit]
Description=n8n - Workflow Automation
After=network.target

[Service]
Type=simple
User=n8n
EnvironmentFile=/opt/n8n/.env
WorkingDirectory=/opt/n8n/
ExecStart=/usr/local/bin/n8n
Restart=always
RestartSec=10
LimitNOFILE=65535
StandardOutput=append:/var/log/n8n/n8n.log
StandardError=append:/var/log/n8n/n8n.log

[Install]
WantedBy=multi-user.target
```

## 5. Запуск сервиса
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now n8n
```
Проверка:
```bash
systemctl status n8n
```

## 6. Обновление n8n
Остановка сервиса:
```bash
sudo systemctl stop n8n
```
Обновление Node.js до версии ≥ 20 (требуется для новых версий n8n):
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```
Проверка:
```bash
node -v
npm -v
```
Обновление n8n через npm:
```bash
sudo npm install -g n8n
```
Проверка версии:
```bash
n8n --version
```
Запуск:
```bash
sudo systemctl start n8n
```

## 7. Резервное копирование
Конфигурация:
```bash
sudo cp -a /opt/n8n/.n8n /opt/n8n/.n8n.bak.$(date +%F)
```
База данных:
```bash
sudo -u postgres pg_dump n8n > ~/n8n_backup_$(date +%F).sql
```

## 8. Логи и отладка
```bash
journalctl -u n8n -e
```
