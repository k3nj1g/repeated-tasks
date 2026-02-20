# MVP сервиса повторяющихся списаний трудозатрат в Redmine

## 1) Два стека реализации

### Стек A (рекомендованный): Python + FastAPI + APScheduler + SQLAlchemy + PostgreSQL

**Состав:**
- **Frontend**: Jinja/HTMX или React (минимум можно начать с server-rendered HTML)
- **Backend API**: FastAPI
- **Scheduler/Worker**: APScheduler (в отдельном процессе)
- **DB**: PostgreSQL
- **Queue (опционально)**: без отдельной очереди на MVP

**Плюсы:**
- Быстрый старт и минимальная сложность для MVP.
- APScheduler легко работает с cron и timezone.
- FastAPI даёт удобную валидацию (Pydantic) и OpenAPI.
- В Python удобно писать ретраи, интеграции и cron-логику.

**Минусы:**
- Для высоких нагрузок/сложных пайплайнов потребуется переход на очередь (Celery/RQ).
- Нужно аккуратно организовать single-instance scheduler (не запускать дублирующие инстансы).

---

### Стек B: Node.js + NestJS/Express + BullMQ + Redis + PostgreSQL

**Состав:**
- **Frontend**: React/Vue (или server-side шаблоны)
- **Backend API**: NestJS (или Express)
- **Scheduler/Worker**: BullMQ repeatable jobs
- **DB**: PostgreSQL
- **Queue**: Redis + BullMQ

**Плюсы:**
- Хорошая модель фоновых задач (очереди, retries, delayed jobs из коробки).
- Удобно масштабировать workers горизонтально.
- Единый стек TypeScript для frontend/backend.

**Минусы:**
- Выше инфраструктурная сложность (нужен Redis).
- Для маленького сервиса избыточно по компонентам.

---

## 2) Выбор для MVP

Для MVP на 1–3 дня оптимален **Стек A**:
- меньше инфраструктуры (без Redis/Celery),
- достаточно надёжности для задач “несколько повторов в день/неделю”,
- проще деплой через `docker-compose`.

## 3) Архитектура и взаимодействие компонентов

Компоненты:
1. **Web UI + API (FastAPI)**
   - CRUD расписаний
   - ручной запуск
   - просмотр логов
   - простая авторизация (single-user password)
2. **Scheduler (APScheduler, отдельный процесс)**
   - читает активные расписания
   - вычисляет локальное время Europe/Riga
   - запускает job execution
3. **Execution service**
   - валидация issue в Redmine
   - дедупликация
   - создание time entry
   - retry с backoff
   - запись результата в таблицу запусков
4. **PostgreSQL**
   - хранение расписаний, секретов, логов, дедуп-ключей
5. **Redmine REST API**
   - `GET /issues/{id}.json`
   - `POST /time_entries.json`
   - (опц.) `GET /time_entries.json` для дедуп-проверки

Поток выполнения:
1. Пользователь создаёт расписание через UI.
2. API сохраняет расписание, проверяет issue и (опционально) activity.
3. Scheduler по cron поднимает задачу.
4. Execution service проверяет дедуп-условие.
5. Если дубля нет — создаёт time entry в Redmine.
6. Пишет `job_runs` (success/error, payload, response, error text).

## 4) Схема БД (PostgreSQL)

### Таблица `users`
- `id` (PK)
- `username` (unique)
- `password_hash`
- `created_at`

### Таблица `redmine_connections`
- `id` (PK)
- `user_id` (FK -> users)
- `redmine_base_url`
- `api_key_encrypted` (AES-GCM ciphertext)
- `api_key_iv`
- `api_key_tag`
- `created_at`
- `updated_at`

### Таблица `schedules`
- `id` (PK)
- `user_id` (FK)
- `name` (text)
- `enabled` (bool)
- `timezone` (text, default `Europe/Riga`)
- `cron_expr` (text, nullable)
- `days_of_week` (int[] или text, nullable) — для упрощённого UI
- `run_time_local` (time, nullable) — для упрощённого UI
- `issue_id` (int)
- `project_id` (int, nullable)
- `hours` (numeric(4,2))
- `rounding_step` (numeric(3,2), default 0.25)
- `activity_id` (int, nullable)
- `activity_name` (text, nullable)
- `comments` (text)
- `dedupe_mode` (text: `remote`, `local`, `hybrid`)
- `skip_weekends` (bool default false)
- `charge_user_id` (int, nullable)
- `last_run_at` (timestamptz, nullable)
- `last_status` (text, nullable)
- `created_at`
- `updated_at`

### Таблица `job_runs`
- `id` (PK)
- `schedule_id` (FK -> schedules)
- `trigger_type` (text: `cron`/`manual`)
- `run_at` (timestamptz)
- `local_entry_date` (date) — дата списания в Europe/Riga
- `status` (text: `success`/`error`/`skipped_duplicate`/`skipped_weekend`)
- `attempt` (int)
- `request_payload` (jsonb)
- `response_payload` (jsonb)
- `error_text` (text)
- `redmine_time_entry_id` (int, nullable)
- `created_at`

### Таблица `dedupe_locks`
- `id` (PK)
- `schedule_id` (FK)
- `entry_date` (date)
- `dedupe_key` (text)
- `created_at`
- `UNIQUE(schedule_id, entry_date, dedupe_key)`

> Почему PostgreSQL, а не SQLite: лучше конкуренция, транзакции и блокировки для scheduler/worker + web, надёжнее для многопроцессной работы и масштабирования.

## 5) API эндпойнты

### Auth
- `POST /api/auth/login`
- `POST /api/auth/logout`
- `GET /api/auth/me`

### Schedules CRUD
- `GET /api/schedules`
- `POST /api/schedules`
- `GET /api/schedules/{id}`
- `PUT /api/schedules/{id}`
- `DELETE /api/schedules/{id}`
- `POST /api/schedules/{id}/enable`
- `POST /api/schedules/{id}/disable`
- `POST /api/schedules/{id}/run-now`

### Logs / Runs
- `GET /api/schedules/{id}/runs?limit=50`
- `GET /api/runs?status=error&from=...&to=...`

### Validation helpers
- `POST /api/redmine/validate-issue` (issue_id)
- `GET /api/redmine/activities` (опц.)

## 6) Ключевые сценарии

### Сценарий A: создание расписания
1. UI отправляет форму (`name`, `issue_id`, `hours`, `cron_expr` или days+time, timezone).
2. Backend округляет `hours` по шагу 0.25.
3. Backend валидирует issue через Redmine API.
4. Сохраняет расписание.

### Сценарий B: выполнение по расписанию
1. APScheduler триггерит job.
2. Вычисляется локальная дата `Europe/Riga`.
3. Если `skip_weekends=true` и выходной — `skipped_weekend`.
4. Проверка дубля.
5. `POST /time_entries.json`.
6. Логирование результата в `job_runs`.

### Сценарий C: дедупликация
Гибридный подход:
- **Локальный ключ**: `hash(issue_id, date, hours, normalized_comments, user_id)` + unique lock.
- **Удалённая проверка** (если разрешена ролью): query `time_entries` за дату и сравнение ключа.
- **Fallback**, если чтение time_entries запрещено: только локальный lock + marker в комментарии (`[rt:<schedule_id>:<YYYY-MM-DD>]`).

### Сценарий D: обработка ошибок
- Сетевые/5xx: retry 3 раза (экспоненциально: 2s, 5s, 15s).
- 4xx (валидация/доступ): без retry, сразу error.
- Все попытки и ответы пишутся в `job_runs`.

## 7) Минимальный код (скелет)

### 7.1 Создание time entry в Redmine (Python)

```python
import httpx
from datetime import date


class RedmineClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {"X-Redmine-API-Key": api_key}

    async def get_issue(self, issue_id: int) -> dict:
        url = f"{self.base_url}/issues/{issue_id}.json"
        async with httpx.AsyncClient(timeout=20) as client:
            r = await client.get(url, headers=self.headers)
            r.raise_for_status()
            return r.json()["issue"]

    async def create_time_entry(
        self,
        issue_id: int,
        hours: float,
        spent_on: date,
        comments: str,
        activity_id: int | None = None,
        user_id: int | None = None,
    ) -> dict:
        payload = {
            "time_entry": {
                "issue_id": issue_id,
                "hours": hours,
                "spent_on": spent_on.isoformat(),
                "comments": comments,
            }
        }
        if activity_id:
            payload["time_entry"]["activity_id"] = activity_id
        if user_id:
            payload["time_entry"]["user_id"] = user_id

        url = f"{self.base_url}/time_entries.json"
        async with httpx.AsyncClient(timeout=20) as client:
            r = await client.post(url, headers=self.headers, json=payload)
            r.raise_for_status()
            return r.json()
```

### 7.2 Планировщик APScheduler

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from zoneinfo import ZoneInfo

scheduler = AsyncIOScheduler(timezone=ZoneInfo("Europe/Riga"))


def register_schedule(schedule):
    trigger = CronTrigger.from_crontab(
        schedule.cron_expr,
        timezone=ZoneInfo(schedule.timezone or "Europe/Riga"),
    )
    scheduler.add_job(
        func=execute_schedule,
        trigger=trigger,
        id=f"schedule:{schedule.id}",
        kwargs={"schedule_id": schedule.id, "trigger_type": "cron"},
        replace_existing=True,
        coalesce=True,
        max_instances=1,
        misfire_grace_time=300,
    )
```

## 8) Пример docker-compose

```yaml
version: "3.9"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: redmine_repeater
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d redmine_repeater"]
      interval: 5s
      timeout: 3s
      retries: 20

  web:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000
    environment:
      DATABASE_URL: postgresql+psycopg://app:app@db:5432/redmine_repeater
      APP_TZ: Europe/Riga
      SECRET_KEY: change_me
      ENCRYPTION_KEY: change_me_32bytes
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

  scheduler:
    build: .
    command: python -m app.scheduler
    environment:
      DATABASE_URL: postgresql+psycopg://app:app@db:5432/redmine_repeater
      APP_TZ: Europe/Riga
      SECRET_KEY: change_me
      ENCRYPTION_KEY: change_me_32bytes
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

## 9) Таймзона Europe/Riga (важные правила)

- Хранить все timestamps в БД в UTC (`timestamptz`).
- Локальную дату списания (`spent_on`) вычислять через `ZoneInfo("Europe/Riga")` в момент запуска.
- Cron интерпретировать в timezone пользователя (`Europe/Riga`), не в UTC.
- Учитывать DST автоматически через IANA timezone (`Europe/Riga`).

Пример:
- `now_utc -> now_local = now_utc.astimezone(ZoneInfo("Europe/Riga"))`
- `spent_on = now_local.date()`

## 10) Практика дедупликации

Рекомендуемый порядок:
1. Сформировать `dedupe_key`:
   - `issue_id|spent_on|hours_rounded|comment_norm|charge_user_id`
2. Поставить локальный lock в БД (`INSERT ... ON CONFLICT DO NOTHING`).
3. Если доступно чтение Redmine time entries — сделать remote check.
4. Если найден дубль — пометить run как `skipped_duplicate`.
5. Если нет — отправить `POST /time_entries`.
6. Добавлять marker в комментарий: `"Daily meeting [rt:12:2025-01-15]"`.

Если у Redmine нет доступа к чтению time entries:
- полагаться на локальный lock + marker,
- и на идемпотентность job run (повторные попытки для одного run не создают новую запись).

## 11) Примеры запросов к Redmine

Проверка issue:
```bash
curl -H "X-Redmine-API-Key: <API_KEY>" \
  "<REDMINE_URL>/issues/123.json"
```

Создание time entry:
```bash
curl -X POST -H "Content-Type: application/json" \
  -H "X-Redmine-API-Key: <API_KEY>" \
  -d '{
    "time_entry": {
      "issue_id": 123,
      "hours": 0.5,
      "spent_on": "2026-02-20",
      "comments": "Daily standup [rt:5:2026-02-20]",
      "activity_id": 9
    }
  }' \
  "<REDMINE_URL>/time_entries.json"
```

## 12) План MVP на 1–3 дня

### День 1
- Инициализация FastAPI проекта + SQLAlchemy/Alembic.
- Таблицы: users, redmine_connections, schedules, job_runs, dedupe_locks.
- Базовый login (single-user), CRUD расписаний.
- Валидация issue при создании/изменении.

### День 2
- APScheduler процесс + загрузка расписаний из БД.
- Execution service: создание time entry + retry/backoff.
- Дедуп (локальный lock + marker).
- Логи запусков и страница истории.

### День 3
- Ручной запуск “Run now”.
- Enable/disable, last status в таблице расписаний.
- Dockerfile + docker-compose.
- Полировка UI (таблица + форма + фильтр ошибок).
- Smoke-тесты и чек-лист деплоя.

## 13) Допущения и расширения

- Шаг округления часов (`0.25`) хранится как настройка расписания.
- Если нужен multi-user, можно расширить модель прав и связей пользователя с Redmine ключом.
- Для production: добавить rate limit, audit logs, structured logging и Sentry.
