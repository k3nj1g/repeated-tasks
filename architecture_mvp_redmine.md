# MVP: локальный сервис повторяющихся списаний трудозатрат в Redmine

## Что меняется относительно предыдущей версии

- Приложение **не публикуется наружу** (не нужен публичный деплой).
- Нужна только **удобная локальная настройка заданий**.
- Автоматический запуск делаем через **локальный `crontab`**, а не через встроенный scheduler-сервис в docker-compose.

---

## 1) Два стека реализации

### Стек A (рекомендованный): Python + FastAPI (локально) + SQLite + crontab

**Состав:**
- **UI/API**: FastAPI + простые HTML-шаблоны (или HTMX)
- **Хранение**: SQLite (один локальный пользователь)
- **Запуск по расписанию**: системный `crontab` (каждую минуту вызывает runner)
- **Интеграция**: Redmine REST API

**Плюсы:**
- Минимум инфраструктуры, очень быстро собрать.
- Удобно конфигурировать задания через локальный веб-интерфейс.
- Не нужен отдельный worker/broker.

**Минусы:**
- Не рассчитано на многопользовательскую нагрузку.
- Для сложных очередей/распределённого запуска потребуется миграция на более тяжёлый стек.

---

### Стек B: Node.js + Express + SQLite/PostgreSQL + node-cron + crontab-trigger

**Состав:**
- **UI/API**: Express + EJS/React
- **Хранение**: SQLite (локально) или PostgreSQL
- **Запуск**: `crontab` вызывает Node-скрипт `run-due-jobs`

**Плюсы:**
- TS/JS-экосистема, удобно если команда уже на Node.
- Просто расширять API.

**Минусы:**
- Для MVP чуть больше шаблонного кода, чем в FastAPI.

---

## 2) Рекомендация для текущего сценария

Так как запуск локальный и без публикации, лучший вариант — **Стек A (Python + FastAPI + SQLite + crontab)**.

Почему SQLite:
- локальный single-user сценарий;
- нулевой overhead на администрирование БД;
- достаточно надёжно для cron-runner с транзакциями и уникальными индексами.

Если позже появится multi-user/серверный режим — перейти на PostgreSQL.

---

## 3) Архитектура (локальный режим)

Компоненты:
1. **Локальный UI/API (FastAPI)**
   - CRUD расписаний
   - ручной запуск
   - просмотр логов
   - хранение Redmine API key (шифрованно)
2. **Cron runner (CLI-скрипт)**
   - запускается `crontab` каждую минуту
   - выбирает due-задания
   - выполняет создание time entry
   - пишет результат в лог-таблицу
3. **SQLite БД**
   - расписания, логи, dedupe lock, секреты
4. **Redmine API**
   - `GET /issues/{id}.json`
   - `POST /time_entries.json`
   - (опц.) `GET /time_entries.json` для дедупа

### Поток
1. Пользователь на `http://127.0.0.1:8000` создаёт/меняет расписание.
2. `crontab` каждую минуту вызывает `python -m app.run_due_jobs`.
3. Runner берёт активные задания и проверяет: “должно ли выполниться сейчас в Europe/Riga?”.
4. Делает дедуп-проверку.
5. Создаёт time entry в Redmine.
6. Сохраняет лог выполнения.

---

## 4) Схема БД

### `app_settings`
- `id` (PK)
- `redmine_base_url` (text)
- `api_key_encrypted` (blob/text)
- `created_at`
- `updated_at`

### `schedules`
- `id` (PK)
- `name` (text)
- `enabled` (bool)
- `timezone` (text, default `Europe/Riga`)
- `cron_expr` (text, nullable) — если пользователь ввёл cron
- `days_of_week` (text, nullable) — например `1,2,3,4,5`
- `run_time_local` (text, nullable, `HH:MM`) — для упрощённого UI
- `issue_id` (int)
- `hours` (real/decimal)
- `rounding_step` (real, default `0.25`)
- `activity_id` (int, nullable)
- `comments` (text)
- `skip_weekends` (bool default false)
- `charge_user_id` (int, nullable)
- `dedupe_mode` (text: `local`/`hybrid`)
- `last_run_at` (datetime, nullable)
- `last_status` (text, nullable)
- `created_at`
- `updated_at`

### `job_runs`
- `id` (PK)
- `schedule_id` (FK)
- `trigger_type` (`cron`/`manual`)
- `run_at_utc` (datetime)
- `entry_date_local` (date)
- `status` (`success`/`error`/`skipped_duplicate`/`skipped_weekend`/`not_due`)
- `attempt` (int)
- `request_payload` (json text)
- `response_payload` (json text)
- `error_text` (text)
- `redmine_time_entry_id` (int, nullable)

### `dedupe_locks`
- `id` (PK)
- `schedule_id` (FK)
- `entry_date_local` (date)
- `dedupe_key` (text)
- `created_at`
- `UNIQUE(schedule_id, entry_date_local, dedupe_key)`

---

## 5) API (локальный UI использует эти же эндпойнты)

### Настройки
- `GET /api/settings`
- `PUT /api/settings` — `redmine_base_url`, `api_key`

### Расписания
- `GET /api/schedules`
- `POST /api/schedules`
- `GET /api/schedules/{id}`
- `PUT /api/schedules/{id}`
- `DELETE /api/schedules/{id}`
- `POST /api/schedules/{id}/enable`
- `POST /api/schedules/{id}/disable`
- `POST /api/schedules/{id}/run-now`

### Логи
- `GET /api/schedules/{id}/runs`
- `GET /api/runs?status=error`

### Валидация
- `POST /api/redmine/validate-issue`

---

## 6) Ключевые сценарии

### A. Создание расписания
1. Пользователь вводит имя, `issue_id`, `hours`, cron (или дни+время).
2. Бэкенд округляет `hours` по `rounding_step` (обычно 0.25).
3. Проверяет issue: `GET /issues/{id}.json`.
4. Сохраняет запись.

### B. Запуск через crontab
1. Каждую минуту runner получает список `enabled` расписаний.
2. Для каждого вычисляет локальное время `Europe/Riga`.
3. Проверяет, due ли задача в текущую минуту.
4. Если выходной и `skip_weekends=true` — skip.
5. Дедуп.
6. POST `time_entries`.
7. Запись в `job_runs`.

### C. Дедупликация
Рекомендуемый ключ:
`issue_id|entry_date|hours_rounded|comment_norm|charge_user_id`

Алгоритм:
1. Пытаемся вставить локальный lock (`UNIQUE` защищает от дублей).
2. Если доступно чтение `time_entries` в Redmine — сверяем удалённо.
3. Если чтение недоступно — работаем только на локальном lock + marker в comment: `[rt:<schedule_id>:<YYYY-MM-DD>]`.

### D. Ошибки и retries
- retry: только для network timeout / 5xx (например 3 попытки: 2s, 5s, 15s)
- 4xx: без retry
- все ошибки писать в `job_runs.error_text`

---

## 7) Минимальный скелет кода

### 7.1 Клиент Redmine

```python
import httpx
from datetime import date


class RedmineClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {"X-Redmine-API-Key": api_key}

    async def validate_issue(self, issue_id: int) -> dict:
        url = f"{self.base_url}/issues/{issue_id}.json"
        async with httpx.AsyncClient(timeout=20) as client:
            r = await client.get(url, headers=self.headers)
            r.raise_for_status()
            return r.json().get("issue", {})

    async def create_time_entry(self, *, issue_id: int, hours: float, spent_on: date, comments: str, activity_id: int | None = None, user_id: int | None = None) -> dict:
        payload = {
            "time_entry": {
                "issue_id": issue_id,
                "hours": hours,
                "spent_on": spent_on.isoformat(),
                "comments": comments,
            }
        }
        if activity_id is not None:
            payload["time_entry"]["activity_id"] = activity_id
        if user_id is not None:
            payload["time_entry"]["user_id"] = user_id

        url = f"{self.base_url}/time_entries.json"
        async with httpx.AsyncClient(timeout=20) as client:
            r = await client.post(url, headers=self.headers, json=payload)
            r.raise_for_status()
            return r.json()
```

### 7.2 Runner, который вызывается crontab

```python
# python -m app.run_due_jobs
from datetime import datetime
from zoneinfo import ZoneInfo

TZ = ZoneInfo("Europe/Riga")


def run_due_jobs(db):
    now_local = datetime.now(tz=TZ)
    schedules = db.get_enabled_schedules()

    for s in schedules:
        if not is_due_now(schedule=s, now_local=now_local):
            continue

        execute_schedule(schedule=s, now_local=now_local, trigger_type="cron")
```

### 7.3 crontab

```cron
* * * * * cd /opt/redmine-repeater && /opt/redmine-repeater/.venv/bin/python -m app.run_due_jobs >> /opt/redmine-repeater/logs/cron.log 2>&1
```

---

## 8) Локальный запуск (без публикации наружу)

### Вариант без Docker
1. `python -m venv .venv && source .venv/bin/activate`
2. `pip install -r requirements.txt`
3. `uvicorn app.main:app --host 127.0.0.1 --port 8000`
4. Открыть локально `http://127.0.0.1:8000`
5. Добавить `crontab -e` строку из примера выше

### Вариант с Docker (только локально)

```yaml
version: "3.9"
services:
  web:
    build: .
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000
    environment:
      APP_TZ: Europe/Riga
      DB_PATH: /data/app.db
      ENCRYPTION_KEY: change_me_32bytes
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      - ./data:/data
```

> В этом сценарии cron можно оставить на хосте и вызывать `docker exec <container> python -m app.run_due_jobs`.

---

## 9) Таймзона Europe/Riga

Правила:
- В БД хранить timestamps в UTC.
- Для `spent_on` всегда брать локальную дату `Europe/Riga` в момент выполнения.
- Проверку “due now” считать в `Europe/Riga`.
- DST закрывается автоматически через `zoneinfo`.

Пример:

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo

now_utc = datetime.now(timezone.utc)
now_riga = now_utc.astimezone(ZoneInfo("Europe/Riga"))
spent_on = now_riga.date()
```

---

## 10) Если Redmine не даёт читать time_entries

Fallback-стратегия:
1. строгий локальный dedupe lock по уникальному ключу;
2. marker в комментарии (`[rt:schedule_id:date]`);
3. лог `job_runs` как источник факта списания;
4. ручная кнопка “Run now” должна проверять lock и не дублировать запись.

---

## 11) Пошаговый MVP план (1–3 дня)

### День 1
- FastAPI + SQLite + миграции
- формы CRUD расписаний
- сохранение Redmine URL/API key (шифрование)
- validate issue endpoint

### День 2
- `run_due_jobs` + crontab
- создание time entry + retries
- dedupe lock + marker
- таблица логов запусков

### День 3
- кнопка “выполнить сейчас”
- enable/disable
- полировка UI (простая таблица + фильтры ошибок)
- smoke-тесты локального сценария

