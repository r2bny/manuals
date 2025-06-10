# Установка и полная замена Python 3.7 на 3.8 в Astra Linux SE 1.7.x

> **Внимание:** Эта инструкция заменяет системную версию Python 3.7.3 на 3.8.18. Убедитесь, что у вас есть root-доступ и возможность восстановления системы.

## 1. Подготовка окружения
Обновите список пакетов и установите необходимые библиотеки для сборки Python:
```bash
sudo apt update
sudo apt install -y build-essential \
  libssl-dev zlib1g-dev libbz2-dev \
  libreadline-dev libsqlite3-dev wget curl \
  llvm libncurses5-dev libncursesw5-dev \
  xz-utils tk-dev libffi-dev liblzma-dev \
  libgdbm-dev libnss3-dev uuid-dev
```

## 2. Установка Python 3.8 из исходников
Скачайте и распакуйте исходники Python 3.8
```bash
cd /usr/src
sudo wget https://www.python.org/ftp/python/3.8.18/Python-3.8.18.tgz
sudo tar xzf Python-3.8.18.tgz
cd Python-3.8.18
```
Соберите и установите Python 3.8:
```bash
sudo ./configure --enable-optimizations --prefix=/usr/local
sudo make -j$(nproc)
sudo make altinstall
```
Убедитесь, что Python 3.8 установлен корректно, вызвав его напрямую:
```bash
/usr/local/bin/python3.8 --version
```

## 4. Настройка альтернатив и замена python3
Зарегистрируйте в системе две версии python3 — системную 3.7 и новую 3.8:
```bash
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 1
sudo update-alternatives --install /usr/bin/python3 python3 /usr/local/bin/python3.8 2
```
Выберите активную версию python3.8 (номер 2):
```bash
sudo update-alternatives --config python3
```
Проверьте текущую версию python3:
```bash
python3 --version
# Ожидаемый результат: Python 3.8.18
```
При необходимости переключитесь обратно на python3.7 (номер 1):
```bash
sudo update-alternatives --config python3
```

## 5. Установка pip для Python 3.8
Загрузите и установите pip для текущей версии python3 (3.8).
```bash
curl -sS https://bootstrap.pypa.io/pip/3.8/get-pip.py | sudo python3
```
Проверьте версию:
```bash
pip3 --version
```

## 6. Проверка системных утилит
Убедитесь, что системные команды работают корректно после смены python:
```bash
apt --version
lsb_release -a
python3 -c "import sys; print(sys.executable)"
```
```bash
sudo ln -s /usr/bin/python3.7 /usr/local/bin/python3.7
```
