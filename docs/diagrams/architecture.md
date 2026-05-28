# Architecture Diagrams

## System Overview

```
┌─────────────────────────────────────────────────────┐
│                  Interface Layer                     │
│         Telegram Bot  (Phase 1 → Voice Phase 5)      │
└────────────────────┬────────────────────────────────┘
                     │ message
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

---

## Message Flow — One-shot Reminder

```
User (Telegram)
    │
    │  "recuérdame llamar al médico mañana a las 10"
    ▼
Bot Handler (python-telegram-bot)
    │
    │  raw text + user_id
    ▼
Orchestrator (llm/orchestrator.py)
    │  1. Load history from Redis
    │  2. Build LLM request (system prompt + history + message)
    ▼
Ollama (llama3.1:8b)
    │
    │  tool_call: create_reminder(message="Llamar al médico",
    │                              trigger_at="2025-XX-XX 10:00")
    ▼
Tool Dispatcher (llm/tools.py)
    │
    ▼
Reminders Service (services/reminders.py)
    │  1. INSERT into reminders (PostgreSQL)
    │  2. Schedule job in APScheduler
    │  3. Return confirmation text
    ▼
Orchestrator
    │  Save exchange to Redis history
    ▼
Bot Handler
    │
    │  "¡Anotado! Te recuerdo llamar al médico mañana a las 10:00 ⏰"
    ▼
User (Telegram)

--- Later, at 10:00 ---

APScheduler fires job
    │
    ▼
Telegram: "⏰ Llamar al médico"
```

---

## Message Flow — Shopping Session

```
User: "voy a hacer la compra"
    │
    ▼
Orchestrator → tool: start_shopping_session()
    │
    ▼
Shopping Service
    │  1. Create shopping_session in PG
    │  2. Query pending shopping_items
    │  3. Group by category via LLM
    ▼
Bot: "🛒 Tienes 5 cosas pendientes:
      Lácteos: leche, yogur
      Frutas: plátanos
      Otros: papel de cocina, detergente"

User: "ya he vuelto"
    │
    ▼
Orchestrator → tool: finish_shopping_session(bought_ids=[...], skipped_ids=[...])
    │
    ▼
Shopping Service: mark items bought/skipped, close session
    │
    ▼
Bot: "✅ ¡Perfecto! He marcado todo como comprado."
```

---

## Docker Compose Services

```
┌─────────────────────────────────────────────┐
│              Docker Compose                  │
│                                             │
│  ┌──────────┐    ┌──────────┐               │
│  │  hermes  │───▶│ postgres │               │
│  │ FastAPI  │    │  :5432   │               │
│  │  :8000   │    └──────────┘               │
│  │          │    ┌──────────┐               │
│  │          │───▶│  redis   │               │
│  │          │    │  :6379   │               │
│  │          │    └──────────┘               │
│  │          │    ┌──────────┐               │
│  │          │───▶│  ollama  │               │
│  │          │    │ :11434   │               │
│  └──────────┘    └──────────┘               │
│                                             │
└─────────────────────────────────────────────┘
              │
         Raspberry Pi 5
```
