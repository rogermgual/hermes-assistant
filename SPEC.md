# Hermes — Technical Specification

> Self-hosted personal assistant on Raspberry Pi 5.  
> Stack: Python · FastAPI · PostgreSQL · Redis · Ollama · Telegram · Docker Compose

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [System Architecture](#2-system-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Data Model](#4-data-model)
5. [Core Services](#5-core-services)
6. [Intent System (LLM + tools)](#6-intent-system-llm--tools)
7. [Development Phases](#7-development-phases)
8. [Repository Structure](#8-repository-structure)
9. [Infrastructure & Deployment](#9-infrastructure--deployment)
10. [Design Decisions](#10-design-decisions)

---

## 1. System Overview

**Hermes** is a self-hosted personal assistant that allows the user to interact in natural language to manage reminders, shopping lists, events, and daily tasks. All processing runs locally on a Raspberry Pi 5 (16 GB RAM), with no dependencies on paid external services.

### Target Features

| Feature | Description |
|---|---|
| One-shot reminders | "Recuérdame poner la lavadora en una hora" |
| Recurring reminders | "Cada día a las 17:00 pregúntame qué he hecho" |
| Shopping list | Item accumulation, list on shopping, confirmation on return |
| Events & calendar | Birthdays, appointments, events synced with Google Calendar |
| Conversational history | Context across messages within a session |

### Design Principles

- **Fully local**: no data leaves the home network (except Telegram API for message delivery).
- **Conversational**: the user speaks naturally; Hermes interprets intent and acts.
- **Extensible**: tool-based architecture that allows adding new capabilities without touching the orchestrator.
- **Persistent**: all reminders and data survive system restarts.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Interface Layer                     │
│         Telegram Bot  (Phase 1 → Voice Phase 5)      │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              LLM Orchestrator                        │
│   Ollama (llama3.1:8b)  ·  Function Calling          │
│   Interprets intent → selects tool                   │
└──────┬─────────────┬──────────────┬─────────────────┘
       │             │              │
┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────────┐
│  Reminders  │ │ Shopping  │ │   Calendar    │
│ APScheduler │ │   CRUD    │ │  Google Cal   │
│  one-shot   │ │  states   │ │  CalDAV       │
│ recurrence  │ │  history  │ │  birthdays    │
└──────┬──────┘ └────┬──────┘ └────┬──────────┘
       │             │              │
┌──────▼─────────────▼──────────────▼─────────────────┐
│                 Sync Layer                           │
│          CalDAV / Google Calendar API                │
│          Webhooks · Push notifications               │
└──────────────┬──────────────────┬───────────────────┘
               │                  │
┌──────────────▼──────┐  ┌────────▼───────────────────┐
│     PostgreSQL       │  │          Redis             │
│  reminders           │  │  conversational context    │
│  events              │  │  pending jobs              │
│  shopping list       │  │  session cache             │
│  history             │  └────────────────────────────┘
└─────────────────────┘
```

All services run via **Docker Compose** on the Raspberry Pi 5.

---

## 3. Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Language | Python 3.12 | Developer's primary language |
| API backend | FastAPI | Native async, Pydantic validation, easy to test |
| ORM | SQLAlchemy 2.x (async) | Mature async ORM |
| Migrations | Alembic | Standard with SQLAlchemy |
| Database | PostgreSQL 16 | Robust, JSON support, developer familiarity |
| Cache / context | Redis 7 | Ephemeral conversational context, lightweight job queue |
| LLM | Ollama + llama3.1:8b | 100% local, no per-call cost, full privacy |
| Scheduling | APScheduler 3.x | Jobs persisted in DB, one-shot and cron |
| Bot interface | python-telegram-bot v21 | Immediate text channel in Phase 1 |
| Calendar | Google Calendar API v3 | Bidirectional event sync |
| STT (Phase 5) | faster-whisper | Local, good Spanish accuracy |
| TTS (Phase 5) | Piper TTS | Lightweight, works on ARM64 |
| Containers | Docker Compose | Reproducible deployment on the Pi |

### Ollama Model Versions

- **Primary**: `llama3.1:8b` — good quality/speed balance on Pi 5 with 16 GB
- **Lightweight fallback**: `qwen2.5:3b` — for fast responses or high load
- Models are downloaded once and persisted in a Docker volume

---

## 4. Data Model

### PostgreSQL Tables

```sql
-- Reminders
CREATE TABLE reminders (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT NOT NULL,           -- Telegram user ID
    message     TEXT NOT NULL,             -- Reminder text
    trigger_at  TIMESTAMPTZ,               -- For one-shot
    cron_expr   TEXT,                      -- For recurring (e.g. "0 17 * * *")
    is_active   BOOLEAN DEFAULT TRUE,
    job_id      TEXT,                      -- APScheduler job ID
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    fired_at    TIMESTAMPTZ                -- Last time it fired
);

-- Shopping items
CREATE TABLE shopping_items (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT NOT NULL,
    name        TEXT NOT NULL,
    quantity    TEXT,                      -- "2 kg", "a dozen", free-form
    category    TEXT,                      -- Normalised by LLM
    status      TEXT DEFAULT 'pending',    -- pending | bought | skipped
    added_at    TIMESTAMPTZ DEFAULT NOW(),
    bought_at   TIMESTAMPTZ,
    session_id  UUID                       -- Shopping session this item belongs to
);

-- Shopping sessions
CREATE TABLE shopping_sessions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT NOT NULL,
    started_at  TIMESTAMPTZ DEFAULT NOW(),
    finished_at TIMESTAMPTZ,
    status      TEXT DEFAULT 'active'      -- active | finished
);

-- Events / calendar
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         BIGINT NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ,
    is_recurring    BOOLEAN DEFAULT FALSE,
    recurrence_rule TEXT,                  -- iCal RRULE
    gcal_event_id   TEXT,                  -- Google Calendar event ID
    reminder_before INTERVAL,             -- e.g. '30 minutes'
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Conversation history (per session)
CREATE TABLE conversation_history (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     BIGINT NOT NULL,
    session_id  TEXT NOT NULL,             -- Redis key prefix
    role        TEXT NOT NULL,             -- 'user' | 'assistant'
    content     TEXT NOT NULL,
    timestamp   TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis Keys

| Key pattern | Content | TTL |
|---|---|---|
| `session:{user_id}` | Last N conversation messages (JSON) | 2 hours |
| `job:pending:{user_id}` | Reminder jobs pending confirmation | 5 minutes |
| `shopping:active:{user_id}` | Active shopping session ID | 24 hours |

---

## 5. Core Services

### 5.1 Reminders Service

Responsibilities:
- Create one-shot (`trigger_at`) and recurring (`cron_expr`) jobs in APScheduler
- Persist jobs in PostgreSQL to survive restarts
- Fire Telegram notification when a reminder is due
- CRUD: list, cancel, modify reminders

One-shot flow:
```
User: "recuérdame llamar al médico mañana a las 10"
→ LLM extracts: { type: "one-shot", message: "Llamar al médico", trigger_at: "2025-XX-XX 10:00" }
→ Reminder created in PG + job in APScheduler
→ At 10:00 → Telegram: "⏰ Llamar al médico"
```

Recurring flow:
```
User: "cada día a las 17:00 pregúntame qué he hecho hoy"
→ LLM extracts: { type: "cron", message: "¿Qué has hecho hoy?", cron: "0 17 * * *" }
→ Reminder + persisted cron job created
→ Every day at 17:00 → Telegram: "📋 ¿Qué has hecho hoy?"
```

### 5.2 Shopping List Service

Item states:
```
pending → bought
       → skipped
```

Main flows:
- **Add**: "necesitamos huevos", "apunta leche y pan"
- **Go shopping**: "voy a hacer la compra" → lists `pending` items grouped by category
- **Return**: "ya he vuelto de comprar" → Hermes confirms items one by one
- **History**: what was bought in previous sessions

### 5.3 Calendar & Events Service

- Sync with Google Calendar (OAuth2, credentials stored locally)
- Automatic reminders X minutes before an event
- Natural language queries: "¿qué tengo mañana?", "¿cuándo es el cumpleaños de Ana?"
- Event creation: "el viernes 13 tengo cita con el dentista a las 11"

---

## 6. Intent System (LLM + tools)

The orchestrator uses **Ollama with function calling**. The LLM does not generate free-form text for actions — it selects tools with typed parameters.

### Registered Tools

```python
tools = [
    {
        "name": "create_reminder",
        "description": "Creates a one-shot or recurring reminder",
        "parameters": {
            "message": "str",
            "trigger_at": "datetime | None",
            "cron_expr": "str | None"
        }
    },
    {
        "name": "list_reminders",
        "description": "Lists the user's active reminders"
    },
    {
        "name": "delete_reminder",
        "parameters": { "reminder_id": "uuid" }
    },
    {
        "name": "add_shopping_item",
        "parameters": {
            "items": "list[{ name: str, quantity: str | None }]"
        }
    },
    {
        "name": "list_shopping_items",
        "description": "Lists pending items to buy"
    },
    {
        "name": "start_shopping_session",
        "description": "Starts a shopping session and lists items"
    },
    {
        "name": "finish_shopping_session",
        "parameters": {
            "bought_ids": "list[uuid]",
            "skipped_ids": "list[uuid]"
        }
    },
    {
        "name": "create_event",
        "parameters": {
            "title": "str",
            "starts_at": "datetime",
            "ends_at": "datetime | None",
            "reminder_before_minutes": "int | None"
        }
    },
    {
        "name": "list_events",
        "parameters": {
            "from_date": "date",
            "to_date": "date"
        }
    },
    {
        "name": "reply_text",
        "description": "Replies to the user with free text (questions, confirmations, conversation)",
        "parameters": { "text": "str" }
    }
]
```

### Conversational Context Management

History is stored in Redis (sliding window of the last 10 messages) and injected into every LLM call for conversational coherence.

```python
# LLM message structure
{
    "model": "llama3.1:8b",
    "messages": [
        {"role": "system", "content": SYSTEM_PROMPT},
        *history_from_redis,        # Last N turns
        {"role": "user", "content": user_message}
    ],
    "tools": TOOLS_DEFINITION
}
```

### Base System Prompt

```
Eres Hermes, un asistente personal local y conversacional.
Ayudas al usuario con recordatorios, lista de la compra, eventos y tareas del día a día.
Cuando el usuario pida una acción, usa siempre la herramienta correspondiente.
Para respuestas conversacionales, usa reply_text.
La fecha y hora actual es: {current_datetime}.
Responde siempre en español, de forma concisa y amigable.
```

---

## 7. Development Phases

See [PROGRESS.md](PROGRESS.md) for the full task checklist.

| Phase | Goal | Estimated Duration |
|---|---|---|
| 0 | Base infrastructure | 1–2 weeks |
| 1 | Text conversation core | 2–3 weeks |
| 2 | Recurring reminders & full shopping | 1–2 weeks |
| 3 | Google Calendar integration | 1–2 weeks |
| 4 | Polish & observability | 1 week |
| 5 | Voice interface | 3–4 weeks |

---

## 8. Repository Structure

```
hermes-assistant/
├── README.md
├── SPEC.md                        # This document
├── PROGRESS.md                    # Development progress tracker
├── Makefile
├── docker-compose.yml
├── docker-compose.dev.yml         # Development overrides
├── .env.example
│
├── docs/
│   └── diagrams/                  # Architecture and flow diagrams
│
├── alembic/
│   ├── env.py
│   └── versions/
│
├── hermes/                        # Main Python package
│   ├── __init__.py
│   ├── main.py                    # FastAPI entrypoint
│   ├── config.py                  # Settings (pydantic-settings)
│   │
│   ├── api/
│   │   ├── routes/
│   │   │   ├── health.py
│   │   │   ├── reminders.py
│   │   │   ├── shopping.py
│   │   │   └── events.py
│   │   └── deps.py                # FastAPI dependencies (DB session, etc.)
│   │
│   ├── bot/
│   │   ├── telegram.py            # Main bot handler
│   │   └── handlers/
│   │       ├── message.py         # Message reception and routing
│   │       └── commands.py        # /start, /help, etc.
│   │
│   ├── llm/
│   │   ├── client.py              # Async Ollama client
│   │   ├── orchestrator.py        # Main loop: message → tool → response
│   │   ├── tools.py               # Tool definitions and registry
│   │   └── prompts.py             # System prompts
│   │
│   ├── services/
│   │   ├── reminders.py           # Reminder business logic
│   │   ├── shopping.py            # Shopping list business logic
│   │   ├── events.py              # Events and calendar logic
│   │   └── scheduler.py           # APScheduler setup and job management
│   │
│   ├── models/
│   │   ├── db/                    # SQLAlchemy models
│   │   │   ├── reminder.py
│   │   │   ├── shopping.py
│   │   │   └── event.py
│   │   └── schemas/               # Pydantic schemas (request/response)
│   │       ├── reminder.py
│   │       ├── shopping.py
│   │       └── event.py
│   │
│   ├── db/
│   │   ├── session.py             # AsyncSession factory
│   │   └── redis.py               # Async Redis client
│   │
│   └── utils/
│       ├── datetime.py            # Date and timezone helpers
│       └── text.py                # Text normalisation
│
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

---

## 9. Infrastructure & Deployment

### Hardware

- **Board**: Raspberry Pi 5, 16 GB RAM
- **Storage**: M.2 NVMe (system + data) + external drives (backups)
- **OS**: Debian Bookworm (ARM64)

### docker-compose.yml (base structure)

```yaml
version: "3.9"

services:
  hermes:
    build: .
    restart: unless-stopped
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      ollama:
        condition: service_started
    ports:
      - "8000:8000"

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: hermes
      POSTGRES_USER: hermes
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U hermes"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    volumes:
      - ollama_models:/root/.ollama

volumes:
  postgres_data:
  redis_data:
  ollama_models:
```

### Environment Variables (.env.example)

```env
# App
APP_ENV=production
LOG_LEVEL=INFO
TZ=Europe/Madrid

# PostgreSQL
POSTGRES_PASSWORD=changeme
DATABASE_URL=postgresql+asyncpg://hermes:changeme@postgres:5432/hermes

# Redis
REDIS_URL=redis://redis:6379/0

# Ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_MODEL=llama3.1:8b
OLLAMA_TIMEOUT=120

# Telegram
TELEGRAM_BOT_TOKEN=your_token_here
TELEGRAM_ALLOWED_USER_IDS=your_user_id

# Google Calendar (Phase 3)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=http://localhost:8080/auth/google/callback
```

---

## 10. Design Decisions

### Why local Ollama instead of an external API?

- **Full privacy**: messages never leave the local network.
- **No per-call cost**: the model is downloaded once.
- **Acceptable latency**: llama3.1:8b on Pi 5 with 16 GB takes 2–5 seconds per response, acceptable for a personal assistant.
- **Risk**: if the model struggles with function calling in Spanish, the system prompt can be tuned or `qwen2.5:7b` / `mistral:7b` can be tried.

### Why Telegram and not a custom app?

- Eliminates weeks of frontend work in Phase 1.
- Native push notifications on mobile without additional infrastructure.
- Stable, well-documented API with a mature Python client.
- Does not require exposing ports externally (uses long polling or webhooks over Ngrok/Tailscale).

### Why APScheduler and not Celery?

- Celery is more robust but adds significant complexity (workers, broker, flower).
- APScheduler with `SQLAlchemyJobStore` covers the use case perfectly (low volume, single user).
- If it needs to scale, migration is straightforward.

### Timezone handling

- Everything stored in UTC in the database.
- Converted to `Europe/Madrid` in the presentation layer and when parsing LLM dates.
- The system prompt includes the current local datetime on every call.
