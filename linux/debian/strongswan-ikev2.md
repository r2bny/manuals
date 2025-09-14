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

## 6. Настройка клиентов

