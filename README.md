# Hermes — Personal Assistant

> Self-hosted, conversational, extensible personal assistant deployed on a Raspberry Pi 5.  
> Stack: Python · FastAPI · PostgreSQL · Redis · Ollama · Telegram · Docker Compose

---

## Overview

**Hermes** is a self-hosted personal assistant that lets you interact in natural language to manage reminders, shopping lists, events, and daily tasks. All processing runs locally on a Raspberry Pi 5 (16 GB RAM) — no paid external services required.

The bot speaks Spanish and understands natural language commands like:
- *"Recuérdame poner la lavadora en una hora"*
- *"Apunta leche, huevos y pan a la lista"*
- *"¿Qué tengo mañana?"*

---

## Features

| Feature | Description |
|---|---|
| One-shot reminders | Set a reminder for a specific date and time |
| Recurring reminders | Cron-style recurring reminders ("every day at 17:00") |
| Shopping list | Add items, group by category, confirm when back from shopping |
| Events & calendar | Create events, sync with Google Calendar |
| Conversational history | Context maintained across messages within a session |

---

## Architecture

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

## Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Language | Python 3.12 | Developer's primary language |
| API backend | FastAPI | Native async, Pydantic validation |
| ORM | SQLAlchemy 2.x (async) | Mature async ORM |
| Migrations | Alembic | Standard with SQLAlchemy |
| Database | PostgreSQL 16 | Robust, JSON support |
| Cache / context | Redis 7 | Ephemeral conversational context, lightweight job queue |
| LLM | Ollama + llama3.1:8b | 100% local, free, fully private |
| Scheduling | APScheduler 3.x | Jobs persisted in DB, one-shot and cron |
| Bot interface | python-telegram-bot v21 | Immediate text channel in Phase 1 |
| Calendar | Google Calendar API v3 | Bidirectional event sync |
| STT (Phase 5) | faster-whisper | Local, good Spanish accuracy |
| TTS (Phase 5) | Piper TTS | Lightweight, works on ARM64 |
| Containers | Docker Compose | Reproducible deployment on the Pi |

---

## Repository Structure

```
hermes-assistant/
├── README.md
├── SPEC.md                        # Full project specification
├── PROGRESS.md                    # Development progress tracker
├── Makefile
├── docker-compose.yml
├── docker-compose.dev.yml
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
│   ├── main.py                    # FastAPI entrypoint
│   ├── config.py                  # Settings (pydantic-settings)
│   ├── api/
│   ├── bot/
│   ├── llm/
│   ├── services/
│   ├── models/
│   ├── db/
│   └── utils/
│
└── tests/
    ├── conftest.py
    ├── unit/
    └── integration/
```

---

## Getting Started

### Prerequisites

- Docker and Docker Compose
- A Telegram bot token ([create one via @BotFather](https://t.me/BotFather))
- Your Telegram user ID

### Setup

1. Clone the repo and copy the env file:
   ```bash
   git clone <repo-url>
   cd hermes-assistant
   cp .env.example .env
   # Edit .env with your values
   ```

2. Start the stack:
   ```bash
   make up
   # or: docker compose up -d
   ```

3. Run database migrations:
   ```bash
   make migrate
   # or: docker compose exec hermes alembic upgrade head
   ```

4. Pull the LLM model:
   ```bash
   docker compose exec ollama ollama pull llama3.1:8b
   ```

### Useful commands

```bash
make up        # Start all services
make down      # Stop all services
make logs      # Follow logs
make migrate   # Run Alembic migrations
make shell     # Open a shell in the hermes container
```

---

## Environment Variables

See [.env.example](.env.example) for the full list. Key variables:

| Variable | Description |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Bot token from @BotFather |
| `TELEGRAM_ALLOWED_USER_IDS` | Your Telegram user ID (single-user setup) |
| `OLLAMA_MODEL` | LLM model to use (default: `llama3.1:8b`) |
| `DATABASE_URL` | PostgreSQL async connection string |
| `REDIS_URL` | Redis connection string |

---

## Development Phases

| Phase | Goal | Status |
|---|---|---|
| 0 — Base infrastructure | Docker stack, DB schema, FastAPI skeleton | Not started |
| 1 — Text conversation core | Telegram bot + reminders + basic shopping list | Not started |
| 2 — Full reminders & shopping | Recurring reminders, full shopping flow | Not started |
| 3 — Google Calendar | Bidirectional event sync | Not started |
| 4 — Polish & observability | Logging, metrics, tests | Not started |
| 5 — Voice interface | Wake word, STT, TTS | Not started |

See [PROGRESS.md](PROGRESS.md) for detailed task tracking.

---

## Hardware

- **Board**: Raspberry Pi 5, 16 GB RAM
- **Storage**: M.2 NVMe (system + data) + external drives (backups)
- **OS**: Debian Bookworm (ARM64)

---

## Design Decisions

**Why local Ollama instead of an external API?**  
Full privacy — messages never leave the local network. No per-call cost. Acceptable latency (~2–5s per response on Pi 5 with 16 GB RAM).

**Why Telegram and not a custom app?**  
Eliminates weeks of frontend work. Native push notifications on mobile. Stable, well-documented API with a mature Python client.

**Why APScheduler and not Celery?**  
Celery adds significant complexity. APScheduler with `SQLAlchemyJobStore` covers the use case perfectly for a single-user, low-volume assistant.

---

*See [SPEC.md](SPEC.md) for the full technical specification.*
