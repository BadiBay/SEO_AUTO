# 🔍 SEO Analyzer — Интеллектуальная платформа семантического анализа

Веб-приложение для автоматизированного SEO-анализа, сбора семантического ядра и генерации контента с помощью ИИ.

---

## 📋 Содержание

- [О проекте](#-о-проекте)
- [Функциональные возможности](#-функциональные-возможности)
- [Архитектура системы](#-архитектура-системы)
- [Стек технологий](#-стек-технологий)
- [Структура проекта](#-структура-проекта)
- [Быстрый старт (Docker)](#-быстрый-старт-docker)
- [Ручной запуск (локально)](#-ручной-запуск-локально)
- [Переменные окружения](#-переменные-окружения)
- [API документация](#-api-документация)
- [Модели данных](#-модели-данных)
- [Тестирование](#-тестирование)

---

## 📖 О проекте

**SEO Analyzer** — это полнофункциональная платформа для SEO-специалистов, которая автоматизирует рутинные задачи:

- Автоматически **собирает семантическое ядро** для любого сайта или конкретной страницы, используя сервис Bukvarix, поисковые подсказки Google и YouTube.
- **Кластеризует ключевые слова** по смысловой близости с помощью нейросетевой модели `sentence-transformers`.
- **Генерирует SEO-контент** (статьи, метатеги) через API GigaChat.
- Проводит **технический SEO-аудит** сайта (проверка доступности, SSL, скорости, Title, Description, H1).
- Экспортирует готовые **технические задания и статьи** в формат `.docx`.

---

## ✨ Функциональные возможности

### 📊 Управление проектами
- Создание, редактирование и удаление SEO-проектов
- Поддержка **вложенных подпроектов** (иерархия: главная страница → внутренние страницы)
- Привязка URL к проекту для автоматического сбора данных

### 🌱 Сбор семантического ядра
- **Автоматический сбор** ключевых слов по URL через Bukvarix API
- Для главной страницы: полный выгруз по домену (до 300 запросов)
- Для внутренних страниц: умный ИИ-анализ контента (Title, H1) → генерация «семенных» запросов → расширение через Bukvarix и Google Suggestions
- **Ручной импорт** ключевых слов из файлов `.csv` и `.xlsx`
- **Массовое удаление** по частотности и стоп-словам
- Лимит: до **15 000 ключевых слов** на один проект

### 🧠 Семантическая кластеризация
- Кластеризация на основе BERT-эмбеддингов (`paraphrase-multilingual-MiniLM-L12-v2`)
- Алгоритм: **Agglomerative Clustering** с косинусным расстоянием
- Настраиваемый порог сходства (по умолчанию 0.6)
- **ИИ-фильтрация нерелевантных кластеров** через GigaChat (для подпроектов)
- Ручное управление: перемещение ключевых слов между кластерами, изменение статусов

### 📝 Генерация контента с ИИ (GigaChat)
- **Генерация SEO-статей**: объём от 8 000 символов, структура H2/H3, Markdown-формат
- **Генерация метатегов**: 3 варианта Title + Description под каждый кластер
- **Настраиваемые AI-роли**: задайте системный промпт (роль эксперта, копирайтера и т.д.)
- Кастомный список LSI-слов для включения в статью

### 🔎 Анализ SERP и позиций
- **Анализ конкурентов** в поисковой выдаче для любого запроса
- **Проверка позиций** сайта по ключевым словам (через Bukvarix API)
- Кэширование результатов в Redis (1 час)

### 🛡️ SEO-аудит
- Проверка доступности (HTTP статус-код)
- Наличие SSL/HTTPS
- Скорость загрузки (в мс)
- Оптимальность Title (10–70 символов)
- Наличие мета-Description
- Корректное использование H1 (один на странице)
- Интегральный **SEO-счёт** (0–100)
- Сохранение истории аудитов

### 📁 Экспорт
- Выгрузка **технического задания (ТЗ)** или **статьи** в формат `.docx`
- Сохранение сгенерированных документов в историю проекта

### 🖥️ Административная панель
- Управление API-ключами для GigaChat
- Просмотр логов API-запросов (эндпоинт, IP, время ответа, статус, токены)
- Управление AI-ролями

---

## 🏗️ Архитектура системы

```
+----------------------------------------------------------+
|                   Клиент (Vue 3 / Vite)                  |
|                  http://localhost:5173                    |
+-----------------------------+----------------------------+
                              | REST API (CORS)
                              v
+----------------------------------------------------------+
|              Django REST Framework                        |
|               http://localhost:8000                       |
|                                                          |
|  +-------------+  +---------------+  +---------------+  |
|  | ProjectView |  |  ClusterView  |  |   AuditView   |  |
|  +-------------+  +---------------+  +---------------+  |
|                                                          |
|  +----------------------------------------------------+  |
|  |                  Services Layer                    |  |
|  |  AuditService | GenerationService | ParsingService |  |
|  +----------------------------------------------------+  |
+----------+--------------------------+-------------------+
           |                          |
           v                          v
+-------------------+     +------------------------+
|  PostgreSQL 15    |     |   Celery + Redis        |
|                   |     |                         |
|  +-------------+  |     |  fetch_keywords_task    |
|  |   seo_db    |  |     |  cluster_keywords_task  |
|  |  (default)  |  |     +------------------------+
|  +-------------+  |
|  +-------------+  |
|  | seo_db_shard1|  |   <- Горизонтальное шардирование по project_id
|  +-------------+  |
+-------------------+
```

### Шардирование базы данных

Проект реализует **горизонтальное шардирование** данных между двумя PostgreSQL-инстансами:

- `default` (порт 5434) — хранит проекты с чётным `id`
- `shard_1` (порт 5433) — хранит проекты с нечётным `id`

Маршрутизация осуществляется через кастомный `ShardRouter` (`analyzer/routers.py`). Индексы оптимизированы под шардирование (составной индекс `shard_key + project`).

---

## 🛠️ Стек технологий

### Backend
| Технология | Версия | Назначение |
|---|---|---|
| Python | 3.11 | Основной язык |
| Django | >=4.2, <5.3 | Веб-фреймворк |
| Django REST Framework | >=3.14 | REST API |
| Celery | >=5.3 | Асинхронные задачи |
| Redis | >=5.0 | Брокер Celery + кэш |
| PostgreSQL | 15 | Основная БД (2 шарда) |
| GigaChat SDK | >=0.1.28 | AI-генерация |
| sentence-transformers | >=2.7 | Векторизация текста |
| scikit-learn | >=1.4 | Кластеризация |
| BeautifulSoup4 + lxml | — | Веб-скрапинг |
| python-docx | >=1.1 | Генерация .docx |
| Gunicorn | >=21.2 | Production-сервер |

### Frontend
| Технология | Назначение |
|---|---|
| Vue 3 | UI-фреймворк |
| Vite | Сборщик / dev-сервер |
| Vue Router | Маршрутизация |
| Vitest | Unit-тесты |

### Инфраструктура
| Инструмент | Назначение |
|---|---|
| Docker + Docker Compose | Контейнеризация |
| pg_trgm (PostgreSQL) | GIN-индексы для полнотекстового поиска |

---

## 📁 Структура проекта

```
VKR/
├── seo_project/               # Конфигурация Django-проекта
│   ├── settings.py            # Настройки (БД, Redis, Celery, CORS)
│   ├── celery_app.py          # Инициализация Celery
│   ├── urls.py                # Корневые URL-маршруты
│   └── ...
│
├── analyzer/                  # Основное Django-приложение
│   ├── models.py              # Модели данных (Project, Cluster, Keyword, ...)
│   ├── views.py               # API ViewSets (ProjectViewSet, ClusterViewSet, ...)
│   ├── serializers.py         # DRF Serializers
│   ├── urls.py                # URL-маршруты приложения
│   ├── routers.py             # ShardRouter для шардирования БД
│   ├── tasks.py               # Celery-задачи (сбор семантики, кластеризация)
│   ├── tests.py               # Unit-тесты
│   ├── migrations/            # Миграции БД
│   └── services/
│       ├── audit.py           # SEO-аудит сайта
│       ├── clustering.py      # Семантическая кластеризация (BERT + sklearn)
│       ├── generation.py      # Генерация контента через GigaChat
│       ├── export.py          # Экспорт в .docx
│       └── parsing.py         # Анализ SERP / конкурентов
│
├── seo-frontend/              # Vue 3 фронтенд
│   ├── src/
│   │   ├── views/
│   │   │   ├── HomeView.vue          # Список проектов
│   │   │   ├── ProjectDetailView.vue # Детальный вид проекта
│   │   │   └── AdminView.vue         # Панель администратора
│   │   ├── components/        # Переиспользуемые компоненты
│   │   ├── router/            # Vue Router
│   │   └── main.js
│   ├── Dockerfile
│   └── vite.config.js
│
├── docker/
│   └── postgres/
│       └── init.sql           # Инициализация расширений PostgreSQL (pg_trgm)
│
├── Dockerfile                 # Docker-образ для Django + Celery
├── docker-compose.yml         # Полный стек: db, db_shard1, redis, web, celery, frontend
├── requirements.txt           # Python-зависимости
├── manage.py
└── .env                       # Переменные окружения (не в репозитории)
```

---

## 🚀 Быстрый старт (Docker)

Самый простой способ запустить весь стек одной командой.

### Предварительные требования

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (включает Docker Compose)

### 1. Клонирование репозитория

```bash
git clone <URL_репозитория>
cd VKR
```

### 2. Настройка переменных окружения

Создайте файл `.env` в корне проекта (см. раздел [Переменные окружения](#-переменные-окружения)).

### 3. Запуск всего стека

```bash
docker-compose up --build
```

Это запустит:

| Сервис | URL |
|---|---|
| 🌐 Фронтенд (Vue) | http://localhost:5173 |
| ⚙️ Backend API (Django) | http://localhost:8000/api/ |
| 🗄️ PostgreSQL (основная) | localhost:5434 |
| 🗄️ PostgreSQL (shard_1) | localhost:5433 |
| 📨 Redis | localhost:6379 |

> **Примечание:** При первом запуске автоматически выполняются миграции БД и тесты.

### 4. Остановка

```bash
docker-compose down          # остановить контейнеры
docker-compose down -v       # + удалить тома БД
```

---

## 🛠️ Ручной запуск (локально)

Для разработки удобно запускать каждый компонент отдельно.

### Предварительные требования

- Python 3.11+
- Node.js 18+
- PostgreSQL 15 (2 инстанса)
- Redis

### 1. Создание и активация виртуального окружения

```bash
python3 -m venv venv
source venv/bin/activate        # macOS / Linux
# venv\Scripts\activate         # Windows
```

### 2. Установка Python-зависимостей

```bash
pip install -r requirements.txt
```

### 3. Применение миграций

```bash
python manage.py migrate
python manage.py migrate --database=shard_1
```

### 4. Запуск Django-сервера

```bash
python manage.py runserver
# Сервер доступен по адресу: http://127.0.0.1:8000
```

### 5. Запуск Celery Worker (в отдельном терминале)

```bash
celery -A seo_project.celery_app:app worker -l info --pool=threads
```

### 6. Запуск фронтенда (в отдельном терминале)

```bash
cd seo-frontend
npm install
npm run dev
# Фронтенд доступен по адресу: http://localhost:5173
```

---

## 🔧 Переменные окружения

Создайте файл `.env` в корне проекта:

```dotenv
# === PostgreSQL (основная БД) ===
POSTGRES_DB=seo_db
POSTGRES_USER=seo_user
POSTGRES_PASSWORD=seo_password
POSTGRES_HOST=localhost
POSTGRES_PORT=5432

# === PostgreSQL (Shard 1) ===
POSTGRES_SHARD1_HOST=localhost

# === Redis ===
REDIS_URL=redis://localhost:6379/0

# === GigaChat API ===
# Ключ также можно добавить через AdminView в интерфейсе приложения
GIGACHAT_API_KEY=ваш_ключ_gigachat
```

> **Важно:** Файл `.env` добавлен в `.gitignore` и не должен попасть в репозиторий.

---

## 📡 API документация

Базовый URL: `http://localhost:8000/api/`

### Проекты

| Метод | Эндпоинт | Описание |
|---|---|---|
| `GET` | `/api/projects/` | Список всех проектов (только корневые) |
| `POST` | `/api/projects/` | Создать новый проект |
| `GET` | `/api/projects/{id}/` | Детали проекта (с кластерами и ключами) |
| `PATCH` | `/api/projects/{id}/` | Обновить проект |
| `DELETE` | `/api/projects/{id}/` | Удалить проект |
| `POST` | `/api/projects/{id}/run_fetch/` | 🚀 Запустить сбор семантики (Celery) |
| `POST` | `/api/projects/{id}/run_clustering/` | 🧠 Запустить кластеризацию (Celery) |
| `GET` | `/api/projects/{id}/audit/` | 🛡️ SEO-аудит |
| `POST` | `/api/projects/{id}/check_position/` | 🔎 Проверить позицию по запросу |
| `POST` | `/api/projects/{id}/generate_content/` | ✍️ Сгенерировать статью |
| `POST` | `/api/projects/{id}/generate_meta/` | 🏷️ Сгенерировать метатеги |
| `POST` | `/api/projects/{id}/download_docx/` | 📄 Скачать .docx |
| `POST` | `/api/projects/{id}/import_keywords/` | 📥 Импорт ключей из CSV/XLSX |
| `POST` | `/api/projects/{id}/mass_delete/` | 🗑️ Массовое удаление ключей |
| `POST` | `/api/projects/{id}/reset_clusters/` | Сброс кластеров |
| `POST` | `/api/projects/{id}/clear_project/` | Очистить проект |
| `POST` | `/api/projects/{id}/save_document/` | Сохранить документ в историю |
| `POST` | `/api/projects/analyze_competitors/` | 🔍 Анализ конкурентов по запросу |

### Кластеры

| Метод | Эндпоинт | Описание |
|---|---|---|
| `GET` | `/api/clusters/` | Список кластеров |
| `POST` | `/api/clusters/{id}/change_status/` | Изменить статус кластера |
| `POST` | `/api/clusters/{id}/move_keyword/` | Переместить ключ в кластер |

### Прочие ресурсы

| Эндпоинт | Описание |
|---|---|
| `/api/apikeys/` | Управление API-ключами GigaChat |
| `/api/apilogs/` | Журнал API-запросов |
| `/api/airoles/` | Управление AI-ролями |
| `/api/tasks/{task_id}/status/` | Статус выполнения Celery-задачи |

---

## 🗃️ Модели данных

```
Project
  ├── id, name, url
  ├── parent -> Project (самоссылка, для подпроектов)
  └── created_at

Cluster
  ├── id, name, status (draft | ready)
  ├── project -> Project
  ├── shard_key (для маршрутизации)
  └── created_at

Keyword
  ├── id, query, volume
  ├── project -> Project
  ├── cluster -> Cluster (nullable)
  ├── shard_key
  └── search_vector (tsvector, GIN-индекс для FTS)

AuditLog
  ├── project -> Project
  ├── score (0-100)
  ├── checks_json
  └── created_at

DocumentHistory
  ├── project -> Project
  ├── cluster -> Cluster (nullable)
  ├── doc_type (tz | article)
  ├── title, content
  └── created_at

APIKey
  └── name, key, is_active

APILog
  └── user_ip, endpoint, tokens_used, status, duration_ms, created_at

AIRole
  └── name, prompt_addition
```

---

## 🧪 Тестирование

### Backend (Django)

```bash
# Запустить все тесты
python manage.py test analyzer --noinput

# С подробным выводом
python manage.py test analyzer -v 2
```

Тесты покрывают:
- CRUD операции с проектами и кластерами
- Логику ShardRouter
- Сервисы аудита и кластеризации

### Frontend (Vitest)

```bash
cd seo-frontend

# Запустить unit-тесты один раз
npm run test:unit -- --run

# Запустить в watch-режиме
npm run test:unit
```

---

## 🔄 Жизненный цикл Celery-задачи

При нажатии кнопки «Собрать семантику» или «Кластеризовать» в UI:

1. Backend создаёт Celery-задачу и возвращает `task_id`
2. Фронтенд периодически опрашивает `/api/tasks/{task_id}/status/`
3. Celery-воркер обновляет статус задачи (`PROGRESS` → `SUCCESS` / `FAILURE`)
4. Состояния отображаются в виде прогресс-бара в UI

---

## 📄 Лицензия

Проект разработан в рамках выпускной квалификационной работы. Все права защищены.
