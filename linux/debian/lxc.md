# Установка и настройка LXC на Debian GNU/Linux
**Важно:** В данной инструкции для наглядности в примерах и конфигурационных файлах **используется домен `r2bny.com`**. Перед применением обязательно **замените его на ваш собственный домен** во всех упоминаниях в конфигурациях сервисов, путях к файлам и директориях, ссылках и сертификатах.

## 1. Подготовка хоста
Выполните обновление системы и установите пакеты:
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y lxc lxc-templates debootstrap bridge-utils uidmap dnsmasq-base iptables net-tools curl rsync apparmor apparmor-utils
```
Проверьте поддержку user namespaces (в Debian 13 по умолчанию включено):
```bash
sysctl kernel.unprivileged_userns_clone
# должно быть 1
```
Если значение 0 — включите:
```bash
echo "kernel.unprivileged_userns_clone=1" | sudo tee /etc/sysctl.d/99-lxc-userns.conf
sudo sysctl --system
```
Настройка subordinate UID/GID (обязательно для unprivileged)
```bash
# Для root (рекомендуется запускать контейнеры от root)
echo "root:100000:65536" | sudo tee -a /etc/subuid
echo "root:100000:65536" | sudo tee -a /etc/subgid
```
Проверьте, что записи появились:
```bash
cat /etc/subuid /etc/subgid
```

## 2. Настройка сети
Включите и настрайте службу lxc-net. Контейнеры будут получать адреса из подсети 10.0.3.0/24, а выход в интернет пойдёт через NAT:
```bash
sudo systemctl enable --now lxc-net
```
Создайте резервную копию и запишите рекомендованную конфигурацию:
```bash
sudo cp /etc/default/lxc-net /etc/default/lxc-net.bak
```
```bash
sudo tee /etc/default/lxc-net << 'EOF'
USE_LXC_BRIDGE="true"
LXC_BRIDGE="lxcbr0"
LXC_ADDR="10.0.3.1"
LXC_NETMASK="255.255.255.0"
LXC_NETWORK="10.0.3.0/24"
LXC_DHCP_RANGE="10.0.3.2,10.0.3.254"
LXC_DHCP_MAX="253"
LXC_DHCP_CONFILE=""
LXC_DOMAIN=""
EOF
```
Перезапустите службу:
```bash
sudo systemctl restart lxc-net
```
Проверьте, что мост поднялся и NAT работает:
```bash
ip a show lxcbr0                    # должен быть 10.0.3.1/24
sudo iptables -t nat -L -n | grep MASQUERADE
```

## 3. Настройка конфигурации LXC
Отредактируйте файл /etc/lxc/default.conf. Все новые контейнеры будут создаваться unprivileged и подключены к мосту lxcbr0:
```bash
sudo tee /etc/lxc/default.conf << 'EOF'
lxc.include = /usr/share/lxc/config/common.conf
lxc.include = /usr/share/lxc/config/userns.conf

lxc.idmap = u 0 100000 65536
lxc.idmap = g 0 100000 65536

lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.net.0.name = eth0

lxc.apparmor.profile = unconfined
lxc.apparmor.allow_nesting = 1
EOF
```

## 4. Создание базового контейнера (шаблон-эталон)
Создайте базовый контейнер Debian 13 (trixie):
```bash
sudo lxc-create -n debian13 -t download -- \
  --dist debian \
  --release trixie \
  --arch amd64
```
После скачивания проверьте, что контейнер появился:
```bash
lxc-ls -f
```
Запустите контейнер в фоне и зайдите внутрь (вы должны оказаться внутри контейнера от имени root):
```bash
sudo lxc-start -n debian13 -d
sudo lxc-attach -n debian13 --clear-env
```
Принудительно получите адрес и пропишите DNS:
```bash
# Получаем IP через DHCP
dhclient -v eth0 || true

# Прописываем надёжные DNS
cat > /etc/resolv.conf << EOF
nameserver 10.0.3.1
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

# Проверка
ping -c 2 8.8.8.8
ping -c 2 r2bny.com
```
Выполните настройку сети контейнера через systemd-networkd (актуальный способ). Для начала выполните установку resolved:
```bash
apt update
apt install -y systemd-resolved
```
Создайте конфигурацию сети:
```bash
mkdir -p /etc/systemd/network
cat > /etc/systemd/network/eth0.network << 'EOF'
[Match]
Name=eth0

[Network]
DHCP=yes
EOF
```
Если хотите сразу статический адрес вместо DHCP:
```bash
cat > /etc/systemd/network/eth0.network << 'EOF'
[Match]
Name=eth0

[Network]
Address=10.0.3.XX/24          # замените XX на нужный IP
Gateway=10.0.3.1
DNS=10.0.3.1 8.8.8.8 1.1.1.1
EOF
```
Настройте DNS:
```bash
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```
Включите службы
```bash
systemctl enable --now systemd-networkd
systemctl enable --now systemd-resolved
```
Выполните полное обновление системы
```bash
apt update
apt full-upgrade -y
```
После обновления рекомендуется перезагрузить контейнер (из хоста):
```bash
# Выйдите из контейнера (exit), затем:
sudo lxc-stop -n debian13
sudo lxc-start -n debian13 -d
sudo lxc-attach -n debian13 --clear-env
```
Выполните установку базового набора пакетов для удобной работы:
```bash
apt install -y locales curl ca-certificates systemd-timesyncd net-tools dnsutils nano
```
Настройте часовой пояс (например Москва):
```bash
timedatectl set-timezone Europe/Moscow
```
Включите синхронизацию времени:
```bash
systemctl enable --now systemd-timesyncd
```
Очень важно хорошо почистить контейнер перед сохранением шаблона. Выполните финальную чистку:
```bash
#Очистка пакетов
apt autoremove -y
apt clean
apt autoclean

# Удаляем кэши и временные файлы
rm -rf /var/lib/apt/lists/*
rm -rf /tmp/*
rm -rf /var/tmp/*
rm -rf /root/.cache

# Очищаем историю команд
cat /dev/null > /root/.bash_history
history -c

# Выходим
exit
```
Выполните остановку контейнера:
```bash
sudo lxc-stop -n debian13
```
Проверьте статус. Контейнер должен быть в состоянии STOPPED:
```bash
lxc-ls -f
```
Cкопируйте подготовленный контейнер в каталог шаблонов:
```bash
sudo mkdir -p /var/cache/lxc/templates
sudo cp -a /var/lib/lxc/debian13 /var/cache/lxc/templates/debian13
```

## 5. Создание нового контейнера из шаблона
Создайте пустой контейнер (например postgresql):
```bash
sudo lxc-create -n postgresql -t none
```
Скопируйте файловую систему из шаблона:
```bash
sudo rsync -aHAX --delete /var/cache/lxc/templates/debian13/rootfs/ /var/lib/lxc/postgresql/rootfs/
```
Cкопируйте конфиг шаблона в новый контейнер:
```bash
sudo cp /var/cache/lxc/templates/debian13/config /var/lib/lxc/postgresql/config
sudo sed -i "s/lxc.uts.name = .*/lxc.uts.name = postgresql/" /var/lib/lxc/postgresql/config
```
Выполните запуск и вход:
```bash
sudo lxc-start -n postgresql -d
sudo lxc-attach -n postgresql
```
