# Creating Dockerfile, docker-compose.yml

## Основные команды (обязательные)
___
### FROM — базовый образ
Всегда первая команда. Задает основу для контейнера.

```dockerfile
# Полная версия Debian 12 (Bookworm)                1.02 GB
FROM python:3.13-bookworm
```

**Когда использовать:** практически никогда для продакшена. Этот образ включает в себя все пакеты Debian.

```dockerfile
# Урезанная версия Debian 12 (Bookworm)             ~124 MB
FROM python:3.13-slim-bookworm
```

**Когда использовать:** оптимальный выбор для продакшена.
- Все преимущества Debian Bookworm (стабильность, долгосрочная поддержка до 2028 года, glibc)
- Совместимость со всеми зависимостями (psycopg2-binary, Celery, DRF)
- Минимальный размер без потери функциональности
- Python 3.13 уже стабилен и готов к production

```dockerfile
# Alpine Linux                                      ~44 MB
FROM python:3.13-alpine
```

**Когда использовать:** только если размер образа критичен и вы готовы к дополнительным сложностям.
- Использует musl libc вместо glibc
- Может потребоваться компиляция некоторых пакетов
- Для PostgreSQL нужно будет устанавливать дополнительные пакеты:

*glibc (GNU C Library) и musl — это две разные реализации стандартной библиотеки языка C. Они выполняют одну и ту же базовую работу, но их различия критически важны для выбора базового образа вашего Docker-контейнера. 
Для Django-приложения: когда Celery читает задачу из Redis, PostgreSQL сохраняет данные, или Nginx отдает статику — все это работает через glibc.*

```dockerfile
FROM python:3.13-alpine
RUN apk add --no-cache postgresql-dev gcc musl-dev python3-dev
RUN pip install psycopg2  # psycopg2-binary может не работать
```

**Совет:** для продакшена используйте ```-slim-bookworm``` или ```-alpine``` версии — они значительно меньше по размеру.

**Рекомендация:** для нового проекта смело используйте ```FROM python:3.13-slim-bookworm```. Это современный, быстрый и стабильный выбор. Для существующего проекта, который работает на 3.11, переходите на 3.13 после тестирования всех зависимостей.

**Если очень хочется использовать Python 3.14**, технически это сделать можно — образы уже существуют. Но для продакшена с Django, Celery и PostgreSQL это пока что похоже на игру в русскую рулетку. 
Образы Python 3.14 доступны, например ```python:3.14-slim``` или ```sourcemation/python-3.14:latest```. Однако есть важный нюанс: образы с тегом ```slim``` по умолчанию базируются на новом ```Debian Trixie```, который еще не вышел как стабильный релиз.

**Алгоритм выбора версии Python и базового образа**

1. **Определите требования проекта:**
   - Какие версии Python поддерживают ваши зависимости?
   - Есть ли C-расширения (psycopg2, numpy, pillow)?

2. **Выберите версию Python:**
   - **Python 3.11** — максимальная стабильность, LTS до 2027
   - **Python 3.13** — современная, стабильная, готова к production
   - **Python 3.14** — только для экспериментов (Debian Trixie не стабилен)

3. **Выберите базовый образ:**
   - **slim-bookworm** — для production (стабильность + совместимость)
   - **alpine** — только если размер критичен (готовьтесь к проблемам)

4. **Проверьте совместимость:**
   ```bash
   # Локально проверяем, что все работает
   docker run --rm python:3.13-slim-bookworm python -c "import django; print(django.__version__)"
___

### WORKDIR — рабочая директория
Задает папку, в которой будут выполняться все последующие команды.

```dockerfile
WORKDIR /app
# Все команды дальше будут выполняться внутри /app
```
Совет: 
- **Всегда начинайте с /:** WORKDIR /app или WORKDIR /usr/src/app.
- **Не используйте cd в RUN:** для смены контекста есть WORKDIR.
- **Комбинируйте с ENV:** для большей гибкости используйте переменные окружения.
- **Помните о правах:** если вы создаете непривилегированного пользователя (что является лучшей практикой), убедитесь, что он имеет права на директорию WORKDIR.

В итоге, Dockerfile может выглядеть так:

``` dockerfile
# Задаем основу для контейнера
FROM python:3.13-slim-bookworm

# Переменная для домашней директории приложения
ENV APP_HOME /app/my_app

# Создаем и переключаемся в абсолютную папку
WORKDIR ${APP_HOME}

# ... остальные инструкции
```
Что произойдет при сборке:
```text
Шаг 1: ENV APP_HOME /app/my_app
        → Переменная APP_HOME = "/app/my_app"

Шаг 2: WORKDIR ${APP_HOME}
        → Docker проверяет: есть ли /app/my_app?
        → Нет? Создает!
        → Переключается в /app/my_app
        → Все следующие команды (COPY, RUN, CMD) будут выполняться внутри /app/my_app
```

**Можно еще лучше — использовать более стандартные имена**
На практике чаще используют такие варианты:

```dockerfile
# Вариант 1: Самый распространенный
ENV APP_HOME /app
WORKDIR ${APP_HOME}

# Вариант 2: С указанием проекта
ENV APP_HOME /app/my_app
WORKDIR ${APP_HOME}

# Вариант 3: Стандарт для Python-проектов
ENV APP_HOME /usr/src/app
WORKDIR ${APP_HOME}
```

**Какой выбрать?**

```/app``` — если у вас один проект в контейнере (рекомендую)

```/app/my_app``` — если вы планируете держать несколько проектов в одном контейнере

```/usr/src/app``` — классический путь для исходников в Unix-системах
___

### RUN — выполнение команд

**RUN** выполняет команды во время сборки образа. Это основная инструкция для установки пакетов, настройки системы и подготовки окружения.

**Синтаксис:**
```dockerfile
# Shell-форма (выполняется через /bin/sh)
RUN command param1 param2

# Exec-форма (рекомендуется для сложных команд)
RUN ["executable", "param1", "param2"]
```
**Использование:**


1. Установка системных пакетов
```dockerfile
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

2. Установка Python-пакетов
```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

3. Установка Poetry
```dockerfile
RUN pip install poetry==1.8.3
```

4. Настройка системы
```dockerfile
RUN adduser --disabled-password --no-create-home appuser
```

5. Создание директорий
```dockerfile
RUN mkdir -p /app/static /app/media /app/logs
```
**Важно:**

1. Объединяйте команды в один RUN:

```dockerfile
# ❌ Плохо: 3 слоя
RUN apt-get update
RUN apt-get install -y gcc
RUN apt-get install -y libpq-dev

# ✅ Хорошо: 1 слой
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```
2. Очищайте кеш:

```dockerfile
# ✅ Очищаем кеш apt
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# ✅ Очищаем кеш pip
RUN pip install --no-cache-dir -r requirements.txt
```
3. Используйте кеширование:

```dockerfile
# ✅ Копируем зависимости ДО кода
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt  # ← Кешируется

COPY . .  # ← Меняется часто
```
**Когда использовать:**

- Установка пакетов (apt, pip, poetry)

- Создание пользователей

- Создание директорий

- Настройка прав доступа

- Любые операции, которые должны быть выполнены во время сборки

**Важно:** `RUN` выполняется только во время сборки, а не при запуске контейнера.

**Золотое правило:** `RUN` — это сердце сборки. Он выполняет все команды по установке и настройке. Без него контейнер не будет содержать ни пакетов, ни кода, ни пользователя.
___

### COPY и ADD — копирование файлов

**COPY** — стандартный и предпочтительный способ
```dockerfile
COPY <источник> <назначение>
```
**Что умеет:**
- Копировать файлы и папки с хоста в контейнер
- Копировать из предыдущих этапов сборки (multi-stage)

**Что НЕ умеет:**
- Распаковывать архивы
- Скачивать файлы по URL

**ADD** — расширенная версия (используйте осторожно)
```dockerfile
ADD <источник> <назначение>
```
**Дополнительные возможности (кроме COPY):**
- Распаковка архивов — если источник — локальный архив (.tar, .tar.gz, .tgz, .bzip2, .xz), он автоматически распакуется
- Скачивание по URL — может скачать файл из интернета

```dockerfile
# COPY так НЕ СМОЖЕТ
ADD https://example.com/file.tar.gz /tmp/   # Скачает файл
ADD archive.tar.gz /app/                    # Распакует архив в /app

# COPY умеет только копировать
COPY file.tar.gz /app/                     # Просто скопирует архив как есть
```

**Оптимизация кеширования**

**Проблема без оптимизации:**
```dockerfile
FROM python:3.13-slim-bookworm
ENV APP_HOME /app/my_app
WORKDIR ${APP_HOME}

# ❌ ПЛОХО: копируем ВСЕ сразу
COPY . /app/
RUN pip install -r requirements.txt
```
**Что произойдет:**

1. Docker копирует все файлы проекта
2. Запускает pip install
3. Если вы измените ЛЮБОЙ файл в проекте (даже README.md) — весь слой COPY . инвалидируется
4. Docker пересоберет ВСЕ: и копирование, и установку зависимостей
5. Сборка будет медленной (особенно с тяжелыми зависимостями)

**Решение: Разделите копирование**
```dockerfile
FROM python:3.13-slim-bookworm
ENV APP_HOME /app/my_app
WORKDIR ${APP_HOME}

# ✅ ШАГ 1: Копируем ТОЛЬКО requirements
COPY requirements.txt /app/
RUN pip install --no-cache-dir -r requirements.txt

# ✅ ШАГ 2: Копируем ВСЁ ОСТАЛЬНОЕ
COPY . /app/
```
**Почему это работает:**
1. Docker кеширует слой с ```pip install...```
2. Если вы меняете код (но НЕ меняете ```requirements.txt```) — Docker НЕ ПЕРЕУСТАНАВЛИВАЕТ зависимости
3. Он просто копирует новый код и использует закешированный слой с зависимостями
4. Сборка занимает секунды, а не минуты!

**Важно:** для оптимизации сборки сначала копируйте файлы с зависимостями, потом все остальное (это использует кеш Docker).

**Копирование нескольких файлов**
```dockerfile
# Можно перечислить несколько источников
COPY requirements.txt setup.py README.md /app/
```
**Использование .dockerignore**
Создайте файл .dockerignore в корне проекта:

```text
# .dockerignore
.git
.gitignore
.env
.env.local
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so
*.egg
*.egg-info
dist
build
venv
.venv
env
.env
.coverage
htmlcov
.pytest_cache
.mypy_cache
.DS_Store
static/
media/
*.log
*.pid
*.sqlite3
```
Это ускорит сборку, так как Docker не будет копировать ненужные файлы.

**Использование Poetry вместо чистого pip** — это хорошее решение для управления зависимостями. Однако оно требует изменения подхода в Dockerfile, чтобы не потерять в скорости сборки и не раздуть итоговый образ.

Вот как перестроить Dockerfile под Poetry, используя лучшие практики.

**Ключевые отличия при работе с Poetry**
Чтобы сборка была эффективной, нужно учитывать несколько моментов:

- **Виртуальное окружение:** Poetry по умолчанию создает виртуальное окружение. Лучшая практика — создать его внутри проекта, установив ```POETRY_VIRTUALENVS_IN_PROJECT=true```. Так все зависимости будут лежать в папке .venv, которую легко скопировать на финальный этап.

- **Разделение зависимостей и кода:** Главное правило ускорения сборки — копировать ```pyproject.toml``` и ```poetry.lock``` до установки зависимостей, а сам код — после. Это позволяет кешировать слой с ```poetry install...```, пока не изменились файлы с зависимостями.


**Простой Dockerfile c poetry**
```dockerfile
FROM python:3.13-slim-bookworm
ENV APP_HOME /app/my_app

# 1. Отключаем создание виртуального окружения (главный трюк!)
ENV POETRY_VIRTUALENVS_CREATE=false

# 2. Устанавливаем Poetry и зависимости системы
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

RUN pip install poetry==1.8.3

# 3. Рабочая директория
WORKDIR ${APP_HOME}

# 4. Копируем зависимости (для кеширования)
COPY pyproject.toml poetry.lock* ./
# Звездочка (*) корректно обрабатывает отсутствие файла. 
# Если poetry.lock нет — команда просто скопирует pyproject.toml.

# 5. Устанавливаем зависимости (без виртуального окружения)
RUN poetry install --no-interaction --no-ansi --no-root --only main

# 6. Копируем код
COPY . .

# 7. Запускаем
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

**Пошаговое описание работы Dockerfile (Poetry-версия)**

**Шаг 1: Берется образ python:3.13-slim-bookworm**
``` dockerfile
FROM python:3.13-slim-bookworm
```
- Docker скачивает из Docker Hub официальный образ Python 3.13

- Базовый дистрибутив — Debian 12 (Bookworm)

- Версия slim — урезанная, без лишних пакетов

- **Итог:** Легкий контейнер с Python 3.13 (~124 МБ)

**Шаг 2: Создается переменная APP_HOME=/app/my_app**
``` dockerfile
ENV APP_HOME /app/my_app 
```
- Создает переменную окружения
- Переменная доступна внутри контейнера

- **Итог:** Теперь можно использовать ```${APP_HOME}``` в других командах

**Шаг 3: Отключается создание виртуального окружения Poetry**
``` dockerfile
ENV POETRY_VIRTUALENVS_CREATE=false
``` 
- Poetry по умолчанию создает виртуальное окружение в случайной папке. С этой переменной Poetry устанавливает все пакеты глобально (в системный Python)

- **Итог:** Пакеты становятся сразу доступны без активации виртуального окружения

**Шаг 4: Устанавливаются системные зависимости**
```dockerfile
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```
- ```apt-get update``` — обновляет список доступных пакетов

- ```apt-get install -y gcc libpq-dev``` — устанавливает:

  - ```gcc``` — компилятор C (нужен для сборки некоторых Python-пакетов)

  - ```libpq-dev``` — заголовочные файлы PostgreSQL (нужны для psycopg2)

- ```rm -rf /var/lib/apt/lists/*``` — удаляет кеш apt (экономит место)

- Все команды объединены в один RUN — это уменьшает количество слоев

- **Итог:** Установлены инструменты для сборки Python-пакетов

**Шаг 5: Устанавливается Poetry**
``` dockerfile
RUN pip install poetry==1.8.3
```

- Устанавливается конкретная версия Poetry (1.8.3). Фиксация версии важна для воспроизводимости сборок

- **Итог:** Poetry установлен и готов к использованию

**Шаг 6: Создается папка /app/my_app и Docker переключается в нее**
``` dockerfile
WORKDIR ${APP_HOME}
``` 
- Docker проверяет, существует ли папка ```/app/my_app```

- Если нет — создает её (вместе с родительской папкой ```/app```)

- Переключает контекст на эту папку

- **Итог:** Все следующие команды будут выполняться внутри ```/app/my_app```

**Шаг 7: Копируются файлы зависимостей**
``` dockerfile
COPY pyproject.toml poetry.lock* ./
```

- Копирует ```pyproject.toml``` (обязательный файл)

- Копирует ```poetry.lock``` (если есть — звездочка обрабатывает отсутствие)

- В текущую директорию контейнера ```/app/my_app/```

- **Итог:** В контейнере появились файлы с описанием зависимостей

**Шаг 8: Poetry устанавливает все зависимости глобально**
``` dockerfile
RUN poetry install --no-interaction --no-ansi --no-root --only main
```

- ```--no-interaction``` — не задавать вопросы в интерактивном режиме

- ```--no-ansi``` — отключает цветной вывод (упрощает логи)

- ```--no-root``` — не устанавливать сам проект как пакет (только зависимости)
- ```--only main``` - установка только основных зависимостей без dev

- Благодаря ```POETRY_VIRTUALENVS_CREATE=false``` пакеты устанавливаются глобально

- **Итог:** Все зависимости (Django, DRF, Celery, psycopg2 и др.) установлены в системный Python

**Шаг 9: Копируется весь код проекта**
``` dockerfile
COPY . .
``` 
- Копирует все файлы из локальной папки в текущую директорию контейнера ```/app/my_app/```

- Первая точка — "все файлы из текущей папки на хосте"

- Вторая точка — "скопируй в текущую папку контейнера"

- **Итог:** Весь Django-проект (manage.py, приложения, настройки) оказался внутри контейнера

**Шаг 10: При запуске контейнера выполняется python manage.py runserver 0.0.0.0:8000**
``` dockerfile
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

- Это команда по умолчанию, которая выполняется при docker run

- Django запускает встроенный сервер разработки

- 0.0.0.0 — слушает все сетевые интерфейсы (чтобы было доступно с хоста)

- 8000 — порт, на котором работает приложение

- **Итог:** Django-приложение запускается на порту 8000

**Результат**
- Django-приложение запущено и доступно по адресу http://localhost:8000

*Отключение виртуального окружения ```ENV POETRY_VIRTUALENVS_CREATE=false``` - это самый важный трюк! Без него Poetry создает виртуальное окружение в случайной папке, и Python не видит пакеты. С этой переменной Poetry устанавливает все пакеты глобально (в системный Python), и они сразу доступны.*

**Еще проще: без Poetry**
Если вы хотите максимально упростить и ускорить, откажитесь от Poetry в Docker и используйте классический ```requirements.txt```:

```dockerfile
FROM python:3.13-slim-bookworm
ENV APP_HOME /app/my_app

WORKDIR ${APP_HOME}

# Копируем и устанавливаем зависимости
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем код
COPY . .

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
**Плюсы:**
- На 70% меньше строк кода
- Не нужно устанавливать Poetry в контейнер
- Сборка быстрее

**Минусы:**
- Нет poetry.lock для точного контроля версий
- Управление зависимостями вручную через ```pip freeze > requirements.txt```

**Пошаговое описание работы Dockerfile (pip-версия)**

**Шаг 1: Берется образ python:3.13-slim-bookworm**
``` dockerfile
FROM python:3.13-slim-bookworm
```
- Docker скачивает из Docker Hub официальный образ Python 3.13

- Базовый дистрибутив — Debian 12 (Bookworm)

- Версия slim — урезанная, без лишних пакетов

- **Итог:** Легкий контейнер с Python 3.13 (~124 МБ)

**Шаг 2: Создается переменная APP_HOME=/app/my_app**
``` dockerfile
ENV APP_HOME /app/my_app 
```
- Создает переменную окружения

- Переменная доступна внутри контейнера

- **Итог:** Теперь можно использовать ${APP_HOME} в других командах

**Шаг 3: Создается папка /app/my_app и Docker переключается в нее**
``` dockerfile
WORKDIR ${APP_HOME}
```

- Docker проверяет, существует ли папка /app/my_app

- Если нет — создает её (вместе с родительской папкой /app)

- Переключает контекст на эту папку

- **Итог:** Все следующие команды будут выполняться внутри /app/my_app

**Шаг 4: Копируется requirements.txt**
``` dockerfile
COPY requirements.txt . 
```
- Копирует файл с зависимостями из локальной папки проекта в текущую директорию контейнера

- **Итог:** В контейнере появился файл /app/my_app/requirements.txt

**Шаг 5: Устанавливаются все Python-зависимости**
``` dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

- pip читает список пакетов из requirements.txt

- Скачивает и устанавливает все зависимости (Django, DRF, Celery, psycopg2 и др.)

- --no-cache-dir — не сохраняет кеш (экономит место)

- **Итог:** Все пакеты установлены глобально в системный Python контейнера

**Шаг 6: Копируется весь код проекта**
``` dockerfile
COPY . . 
```
- Копирует все файлы из локальной папки в текущую директорию контейнера ```/app/my_app/```

- Первая точка — "все файлы из текущей папки на хосте"

- Вторая точка — "скопируй в текущую папку контейнера"

- **Итог:** Весь Django-проект (manage.py, приложения, настройки) оказался внутри контейнера

**Шаг 7: При запуске контейнера выполняется python manage.py runserver 0.0.0.0:8000**
``` dockerfile
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

- Это команда по умолчанию, которая выполняется при docker run

- Django запускает встроенный сервер разработки

- 0.0.0.0 — слушает все сетевые интерфейсы (чтобы было доступно с хоста)

- 8000 — порт, на котором работает приложение

- **Итог:** Django-приложение запускается на порту 8000

**Результат**
- Django-приложение запущено и доступно по адресу http://localhost:8000

___
### CMD — команда запуска

**CMD** задает команду по умолчанию, которая выполняется при запуске контейнера.

**Синтаксис:**
```dockerfile
# Exec-форма (рекомендуется)
CMD ["executable", "param1", "param2"]

# Shell-форма (не рекомендуется)
CMD executable param1 param2
```
**Примеры:**

```dockerfile
# Разработка
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]

# Продакшен (Gunicorn)
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "config.wsgi:application"]

# С ENTRYPOINT (использование ENTRYPOINT описано в другом разделе)
ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "config.wsgi:application"]
```
**Важно:**

- Только один CMD в Dockerfile (последний побеждает)

- CMD легко переопределяется при запуске

- Всегда используйте Exec-форму

- Без CMD контейнер падает сразу

- Размещайте CMD в конце Dockerfile

**Переопределение при запуске:**

```bash
# Используем CMD из Dockerfile
docker run myapp

# Переопределяем CMD
docker run myapp python manage.py shell

# Переопределяем CMD с аргументами
docker run myapp python manage.py createsuperuser --noinput
```

___

## Дополнительные команды

### EXPOSE

EXPOSE — это информационная инструкция, которая НЕ открывает порты, а документирует, какие порты приложение будет слушать внутри контейнера.

```dockerfile
# Для Django/DRF
EXPOSE 8000

# Для Nginx
EXPOSE 80
EXPOSE 443
```

**Зачем:**

- Это ничего не ломает

- Это помогает другим (и вам в будущем) понять, какой порт использует приложение

- Это позволяет использовать docker run -P и Docker Compose с expose

- Это стандарт, которого придерживаются все официальные образы

***Но если забудете — ничего страшного, всё будет работать!***

Пример использования:
``` dockerfile
# Dockerfile
FROM python:3.13-slim-bookworm

... Остальные инструкции

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
___
### USER
USER — переключает контейнер на непривилегированного пользователя для выполнения всех последующих команд.

```dockerfile
# Создаем пользователя
RUN adduser --disabled-password --no-create-home appuser

# Переключаемся на него
USER appuser

# Все команды после этого выполняются от appuser
CMD ["gunicorn", ...]
```

Представьте, что контейнер — это квартира:

- ```root``` — это хозяин квартиры, у которого есть доступ ко всем комнатам, сейфам и документам

- ```appuser``` — это гость, который может только сидеть в гостиной и пользоваться телевизором

Запускать приложение от root — это как дать гостю ключи от всех сейфов в квартире. Он может случайно (или намеренно) что-то сломать.

**Важно:** в продакшене никогда не запускайте приложение от root. Приложению нужен доступ только к своим файлам и ресурсам, не больше. ```root``` имеет доступ ко всему.

**Создание пользователя**
```dockerfile
# Вариант 1: Самый простой
RUN adduser --disabled-password --no-create-home appuser

# Вариант 2: С конкретным UID (рекомендуется)
RUN adduser --disabled-password --no-create-home --uid 1000 appuser

# Вариант 3: С созданием домашней директории
RUN adduser --disabled-password --gecos '' appuser

# Вариант 4: Для Alpine Linux
RUN adduser -D -H -s /sbin/nologin appuser
```
Что означают флаги:

- ```--disabled-password``` — не запрашивать пароль (не нужен)

- ```--no-create-home``` — не создавать домашнюю папку (экономит место)

- ```--uid 1000``` — задать конкретный ID пользователя

- ```--gecos ''``` — пропустить заполнение информации о пользователе

Переключение на пользователя
```dockerfile
# Переключаемся на appuser
USER appuser

# Все команды после этого выполняются от appuser
CMD ["gunicorn", ...]
```

**Полный пример с USER**
```dockerfile
FROM python:3.13-slim-bookworm

# ============================================
# Переменные
# ============================================
ENV APP_HOME /app/my_app/
ENV APP_USER=appuser
ENV APP_UID=1000
ENV APP_GID=1000

# ============================================
# Создаем группу и пользователя
# ============================================
RUN groupadd --gid ${APP_GID} ${APP_USER} && \
    useradd --uid ${APP_UID} --gid ${APP_GID} --no-create-home ${APP_USER}

# ============================================
# Устанавливаем системные зависимости
# ============================================
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# ============================================
# Копируем requirements.txt и устанавливаем зависимости
# ============================================
WORKDIR ${APP_HOME}

# Копируем только зависимости (для кеширования)
COPY requirements.txt .

# ✅ Устанавливаем зависимости через pip
RUN pip install --no-cache-dir -r requirements.txt

# ============================================
# Копируем код и даем права
# ============================================
COPY . .

# Даем права пользователю на все файлы
RUN chown -R ${APP_USER}:${APP_USER} ${APP_HOME}

# ============================================
# Документируем порт
# ============================================
EXPOSE 8000

# ============================================
# Переключаемся на непривилегированного пользователя
# ============================================
USER ${APP_USER}

# ============================================
# Запускаем приложение
# ============================================
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

**Важные нюансы с USER**

**Нюанс 1: Права на файлы**

**Проблема:** Вы копируете файлы от root, а запускаете от appuser → приложение не может писать в файлы.

**Решение 1:** Дать права на папку:

```dockerfile
RUN chown -R appuser:appuser /app
USER appuser
```
**Решение 2:** Копировать с указанием владельца:

```dockerfile
COPY --chown=appuser:appuser . /app
USER appuser
```

**Нюанс 2: Установка зависимостей**

**Проблема:** Некоторые пакеты требуют установку в системную папку, куда appuser не может писать.

**Решение:** Устанавливать зависимости ДО переключения на ```appuser```:

```dockerfile
# ✅ Правильно
RUN pip install poetry==1.8.3  # от root
USER appuser

# ❌ Неправильно
USER appuser
RUN pip install poetry==1.8.3  # ошибка: permission denied
```
**Нюанс 3: Порт ниже 1024**

**Проблема:** Не ```root``` пользователь не может слушать порты ниже 1024:

```dockerfile
# ❌ Ошибка: Permission denied
USER appuser
CMD ["python", "manage.py", "runserver", "0.0.0.0:80"]
```
**Решение:** Использовать порты выше 1024:

```dockerfile
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]  # ✅
```
**Нюанс 4: Обновление пакетов**

**Проблема:** Appuser не может выполнять apt-get update (нужны права root).

**Решение:** Все системные операции выполнять ДО переключения:

```dockerfile
# ✅ Правильно: системные операции от root
RUN apt-get update && apt-get install -y curl
USER appuser

# ❌ Неправильно: appuser не может ставить пакеты
USER appuser
RUN apt-get update  # ошибка!
```

**Золотое правило:** ```root``` — только для установки пакетов и настройки. Приложение запускаем от ```appuser```. 

**Инструкция USER не обязательна, потому что:**

1. Простота важнее безопасности: Docker начинался как инструмент для разработчиков, где приоритетом была скорость и удобство .

2. Это рекомендация, а не закон: Безопасность — это всегда баланс риска и удобства. Для локальной разработки риск минимален .

3. Есть другие способы: Вы можете добиться того же уровня безопасности на этапе запуска контейнера, не изменяя сам Dockerfile .

Параметр **USER** — это важная часть лучших практик, но не жесткое требование. Решение о его использовании остается за разработчиком и зависит от того, куда и как будет развертываться приложение 
___

### VOLUME
**VOLUME** — это инструкция, которая создает точку монтирования внутри контейнера для хранения данных или обмена между контейнерами.

**Синтаксис**
```dockerfile
# Базовый синтаксис
VOLUME <путь>

# Примеры
VOLUME /app/static
VOLUME /app/media
VOLUME /var/log/nginx

# Можно указать несколько
VOLUME ["/app/static", "/app/media", "/var/log/nginx"]
```

**VOLUME в Dockerfile — это "Документация + Подстраховка"**
Использование VOLUME в Dockerfile без дополнительных команд — это, по сути, создание временного (анонимного) хранилища.

**Главная цель:** Указать другим разработчикам (и себе в будущем): "В этой папке лежат важные данные, которые не должны храниться внутри контейнера!"

**На практике:** Вы редко будете полагаться на анонимный том, который создается по умолчанию. Вместо этого вы будете использовать ```docker run -v my_volume:/app/data``` или ```docker-compose.yml```, чтобы подменить этот анонимный том на свой именованный или привязать папку с хоста.

**Docker Compose — это "Центр управления"**

**Один файл — вся инфраструктура:** Вы запускаете одним кликом и Nginx, и ваше Django-приложение, и базу данных, и Redis.

**Четкое объявление томов:** Именно в docker-compose.yml вы объявляете именованные тома или bind mount (для разработки). Это дает вам полный контроль над данными.

```yaml
# docker-compose.yml
services:
  django:
    build: .
    volumes:
      # Именованный том для статики
      - static_volume:/app/static  
      # Именованный том для медиа
      - media_volume:/app/media   
      # Bind mount для разработки (чтобы не пересобирать образ при  изменении кода) 
      - ./local_code:/app          

  nginx:
    image: nginx:alpine
    volumes:
      # Тот же том, что и у Django
      - static_volume:/var/www/static  
      # Bind mount, чтобы править конфиг на хосте
      - ./nginx.conf:/etc/nginx/conf.d/default.conf 

volumes:
  static_volume:  # <-- Просто объявляем, Docker сам создаст их при первом запуске
  media_volume:
```
**Как это работает на практике:**
1. Сборка статики (Django):

- Внутри контейнера django вы выполняете команду: ```python manage.py collectstatic```.

- Эта команда собирает все статические файлы (CSS, JS, картинки) и складывает их в папку ```/app/static```.

- Но ```/app/static``` внутри контейнера django — это не просто папка на диске, это вход в том ```static_volume```.

- **Результат:** Все файлы попадают в общее хранилище ```static_volume```.

2. Раздача статики (Nginx):

- Контейнер nginx настроен на то, чтобы отдавать статические файлы из папки ```/var/www/static```.

- ```/var/www/static``` внутри контейнера nginx — это точно такой же вход в том ```static_volume```.

- **Результат:** Nginx видит те же самые файлы, которые туда положил Django.

**Если бы мы создали два разных тома:**

```yaml
volumes:
  django_static:
  nginx_static:
```
То Django бы складывал файлы в ```django_static```, а Nginx пытался бы читать из ```nginx_static``` — и ничего бы не работало. Nginx бы просто отдавал ```404``` ошибки.

***Поэтому один том на двоих — это правильное и элегантное решение для обмена данными между контейнерами.***

**Итоговое понимание**
```VOLUME /app/static``` в Dockerfile — это как подпись на коробке: "Здесь хрупкое". Вы знаете, что это место для данных.

```docker-compose.yml``` с ```volumes:``` — это как инструкция для грузчиков: "Эту коробку (том) положите сюда, а вот эту часть кода (bind mount) мы будем менять на ходу".

***"Не заморачивайтесь с управлением данными внутри Dockerfile, а делайте это централизованно через docker-compose.yml."***

___
### HEALTHCHECK
**HEALTHCHECK** — это инструкция, которая говорит Docker: "Регулярно проверяй, работает ли приложение внутри контейнера".

**HEALTHCHECK** — это не магия. Это просто команда, которая выполняется внутри контейнера. Если в команде нет инструмента для проверки (например, ```curl``` или ```wget```) и нет **эндпоинта**, который этот инструмент может проверить — healthcheck работать не будет.



```dockerfile
# Проверка Django приложения (нужен endpoint)
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1

# Проверка PostgreSQL (не нужен endpoint)
HEALTHCHECK --interval=10s --timeout=5s \
  CMD pg_isready -U postgres || exit 1
```

**Представьте, что контейнер — это сотрудник в офисе:**

**Без HEALTHCHECK:** Вы просто видите, что сотрудник сидит на месте (контейнер запущен), но не знаете, работает ли он (приложение отвечает на запросы).

**С HEALTHCHECK:** Вы каждые 30 секунд подходите к сотруднику и спрашиваете: "Ты работаешь?" Если он отвечает — всё ок. Если нет — вы знаете, что что-то не так.

**Синтаксис**
```dockerfile
# Базовый синтаксис
HEALTHCHECK [опции] CMD <команда>

# Опции:
# --interval=30s    - как часто проверять (по умолчанию 30s)
# --timeout=5s      - таймаут на выполнение проверки (по умолчанию 30s)
# --start-period=5s - время на запуск приложения перед проверкой
# --retries=3       - сколько раз повторить перед пометкой unhealthy

# Пример
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1
```

**Что происходит под капотом**

Docker отслеживает статус контейнера:

|Статус	|Описание|
|--------|--------|
|starting	|Контейнер запускается (ждет start-period)|
|healthy	|✅ Приложение работает нормально|
|unhealthy|	❌ Приложение не отвечает на проверки|
```bash
docker ps
# CONTAINER ID   STATUS
# a1b2c3d4       Up 2 minutes (healthy)   ← ✅ Все хорошо
# e5f6g7h8       Up 30 seconds (starting) ← ⏳ Запускается
# i9j0k1l2       Up 1 hour (unhealthy)    ← ❌ Что-то сломалось
```

**Что нужно для работы HEALTHCHECK**
1. Инструмент внутри контейнера
```dockerfile
# В контейнере должен быть curl 
RUN apt-get update && apt-get install -y curl
# или wget
RUN apt-get update && apt-get install -y wget
```
2. Эндпоинт в приложении
```python
# Django должен отдавать /health/
urlpatterns = [
    path('health/', health_check, name='health_check'),
]
```
3. Команда, которая проверяет этот эндпоинт
```dockerfile
HEALTHCHECK CMD curl -f http://localhost:8000/health/ || exit 1
# или
HEALTHCHECK CMD wget --spider http://localhost:8000/health/ || exit 1
# Оба делают одно и то же: проверяют, что эндпоинт /health/ 
# возвращает успешный HTTP-статус (200 OK). 
```

**Healthcheck — это не про то, "что внутри контейнера", а про то, "как мы запускаем контейнер в конкретном окружении". Поэтому его место в docker-compose.yml, а не в Dockerfile.**

1. Разные окружения — разные требования. В разработке проверки могут быть реже и с большим таймаутом, а в продакшене — чаще и строже.

2. Без пересборки образа. Чтобы изменить интервал проверки, достаточно отредактировать ```docker-compose.yml``` и перезапустить контейнер. Не нужно пересобирать образ.

3. Все настройки сервиса в одном месте. Healthcheck — это часть конфигурации запуска, наравне с портами, томами и переменными окружения.

4. Зависимости от других сервисов. ```depends_on``` с ```condition: service_healthy``` работает только на уровне композа.
___
### ENTRYPOINT
**ENTRYPOINT** — это инструкция, которая задает фиксированную команду, которая всегда будет выполняться при запуске контейнера.

**Синтаксис**
```dockerfile
# Форма 1: Shell-форма (выполняется через /bin/sh)
ENTRYPOINT command param1 param2

# Форма 2: Exec-форма (рекомендуется)
ENTRYPOINT ["executable", "param1", "param2"]
```
Рекомендуется Exec-форма, так как она:

- Не создает лишний процесс оболочки

- Позволяет корректно обрабатывать сигналы (SIGTERM, SIGINT)

Чаще всего используется комбинация ```ENTRYPOINT``` + ```CMD```
```dockerfile
ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "config.wsgi:application"]
```
Что происходит:

1. Выполняется /entrypoint.sh (миграции, сбор статики)

2. Затем запускается gunicorn config.wsgi:application (из CMD)

**Пример entrypoint.sh для Django**
1. Файл entrypoint.sh
```bash
#!/bin/bash
set -e  # Останавливаем скрипт при ошибке

echo "=== Django Entrypoint ==="

# Ожидаем PostgreSQL (healthcheck)
echo "Waiting for PostgreSQL..."
while ! nc -z postgres 5432; do
  sleep 1
done
echo "PostgreSQL is ready!"

# Выполняем миграции
echo "Running migrations..."
python manage.py migrate --noinput

# Собираем статику
echo "Collecting static files..."
python manage.py collectstatic --noinput

# Запускаем команду из CMD
echo "Executing: $@"
exec "$@"
```

2. Dockerfile
```dockerfile
FROM python:3.13-slim-bookworm

... Остальные инструкции

# Копируем entrypoint и делаем исполняемым
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# ENTRYPOINT — фиксированная часть
ENTRYPOINT ["/entrypoint.sh"]

# CMD — изменяемая часть (может быть переопределена)
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "config.wsgi:application"]

EXPOSE 8000
```


**Итоговая стратегия**
|Сценарий	|ENTRYPOINT	|docker exec |Почему|
|-----------|----------|------------|------|
|Локальная разработка	|✅ Да	|❌ Нет	|Разработчик не должен помнить команды|
|Ручной деплой на сервер	|✅ Да	|❌ Нет	|Админ не должен помнить команды|
|CI/CD	|⚠️ Опционально|✅ Да	|CI/CD сам выполнит команды|

**Три подхода к развертыванию**

**1. Только ENTRYPOINT (рекомендуется)**
```dockerfile
# Dockerfile
ENTRYPOINT ["/entrypoint.sh"]
```
```yaml
# docker-compose.yml
services:
  django:
    build: .
    entrypoint: ["/entrypoint.sh"]
```
**Плюсы:**

- Единый подход

- Нельзя забыть команды

- Работает локально, на сервере и в CI/CD

**Минусы:**

- CI/CD не может отключить миграции (иногда нужно)

**2. Только ```docker exec```**
```dockerfile
# Dockerfile (нет ENTRYPOINT)
CMD ["gunicorn", "config.wsgi:application"]
```
```yaml
# Локально
services:
  django:
    build: .
    # Без entrypoint
```
```bash
# Ручной деплой (опасно!)
docker-compose up -d
docker-compose exec django python manage.py migrate
docker-compose exec django python manage.py collectstatic
```

```yaml
# CI/CD (безопасно)
jobs:
  deploy:
    steps:
      - run: |
          docker-compose up -d
          docker-compose exec django python manage.py migrate
          docker-compose exec django python manage.py collectstatic
```
**Плюсы:**

- Простой Dockerfile

- Гибкость в CI/CD

**Минусы:**

- ❌ Опасно для ручного деплоя

- ❌ Можно забыть команды

- ❌ Перезапуск контейнера требует повторения

**3. Комбинированный подход (рекомендуется для продакшена)**
```dockerfile
# Dockerfile
ENTRYPOINT ["/entrypoint.sh"]  # ← Всегда!
CMD ["gunicorn", "config.wsgi:application"]
```
```bash
# entrypoint.sh (базовый минимум)
#!/bin/bash
set -e

# ✅ Только ОБЯЗАТЕЛЬНЫЕ операции
python manage.py migrate --noinput
python manage.py collectstatic --noinput

exec "$@"
```
```yaml
# CI/CD (дополнительные команды)
jobs:
  deploy:
    steps:
      - run: |
          # entrypoint.sh выполняется автоматически
          docker-compose up -d  
          # Дополнительные операции
          docker-compose exec django python manage.py createsuperuser --noinput
          docker-compose exec django python manage.py loaddata fixtures.json
```
**Сводная таблица решений**
|Команда	|Где выполняется	|Когда|
|--------|----------------|-----|
|migrate	|В entrypoint.sh + CI/CD	|✅ При каждом запуске|
|collectstatic	|В entrypoint.sh + CI/CD	|✅ При каждом запуске
|createsuperuser	|В entrypoint.sh (условно) или CI/CD	|✅ Только при первом запуске|
|loaddata fixtures	|Только в CI/CD	|✅ Только при деплое|
|flush / reset_db	|Только в CI/CD	|✅ Только при необходимости|

***ENTRYPOINT — это страховка для ручных действий. В CI/CD мы можем делать всё что угодно, но базовые операции (миграции, статика) должны быть автоматизированы через ENTRYPOINT, чтобы разработчики и администраторы не забывали их выполнять.***
___

## Шпаргалка по командам
|Команда	|Когда использовать	|Пример|
|--------|-------------------|------|
|FROM|	Всегда первой	|FROM python:3.11-slim|
|WORKDIR|	Сразу после FROM	|WORKDIR /app|
|COPY|	Для копирования файлов	|COPY . /app|
|RUN|	Для установки пакетов	|RUN pip install -r requirements.txt|
|ENV|	Для постоянных переменных	|ENV DEBUG=False|
|EXPOSE|	Для документации портов	|EXPOSE 8000|
|VOLUME|	Для постоянных данных	|VOLUME /app/media|
|USER|	Для безопасности	|USER appuser|
|HEALTHCHECK|	Для мониторинга	|HEALTHCHECK CMD curl ...|
|ENTRYPOINT|	Для фиксированной команды	|ENTRYPOINT ["entrypoint.sh"]|
|CMD|	Для команды по умолчанию	|CMD ["gunicorn"]|
|ADD|	Редко (вместо COPY)	|ADD archive.tar.gz /tmp/|

___
## Бонус

### 📦 Вариант 1: pip + requirements.txt
**Dockerfile**
```dockerfile
FROM python:3.13-slim-bookworm

# Переменные окружения
ENV PYTHONUNBUFFERED=1 \ 
    # Отключает буферизацию вывода Python. 
    # Без нее логи появляются с задержкой
    PYTHONDONTWRITEBYTECODE=1 \
    # Запрещает создавать файлы *.pyc
    # Без нее в контейнере копится мусор
    APP_HOME=/app \
    APP_USER=appuser \
    APP_UID=1000

# Создаем непривилегированного пользователя
RUN groupadd --gid ${APP_UID} ${APP_USER} && \
    useradd --uid ${APP_UID} --gid ${APP_UID} --no-create-home ${APP_USER}

# Устанавливаем системные зависимости
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    curl \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

WORKDIR ${APP_HOME}

# Копируем зависимости (оптимизация кеширования)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем код
COPY . .

# Даем права пользователю на папку приложения
RUN chown -R ${APP_USER}:${APP_USER} ${APP_HOME}

# Копируем entrypoint и делаем исполняемым
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Переключаемся на непривилегированного пользователя
USER ${APP_USER}

EXPOSE 8000

# ENTRYPOINT для автоматизации миграций и статики
ENTRYPOINT ["/entrypoint.sh"]

# CMD для запуска приложения
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "config.wsgi:application"]
```

**requirements.txt**
```text
Django==5.1.7
djangorestframework==3.15.2
psycopg2-binary==2.9.10
python-dotenv==1.0.1
django-filter==24.3
djangorestframework-simplejwt==5.3.1
drf-spectacular==0.27.2
stripe==11.3.0
celery==5.4.0
redis==5.2.1
django-celery-beat==2.7.0
flower==2.0.1
gunicorn==23.0.0
```
### 📦 Вариант 2: Poetry
**Dockerfile**
```dockerfile
FROM python:3.13-slim-bookworm

# Переменные окружения
ENV PYTHONUNBUFFERED=1 \ 
    # Отключает буферизацию вывода Python. 
    # Без нее логи появляются с задержкой
    PYTHONDONTWRITEBYTECODE=1 \
    # Запрещает создавать файлы *.pyc
    # Без нее в контейнере копится мусор
    APP_HOME=/app \
    APP_USER=appuser \
    APP_UID=1000 \
    POETRY_VERSION=1.8.3 \
    POETRY_VIRTUALENVS_CREATE=false \
    POETRY_NO_INTERACTION=1

# Создаем непривилегированного пользователя
RUN groupadd --gid ${APP_UID} ${APP_USER} && \
    useradd --uid ${APP_UID} --gid ${APP_UID} --no-create-home ${APP_USER}

# Устанавливаем системные зависимости
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    curl \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

# Устанавливаем Poetry
RUN pip install "poetry==${POETRY_VERSION}"

WORKDIR ${APP_HOME}

# Копируем зависимости (оптимизация кеширования)
COPY pyproject.toml poetry.lock* ./
RUN poetry install --no-interaction --no-ansi --no-root

# Копируем код
COPY . .

# Даем права пользователю на папку приложения
RUN chown -R ${APP_USER}:${APP_USER} ${APP_HOME}

# Копируем entrypoint и делаем исполняемым
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Переключаемся на непривилегированного пользователя
USER ${APP_USER}

EXPOSE 8000

# ENTRYPOINT для автоматизации миграций и статики
ENTRYPOINT ["/entrypoint.sh"]

# CMD для запуска приложения
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "config.wsgi:application"]
```
**pyproject.toml**
```toml
[tool.poetry]
name = "myproject"
version = "0.1.0"
description = "Django DRF project with Celery"
authors = ["Your Name <email@example.com>"]

[tool.poetry.dependencies]
python = "^3.13"
Django = "^5.1.7"
djangorestframework = "^3.15.2"
psycopg2-binary = "^2.9.10"
python-dotenv = "^1.0.1"
django-filter = "^24.3"
djangorestframework-simplejwt = "^5.3.1"
drf-spectacular = "^0.27.2"
stripe = "^11.3.0"
celery = "^5.4.0"
redis = "^5.2.1"
django-celery-beat = "^2.7.0"
flower = "^2.0.1"
gunicorn = "^23.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.3.4"
black = "^24.10.0"
flake8 = "^7.1.1"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
### docker/entrypoint.sh (одинаков для обоих вариантов)
```bash
#!/bin/bash
set -e

echo "=== Entrypoint: Starting ==="

# Ожидание PostgreSQL
echo "Waiting for PostgreSQL..."
while ! nc -z postgres 5432; do
  sleep 1
done
echo "PostgreSQL is ready!"

# Ожидание Redis
echo "Waiting for Redis..."
while ! nc -z redis 6379; do
  sleep 1
done
echo "Redis is ready!"

# Применяем миграции
echo "Running migrations..."
python manage.py migrate --noinput

# Собираем статику
echo "Collecting static files..."
python manage.py collectstatic --noinput

# Создаем суперпользователя (если не существует)
echo "Creating superuser..."
python manage.py shell -c "
from django.contrib.auth import get_user_model;
User = get_user_model();
if not User.objects.filter(username='admin').exists():
    User.objects.create_superuser('admin', 'admin@example.com', 'admin123')
" || true

echo "=== Entrypoint: Done ==="

# Запускаем команду из CMD
exec "$@"
```
### 🐳 docker-compose.yml (общий для обоих вариантов)
```yaml
version: '3.8'

services:
  # ==========================================
  # Django + DRF
  # ==========================================
  django:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp-django:latest
    container_name: django
    restart: unless-stopped
    entrypoint: ["/entrypoint.sh"]
    command: ["gunicorn", "--bind", "0.0.0.0:8000", "config.wsgi:application"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
      - logs_volume:/app/logs
    env_file:
      - .env
    ports:
      - "8000:8000"
    networks:
      - app-network

  # ==========================================
  # Celery Worker
  # ==========================================
  celery:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp-celery:latest
    container_name: celery
    restart: unless-stopped
    entrypoint: ["/entrypoint.sh"]
    command: ["celery", "-A", "config", "worker", "--loglevel=info", "--concurrency=4"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - logs_volume:/app/logs
    env_file:
      - .env
    networks:
      - app-network

  # ==========================================
  # Celery Beat
  # ==========================================
  celery-beat:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp-celery-beat:latest
    container_name: celery-beat
    restart: unless-stopped
    entrypoint: ["/entrypoint.sh"]
    command: ["celery", "-A", "config", "beat", "--loglevel=info"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - logs_volume:/app/logs
    env_file:
      - .env
    networks:
      - app-network

  # ==========================================
  # PostgreSQL
  # ==========================================
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    networks:
      - app-network

  # ==========================================
  # Redis
  # ==========================================
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 15s
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - app-network

  # ==========================================
  # Nginx
  # ==========================================
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health/"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 5s
    depends_on:
      django:
        condition: service_healthy
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_volume:/var/www/static
      - media_volume:/var/www/media
    ports:
      - "80:80"
      - "443:443"
    networks:
      - app-network

  # ==========================================
  # Flower (мониторинг Celery)
  # ==========================================
  flower:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp-flower:latest
    container_name: flower
    restart: unless-stopped
    entrypoint: ["/entrypoint.sh"]
    command: ["celery", "-A", "config", "flower", "--port=5555"]
    depends_on:
      redis:
        condition: service_healthy
    ports:
      - "5555:5555"
    env_file:
      - .env
    networks:
      - app-network

# ==========================================
# Тома
# ==========================================
volumes:
  static_volume:
    name: myapp_static
  media_volume:
    name: myapp_media
  logs_volume:
    name: myapp_logs
  postgres_data:
    name: myapp_postgres
  redis_data:
    name: myapp_redis

# ==========================================
# Сеть
# ==========================================
networks:
  app-network:
    name: myapp_network
    driver: bridge
```
### 📁 Структура проекта
```text
myproject/
├── .env
├── .dockerignore
├── docker-compose.yml
├── Dockerfile
├── requirements.txt          # для pip
├── pyproject.toml            # для Poetry
├── poetry.lock               # для Poetry
├── manage.py
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── celery.py
├── docker/
│   └── entrypoint.sh
├── nginx/
│   └── nginx.conf
├── apps/
│   └── health/
│       ├── __init__.py
│       ├── views.py
│       └── urls.py
└── .github/
    └── workflows/
        └── deploy.yml
```
### 🔧 .env файл
```env
# Django
DJANGO_SETTINGS_MODULE=config.settings
DJANGO_SECRET_KEY=your-secret-key
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1,example.com

# Database
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=mydb

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0

# Celery
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0
```
### 📝 .dockerignore
```text
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.so
*.egg
*.egg-info
dist
build
.env
.venv
venv
.git
.gitignore
README.md
LICENSE
.dockerignore
Dockerfile
docker-compose*.yml
node_modules
*.log
.DS_Store
coverage
htmlcov
.pytest_cache
.mypy_cache
.ruff_cache
```
### 🚀 Запуск
**Разработка**
```bash
docker-compose up -d
docker-compose logs -f django
```
**Продакшен**
```bash
docker-compose -f docker-compose.prod.yml up -d
```
**Миграции (если не через entrypoint)**
```bash
docker-compose exec django python manage.py migrate
```
**Сбор статики**
```bash
docker-compose exec django python manage.py collectstatic
```
**Создание суперпользователя**
```bash
docker-compose exec django python manage.py createsuperuser
```
___
## Продолжение следует...
