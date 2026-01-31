# Установка и настройка strongSwan IKEv2/IPsec на Debian GNU/Linux
Замените `tun.r2bny.com` на ваше доменное.
## 1. Установка необходимых пакетов
Обновите список пакетов и установите необходимые утилиты, включая StrongSwan и утилиты для работы с сертификатами:
```bash
sudo apt update
sudo apt install strongswan strongswan-pki libstrongswan-standard-plugins libstrongswan-extra-plugins libcharon-extra-plugins libcharon-extauth-plugins
```

## 2. Настройка сертификатов
### 2.1 Генерация SSL-сертификатов с Let's Encrypt
Установите certbot и получите сертификат:
```bash
sudo apt install certbot -y
sudo certbot certonly --standalone --agree-tos --no-eff-email --email e-mail@r2bny.com -d tun.r2bny.com
```

### 2.2 Настройка SSL-сертификатов
Создайте символические ссылки на сертификаты Let's Encrypt в окружение strongSwan:
```bash
sudo ln -sf /etc/letsencrypt/live/tun.r2bny.ru/fullchain.pem /etc/swanctl/x509/cert.pem
sudo ln -sf /etc/letsencrypt/live/tun.r2bny.ru/privkey.pem /etc/swanctl/private/privkey.pem
sudo ln -sf /etc/letsencrypt/live/tun.r2bny.ru/chain.pem /etc/swanctl/x509ca/chain.pem
```
Убедитесь, что права доступа к приватному ключу правильные:
```bash
sudo chmod 600 /etc/swanctl/private/privkey.pem
```

## 3. Настройка StrongSwan
### 3.1 Конфигурация StrongSwan
Создайте или отредактируйте файл конфигурации /etc/swanctl/conf.d/tun.r2bny.com.conf:
```bash
sudo nano /etc/swanctl/conf.d/tun.r2bny.com.conf
```
```conf
connections {
    eap-ikev2 {
        local {
            auth = pubkey
            certs = /etc/swanctl/x509/cert.pem
            id = tun.r2bny.com
        }
        remote {
            auth = eap-mschapv2
            eap_id = %any
        }
        children {
            eap-sa {
                local_ts  = 0.0.0.0/0
                start_action = start
        }
    }
    unique = never
    version = 2
    dpd_delay = 30s
    send_cert = always
    proposals=aes128-aes192-aes256-sha1-sha256-sha384-modp1024,default
    pools = ipv4-pool
    }
}

pools {
    ipv4-pool {
        addrs = 10.0.0.0/25
        dns = 8.8.8.8
    }
}

secrets {
    eap-user1 {
        id = user1
        secret = "P@ssw0rdUser1"
    }
    # eap-user2 {
    #    id = user2
    #    secret = "P@ssw0rdUser2"
    # }    
}
```
### 3.1 Конфигурация AppArmor 
Отредактируйте файл конфигурации AppArmor для утилиты swanctl (которая загружает конфигурацию strongSwan):
```bash
sudo nano /etc/apparmor.d/local/usr.sbin.swanctl
```
Добавьте строку:
```bash
/etc/letsencrypt/** r,
/etc/swanctl/** rwk,
```
После редактирования файлов перезагрузите профили AppArmor:
```bash
sudo systemctl reload apparmor
```
Проверьте, что профили загружены корректно и нет ошибок:
```bash
sudo apparmor_status
```

## 4. Настройка сети
### 4.1 Включение форвардинга пакетов
Активируйте пересылку пакетов на уровне ядра, в директории /etc/sysctl.d/ для отдельных настроек создав файл '99-tun.r2bny.com.conf':
```bash
sudo nano /etc/sysctl.d/99-tun.r2bny.com.conf
```
Добавьте базовое содержимое:
```conf
# Enable IP forwarding
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Additional security settings (optional)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1
```
Примените изменения:
```bash
sudo sysctl --system
```
### 4.2 Настройка правил nftables 
Отредактируйте конфигурационный файл nftables:
```bash
sudo nano /etc/nftables.conf
```
Вставьте эти строки в конец файла, заменив eth0 на ваш внешний интерфейс (например, ens3):
```bash
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        
        # Добавьте эту строку для MASQUERADE VPN-трафика
        oifname "eth0" ip saddr 10.0.0.0/24 masquerade
    }
}
```
Если у вас "жёсткие" правила вставьте эти строки в соответствующие разделы, сохраняя существующие правила, заменив eth0 на ваш внешний интерфейс (например, ens3):
```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter;
        
        # Добавьте эти строки для разрешения VPN-трафика
        udp dport {500, 4500} accept
        ip protocol esp accept
    }
    
    chain forward {
        type filter hook forward priority filter;
        
        # Добавьте эти строки для пересылки VPN-трафика
        ip saddr 10.0.0.0/24 oifname "eth0" accept
        iifname "eth0" ip daddr 10.0.0.0/24 ct state established,related accept
    }
    
    chain output {
        type filter hook output priority filter;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        
        # Добавьте эту строку для MASQUERADE VPN-трафика
        oifname "eth0" ip saddr 10.0.0.0/24 masquerade
    }
}
```
Включите и запустите nftables:
```bash
sudo systemctl enable nftables
sudo systemctl start nftables
```
Примените конфигурацию:
```bash
sudo nft -f /etc/nftables.conf
```
Просмотрите текущие правила:
```bash
sudo nft list ruleset
```

## 5. Запуск StrongSwan
Загрузите конфигурацию в StrongSwan:
```bash
sudo swanctl --load-all
```
Включите автозагрузку StrongSwan и перезагрузите сервер для применения всех изменений:
```bash
sudo systemctl enable strongswan
sudo reboot
```
Проверьте статус сервиса:
```bash
sudo systemctl status strongswan
```
Просмотрите список загруженных подключений:
```bash
sudo swanctl --list-conns
```
Проверьте пулы адресов:
```bash
sudo swanctl --list-pools
```

## 6. Настройка VPN-клиентов (IKEv2 EAP)
StrongSwan настроен на использование протокола **IKEv2 с EAP-MSCHAPv2**, который поддерживается большинством современных операционных систем без установки дополнительного ПО.
### 6.1 Общие параметры подключения
Эти параметры одинаковы для всех клиентов:
| Параметр           | Значение                  |
|--------------------|---------------------------|
| Тип VPN            | IKEv2                     |
| Сервер             | `tun.r2bny.com`           |
| Тип аутентификации | Имя пользователя + пароль |
| Имя пользователя   | `user1`                   |
| Пароль             | `P@ssw0rdUser1`           |
| Сертификат сервера | Проверяется автоматически |
### 6.2 Windows 10 / 11
1. **Параметры → Сеть и Интернет → VPN**
2. Нажмите **Добавить VPN-подключение**
3. Заполните поля:
   - Поставщик VPN: **Windows (встроенный)**
   - Имя подключения: `IKEv2 VPN`
   - Имя сервера: `tun.r2bny.com`
   - Тип VPN: **IKEv2**
   - Тип входа: **Имя пользователя и пароль**
4. Сохраните профиль и подключитесь
**Рекомендуется**:  
В свойствах подключения → **Безопасность**:
- Шифрование: *Требовать шифрование*
- Аутентификация: *EAP (MSCHAPv2)*
### 6.3 Android
1. **Настройки → Сеть и Интернет → VPN**
2. **Добавить VPN**
3. Тип: **IKEv2 EAP**
4. Укажите:
   - Сервер: `tun.r2bny.com`
   - Идентификатор IPSec: `tun.r2bny.com`
   - Имя пользователя и пароль
5. Сохраните и подключитесь
Работает со стандартным VPN-клиентом Android.
### 6.4 iOS
1. **Настройки → Основные → VPN и управление устройством → VPN**
2. **Добавить конфигурацию VPN**
3. Тип: **IKEv2**
4. Параметры:
   - Сервер: `tun.r2bny.com`
   - Remote ID: `tun.r2bny.com`
   - Local ID: *(пусто)*
   - Аутентификация: **Имя пользователя**
5. Введите логин и пароль
6. Сохраните и подключитесь
### 6.5 macOS
1. **Системные настройки → Сеть**
2. **Добавить интерфейс → VPN**
3. Тип VPN: **IKEv2**
4. Имя сервера: `tun.r2bny.com`
5. Remote ID: `tun.r2bny.com`
6. Аутентификация: **Имя пользователя**
7. Введите логин и пароль
8. Сохраните и подключитесь
### 6.6 Проверка подключения на сервере
После подключения клиента выполните:
```bash
sudo swanctl --list-sas
```

## 7. Проверка и автоматическое обновление сертификата Let’s Encrypt
Сертификаты Let’s Encrypt действуют **90 дней** и должны регулярно обновляться.  
При корректной настройке обновление происходит **автоматически**, без участия администратора.
### 7.1 Проверка автообновления certbot
Убедитесь, что systemd-таймер certbot активен:
```bash
systemctl status certbot.timer
```
Ожидаемый статус:
```
Active: active (waiting)
```
Если таймер не активен, включите его:
```bash
sudo systemctl enable --now certbot.timer
```
### 7.2 Проверка срока действия текущего сертификата
Проверьте значение `notAfter` — это дата окончания действия сертификата:
```bash
sudo openssl x509 -in /etc/letsencrypt/live/tun.r2bny.com/fullchain.pem -noout -dates
```
### 7.3 Тестовое обновление сертификата (dry-run)
Для проверки механизма обновления выполните безопасный тест:
```bash
sudo certbot renew --dry-run
```
При успешной проверке вы увидите:
```
Congratulations, all simulated renewals succeeded!
```
Если возникает ошибка — **автообновление работать не будет**.
### 7.4 Автоматическая перезагрузка сертификатов в strongSwan
StrongSwan **не перечитывает сертификаты автоматически** после их обновления.  
Необходимо добавить deploy-hook для certbot.
Создайте файл:
```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/strongswan.sh
```
Содержимое:
```bash
#!/bin/bash
/usr/sbin/swanctl --load-creds
/usr/sbin/swanctl --load-conns
```
Сделайте файл исполняемым:
```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/strongswan.sh
```
Теперь после каждого автоматического обновления:
- certbot обновит файлы в `/etc/letsencrypt/live/`
- strongSwan автоматически перечитает сертификаты и ключи
- VPN-подключения продолжат работать без перезапуска сервиса
### 7.5 Проверка, что strongSwan использует актуальный сертификат
```bash
sudo swanctl --list-certs
```
В выводе должен присутствовать сертификат с:
- `subject: CN=tun.r2bny.com`
- `issuer: Let's Encrypt`
### 7.6 Проверка работы VPN после обновления
1. Подключитесь VPN-клиентом
2. На сервере выполните:
```bash
sudo journalctl -u strongswan -f
```
3. Переподключите клиента
Если нет ошибок типа:
- `certificate expired`
- `no trusted RSA public key`
то сертификат корректно обновлён и используется.
### 7.7 Важно (standalone-режим certbot)
Сертификат был получен с использованием:
```bash
certbot certonly --standalone
```
Это означает, что при обновлении:
- порт **80 должен быть свободен**
- веб-сервер (nginx/apache) не должен работать
Если порт 80 занят — рекомендуется использовать `--webroot` или DNS-challenge.
