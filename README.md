# ⏱️ Timely — Slack-based Oracle HCM Timecard Assistant

> **Oracle Code Assist Hackathon Project**
> A Slack bot that acts as your Oracle HCM digital twin — reads last week's timecard, drafts this week's entries, and submits them to Oracle HCM, all without leaving Slack.

---

## 🧠 What It Does

Timely eliminates the weekly friction of logging into Oracle HCM to submit timecards. Instead, it meets employees where they already are — in Slack — and guides them through a simple, conversational flow.

Every week the bot fires a scheduled reminder, optionally pre-fills a draft from the previous week, lets the user review and edit inline, and submits directly to Oracle HCM REST APIs with instant confirmation.

---

## ✨ Key Features

| Feature | Description |
|---|---|
| **Smart reuse** | Fetches last week's approved entries and auto-drafts this week's timecard |
| **Editable draft** | Review the draft inline in Slack before submitting; edit per-day times in a modal |
| **Manual entry** | Full Oracle-style entry modal with searchable Project, Task, Type, Country, and State/Province fields |
| **Snooze & reminders** | Snooze for 15, 30, 60, 120 mins, or pick a custom date + time — the bot re-pings you automatically |
| **Skip this week** | Acknowledge and receive a direct link to Oracle HCM for one-off manual entry |
| **Direct HCM submission** | Posts to Oracle HCM REST APIs with `processMode=TIME_SAVE` and confirms success or failure in Slack |
| **Double-tap protection** | Idempotency guard prevents duplicate submissions if Approve is clicked twice |
| **HCM retries** | 5xx errors are retried 3× with exponential backoff (1s / 2s / 4s) |

---

## 💬 Conversation Flow

```
Scheduler fires (e.g. Tuesday 12:00 PM)
    │
    ▼
Bot DMs the user:
  "Hi! It's timecard day. What would you like to do?"
  [ Reuse Last Week ]  [ Enter Manually ]  [ Remind Me Later ]  [ Skip This Week ]
    │
    ├─ Reuse Last Week
    │     └─ Fetches previous week entries → shifts dates +7 days → shows draft
    │           [ Approve ]  [ Edit ]  [ Cancel ]
    │               │
    │               ├─ Approve → submits to Oracle HCM → confirmation in Slack ✅
    │               └─ Edit   → opens per-day time-picker modal → rebuild draft
    │
    ├─ Enter Manually
    │     └─ Opens blank 5-day modal (Project / Task / Type / Location)
    │           → review preview → Submit → Oracle HCM → confirmation ✅
    │
    ├─ Remind Me Later
    │     └─ Snooze modal (15 / 30 / 60 / 120 min or custom date+time)
    │           → bot reschedules and resends the reminder
    │
    └─ Skip This Week
          └─ Acknowledges + links to Oracle HCM for manual entry
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────┐
│  APScheduler (weekly cron)                               │
│    └─ send_weekly_reminders() → Slack DM per user        │
└──────────────────┬───────────────────────────────────────┘
                   │ Socket Mode WebSocket
┌──────────────────▼───────────────────────────────────────┐
│  Slack Bolt App (app.py)                                 │
│    handlers.py ── blocks.py ── draft_engine.py           │
│         │               │              │                 │
│    user_registry   status_tracker   hcm_client           │
│    (JSON file)     (JSON file)      (REST)               │
└──────────────────────────────────────────────────────────┘
```

**Module breakdown**

| File | Responsibility |
|---|---|
| `app.py` | Entry point, APScheduler setup, Socket Mode handler |
| `config.py` | Typed env-var constants via python-dotenv |
| `hcm_client.py` | All 6 Oracle HCM REST endpoints with retry / backoff |
| `draft_engine.py` | Replicate, edit, and serialise weekly drafts |
| `blocks.py` | Slack Block Kit message builders |
| `handlers.py` | All Slack action and view-submission handlers |
| `user_registry.py` | Slack ID ↔ HCM profile mapping (JSON) |
| `status_tracker.py` | Submission log + duplicate-tap detection |
| `models.py` | Pydantic types for all data shapes |
| `utils.py` | Date helpers and structured logging |

---

## 🔌 Oracle HCM API Integration

The bot integrates with **6 Oracle HCM REST endpoints** under API version `11.13.18.05`.

| # | API | Endpoint | Purpose |
|---|---|---|---|
| 1 | Payroll Time Types | `timeAttributeValues?finder=filterByDataSourceUsage` | Populate dropdown options for the user's assignment |
| 2 | Time Attributes | `timeAttributes?finder=filterByAttrContext` | Retrieve attribute metadata (absence types, project codes) |
| 3 | Time Cards | `timeRecordGroups?finder=filterByPerNumTimeGrp` | Check previous week's hours and submission status |
| 4 | Time Entries | `timeRecords?finder=filterByPerNumTimeGrp` | Fetch individual daily entries for replication |
| 5 | Update Time Entries | `timeRecordEventRequests/` | Create / update records with `processMode=TIME_SAVE` |
| 6 | Update Status | `statusChangeRequests` | Mark submission as `SUCCESS` or `FAILURE` for payroll |

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- A Slack workspace with a Bot App configured for **Socket Mode**
  - Required scopes: `chat:write`, `im:write`, `users:read`, `users:read.email`
  - App-Level Token with `connections:write`
- Oracle HCM Cloud access with a valid OAuth Bearer token

### Local Setup

```bash
# 1. Clone the repo
git clone <repo-url>
cd Timely_Digital_Twin/src/services

# 2. Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env
# Edit .env with your Slack tokens and Oracle HCM credentials

# 5. Register a user
python - <<'EOF'
from src.services.user_registry import UserRegistry
from src.services.models import UserProfile
r = UserRegistry()
r.add(UserProfile(
    slack_id="YOUR_SLACK_USER_ID",
    person_number="YOUR_PERSON_NUMBER",
    person_id="YOUR_PERSON_ID",
    assignment_id="YOUR_ASSIGNMENT_ID",
    assignment_number="YOUR_ASSIGNMENT_NUMBER",
    hcm_token="Bearer YOUR_HCM_TOKEN",
    name="Your Name",
    timezone="Asia/Kolkata",
))
EOF

# 6. Run the bot
python -m src.services.app
```

### Docker

```bash
cd Timely_Digital_Twin/src/services
cp .env.example .env          # fill in real values
docker compose up --build
```

The container mounts a named volume (`registry-data`) to persist `user_registry.json` and `submission_log.json` across restarts.

---

## ⚙️ Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `SLACK_BOT_TOKEN` | Yes | — | `xoxb-…` bot token |
| `SLACK_APP_TOKEN` | Yes | — | `xapp-…` Socket Mode app-level token |
| `HCM_BASE_URL` | Yes | — | Oracle HCM tenant base URL |
| `HCM_AUTH_TOKEN` | Yes | — | `Bearer <token>` for HCM API calls |
| `HCM_DATA_SOURCE_USAGE_ID` | Yes | — | Payroll time type data source usage ID |
| `SCHEDULE_DAY` | No | `tue` | Day of week for the weekly reminder |
| `SCHEDULE_HOUR` | No | `12` | Hour (24h) for the reminder |
| `SCHEDULE_MINUTE` | No | `0` | Minute for the reminder |
| `SCHEDULE_TIMEZONE` | No | `Asia/Kolkata` | IANA timezone for the scheduler |
| `SEND_REMINDER_ON_STARTUP` | No | `true` | Fire the reminder immediately on each app start |
| `LOG_LEVEL` | No | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `DRAFT_EXPIRY_MINUTES` | No | `60` | Minutes before a pending draft expires |

---

## 🧪 Running Tests

```bash
# From repo root
pytest src/services/tests/ -v

# With coverage
pip install pytest-cov
pytest src/services/tests/ --cov=src/services --cov-report=term-missing
```

---

## 🗺️ Roadmap / Next Steps

- [ ] **Persistent snooze storage** — move in-memory reminders to Redis or a DB so they survive process restarts
- [ ] **OAuth per-user HCM tokens** — replace the shared bearer token in `.env` with individual OAuth 2.0 tokens, one per employee
- [ ] **Multi-assignment support** — let users with more than one active Oracle assignment pick which one to submit against
- [ ] **Approval workflow** — add a manager-facing Slack flow to approve submitted timecards before they reach payroll
- [ ] **Analytics dashboard** — track submission rates, average submission time, and snooze frequency across the team
- [ ] **Slack Home tab** — surface submission history and upcoming deadlines in a persistent App Home view
- [ ] **Enterprise deployment** — replace JSON user registry with a proper database (Postgres / Firestore) and add secret manager integration (AWS Secrets Manager / GCP Secret Manager)
- [ ] **Unit + integration test coverage** — expand pytest suite with mocked HCM API responses and Slack payload fixtures

---

## 📚 What This Project Demonstrates

- **Chat-native UX design** — a conversational flow that is linear and stateful, where context carries forward between turns without requiring a web form
- **End-to-end Oracle HCM integration** — auth, payload shaping, error states, and confirmation handling across all six relevant REST endpoints
- **Reliability patterns** — draft expiry, idempotency guards, and HCM retries make a small productivity tool feel production-ready
- **Adoption by embedding** — a tool that lives inside the user's daily workflow (Slack) drives far higher completion rates than a standalone app for the same task

---

## 🛠️ Tech Stack

- **Python 3.10+**
- **Slack Bolt for Python** — event handling and Socket Mode
- **APScheduler** — cron-based weekly reminders
- **Oracle HCM Cloud REST APIs** v11.13.18.05
- **Pydantic** — data validation and typed models
- **python-dotenv** — environment configuration
- **Docker + Docker Compose** — containerised deployment

---

## 📄 License

MIT
