
# Инструкция по установке PostgreSQL на Debian GNU/Linux

## 1. Обновление списка пакетов
Перед установкой обновите список пакетов на вашем сервере:
```bash
sudo apt update
```

## 2. Установка PostgreSQL
Установите PostgreSQL из официальных репозиториев Debian:
```bash
sudo apt install postgresql postgresql-contrib
```
- `postgresql` — основной пакет PostgreSQL.
- `postgresql-contrib` — дополнительный пакет с утилитами и расширениями.

## 3. Проверка статуса службы
После установки PostgreSQL, служба должна запуститься автоматически. Вы можете проверить её статус с помощью следующей команды:
```bash
sudo systemctl status postgresql
```

Если служба не запустилась, вы можете запустить её вручную:
```bash
sudo systemctl start postgresql
```

Чтобы настроить автоматический запуск PostgreSQL при старте системы:
```bash
sudo systemctl enable postgresql
```

## 4. Доступ к PostgreSQL
По умолчанию, PostgreSQL создает пользователя `postgres`, который имеет доступ к базе данных. Чтобы войти в PostgreSQL, выполните команду:
```bash
sudo -i -u postgres
```
Теперь вы можете использовать команду `psql` для работы с PostgreSQL:
```bash
psql
```
Создание базы данных и пользователя:
```sql
CREATE DATABASE имя_базы_данных ENCODING 'UTF8';
CREATE USER имя_пользователя WITH PASSWORD 'P@ssw0rd';
GRANT ALL PRIVILEGES ON DATABASE имя_базы_данных TO имя_пользователя;
ALTER DATABASE имя_базы_данных OWNER TO имя_пользователя;
```

Чтобы выйти из psql, выполните команду:
```bash
\q
```

## 5. Создание нового пользователя и базы данных
Для создания нового пользователя и базы данных выполните следующие команды:

1. Создайте пользователя:
   ```bash
   createuser --interactive
   ```
   Команда запросит имя нового пользователя и предложит другие настройки.

2. Создайте базу данных:
   ```bash
   createdb имя_базы_данных
   ```

3. Подключитесь к базе данных:
   ```bash
   psql имя_базы_данных
   ```

## 6. Настройка удалённого доступа (по желанию)
Если вы хотите настроить удалённый доступ к PostgreSQL, отредактируйте два файла:

1. **Разрешить подключения в `postgresql.conf`:**
   Откройте файл конфигурации PostgreSQL:
   ```bash
   sudo nano /etc/postgresql/17/main/postgresql.conf
   ```
   Найдите строку:
   ```bash
   #listen_addresses = 'localhost'
   ```
   и замените её на:
   ```bash
   listen_addresses = '*'
   ```
   Это позволит принимать подключения с любых IP-адресов.

2. **Настройка файла `pg_hba.conf`:**
   Откройте файл:
   ```bash
   sudo nano /etc/postgresql/<версия>/main/pg_hba.conf
   ```
   Добавьте строку для разрешения подключения от клиентов:
   ```bash
   host    all             all             0.0.0.0/0            md5
   ```
   Это позволит всем клиентам подключаться к PostgreSQL через сеть.

3. Перезапустите PostgreSQL:
   ```bash
   sudo systemctl restart postgresql
   ```

## 7. Управление PostgreSQL
- Для остановки службы:
  ```bash
  sudo systemctl stop postgresql
  ```

- Для перезапуска службы:
  ```bash
  sudo systemctl restart postgresql
  ```

## 8. Дополнительные утилиты
- Для установки клиента `psql` на удалённый сервер:
  ```bash
  sudo apt install postgresql-client
  ```
