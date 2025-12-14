# Установка и настройка MariaDB на Debian GNU/Linux

## 1. Обновление системы
Выполните обновление системы:
```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Установка MariaDB
MariaDB входит в официальные репозитории Debian, поэтому достаточно одной команды:
```bash
sudo apt install mariadb-server mariadb-client -y
```
После установки включаем и запускаем службу:
```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```
Проверим статус службы:
```bash
sudo systemctl status mariadb
```
> Если статус `active (running)` — сервер работает корректно.

## 3. Первичная настройка безопасности
Запустите встроенный скрипт безопасности:
```bash
sudo mariadb-secure-installation
```
Рекомендуемые ответы:
```
Enter current password for root (enter for none): [нажмите Enter]
Switch to unix_socket authentication [Y/n]: Y
Change the root password? [Y/n]: Y
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]: Y
Reload privilege tables now? [Y/n]: Y
```

## 4. Подключение к MariaDB, создание базы данных и пользователя
Подключение без пароля (через `unix_socket`):
```bash
sudo mysql
```
Подключение с паролем:
```bash
mysql -u root -p
```
В консоли MariaDB выполните команды:
```sql
CREATE DATABASE dbname
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'dbuser'@'localhost'
  IDENTIFIED BY 'strongpassword';

GRANT ALL PRIVILEGES ON dbname.*
  TO 'dbuser'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```
## 5. Работа с дампом из командной строки
### Создание дампа базы данных
```bash
mysqldump -y -f -q \
  --default-character-set=binary \
  --create-options \
  --single-transaction \
  --skip-extended-insert \
  --add-drop-table \
  -h dbhost -u dbuser -pdbpassword dbname > dump.sql
```
Где:

- **dbhost** — адрес сервера баз данных
- **dbuser** — имя пользователя MariaDB/MySQL
- **dbpassword** — пароль пользователя (указывается слитно с `-p`)
- **dbname** — имя базы данных
- **dump.sql** — имя файла дампа (создаётся в текущем каталоге)
### Импорт дампа
```bash
mysql -h dbhost -u dbuser -pdbpassword dbname < dump.sql
```
### Импорт сжатого дампа (`.sql.gz`)
```bash
gunzip < /path/to/r2bny.sql.gz | mysql -u dbuser -p dbname
```

## 6. Проверка работы MariaDB
Проверка списка баз данных:
```bash
mysql -u dbuser -p -e "SHOW DATABASES;"
```
Проверка версии сервера:
```bash
mysql -V
```

## 7. Разрешение внешнего доступа (опционально)
Откройте конфигурацию MariaDB:
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
Найдите строку 'bind-address = 127.0.0.1' и измените на:
```
bind-address = 0.0.0.0
```
Перезапустите сервер:
```bash
sudo systemctl restart mariadb
```
Разрешите пользователю подключаться извне:
```sql
GRANT ALL PRIVILEGES ON wordpress.*
  TO 'dbuser'@'%'
  IDENTIFIED BY 'strongpassword';

FLUSH PRIVILEGES;
```
Проверьте подключение:
```bash
mysqladmin -u wpuser -p version
```

