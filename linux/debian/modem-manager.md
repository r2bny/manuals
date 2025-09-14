# Установка и настройка ModemManager на Debian GNU/Linux

## 1. Установка ModemManager
Обновляем список пакетов и устанавливаем ModemManager:
```bash
sudo apt update
sudo apt install modemmanager
```

## 2. Запуск и автозагрузка
Запускаем сервис, включаем автозапуск и проверяем его состояние:
```bash
sudo systemctl start ModemManager
sudo systemctl enable ModemManager
systemctl status ModemManager
```

## 3. Проверка модема
Показываем список обнаруженных модемов:
```bash
mmcli -L
```
Пример:
```
/org/freedesktop/ModemManager1/Modem/0 [Quectel] EG25-G
```
Проверяем, поддерживает ли модем работу с SMS:
```bash
mmcli -m 0 | grep -i messaging
```

## 4. Приём SMS вручную
Получаем список входящих сообщений:
```bash
mmcli -m 0 --messaging-list-sms
```
Пример:
```
SMS list:
    /org/freedesktop/ModemManager1/SMS/12 (received)
```
Читаем конкретное сообщение:
```bash
mmcli -s 12
```
Удаляем сообщение после прочтения:
```bash
sudo mmcli -s 12 --delete
```

## 5. Автоматический приём и логирование SMS
Создаём скрипт для автоматического опроса модема:
```bash
sudo nano /usr/local/bin/sms-listener.sh
```
Вставляем код скрипта:
```bash
#!/bin/bash
MODEM=0
LOG_FILE="/var/log/sms.log"

while true; do
    LIST=$(mmcli -m $MODEM --messaging-list-sms | grep received | awk '{print $1}')
    for SMS_PATH in $LIST; do
        ID=$(echo $SMS_PATH | awk -F'/' '{print $NF}')
        {
            echo "=== $(date '+%Y-%m-%d %H:%M:%S') ==="
            mmcli -s $ID
            echo
        } >> "$LOG_FILE"
        mmcli -s $ID --delete
    done
    sleep 10
done
```
Делаем скрипт исполняемым:
```bash
sudo chmod +x /usr/local/bin/sms-listener.sh
```

## 6. Настройка прав для логов
Создаём пустой лог-файл и даём права на запись всем пользователям:
```bash
sudo touch /var/log/sms.log
sudo chmod 666 /var/log/sms.log
```

## 7. Автозапуск через systemd
Создаём unit-файл сервиса:
```bash
sudo nano /etc/systemd/system/sms-listener.service
```
Вставляем конфигурацию сервиса:
```ini
[Unit]
Description=SMS Listener via ModemManager
After=network.target ModemManager.service

[Service]
ExecStart=/usr/local/bin/sms-listener.sh
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```
Перечитываем конфигурацию systemd:
```bash
sudo systemctl daemon-reload
```
Включаем автозапуск сервиса:
```bash
sudo systemctl enable sms-listener.service
```
Запускаем сервис:
```bash
sudo systemctl start sms-listener.service
```
Проверяем его состояние:
```bash
systemctl status sms-listener.service
```

## 8. Где искать полученные SMS
Все входящие сообщения сохраняются в файл:
```
/var/log/sms.log
```
Пример содержимого:
```
=== 2025-08-12 14:32:10 ===
SMS '/org/freedesktop/ModemManager1/SMS/12'
  Status  : received
  Number  : +79991234567
  Text    : Код подтверждения: 1234
```
Примечания:
- Если у вас несколько модемов, замените `MODEM=0` в скрипте на нужный ID из `mmcli -L`.
- Интервал опроса (`sleep 10`) можно изменить в скрипте.
- Чтобы смотреть лог в реальном времени:
```bash
tail -f /var/log/sms.log
```
