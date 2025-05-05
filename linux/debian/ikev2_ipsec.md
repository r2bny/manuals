# Установка и настройка IKEv2/IPsec на Debian GNU/Linux

## 1. Установка необходимых пакетов
Обновите список пакетов и установите необходимые утилиты, включая StrongSwan и утилиты для работы с сертификатами:
```bash
sudo apt update
sudo apt install strongswan strongswan-pki libstrongswan-standard-plugins libstrongswan-extra-plugins libcharon-extra-plugins libcharon-extauth-plugins libtss2-tcti-tabrmd0
```

## 2. Настройка сертификатов
### 2.1 Создание инфраструктуры для сертификатов
Создадим директории для хранения ключей и сертификатов:
```bash
mkdir -p /usr/local/lib/pki/{cacerts,certs,private}
chmod 700 /usr/local/lib/pki/
```
### 2.2 Создание корневого сертификата (CA)
Теперь создадим корневой сертификат:
```bash
ipsec pki --gen --type rsa --size 4096 --outform pem > /usr/local/lib/pki/private/ca.pem
pki --self --ca --lifetime 3650 --in /usr/local/lib/pki/private/ca.pem --type rsa --dn "CN=R2BNY CA" --outform pem > /usr/local/lib/pki/cacerts/ca.pem
```
Флаг --lifetime 3650 используется для гарантии того, что корневой сертификат центра сертификации будет действителен в течение 10 лет. Корневой сертификат для центра обычно не меняется, поскольку его придется перераспределять на каждый сервер и клиент, которые полагаются на него, поэтому 10 лет — это безопасное значение срока действия по умолчанию.
Вы можете изменить значение отличительного имени (DN) на другое, если хотите. Общее имя (поле CN) здесь является просто индикатором, поэтому оно не должно соответствовать чему-либо в вашей инфраструктуре.
### 2.3 Создание серверного сертификата
Теперь создадим ключ и сертификат для VPN сервера:
```bash
pki --gen --type rsa --size 4096 --outform pem > /usr/local/lib/pki/private/server.pem
pki --pub --in /usr/local/lib/pki/private/server.pem --type rsa | pki --issue --lifetime 1825 --cacert /usr/local/lib/pki/cacerts/ca.pem --cakey /usr/local/lib/pki/private/ca.pem --dn "CN=tun.r2bny.ru" --san tun.r2bny.ru --flag serverAuth --flag ikeIntermediate --outform pem > /usr/local/lib/pki/certs/server.pem
 ```
Измените поля «Общее имя» (CN) и «Альтернативное имя субъекта» (SAN) на DNS-имя или IP-адрес вашего VPN-сервера.
Параметр --flag serverAuth используется для указания того, что сертификат будет явно использоваться для проверки подлинности сервера до того, как будет установлен зашифрованный туннель. Этот --flag ikeIntermediate параметр используется для поддержки старых клиентов macOS.
### 2.4 Перемещение сертификатов
Переместим ключи и сертификаты в директории StrongSwan:
```bash
sudo cp -r /usr/local/lib/pki/* /etc/ipsec.d/
```
На этом шаге мы создали пару сертификатов, которая будет использоваться для защиты связи между клиентом и сервером. Мы также подписали сертификаты ключом ЦС, поэтому клиент сможет проверить подлинность VPN-сервера с помощью сертификата ЦС. Когда все эти сертификаты готовы, мы перейдем к настройке программного обеспечения.

## 3. Настройка StrongSwan
### 3.1 Конфигурация IPsec
Откройте файл `/etc/ipsec.conf` для редактирования:
```bash
sudo nano /etc/ipsec.conf
```
Замените `tun.r2bny.com` на ваше доменное имя или IP-адрес. Полный файл конфигурации должен выглядеть так:
```conf
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@tun.r2bny.com
    leftcert=server.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.0.0.0/24
    rightdns=1.1.1.1,9.9.9.9
    rightsendcert=never
    eap_identity=%identity
	ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
	esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```
Чтобы обеспечить широкую совместимость между клиентами и сервером IKEv2, нужно использовать алгоритмы шифрования, которые поддерживаются большинством платформ. Универсальные и совместимые шифры IKE и ESP:
```
ike=aes256-sha1-modp2048,aes128-sha1-modp2048,aes256-sha1-modp1024,aes128-sha1-modp1024!
esp=aes256-sha1,aes128-sha1,3des-sha1!
```
### 3.2 Настройка идентификации клиентов
Откройте файл `/etc/ipsec.secrets` и добавьте учетные данные для клиентов:
```bash
sudo nano /etc/ipsec.secrets
```
Добавьте строку:
```conf
: RSA "server.pem"
username : EAP "password"
```
- Замените `username` и `password` на данные пользователя VPN. Убедитесь, что строка начинается с : символа и после него стоит пробел, чтобы вся строка читалась как : RSA "server-key.pem".
### 3.3 Настройка идентификации клиентов через БД
Откройте файл `/etc/ipsec.secrets` и добавьте учетные данные для клиентов:
```bash
sudo nano /etc/ipsec.secrets
```

## 4. Настройка сети
### 4.1 Включение форвардинга пакетов
Для работы VPN нужно разрешить форвардинг пакетов:
```bash
sudo nano /etc/sysctl.conf
```
Найдите строку `net.ipv4.ip_forward` и измените её на:
```conf
net.ipv4.ip_forward = 1
```
Примените изменения:
```bash
sudo sysctl -p
```
### 4.2 Настройка правил IPTables
Теперь добавим правила для NAT:
```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
```
Если вы используете интерфейс с другим именем (например, `ens3`), замените `eth0` на нужный интерфейс.
Для сохранения правил IPTables установим пакет `iptables-persistent`:
```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

## 5. Запуск StrongSwan
После всех изменений перезапустите сервис StrongSwan:
```bash
sudo systemctl restart ipsec
```
Проверьте статус сервиса:
```bash
sudo systemctl status ipsec
```

## 6. Настройка клиентов
Теперь необходимо настроить клиентов (например, устройства на iOS, Android, Windows или macOS) для использования протокола IKEv2 с вашим VPN сервером. На каждом клиенте потребуется:
- Установить корневой сертификат `ca.pem`.
- Настроить подключение к серверу с использованием учетных данных, указанных в файле `ipsec.secrets`.