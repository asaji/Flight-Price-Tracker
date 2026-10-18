# Agent Context & Instructions (Flight Price Tracker)

This file provides context for AI coding assistants (Antigravity, OpenAI Codex, Claude Code, Cursor, etc.).

## 1. Project Overview
- **Purpose:** Self-hosted flight price tracking web application that runs perpetually as a Docker container (port `7002`, e.g., on Unraid). Periodically checks Google Flights prices via the `fli`/`flights` library, stores price history in SQLite, plots price trends over time, organizes routes into named trips with budget tracking, and sends Pushover push notifications on price changes or budget alerts.
- **Stack:**
  - **Backend:** Python 3.12, Flask, APScheduler
  - **Flight data:** `flights` pip package (`punitarani/fli` — wraps Google Flights, no API key needed)
  - **Database:** SQLite at `./data/flights.db` (host) / `/config/flights.db` (container)
  - **Frontend:** Bootstrap 5 dark theme, Chart.js, vanilla JS — single-page, no build step
  - **Notifications:** Pushover API
  - **Deployment:** Docker Compose, port `7002`, `./data/` volume for persistence

## 2. Key Files
- `main.py` — Entry point; starts Flask on `0.0.0.0:7002`.
- `app/__init__.py` — App factory; initializes DB, registers blueprint, starts background scheduler.
- `app/db.py` — All SQLite schema definitions, auto-migrations, and queries (`trips`, `routes`, `price_history`, `settings`).
- `app/search.py` — `fli` library wrapper; always searches one-way legs.
- `app/scheduler.py` — APScheduler background poller + Pushover price-change & trip budget alert logic.
- `app/pushover.py` — Pushover notification helper.
- `app/api.py` — Flask blueprint; all REST endpoints + main UI route.
- `app/templates/index.html` — Full single-page web UI (Bootstrap 5 dark theme).
- `app/static/js/app.js` — Frontend logic: route cards, charts, named trips filter bar, best-combo banner, and Trip Builder.
- `app/static/css/style.css` — Dark theme styles.
- `Dockerfile` & `docker-compose.yml` — Container definition (`python:3.12-slim`, port `7002`), volume mounts, and environment variables.
- `run_local.sh` — Local macOS development launcher script.
- `README.md` & `UNRAID.md` — General documentation and step-by-step Unraid setup guide.
- `CHANGELOG.md` — Dated log of all changes, architectural decisions, and bug fixes.

## 3. Database Schema & Core Mechanics
```text
trips           id, name, notes, budget, color, created_at, last_combo

routes          id, trip_id, named_trip_id (FK -> trips.id), leg_label,
                origin, destination, departure_date, non_stop_only,
                airlines (JSON), adults, seat_type, active, created_at,
                day_offset (-1/0/+1), notify (0/1)

price_history   id, route_id (FK -> routes.id), price, currency,
                flight_details (JSON), checked_at

settings        key, value (poll_interval_minutes)
```

### How Routes & Named Trips Work
- **Date Variants (`-1 / 0 / +1`):** Every time a user adds a route leg, the backend creates **3 route records** sharing a `trip_id` (`day_offset = -1`, `0`, `+1`). Round trips create 6 records (3 outbound + 3 return) under the same `trip_id`.
- **One-Way Polling:** The scheduler always searches each record as a **one-way leg** regardless of trip type.
- **Notifications:** Per-route notifications default to off (`notify = 0`) and can be toggled via the bell icon on each leg group. Price-change alerts only fire for `day_offset = 0` routes with `notify = 1`.
- **Named Trips:** Routes can be assigned to named trips (`trips` table) with custom color tags and optional budgets. The app calculates the best combination price across assigned legs and sends a Pushover alert when the best combo drops under the trip's budget.

## 4. API Endpoints
```text
GET    /                              Web UI
GET    /api/routes                    List routes with latest non-null price, stats, trend (?include_archived=1)
POST   /api/routes                    Add route(s) — creates 3 date variants (-1/0/+1 day) per leg
DELETE /api/routes/<id>               Delete single route
DELETE /api/routes/batch              Delete multiple routes {ids: [...]}
PATCH  /api/routes/notify             Toggle per-route notifications {ids: [...], notify: bool}
PATCH  /api/routes/assign-trip        Assign/unassign routes to a named trip {ids: [...], named_trip_id: int|null}
POST   /api/routes/archive-past       Set active=0 for all past-date routes
GET    /api/routes/archived-count     Count of archived routes
GET    /api/routes/<id>/history       Full price history for one route
GET    /api/named-trips               List named trips with best combo & budget remaining
POST   /api/named-trips               Create named trip {name, notes, budget, color}
PUT    /api/named-trips/<id>          Update named trip
DELETE /api/named-trips/<id>          Delete named trip
GET    /api/named-trips/<id>/combo    Get best combo for a named trip
POST   /api/poll                      Trigger immediate price check (background thread)
POST   /api/notify/test               Send Pushover test notification
GET    /api/settings                  Get poll_interval_minutes
POST   /api/settings                  Update poll_interval_minutes
```

## 5. Common Workflows
- **Local Development:**
  ```bash
  ./run_local.sh
  # or:
  pip install -r requirements.txt
  python main.py          # runs on http://localhost:7002, writes to ./data/flights.db
  ```
- **Docker Compose:**
  ```bash
  docker compose up -d --build    # build and start
  docker logs fp-tracker          # view logs
  docker compose down             # stop
  ```

## 6. Housekeeping Rule for All AI Agents
**Always read `CHANGELOG.md` before making changes, and always append a dated entry (`## YYYY-MM-DD — [Agent / Model Name] — Summary`) to `CHANGELOG.md` specifying which agent and model you are, what changes you made, and why.**
