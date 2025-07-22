# Установка и настройка DRBD на Debian GNU/Linux

## Условия

- Узлы: `node1` и `node2`
- IP-адреса: `192.168.100.1` и `192.168.100.2`
- Устройство: `/dev/sdb`
- DRBD-ресурс: `r0`
- Используется `ext4`, только один Primary

## 1. Обновление и настройка системы на обоих узлах
Перед установкой рекомендуется обновить систему:
```bash
sudo apt update && sudo apt upgrade -y
```
Задать корректное имя хоста на каждом узле:
```bash
sudo nano /etc/hostname
```
Редактирование файла hosts:
```bash
sudo nano /etc/hosts
```

## 2. Установка DRBD

На **обоих узлах** выполните установку DRBD:
```bash
sudo apt install -y drbd-utils
```

Убедитесь, что `/dev/sdb` свободен:
```bash
lsblk
wipefs -a /dev/sdb
```

## 3. Конфигурация DRBD-ресурса
Создайте файл `/etc/drbd.d/r0.res` на **обоих узлах**:
```ini
resource r0 {
    protocol C;

    on node1 {
        device     /dev/drbd0;
        disk       /dev/sdb;
        address    192.168.100.1:7789;
        meta-disk  internal;
    }

    on node2 {
        device     /dev/drbd0;
        disk       /dev/sdb;
        address    192.168.100.2:7789;
        meta-disk  internal;
    }
}
```
Проверьте имя хоста: `uname -n`

## 4. Инициализация ресурса
На **обоих узлах**:
```bash
drbdadm create-md r0
drbdadm up r0
```

## 5. Назначение Primary
На **одном узле**:
```bash
drbdadm primary --force r0
```

## 6. Создание и монтирование ФС
```bash
mkfs.ext4 /dev/drbd0
mkdir -p /mnt/drbd
mount /dev/drbd0 /mnt/drbd
```

## 7. Автомонтирование
Добавьте в `/etc/fstab`:
```fstab
/dev/drbd0 /mnt/drbd ext4 defaults,_netdev 0 0
```

## 8. Проверка DRBD-ресурсов
Статус ресурса:
```bash
drbdadm status
```
```bash
cat /proc/drbd
mount | grep drbd
```
Проверить конкретный ресурс:
```bash
drbdadm status r0
```
Если drbd0 уже используется как блочное устройство, проверь:
```bash
lsblk -f | grep drbd
```
Монтирование drbd0 в /var/www/html/:
mkdir -p /var/www/html/bx-sites
mount /dev/drbd0 /var/www/html/bx-sites

