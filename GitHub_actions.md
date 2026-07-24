```yaml
# ============================================================================
# deploy.yml - Полный CI/CD пайплайн для Django проекта
# ============================================================================
# ЧТО ЭТОТ ФАЙЛ ДЕЛАЕТ:
# 1. Собирает Docker образы приложения
# 2. Загружает их в Docker Hub
# 3. Подключается к серверу по SSH
# 4. Обновляет конфигурацию и перезапускает контейнеры
#
# ДЛЯ КОГО: Для разработчиков, которые хотят автоматизировать деплой
# КОГДА ИСПОЛЬЗОВАТЬ: При каждом пуше в main или вручную
# ============================================================================

# ============================================================================
# НАЗВАНИЕ WORKFLOW
# ============================================================================
# name - отображается в GitHub Actions вкладке "Actions"
# Можно назвать: "Deploy", "CD Pipeline", "Production Deployment"
# ============================================================================
name: Deploy to Production Server

# ============================================================================
# ТРИГГЕРЫ ЗАПУСКА (on)
# ============================================================================
# Определяет, когда workflow будет запускаться
# ============================================================================
on:
  # ---------- АВТОМАТИЧЕСКИЙ ЗАПУСК ----------
  # push - запускается при пуше в репозиторий
  # branches: [ main ] - только при пуше в ветку main
  # 
  # АЛЬТЕРНАТИВЫ:
  #   - branches: [ main, develop ] - несколько веток
  #   - branches-ignore: [ docs/* ] - игнорировать определенные ветки
  #   - paths: - запускать только при изменении определенных файлов
  #     paths:
  #       - 'src/**'
  #       - 'Dockerfile'
  #       - 'docker-compose.yml'
  push:
    branches: [ main ]
    # paths:
    #   - 'app/**'  # Запускать только если изменилась папка app
    #   - 'Dockerfile'  # Или если изменился Dockerfile
  
  # ---------- РУЧНОЙ ЗАПУСК ----------
  # workflow_dispatch - позволяет запустить workflow вручную из интерфейса GitHub
  # 
  # АЛЬТЕРНАТИВЫ:
  #   - schedule: - запуск по расписанию (для бэкапов, например)
  #     schedule:
  #       - cron: '0 2 * * *'  # Каждый день в 2 часа ночи
  workflow_dispatch:
    # Входные параметры для ручного запуска
    inputs:
      environment:
        description: 'Environment to deploy to'  # Описание параметра
        required: true  # Обязательный параметр
        default: 'production'  # Значение по умолчанию
        type: choice  # Тип - выпадающий список
        options:  # Доступные варианты
          - production
          - staging
          - development
      
      skip_migrations:
        description: 'Skip database migrations'
        required: false
        default: false
        type: boolean
      
      tag:
        description: 'Docker image tag to deploy (default: latest)'
        required: false
        default: 'latest'
        type: string

# ============================================================================
# ПЕРЕМЕННЫЕ ОКРУЖЕНИЯ (env)
# ============================================================================
# Глобальные переменные, доступные во всех jobs
# 
# ОТКУДА БЕРУТСЯ:
#   - Статические значения - прописаны здесь
#   - ${{ secrets.XXX }} - из GitHub Secrets (Settings → Secrets)
#   - ${{ vars.XXX }} - из GitHub Variables (Settings → Variables)
# ============================================================================
env:
  # ---------- РЕГИСТРЫ ДЛЯ ОБРАЗОВ ----------
  # DOCKER_REGISTRY - где хранятся образы
  # 
  # ВАРИАНТЫ:
  #   - docker.io - Docker Hub (по умолчанию)
  #   - ghcr.io - GitHub Container Registry (бесплатно для публичных репо)
  #   - registry.gitlab.com - GitLab Container Registry
  #   - your-registry.com - свой приватный реестр
  # 
  # ВЫБОР: docker.io (Docker Hub) - самый простой и популярный
  DOCKER_REGISTRY: docker.io
  
  # ---------- ИМЯ ОБРАЗА ----------
  # DOCKER_IMAGE_NAME - базовое имя для всех образов
  # 
  # ФОРМАТ:
  #   - Для Docker Hub: username/image-name
  #   - Для GHCR: ghcr.io/username/image-name
  # 
  # ОТКУДА: ${{ secrets.DOCKER_USERNAME }} - из GitHub Secrets
  # 
  # ВАРИАНТЫ:
  #   - Использовать имя пользователя из секретов
  #   - Использовать название репозитория: ${{ github.repository }}
  #   - Задать жестко: myapp
  DOCKER_IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/myapp
  
  # ---------- ТЕГ ОБРАЗА ----------
  # Для идентификации версии образа
  # 
  # ВАРИАНТЫ:
  #   - latest - всегда последняя версия (просто, но небезопасно)
  #   - ${{ github.sha }} - хэш коммита (точно идентифицирует версию)
  #   - ${{ github.run_number }} - номер запуска (монотонно возрастает)
  #   - 1.0.0 - ручное версионирование
  # 
  # ВЫБОР: используем SHA для точной идентификации
  IMAGE_TAG: ${{ github.sha }}
  
  # ---------- ПЕРЕМЕННЫЕ ДЛЯ ПОДКЛЮЧЕНИЯ К СЕРВЕРУ ----------
  # SERVER_HOST - IP адрес или домен сервера
  # SERVER_USER - имя пользователя для SSH подключения
  # 
  # ОТКУДА: из GitHub Secrets
  # 
  # ГДЕ БРАТЬ:
  #   1. SERVER_HOST: IP сервера (например, 192.168.1.100)
  #   2. SERVER_USER: deploy или root
  #   3. SSH_PRIVATE_KEY: приватный ключ для подключения
  # 
  # БЕЗОПАСНОСТЬ: Никогда не храните SSH ключи в коде!
  # Используйте GitHub Secrets: Settings → Secrets and variables → Actions
  SERVER_HOST: ${{ secrets.SERVER_HOST }}
  SERVER_USER: ${{ secrets.SERVER_USER }}
  SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}

# ============================================================================
# JOBS (ЗАДАЧИ)
# ============================================================================
# Jobs выполняются последовательно или параллельно
# ============================================================================
jobs:

  # ==========================================================================
  # JOB 1: Линтинг и проверка кода
  # ==========================================================================
  # Зачем: Проверить качество кода до сборки
  # Когда: Всегда, перед сборкой
  # ==========================================================================
  lint:
    # ---------- ИМЯ ----------
    # Отображается в интерфейсе GitHub Actions
    name: Lint & Type Check
    
    # ---------- RUNNER ----------
    # runs-on - на какой ОС запускать job
    # 
    # ВАРИАНТЫ:
    #   - ubuntu-latest - Ubuntu (самый популярный)
    #   - windows-latest - Windows (для .NET, редко)
    #   - macos-latest - macOS (для iOS приложений)
    #   - self-hosted - свой сервер (если есть)
    # 
    # ВЫБОР: ubuntu-latest - бесплатно, быстро, популярно
    runs-on: ubuntu-latest
    
    # ---------- ШАГИ ----------
    steps:
      # --- ШАГ 1: Загрузка кода ---
      - name: Checkout code
        # actions/checkout - официальное действие GitHub
        # Загружает код из репозитория в runner (специальную 
        # виртуальную машину, на которой GitHub выполняет ваш код)
        # 
        # Это стандартный способ
        uses: actions/checkout@v4
        # 
        # ПАРАМЕТРЫ (опционально):
        # with:
        #   fetch-depth: 0  # Загрузить всю историю (для тегов)
        #   ref: main  # Конкретная ветка
      
      # ============================================================================
      # ШАГ 2: Установка Python
      # ============================================================================
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          
          # ---------- КЕШИРОВАНИЕ ДЛЯ PIP ----------
          # cache: 'pip' - автоматическое кеширование пакетов pip
          # 
          # ЗАЧЕМ: Не скачивать пакеты при каждом запуске
          # КАК РАБОТАЕТ: Сохраняет ~/.cache/pip между запусками
          # 
          # ВАЖНО: Работает ТОЛЬКО с pip и requirements.txt
          # ДЛЯ POETRY: используйте отдельную настройку кеша (см. вариант Б)
          cache: 'pip'

      # ============================================================================
      # ШАГ 3: Установка зависимостей
      # ============================================================================
      # 
      # ВАРИАНТ А: PIP + REQUIREMENTS.TXT (простой способ)
      # Используйте если у вас есть файлы requirements.txt
      # ============================================================================
      - name: Install dependencies with pip
        run: |
          # pip install - устанавливает Python пакеты
          # 
          # ОТКУДА БЕРУТСЯ ПАКЕТЫ:
          #   - requirements.txt - основные зависимости
          #   - requirements-dev.txt - для разработки (линтеры, тесты)
          #
          # КОМАНДЫ:
          python -m pip install --upgrade pip  # Обновляем pip
          pip install -r requirements.txt      # Основные пакеты
          pip install -r requirements-dev.txt  # Пакеты для разработки
      
      # ============================================================================
      # ВАРИАНТ Б: POETRY (современный способ)
      # Используйте если у вас есть pyproject.toml и poetry.lock
      # ============================================================================
      # 
      # ВНИМАНИЕ: Для Poetry НЕ РАБОТАЕТ cache: 'pip' в шаге 2!
      # Нужно использовать отдельную настройку кеша (шаг 4)
      # 
      # Раскомментируйте шаги 3-5 и закомментируйте вариант А
      # ============================================================================
      
      # --- ШАГ 3 (для Poetry): Установка Poetry ---
      # - name: Install Poetry
      #   run: |
      #     # Устанавливаем Poetry через pip
      #     pip install poetry==1.8.3
      
      # --- ШАГ 4 (для Poetry): Кеширование зависимостей ---
      # - name: Cache Poetry dependencies
      #   uses: actions/cache@v3
      #   with:
      #     # Кешируем виртуальное окружение и кеш Poetry
      #     path: |
      #       .venv
      #       ~/.cache/pypoetry
      #     # Ключ кеша зависит от содержимого poetry.lock
      #     key: ${{ runner.os }}-poetry-${{ hashFiles('poetry.lock') }}
      #     restore-keys: |
      #       ${{ runner.os }}-poetry-
      
      # --- ШАГ 5 (для Poetry): Установка зависимостей ---
      # - name: Install dependencies with Poetry
      #   run: |
      #     # Настраиваем создание виртуального окружения в папке проекта
      #     poetry config virtualenvs.in-project true
      #     
      #     # Устанавливаем зависимости (без самого проекта)
      #     # --no-root - не устанавливать сам проект (только зависимости)
      #     # --no-interaction - не спрашивать подтверждения
      #     # --no-ansi - без цветов (для логов)
      #     poetry install --no-interaction --no-ansi --no-root

      # ============================================================================
      # ШАГ 4: Проверка форматирования
      # ============================================================================
      - name: Check formatting with Black
        run: |
          # Black - автоматический форматтер кода
          # 
          # ПАРАМЕТРЫ:
          #   --check - только проверить, не изменять
          #   --diff - показать различия
          #   --line-length 120 - длина строки
          # 
          # ВАРИАНТЫ:
          #   - black . --check --diff
          #   - black . --line-length 120 --check
          # 
          # ДЛЯ PIP: используйте black . --check --diff --line-length 120
          # ДЛЯ POETRY: используйте poetry run black . --check --diff --line-length 120
          black . --check --diff --line-length 120
      
      # ============================================================================
      # ШАГ 5: Проверка импортов
      # ============================================================================
      - name: Check imports with isort
        run: |
          # isort - сортировка импортов
          # 
          # ПАРАМЕТРЫ:
          #   --check-only - только проверить
          #   --diff - показать изменения
          #   --profile black - совместимость с Black
          # 
          # ДЛЯ PIP: используйте isort . --check-only --diff --profile black
          # ДЛЯ POETRY: используйте poetry run isort . --check-only --diff --profile black
          isort . --check-only --diff --profile black
      
      # ============================================================================
      # ШАГ 6: Проверка стиля
      # ============================================================================
      - name: Lint with flake8
        run: |
          # flake8 - проверка стиля кода (PEP 8)
          # 
          # ПАРАМЕТРЫ:
          #   --count - показать количество ошибок
          #   --max-complexity=10 - максимальная сложность
          #   --statistics - статистика
          # 
          # ОШИБКИ:
          #   E - ошибки стиля (PEP 8)
          #   W - предупреждения
          #   F - ошибки, которые могут привести к багам
          #   C - сложность кода
          # 
          # ДЛЯ PIP: используйте flake8 . --count --max-complexity=10 --statistics
          # ДЛЯ POETRY: используйте poetry run flake8 . --count --max-complexity=10 --statistics
          flake8 . --count --max-complexity=10 --statistics

  # ==========================================================================
  # JOB 2: Тестирование
  # ==========================================================================
  # Зачем: Убедиться, что код работает
  # Когда: После линтеров, перед сборкой
  # ==========================================================================
  test:
    name: 🧪 Run Tests
    runs-on: ubuntu-latest
    # needs - выполнять только после успешного lint
    # 
    # ВАРИАНТЫ:
    #   - needs: lint  # Последовательно
    #   - needs: [lint, security]  # После нескольких
    #   - без needs  # Параллельно
    needs: lint
    
    # ---------- СЕРВИСЫ ДЛЯ ТЕСТОВ ----------
    # services - дополнительные контейнеры для тестов
    # 
    # ЗАЧЕМ: Запускать БД и Redis для интеграционных тестов
    services:
      # --- PostgreSQL для тестов ---
      postgres:
        image: postgres:16-alpine  # Готовый образ из Docker Hub
        env:
          # Переменные окружения для PostgreSQL
          # 
          # ОТКУДА: Задаем здесь для тестов
          # ВАРИАНТЫ: Можно брать из .env.test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        ports:
          # Пробрасываем порт на host
          # - "5432:5432"  # Стандартный порт
          # 
          # ВАРИАНТЫ:
          #   - "5432:5432" - фиксированный порт
          #   - "5433:5432" - другой порт на host
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      # --- Redis для тестов ---
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      # --- ШАГ 1: Загрузка кода ---
      - name: Checkout code
        uses: actions/checkout@v4
      
      # --- ШАГ 2: Установка Python ---
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          # cache: 'pip' - для pip (работает только с requirements.txt)
          # ДЛЯ POETRY: НЕ РАБОТАЕТ, используйте отдельный кеш
          cache: 'pip'
      
      # ==========================================================================
      # ШАГ 3: Установка зависимостей для тестов
      # ==========================================================================
      # 
      # ВАРИАНТ А: PIP + REQUIREMENTS.TXT
      # ==========================================================================
      - name: Install dependencies with pip
        run: |
          # Устанавливаем зависимости из requirements.txt
          pip install -r requirements.txt
          # Устанавливаем тестовые зависимости
          pip install pytest pytest-cov pytest-django factory-boy
      
      # ==========================================================================
      # ВАРИАНТ Б: POETRY
      # Раскомментируйте шаги 3-5 и закомментируйте вариант А
      # ==========================================================================
      # 
      # --- ШАГ 3 (для Poetry): Установка Poetry ---
      # - name: Install Poetry
      #   run: pip install poetry==1.8.3
      # 
      # --- ШАГ 4 (для Poetry): Кеширование ---
      # - name: Cache Poetry dependencies
      #   uses: actions/cache@v3
      #   with:
      #     path: |
      #       .venv
      #       ~/.cache/pypoetry
      #     key: ${{ runner.os }}-poetry-${{ hashFiles('poetry.lock') }}
      #     restore-keys: |
      #       ${{ runner.os }}-poetry-
      # 
      # --- ШАГ 5 (для Poetry): Установка зависимостей ---
      # - name: Install dependencies with Poetry
      #   run: |
      #     # Настраиваем Poetry
      #     poetry config virtualenvs.in-project true
      #     # Устанавливаем зависимости
      #     poetry install --no-interaction --no-ansi --no-root
      #     # Добавляем тестовые зависимости если их нет в pyproject.toml
      #     poetry add --group dev pytest pytest-cov pytest-django factory-boy
      
      # --- ШАГ 4: Создание .env для тестов ---
      - name: Create test .env
        run: |
          # Создаем .env для тестов
          # 
          # ОТКУДА: Формируем здесь
          # 
          # ВАРИАНТЫ:
          #   - Из .env.test файла
          #   - Из GitHub Secrets
          #   - Из переменных окружения GitHub
          cat > .env << 'EOF'
          DEBUG=True
          SECRET_KEY=test-secret-key-not-for-production
          DATABASE_URL=postgresql://test_user:test_password@localhost:5432/test_db
          REDIS_URL=redis://localhost:6379/0
          CELERY_BROKER_URL=redis://localhost:6379/0
          CELERY_RESULT_BACKEND=redis://localhost:6379/0
          EOF
      
      # --- ШАГ 5: Применение миграций ---
      - name: Run migrations
        run: |
          # Применяем миграции для тестовой БД
          # 
          # ДЛЯ PIP: python manage.py migrate
          # ДЛЯ POETRY: poetry run python manage.py migrate
          python manage.py migrate
      
      # --- ШАГ 6: Запуск тестов ---
      - name: Run tests with coverage
        run: |
          # pytest - запуск тестов
          # 
          # ПАРАМЕТРЫ:
          #   -v - подробный вывод
          #   --cov=. - измерять покрытие кода
          #   --cov-report=xml - отчет в XML формате
          #   --cov-report=html - отчет в HTML формате
          #   -m "not slow" - пропустить медленные тесты
          # 
          # ВАРИАНТЫ:
          #   - pytest -v --cov=. --cov-report=xml
          #   - pytest -v -m "unit"  # Только unit тесты
          #   - pytest -v -k "test_user"  # Тесты с именем
          # 
          # ДЛЯ PIP: pytest -v --cov=. --cov-report=xml --cov-report=html
          # ДЛЯ POETRY: poetry run pytest -v --cov=. --cov-report=xml --cov-report=html
          pytest -v --cov=. --cov-report=xml --cov-report=html
      
      # --- ШАГ 7: Сохранение отчета о покрытии ---
      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: htmlcov/
      
      # --- ШАГ 8: Отправка в Codecov (опционально) ---
      - name: Upload coverage to Codecov
        # Codecov - сервис для отслеживания покрытия кода
        # 
        # ЗАЧЕМ: Визуализировать покрытие тестами
        # КАК: Отправляет отчет на codecov.io
        # 
        # АЛЬТЕРНАТИВЫ:
        #   - Coveralls
        #   - SonarCloud
        #   - Свой сервер
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          flags: unittests
          name: myapp-coverage
          fail_ci_if_error: false

  # ==========================================================================
  # JOB 3: Сборка и публикация образов
  # ==========================================================================
  # Зачем: Собрать Docker образы и загрузить их в реестр
  # Когда: После успешных тестов
  # ==========================================================================
  build-and-push:
    name: 🐳 Build & Push to Docker Hub
    runs-on: ubuntu-latest
    needs: test  # Только после тестов
    
    # ---------- РАЗРЕШЕНИЯ ----------
    # permissions - права доступа
    # 
    # ЗАЧЕМ: Дать права на запись в GitHub Container Registry
    # 
    # ВАРИАНТЫ:
    #   - contents: read  # Только чтение кода
    #   - packages: write  # Запись в GitHub Packages
    permissions:
      contents: read
      packages: write
    
    steps:
      # --- ШАГ 1: Загрузка кода ---
      - name: 📥 Checkout code
        uses: actions/checkout@v4
      
      # --- ШАГ 2: Настройка Docker Buildx ---
      - name: ⚙️ Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        # 
        # ЧТО ДЕЛАЕТ: Настраивает Buildx для продвинутой сборки
        # 
        # ВОЗМОЖНОСТИ:
        #   - Сборка для нескольких архитектур (amd64, arm64)
        #   - Использование кеша
        #   - Сборка в удаленном кластере
        # 
        # АЛЬТЕРНАТИВЫ:
        #   - docker build (обычная сборка, без Buildx)
        #   - Использование Docker Desktop Buildx
      
      # --- ШАГ 3: Логин в Docker Hub ---
      - name: 🔑 Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          # ---------- АРГУМЕНТЫ ----------
          # username - имя пользователя Docker Hub
          # 
          # ОТКУДА: ${{ secrets.DOCKER_USERNAME }}
          # ГДЕ НАСТРОИТЬ: GitHub → Settings → Secrets
          # 
          # ВАРИАНТЫ:
          #   - Использовать переменную окружения
          #   - Задать вручную (НЕ РЕКОМЕНДУЕТСЯ)
          username: ${{ secrets.DOCKER_USERNAME }}
          
          # password - токен доступа Docker Hub
          # 
          # ОТКУДА: ${{ secrets.DOCKER_TOKEN }}
          # ГДЕ ВЗЯТЬ: Docker Hub → Account Settings → Security → New Access Token
          # 
          # ВАЖНО: Используйте ТОКЕН, а не пароль!
          # Токен безопаснее и можно ограничить права
          # 
          # ВАРИАНТЫ:
          #   - Использовать пароль (устаревший способ)
          #   - Использовать токен (рекомендуется)
          password: ${{ secrets.DOCKER_TOKEN }}
      
      # --- ШАГ 4: Извлечение метаданных ---
      - name: 🏷️ Extract Docker metadata
        id: meta  # ID для доступа к результатам
        uses: docker/metadata-action@v5
        with:
          # images - для каких образов генерировать метаданные
          # 
          # ОТКУДА: Используем переменные окружения
          # 
          # ВАРИАНТЫ:
          #   - ${{ env.DOCKER_REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}
          #   - ${{ secrets.DOCKER_USERNAME }}/myapp
          images: ${{ env.DOCKER_REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}
          
          # tags - как будут называться теги
          # 
          # ГЕНЕРИРУЕТ ТЕГИ:
          #   - latest - для main ветки
          #   - sha-{hash} - для каждого коммита
          #   - version-{version} - для релизов
          #   - branch-{branch} - для веток
          # 
          # ВАРИАНТЫ:
          #   - type=sha,format=short - короткий SHA
          #   - type=raw,value=latest - всегда latest
          #   - type=semver,pattern={{version}} - версия из тега
          tags: |
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
            type=sha,format=short
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
          # 
          # labels - дополнительные метки для образа
          # 
          # ДОБАВЛЯЕТ МЕТКИ:
          #   - org.opencontainers.image.title - название
          #   - org.opencontainers.image.version - версия
          #   - org.opencontainers.image.created - дата создания
          #   - org.opencontainers.image.revision - SHA коммита
          labels: |
            org.opencontainers.image.title=${{ github.event.repository.name }}
            org.opencontainers.image.description=${{ github.event.repository.description }}
            org.opencontainers.image.url=${{ github.event.repository.html_url }}
            org.opencontainers.image.source=${{ github.event.repository.clone_url }}
            org.opencontainers.image.version=${{ github.ref_name }}
            org.opencontainers.image.created=${{ github.event.repository.created_at }}
            org.opencontainers.image.revision=${{ github.sha }}
            org.opencontainers.image.licenses=${{ github.event.repository.license.spdx_id }}
      
      # --- ШАГ 5: Сборка и публикация основного образа ---
      - name: 🏗️ Build and push Django image
        uses: docker/build-push-action@v5
        with:
          # context - контекст сборки
          # 
          # ЧТО ЭТО: Папка, которая отправляется в Docker daemon
          # 
          # ВАРИАНТЫ:
          #   - context: . - текущая папка
          #   - context: ./app - папка app
          context: .
          
          # file - путь к Dockerfile
          # 
          # ВАРИАНТЫ:
          #   - file: ./Dockerfile - стандартное имя
          #   - file: ./Dockerfile.prod - для продакшена
          file: ./Dockerfile
          
          # push - публиковать ли образ
          # 
          # ВАРИАНТЫ:
          #   - push: true - публиковать
          #   - push: false - только собрать локально
          push: true
          
          # tags - теги для образа
          # 
          # ОТКУДА: Из предыдущего шага (meta)
          # 
          # ВАРИАНТЫ:
          #   - tags: ${{ steps.meta.outputs.tags }}
          #   - tags: ${{ env.DOCKER_IMAGE_NAME }}:latest
          tags: ${{ steps.meta.outputs.tags }}
          
          # labels - метки для образа
          labels: ${{ steps.meta.outputs.labels }}
          
          # cache-from - откуда брать кеш
          # 
          # ЗАЧЕМ: Ускорить сборку
          # 
          # ВАРИАНТЫ:
          #   - type=gha - использовать GitHub Actions cache
          #   - type=local,src=/tmp/.buildx-cache - локальный кеш
          #   - type=registry,ref=user/app:cache - кеш в реестре
          cache-from: type=gha
          
          # cache-to - куда сохранять кеш
          cache-to: type=gha,mode=max
          
          # build-args - аргументы сборки
          # 
          # ПЕРЕДАЕТСЯ В DOCKERFILE:
          #   - ARG BUILD_ENV=production
          #   - ARG DJANGO_SETTINGS_MODULE=config.settings.prod
          # 
          # ВАРИАНТЫ:
          #   - build-args: |
          #       BUILD_ENV=production
          #       VERSION=${{ github.sha }}
          build-args: |
            BUILD_ENV=production
            VERSION=${{ github.sha }}
      
      # --- ШАГ 6: Сборка дополнительных образов ---
      # (Опционально, если есть другие сервисы)
      - name: 🏗️ Build and push Celery image
        if: false  # Отключаем, если не нужен отдельный образ
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ env.DOCKER_REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}-celery:${{ env.IMAGE_TAG }}
          build-args: |
            SERVICE=celery
      
      # --- ШАГ 7: Сохранение информации о сборке ---
      - name: 📝 Save build info
        run: |
          # Сохраняем информацию о собранном образе
          # 
          # ЗАЧЕМ: Для отслеживания версий
          # 
          # ВАРИАНТЫ:
          #   - Сохранять в файл
          #   - Отправлять в Slack/Telegram
          #   - Сохранять в GitHub Release
          cat > build-info.txt << EOF
          Build Date: $(date)
          Commit SHA: ${{ github.sha }}
          Branch: ${{ github.ref_name }}
          Image Tag: ${{ env.IMAGE_TAG }}
          Repository: ${{ github.repository }}
          Run ID: ${{ github.run_id }}
          Docker Image: ${{ env.DOCKER_REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          EOF
      
      - name: 📤 Upload build info
        uses: actions/upload-artifact@v4
        with:
          name: build-info
          path: build-info.txt

  # ==========================================================================
  # JOB 4: Деплой на сервер
  # ==========================================================================
  # Зачем: Обновить приложение на production сервере
  # Когда: После успешной сборки образов
  # ==========================================================================
  deploy:
    name: 🚀 Deploy to Server
    runs-on: ubuntu-latest
    needs: build-and-push  # Только после сборки
    # if: github.ref == 'refs/heads/main'  # Только для main ветки
    # 
    # УСЛОВИЯ ЗАПУСКА:
    #   - github.ref == 'refs/heads/main' - только для main
    #   - github.event_name == 'push' - только при пуше
    #   - всегда true - для ручного запуска
    
    # ---------- ОКРУЖЕНИЕ ДЛЯ ДЕПЛОЯ ----------
    # environment - настройки окружения
    # 
    # ЗАЧЕМ:
    #   - Визуально разделить окружения
    #   - Управлять approve (подтверждение деплоя)
    #   - Свои секреты для каждого окружения
    # 
    # ВАРИАНТЫ:
    #   - name: production
    #   - name: staging
    #   - name: development
    # 
    # КАК НАСТРОИТЬ:
    #   GitHub → Settings → Environments → New environment
    environment:
      name: production
      # url - ссылка на приложение
      # 
      # ВАРИАНТЫ:
      #   - https://your-domain.com
      #   - http://server-ip:8000
      #   - ${{ vars.APP_URL }}
      url: https://${{ secrets.DOMAIN_NAME }}
    
    steps:
      # ============================================================
      # ШАГ 1: Загрузка кода
      # ============================================================
      - name: 📥 Checkout code
        uses: actions/checkout@v4
        # 
        # ЗАЧЕМ: Нужен для docker-compose.yml и nginx.conf
        # 
        # ВАРИАНТЫ:
        #   - Можно скачать только нужные файлы
        #   - Можно использовать готовый compose файл с сервера
      
      # ============================================================
      # ШАГ 2: Настройка SSH
      # ============================================================
      - name: 🔑 Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          # ssh-private-key - приватный ключ для подключения
          # 
          # ОТКУДА: ${{ secrets.SSH_PRIVATE_KEY }}
          # 
          # КАК СОЗДАТЬ:
          #   1. На сервере: ssh-keygen -t ed25519
          #   2. Добавить публичный ключ в ~/.ssh/authorized_keys
          #   3. Приватный ключ сохранить в GitHub Secrets
          # 
          # ВАЖНО:
          #   - Храните ключ в безопасности
          #   - Используйте ed25519 вместо RSA
          #   - Никогда не передавайте ключ в логах
          # 
          # ВАРИАНТЫ:
          #   - Использовать appleboy/ssh-action (альтернатива)
          #   - Использовать стандартный SSH клиент
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      
      # ============================================================
      # ШАГ 3: Добавление сервера в known_hosts
      # ============================================================
      - name: 📝 Add server to known_hosts
        run: |
          # Создаем папку .ssh если её нет
          mkdir -p ~/.ssh
          
          # ssh-keyscan - сканирует публичный ключ сервера
          # 
          # ЗАЧЕМ: Избежать вопроса "Are you sure you want to continue connecting?"
          # 
          # ВАРИАНТЫ:
          #   - ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts
          #   - вручную добавить ключ на сервер
          #   - использовать StrictHostKeyChecking=no (менее безопасно)
          ssh-keyscan -H ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts
      
      # ============================================================
      # ШАГ 4: Деплой через SSH
      # ============================================================
      - name: 🚀 Deploy via SSH
        run: |
          # Подключаемся к серверу и выполняем команды
          # 
          # ФОРМАТ: ssh user@host "команды"
          # 
          # ВАРИАНТЫ SSH КОМАНД:
          #   - Одинарные кавычки: ssh user@host 'команды'
          #   - Двойные кавычки: ssh user@host "команды"
          #   - HEREDOC: ssh user@host << 'ENDSSH'
          #     команды
          #     ENDSSH
          ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} << 'ENDSSH'
            # ============================================================
            # 1. Переход в папку проекта
            # ============================================================
            # cd - change directory
            # 
            # ГДЕ РАСПОЛОЖЕН ПРОЕКТ:
            #   - /opt/myapp/ -
```