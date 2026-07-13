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
Сгенерируйте ключи:
```bash
xray x25519
```
Пример вывода:
```bash
Private Key: abcdef...
Public Key: 123456...
```
Сохраните ключи:
- На сервер: `abcdef...`
- На клиент: `123456...`
Сгенерируйте UUID который будет использоваться для аутентификации клиента:
```bash
cat /proc/sys/kernel/random/uuid
```
Откройте в редакторе конфигурационный файл:
``` bash
sudo nano /usr/local/etc/xray/config.json
```
Замените <UUID> на сгенерированный UUID и <PRIVATE_KEY> на `abcdef...`:
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
            "kick.com"
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
