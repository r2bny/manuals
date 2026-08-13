# Установка и настройка vsftpd на Debian GNU/Linux
**Важно:** В данной инструкции для наглядности в примерах и конфигурационных файлах **используется домен `r2bny.com`**. Перед применением обязательно **замените его на ваш собственный домен** во всех упоминаниях в конфигурациях сервисов, путях к файлам и директориях, ссылках и сертификатах.

## 1️. Установка vsftpd
Сначала обновляем пакеты и устанавливаем vsftpd:
```bash
sudo apt update
sudo apt install vsftpd -y
```
Проверяем, что сервис запущен:
```bash
sudo systemctl status vsftpd
```
- Статус должен быть `active (running)`.

## 2️. Создание общего каталога `/datastore` и группы
Создаём общую папку для записи всеми пользователями группы:
```bash
sudo mkdir -p /datastore
sudo groupadd -f datastore
sudo chown root:datastore /datastore
sudo chmod 2775 /datastore
```

## 3️. Создание FTP-пользователей
Создаём пользователя для FTP:
```bash
sudo adduser ftpuser
```
Ограничьте home-каталог пользователя:
```bash
sudo chown root:root /home/ftpuser
sudo chmod 755 /home/ftpuser
```
Добавляем пользователя в группу datastore:
```bash
sudo usermod -aG datastore ftpuser
```
Создаём личную папку пользователя:
```bash
sudo mkdir -p /home/ftpuser/private
sudo chown ftpuser:ftpuser /home/ftpuser/private
sudo chmod 700 /home/ftpuser/private
```
Создаём ссылку на общую папку:
```bash
sudo ln -s /datastore /home/ftpuser/share
```
Проверяем группы пользователя:
```bash
groups ftpuser
```
- Должно быть: `ftpuser : ftpuser datastore`.

## 4️. Настройка vsftpd
Редактируем конфиг для включения локальных пользователей, записи, chroot и пассивного режима:
```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.backup
sudo nano /etc/vsftpd.conf
```
Пример рабочей конфигурации:
```conf
# Основные настройки
listen=YES
listen_ipv6=NO

anonymous_enable=NO
local_enable=YES
write_enable=YES

dirmessage_enable=YES
use_localtime=YES

# Права файлов
local_umask=002
file_open_mode=0664

# Изоляция пользователей
# chroot_local_user=YES

# Безопасность
hide_ids=YES
ls_recurse_enable=NO

secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd

guest_enable=NO

# Пассивный режим
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000

# Укажите ваш публичный IP
pasv_address=46.16.15.60

# Логирование
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log

log_ftp_protocol=YES
vsftpd_log_file=/var/log/vsftpd.log
```
- **Важно:** `pasv_address` должен быть числовым IP или публичным IP, на который указывает домен. 
Перезапускаем сервис:
```bash
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
```

## 5️. Настройка фаервола
Открываем порты FTP и пассивного режима на сервере:
```bash
sudo ufw allow 21/tcp
sudo ufw allow 20/tcp
sudo ufw allow 30000:31000/tcp
sudo ufw reload
```
- Эти порты нужны для FTP и пассивного режима.

## 6️. Проверка подключения
Проверяем работу FTP-сервера локально:
```bash
ftp 127.0.0.1
```
По внутреннему IP сети:
```bash
ftp 192.168.0.10
```
По публичному IP или домену:
```bash
ftp ftp.r2bny.com
```
- Логин: `ftpuser`
- Пароль: заданный при создании пользователя

