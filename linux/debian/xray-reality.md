# Установка и настройка Xray (VLESS + REALITY) на Debian GNU/Linux
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
sudo apt install -y uuid-runtime
```

## 3. Конфигурация Xray (VLESS + REALITY)
Сгенерируйте и сохарните ключи Reality:
```bash
xray x25519
```
Пример вывода:
```bash
Private Key: abcdef...
Public Key: 123456...
```
Сгенерируйте UUID который будет использоваться для аутентификации клиента:
```bash
cat /proc/sys/kernel/random/uuid
```
Откройте в редакторе конфигурационный файл:
``` bash
sudo nano /usr/local/etc/xray/config.json
```
Замените '<UUID>' и '<PRIVATE_KEY>' на сгенерированные значения:
``` json
{
  "log": {
    "loglevel": "warning"
  },

  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },

      "streamSettings": {
        "network": "tcp",
        "security": "reality",

        "realitySettings": {
          "show": false,
          "dest": "twitch.tv:443",
          "xver": 0,

          "serverNames": [
            "twitch.tv"
          ],

          "privateKey": "<PRIVATE_KEY>",

          "shortIds": [
            "a1b2c3d4"
          ]
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
Блок 'clients':
``` json
        "clients": [
          {
            "id": "<UUID_1>",
            "flow": "xtls-rprx-vision",
            "email": "user1@r2bny.com"   // для удобства в логах
          },
          {
            "id": "<UUID_2>",
            "flow": "xtls-rprx-vision",
            "email": "user2@example.com"
          }
          // Добавляйте сколько угодно
```
Выполните перезапуск сервисов:
``` bash
sudo systemctl restart xray
```
Просмотр логов Xray в реальном времени:
``` bash
journalctl -u xray -f
```

## 4. Клиентская ссылка
Скопируйте шаблон ниже и замените плейсхолдеры на свои данные:
- `<UUID>` — UUID пользователя  
- `<YOUR_SERVER_IP_OR_DOMAIN>` — IP-адрес или домен вашего сервера  
- `<PUBLIC_KEY>` — публичный ключ Reality (Public Key)
```vless
vless://<UUID>@<YOUR_SERVER_IP_OR_DOMAIN>:443?encryption=none&security=reality&sni=twitch.tv&fp=randomized&pbk=<PUBLIC_KEY>&sid=a1b2c3d4&type=tcp&flow=xtls-rprx-vision#XRAY Server
```
Вставьте полученную ссылку в клиент Xray. Сохраните конфигурацию и подключитесь к серверу.

## 5. Настройка Xray Client как локального SOCKS5-прокси
**Конфигурация клиента (config.json): **
```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "socks-in",
      "port": 1080,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": {
        "udp": true,
        "auth": "noauth"
      }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "<YOUR_SERVER_IP_OR_DOMAIN>",
            "port": 443,
            "users": [
              {
                "id": "<UUID>",
                "flow": "xtls-rprx-vision",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "fingerprint": "randomized",
          "serverName": "twitch.tv",
          "publicKey": "<PUBLIC_KEY>",
          "shortId": "a1b2c3d4",
          "spiderX": "/"
        }
      }
    }
  ]
}
```
### 5.1 Linux (Debian / Ubuntu)
Выполните установку Xray:
```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```
Создайте/отредактируйте файл с следующим содержимым (замените плейсхолдеры):
```bash
sudo nano /usr/local/etc/xray/config.json
```
Запуск сервиса:
```bash
sudo systemctl enable --now xray
journalctl -u xray -f
```
### 5.2 Windows
1. Скачайте [Xray-core](https://github.com/XTLS/Xray-core/releases)
2. Распакуйте в `C:\xray`
3. Создайте `config.json` с конфигом выше
4. Запуск:
```cmd
cd C:\xray
xray.exe run -c config.json
```
### 5.3 macOS
Выполните установку Xray:
```bash
brew install xray
```
Вставьте конфиг в `/usr/local/etc/xray/config.json` и запустите:
```bash
brew services start xray
```

## Полезные ссылки
- Xray Core: https://github.com/XTLS/Xray-core
- Официальная документация: https://xtls.github.io/
