# Установка KDE Plasma на Debian GNU/Linux
## 1. Обновление системы
Перед установкой убедитесь, что система обновлена:
```bash
sudo apt update && sudo apt full-upgrade -y
```
Очистите систему от устаревших пакетов:
```bash
sudo apt autoremove -y && sudo apt autoclean
```

## 2. Установка KDE Plasma
В Debian доступно три основных варианта: plasma-desktop, kde-standard, kde-full. Для экономии ресурсов выберите первый вариант:
```bash
sudo apt -y install sddm plasma-desktop plasma-nm dolphin konsole ark kate firefox-esr
```
Если вы хотите более полную среду выполните:
```bash
sudo apt install -y sddm kde-standard
```
Установите дополнительные компоненты (по желанию):
```bash
sudo apt install -y plasma-widgets-addons spectacle gwenview okular
```
Установите загрузку в графический интерфейс по умолчанию:
```bash
sudo systemctl set-default graphical.target
```
Проверьте диспетчер входа SDDM:
```bash
sudo systemctl status sddm
```
Если SDDM не активен, запустите его:
```bash
sudo systemctl enable --now sddm
```
Выполните перезагрузку:
```bash
sudo reboot
```
После перезагрузки вы должны увидеть экран входа SDDM с KDE Plasma. Если система загружается в текстовый режим или LXDE, настройте SDDM:
```bash
sudo dpkg-reconfigure sddm
```
