# Job Application Tracker — Backend

Django + Django REST Framework backend for tracking job applications.

## Stack

- Python 3.12 / Django 6.1 / Django REST Framework
- SQLite (default; no external database needed)
- [uv](https://docs.astral.sh/uv/) for dependency management
- Docker Compose for containerized runs

## Project layout

```
.
├── .env                  # Secrets (git-ignored)
├── .env.example          # Template for .env
├── docker-compose.yml    # Backend service definition
└── backend/
    ├── manage.py
    ├── pyproject.toml    # Dependencies (uv)
    ├── Dockerfile
    └── config/           # Django project package
        ├── settings.py   # All settings; reads env vars
        ├── urls.py
        ├── wsgi.py
        └── asgi.py
```

## Configuration

Settings are read from environment variables in `backend/config/settings.py`,
with safe dev defaults:

| Variable              | Purpose                        | Default                              |
| --------------------- | ------------------------------ | ------------------------------------ |
| `DJANGO_SECRET_KEY`   | Django signing key             | insecure fallback (dev only)         |
| `DJANGO_DEBUG`        | Debug mode on/off              | `true`                               |
| `DJANGO_ALLOWED_HOSTS`| Comma-separated host list      | `localhost,127.0.0.1`                |

`.env` is git-ignored (`.gitignore` lines for `.env` / `.env.*`) and excluded
from Docker image builds via `backend/.dockerignore`. Create it from the
example file at the repo root (next to `docker-compose.yml`):

```bash
cp .env.example .env
```

Generate a real key for production:

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

## Running the backend

### Option A: Docker (recommended)

```bash
docker compose up --build
```

- App: http://localhost:8000
- Migrations run automatically on container start (see `Dockerfile` `CMD`).
- Source is bind-mounted, so edits reload via the dev server.

Stop with `Ctrl+C` or `docker compose down`.

### Option B: Local, without Docker

Requires [uv](https://docs.astral.sh/uv/) (or plain Python 3.11+).

```bash
cd backend
uv sync                      # create .venv and install deps from uv.lock
uv run python manage.py migrate
uv run python manage.py runserver
```

App: http://localhost:8000

## Common commands

### Docker (container already running)

Migrations run automatically on `docker compose up`, but after adding models:

```bash
docker compose exec backend python manage.py makemigrations   # generate migration
docker compose exec backend python manage.py migrate         # apply migrations
```

Open a shell inside the container (you land in `/app` with the venv on PATH;
exit with `exit` or Ctrl+D):

```bash
docker compose exec -it backend bash
```

Then any `python manage.py ...` command works directly, e.g. `createsuperuser`,
`shell`, `test`, `check`.

Follow server logs (`Ctrl+C` stops following, not the container):

```bash
docker compose logs -f backend
```

### Local (no Docker)

<!-- Run from `backend/` (prefix with `uv run` if not in an activated venv): -->

```bash
docker compose exec backend python manage.py check            # sanity-check the project
docker compose exec backend python manage.py migrate          # apply database migrations
docker compose exec backend python manage.py createsuperuser  # create an admin login
docker compose exec backend python manage.py shell            # interactive Django shell
docker compose exec backend python manage.py test             # run tests
```

Admin site: http://localhost:8000/admin/

## API

Only the Django admin is wired up so far (`backend/config/urls.py`). New
endpoints go through Django REST Framework — add an app with
`uv run django-admin startapp <name>` and register its URLs in
`config/urls.py`.

## Production checklist

- Set a real `DJANGO_SECRET_KEY` in the environment (not the `.env` dev key).
- `DJANGO_DEBUG=false`
- Set `DJANGO_ALLOWED_HOSTS` to your real domain(s).
- Serve with a production WSGI/ASGI server (gunicorn/uvicorn), not
  `runserver`.
- Put TLS and static-file serving behind a reverse proxy.
