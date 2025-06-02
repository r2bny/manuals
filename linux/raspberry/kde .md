# Установка KDE Plasma на Raspberry Pi OS

### 1. Обновите систему
```bash
sudo apt update && sudo apt full-upgrade -y 
```
> Обновляет все пакеты до актуальных версий.

### 2. Установите KDE Plasma и базовые программы
```bash
sudo apt -y install kde-plasma-desktop konsole ark kate
```
- `kde-plasma-desktop` — минимальный KDE

### 3. Установите графическую среду по умолчанию
```bash
sudo systemctl set-default graphical.target
```
> Теперь KDE будет запускаться при старте системы.

### 4. Перезагрузите устройство (по желанию)
```bash
sudo reboot
```
