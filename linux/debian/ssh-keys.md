# Создание и работа с SSH-ключами на Debian GNU/Linux

## 1. Создание ключей
Ключ для **своих серверов** (с паролем):
```bash
ssh-keygen -t ed25519 -a 100 -C "Private SSH Key" -f ~/.ssh/id_ed25519_private
```
Рекомендуется **задать пароль** (passphrase).  

Ключ для **публичных сервисов** (без пароля):
```bash
ssh-keygen -t ed25519 -a 100 -C "Public SSH Key" -f ~/.ssh/id_ed25519_public
```
Можно оставить **без passphrase** для Git/CI/CD.  

## 2. Настройка прав доступа
Выполните ряд команд для применения прав доступа:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_private
chmod 644 ~/.ssh/id_ed25519_private.pub
chmod 600 ~/.ssh/id_ed25519_public
chmod 644 ~/.ssh/id_ed25519_public.pub
```

## 3. Настройка `~/.ssh/config`
Редактируем файл:
```bash
nano ~/.ssh/config
```
Пример:
```sshconfig
# Подключение к личному серверу
Host myserver
    HostName myserver.r2bny.com
    User r2bny
    IdentityFile ~/.ssh/id_ed25519_private
# GitHub (публичные проекты)
Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_public
# GitLab (пример)
Host gitlab.com
    User git
    IdentityFile ~/.ssh/id_ed25519_public
```

## 4. Добавление ключа в SSH-агент
Для добавления ключа выполните команду:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_private
```
Посмотреть подключённые ключи:
```bash
ssh-add -l
```
Удалить ключ:
```bash
ssh-add -d ~/.ssh/id_ed25519_private
```

## 5. Установка публичного ключа на сервер
Автоматически:
```bash
ssh-copy-id -i ~/.ssh/id_ed25519_private.pub user@r2bny.com
```
Вручную:
```bash
cat ~/.ssh/id_ed25519_private.pub | ssh user@r2bny.com "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## 6. Резервное копирование
Сохрани приватные ключи в зашифрованный архив:
```bash
tar -czf - ~/.ssh/id_ed25519_private ~/.ssh/id_ed25519_public | gpg -c > ssh_keys_backup.tar.gz.gpg
```