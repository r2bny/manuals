# Установка и настройка Apache Superset на Debian GNU/Linux

## 1. Обновление системы
Перед установкой рекомендуется обновить все пакеты системы:
```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Установка необходимых зависимостей
Superset и его зависимости требуют ряда системных пакетов:
```bash
sudo apt install -y build-essential libssl-dev libffi-dev python3-dev python3-pip python3-venv libpq-dev libsasl2-dev libldap2-dev default-libmysqlclient-dev git curl nginx certbot python3-certbot-nginx postgresql postgresql-contrib
```

## 3. Настройка PostgreSQL
Запуск и включение в автозагрузку:
```bash
sudo systemctl enable --now postgresql
```
Создание базы данных и пользователя:
```bash
sudo -u postgres psql
```
```sql
CREATE DATABASE superset ENCODING 'UTF8';
CREATE USER superset WITH PASSWORD 'P@ssw0rd';
GRANT ALL PRIVILEGES ON DATABASE superset TO superset;
ALTER DATABASE superset OWNER TO superset;
\q
```

## 4. Создание виртуального окружения Python
Создаём рабочую директорию и виртуальное окружение:
```bash
mkdir /opt/superset
cd /opt/superset
python3 -m venv .venv
source .venv/bin/activate
```
Обновляем pip и производим установку пакета Superset и зависимостей:
```bash
pip install --upgrade pip setuptools wheel
pip install apache-superset psycopg2-binary
```

## 5. Настройка конфигурационного файла
Создайте файл `superset_config.py` в директории `/opt/superset`:
```python
# ~/superset/superset_config.py
SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://superset_user:superset_pass@localhost/superset_db'
ROW_LIMIT = 5000
SUPERSET_WEBSERVER_TIMEOUT = 60
```
Экспортируйте переменную окружения:
```bash
export SUPERSET_CONFIG_PATH=~/superset/superset_config.py
```

## 6. Инициализация Superset
Выполните начальную настройку базы данных Superset:
```bash
superset db upgrade
```
Создайте администратора:
```bash
superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname User \
  --email admin@superset.local \
  --password admin
```
Загрузите начальные метаданные:
```bash
superset init
```

## 7. Запуск Superset
В режиме разработки:
```bash
superset run -h 0.0.0.0 -p 8088 --with-threads
```
В продакшене — через Gunicorn:
```bash
pip install gunicorn gevent
gunicorn -w 4 -k gevent -b 0.0.0.0:8088 "superset.app:create_app()"
```

## 8. Создание systemd-сервиса
Создайте unit-файл:
```bash
sudo nano /etc/systemd/system/superset.service
```
Содержимое:
```ini
[Unit]
Description=Apache Superset
After=network.target

[Service]
User=your_user
Group=your_user
WorkingDirectory=/home/your_user/superset
Environment="SUPERSET_CONFIG_PATH=/home/your_user/superset/superset_config.py"
ExecStart=/home/your_user/superset/venv/bin/gunicorn -w 4 -k gevent -b 0.0.0.0:8088 "superset.app:create_app()"
Restart=always

[Install]
WantedBy=multi-user.target
```
Активируйте сервис:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now superset
```
## 9. Настройка nginx
Создай новый конфигурационный файл:
```
sudo nano /etc/nginx/sites-available/superset.r2bny.com.conf
```
Вставь следующий конфиг (замениnt superset.r2bny.com на свой домен):
```
server {
    listen 80;
    server_name superset.r2bny.com;

    location / {
        proxy_pass http://localhost:8088;
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
sudo ln -s /etc/nginx/sites-available/superset.r2bny.com.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
Получим сертификат HTTPS от Let's Encrypt:
```
sudo certbot --nginx -d superset.r2bny.com
```

## 10. Проверка и вход
Откройте браузер:
- URL: http://superset.r2bny.com
- Логин: `admin`
- Пароль: `admin` (или ваш)
