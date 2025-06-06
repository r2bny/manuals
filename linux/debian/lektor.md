# Установка и настройка Lektor на Debian GNU/Linux

## 1. Обновление и настройка системы
Перед установкой рекомендуется обновить систему и задать корректное имя хоста.
```bash
sudo apt update && sudo apt upgrade -y
```
Редактирование имени хоста:
```bash
sudo nano /etc/hostname
```
Редактирование файла hosts:
```bash
sudo nano /etc/hosts
```

## 2. Установка зависимостей
Lektor требует наличия Python и некоторых дополнительных пакетов:
```bash
sudo apt install nginx certbot python3-certbot-nginx python3 python3-venv gcc curl -y
```

## 3. Развертывание окружения и установка Lektor
Создайте рабочую директорию:
```bash
sudo mkdir /opt/r2bny
```
Перейдите в неё:
```bash
cd /opt/r2bny
```
Создайте виртуальное окружение:
```bash
python3 -m venv .venv
```
Активируйте виртуальное окружение:
```bash
source .venv/bin/activate
```
Обновите pip и setuptools, затем установите Lektor:
```bash
pip install --upgrade pip setuptools
pip install Lektor
```

## 4. Создание проекта Lektor
### 4.1 Инициализация проекта
Теперь, когда Lektor установлен, можно создать первый проект:
```bash
lektor quickstart
```
Пример диалога:
```
Lektor Quickstart
=================
> Project Name: R2BNY
> Author Name [Artem Deev]:
> Project Path [/opt/r2bny/web]: 
> Add Basic Blog [Y/n]: y
That's all. Create project? [Y/n]: y
```
### 4.2 Запуск локального сервера
Перейдите в каталог проекта:
```bash
cd /opt/r2bny/web
```
Запустите сервер командой:
```bash
lektor server
```
Откройте в браузере:
```
http://localhost:5000
```
Для доступа к административной панели:
```
http://localhost:5000/admin
```
## 5. Настройка Nginx для Lektor
### 5.1 Создание конфигурации виртуального хоста
Создайте новый конфигурационный файл для вашего сайта, например:

```bash
sudo nano /etc/nginx/sites-available/dev.r2bny.com.conf
```

Добавьте в файл следующее (замените `r2bny.com` на ваше доменное имя):

```nginx
server {
    listen 80;
    server_name dev.r2bny.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
### 5.2 Активация конфигурации
Создайте символическую ссылку для активации сайта:
```bash
sudo ln -s /etc/nginx/sites-available/dev.r2bny.com.conf /etc/nginx/sites-enabled/
```
Проверьте конфигурацию на наличие ошибок:
```bash
sudo nginx -t
```
Перезапустите Nginx:
```bash
sudo systemctl reload nginx
```
### 5.3 Настройка HTTPS с Let's Encrypt
Запустите команду для получения и автоматической настройки SSL-сертификата:
```bash
sudo certbot --nginx -d dev.r2bny.com
```
После выполнения инструкции Certbot сам добавит конфигурацию HTTPS в Nginx.

### 5.4 Автоматическое обновление сертификатов
Проверьте, настроено ли автоматическое обновление:
```bash
sudo systemctl list-timers | grep certbot
```
Если не настроено, добавьте в cron:
```bash
sudo crontab -e
```
Добавьте строку:
```
0 3 * * * /usr/bin/certbot renew --quiet && systemctl reload nginx
```

## 6. Структура проекта
### 6.1 Базовая структура проекта
Проект Lektor — это папка с файлом проекта и определённой структурой. Также могут присутствовать дополнительные папки, особенно при использовании плагинов.
```
yourproject.lektorproject
content/
models/
templates/
assets/
```
### 6.2 Файл проекта (`*.lektorproject`)
Файл конфигурации проекта с расширением `.lektorproject`. Используется для идентификации проекта в интерфейсе Lektor и содержит базовую информацию о проекте: его имя, версию и пути к основным папкам. Имя файла может быть любым, главное — наличие расширения `.lektorproject`.
```
[project]
# Название проекта
name = My Multilingual Website

# Основная локаль сайта
locale = en_US

# Полный URL сайта 
url = https://www.example.com/

# Стиль генерации ссылок
# Возможные значения
url_style = relative

# Путь к дереву проекта
# path = ./content

# Кастомный путь сборки 
# output_path = ./build_output

# Исключаемые файлы из assets
excluded_assets = *.backup, *~

# Включаемые файлы, несмотря на шаблоны исключений
included_assets = _keep.me, .htaccess

[packages]
# Плагины проекта
lektor-webpack-support = 0.1

[servers.production]
# Сервер для деплоя
name = Production Server
enabled = yes
default = yes
target = rsync://example.com/home/user/site-output

[alternatives.en]
# Альтернатива для английской версии
name = English
primary = yes
locale = en_US
url_prefix = /

[alternatives.ru]
# Альтернатива для русской версии
name = Русский
locale = ru_RU
url_prefix = /ru/

[attachment_types]
# Расширения вложений
.gpx = gpx
.ogv = video
```
### 6.3 Папки проекта
#### 6.3.1 `content/`
Содержит весь контент сайта. Каждая папка в `content/` представляет собой запись (record), а основная информация о записи хранится в файле `contents.lr`.
##### Пример структуры:
```
content/
  contents.lr
  projects/
    contents.lr
    project-a/
      contents.lr
      thumbnail.png
    project-b/
      contents.lr
      thumbnail.png
```
Файлы, отличные от `contents.lr`, считаются вложениями и могут быть использованы как изображения, документы и т.д.
#### 6.3.2 `models/`
Содержит описание моделей данных в формате `.ini`. Каждая модель описывает структуру определённого типа содержимого. Например, модель `page.ini` может описывать тип страницы.
Файлы модели управляют:
- Названиями и типами полей
- Наследованием от других моделей
- Используемыми шаблонами
#### 6.3.3 `templates/`
Хранят HTML-шаблоны для рендеринга контента. Каждый шаблон соответствует модели, определённой в папке `models/`.
Пример:
- модель `page.ini` — шаблон `templates/page.html`
Шаблоны могут использовать Jinja2-синтаксис и обращаться к полям модели напрямую.
#### 6.3.4 `assets/`
Статические файлы, копируемые в финальную сборку проекта без изменений.
Типичные примеры:
- CSS, JavaScript
- favicon.ico
- изображения
- `.htaccess`, `robots.txt`
Эта папка перекрывает пути сайта, и её содержимое доступно напрямую через URL.
#### 6.3.5 `flowblocks/`
Используется для определения блоков контента (Flow Blocks), которые можно вставлять в поля моделей. Каждый блок — это отдельная мини-модель со своей структурой.
Flow Blocks позволяют гибко управлять компоновкой и повторно использовать контент.
#### 6.3.6 `packages/`
Пространство для локальной разработки плагинов. Все плагины, расположенные здесь, автоматически подгружаются Lektor при старте.
#### 6.3.7 `configs/`
Конфигурационные файлы для плагинов. Каждый файл имеет имя в формате `<plugin-id>.ini` и содержит параметры настройки соответствующего плагина.
#### 6.3.8 `databags/`
Папка для хранения универсальных данных, доступных из шаблонов: меню, списки ссылок, параметры и т.п.  
Данные структурированы в виде YAML или JSON-файлов и не связаны с моделями напрямую.

## 7 Контент (Content)
Lektor собирает сайт, используя все файлы из папки `content/`, обрабатывая их согласно правилам моделей и рендеря их с помощью шаблонов.  
Звучит сложно, но на практике всё довольно просто.
### 7.1 Одна папка — одна страница
Каждая страница (или URL) соответствует папке в `content/`.  
Вы можете создать столько папок, сколько необходимо, и вкладывать их друг в друга. В каждой папке должен быть хотя бы один файл — `contents.lr`, в котором содержатся данные страницы.
Все остальные файлы в папке считаются вложениями (attachments) данной страницы.
#### Пример структуры:
```
content/
  contents.lr
  portfolio/
    contents.lr
    project-a/
      contents.lr
      thumbnail.jpg
    project-b/
      contents.lr
      thumbnail.jpg
  about/
    contents.lr
```
### 7.2 Одна страница — один URL
Каждый файл `contents.lr` превращается в одну итоговую страницу по одному URL.  
Например, файл `content/portfolio/project-a/contents.lr` приведёт к странице `/portfolio/project-a/`.
### 7.3 Страница, модель и шаблон
Каждая страница связана с моделью и шаблоном.  
Модель определяет поля данных, а шаблон управляет их отображением.  
По умолчанию имя шаблона соответствует имени модели, но его можно переопределить с помощью поля `_template` в `contents.lr`.
#### Как выбирается модель:
1. Указана явно в поле `_model` файла `contents.lr`
2. Наследуется от родительской модели
3. Соответствует `id` страницы
4. Используется модель с именем `page`
### 7.4 Формат файла `contents.lr`
Файл контента — это обычный текстовый файл в кодировке UTF-8 с расширением `.lr`.  
Он содержит поля, определённые в модели, и легко редактируется вручную.
#### Пример содержимого:
```
_model: page
---
title: Заголовок страницы
---
body:

Основной текст страницы
```
Каждое поле начинается с ключа, затем двоеточие и значение.  
Для многострочных значений — два переноса строки после двоеточия.
- Поля могут быть простым текстом, Markdown и т.д.
- Поля с подчёркиванием (`_model`, `_template`) — системные.
Если нужно использовать `---` в тексте, добавьте лишний дефис:  
`----` отобразится как `---`.

## 8. Модели и шаблоны
Сила Lektor заключается в способности описывать структуру данных, на основе которой строятся страницы.  
Корректно выстроенные модели облегчают создание и поддержку красивого и масштабируемого сайта.
### 8.1 Модели
Модели — это "чертежи" страниц. Они определяют, какие поля есть у страницы и как они отображаются в админке.  
Хранятся модели в папке `models/` в виде INI-файлов с кодировкой UTF-8. Имя файла определяет ID модели.
Если модель явно не указана, Lektor попытается использовать модель по умолчанию — чаще всего `page.ini`.
#### Пример модели `models/page.ini`:
```ini
[model]
name = Page
label = {{ this.title }}

[fields.title]
label = Title
type = string
size = large

[fields.body]
label = Body
type = markdown
```
В этом примере:
- модель имеет ID `page` и отображаемое имя `Page`;
- поле `title` — строка, отображается крупным шрифтом;
- поле `body` — поддерживает Markdown и будет отображаться как текстовая область в UI.
### 8.2 Поля (Fields)
Поля отображаются в админке в том порядке, в котором они указаны в INI-файле модели.  
Каждое поле имеет ряд параметров, 8
- `label` — подпись поля в UI.
- `description` — описание поля (отображается как пояснение).
- `addon_label` — пояснение справа от поля (например, единицы измерения).
- `width` — ширина поля (например, `1/2`, `1/4`).
- `size` — размер поля (`normal`, `small`, `large`).
- `type` — тип поля (определяет 8
Типы полей описаны подробнее в официальной документации Lektor по [типам полей](https://www.getlektor.com/docs/models/fields/).8
### 8.3 Опции модели (
Каждая модель может иметь дополнительные параметры настройки:

- `name` — имя модели, отображаемое в UI.
- `label` — шаблонная строка, отображаемая для страницы с этой моделью (например, `{{ this.title }}`).
- `hidden` — скрывает модель от выбора при создании страниц.
- `protected` — запрещает удаление страниц с этой моделью.
- `inherits` — наследует поля и настройки другой 
Также возможны дополнительные конфигурационные секции, зависящие от структуры проекта.
Модели являются ключевым элементом архитектуры сайта на Lektor.  
Грамотно построенные модели обеспечивают удобство управления и гибкость шаблонов.
### 8.4 Шаблоны
Lektor использует шаблонизатор **Jinja2** для генерации HTML на основе данных страниц. Все шаблоны хранятся в папке `templates/` и обычно имеют расширение `.html`.  
По умолчанию имя шаблона совпадает с именем модели. Например, если у вас есть модель `page`, то должен быть файл `templates/page.html`.
Также можно явно указать другой шаблон для конкретной страницы с помощью настройки в модели.
### 8.5 Контекст шаблона
При рендеринге шаблона доступны следующие переменные:
| Переменная | Описание |
|------------|----------|
| `this`     | Текущая запись (Record), которая рендерится |
| `site`     | Объект базы данных Pad, используется для запросов к другим частям сайта |
| `alt`      | Строка, идентифицирующая альтернативу (Alternative) страницы |
| `config`   | Доступ к конфигурации проекта Lektor |
### 8.6 Первый шаблон
Создадим первый шаблон, если вы уже настроили модель `page`. В папке `templates/` создайте файл `page.html`:
```jinja
{% extends "layout.html" %}
{% block title %}{{ this.title }}{% endblock %}
{% block body %}
  <h2>{{ this.title }}</h2>
  {{ this.body }}
{% endblock %}
```
#### Объяснение синтаксиса:
- `{% ... %}` — управляющие конструкции Jinja
- `extends` — расширяет другой шаблон (наследование)
- `block` — объявляет или переопределяет блок из базового шаблона
- `{{ ... }}` — вывод значения переменной
### 8.7 Шаблон макета (Layout Template)
Теперь создадим базовый шаблон `layout.html`, от которого будут наследоваться другие:
```html
<!doctype html>
<meta charset="utf-8">
<title>{% block title %}Welcome{% endblock %} — My Website</title>
<body>
  <header>
    <h1>My Website</h1>
    <nav>
      Navigation can go here.
    </nav>
  </header>
  <div class="page">
    {% block body %}{% endblock %}
  </div>
</body>
```
Это демонстрирует, как работает механизм блоков и наследования в Jinja2.
### 8.8 Всё о шаблонах
Шаблоны — один из ключевых компонентов при создании сайтов на Lektor.  
Они дают гибкость, контроль и мощные возможности для построения динамического контента.
Дополнительные материалы:
- [Документация по шаблонам Lektor](https://www.getlektor.com/docs/templates/)
- [Jinja2 документация](https://jinja.palletsprojects.com/)

## 9 Публикация сайта (Deployment)
Сайт становится полезным только тогда, когда его могут увидеть другие.  
Во время локальной разработки это не критично, но для публикации проекта на веб-хостинге вам понадобится команда `lektor deploy`.
### 9.1 Публикация в два шага
Публикация сайта в Lektor состоит из двух этапов:
1. **Сборка** (`build`)
2. **Публикация** (`deploy`)
> ⚠️ Важно: команда `deploy` **не запускает сборку автоматически**. Если вы забудете выполнить сборку перед публикацией, вы можете загрузить устаревшую или пустую версию сайта.
### 9.2 Сборочный конвейер (Build Pipeline)
При запуске локального сервера Lektor автоматически собирает HTML-файлы в фоновом режиме.  
По умолчанию сайт собирается в системную временную папку.
Узнать путь можно так:
```bash
lektor project-info --output-path
```
Для ручной сборки с указанием собственного пути:
```bash
lektor build --output-path my-folder
```
> Рекомендуется использовать путь по умолчанию при локальной публикации — это быстрее.
### 9.3 Автоматическая публикация с помощью Lektor
Lektor поддерживает автоматическую публикацию через `rsync` и `FTP`. Для этого настройте файл конфигурации проекта:
```ini
[servers.production]
name = Production
enabled = yes
default = yes
target = rsync://server/path/to/folder
```
Запуск публикации:
```bash
lektor deploy production
```
Если сервер помечен как `default`, достаточно выполнить:
```bash
lektor deploy
```
Для публикации сразу после сборки:
```bash
lektor build && lektor deploy
```
> Путь к сборке можно задать через флаг `--output-path` или переменную окружения `LEKTOR_OUTPUT_PATH`.
### 9.4 Поддерживаемые цели публикации
Нативно поддерживаются:
- `rsync`
- `FTP`
- `GitHub Pages`
Для других целей публикации можно использовать сторонние плагины.
### 9.5 Ручная публикация
Если вы предпочитаете собственный способ публикации, это не проблема. Например, публикация на Amazon S3 с помощью `s3cmd`:
```bash
lektor build && s3cmd sync "$(lektor project-info --output-path)"/* "s3://my-bucket/some/path"
```
### 9.6 Публикация через веб-интерфейс
Публикация также поддерживается напрямую из интерфейса администратора. Там доступна кнопка **Publish**, которая запускает процесс отправки файлов на сервер.

## 10. Командная строка
Lektor предоставляет утилиту командной строки `lektor`, с помощью которой можно управлять всеми функциями системы прямо из терминала — в дополнение к графическому интерфейсу.
Если команда `lektor` недоступна, возможно установлена только GUI-версия. Ознакомьтесь с инструкцией по установке, чтобы добавить CLI в вашу систему.
### 10.1 Общие параметры
Все команды вызываются как подкоманды к основной команде `lektor`. Например:
```bash
lektor build
```
#### Доступные параметры:
- `--project PATH` — путь к проекту. Если не указан, Lektor ищет проект вверх по директориям до тех пор, пока не найдёт `.lektorproject`.
- `--language LANG` — язык интерфейса администратора (по умолчанию `en`). Поддерживаются: `ca`, `de`, `en`, `es`, `fr`, `it`, `ja`, `ko`, `nl`, `pl`, `pt`, `ru`, `zh`.
- `--version` — выводит текущую версию Lektor и завершает выполнение.
- `--help` — выводит справку по интерфейсу командной строки.
### 10.2 Переменные окружения
Lektor поддерживает несколько переменных окружения, влияющих на поведение CLI.
#### `LEKTOR_PROJECT`
Альтернатива флагу `--project`. Указывает путь к проекту, который следует использовать. Если не задано ни это, ни `--project`, поиск проекта будет выполнен от текущей директории вверх.
#### `LEKTOR_OUTPUT_PATH`
Позволяет задать нестандартный путь для выходной сборки.  
По умолчанию используется уникальный путь внутри кэш-папки Lektor. Параметр `--output-path` (если указан) имеет приоритет над этой переменной.
