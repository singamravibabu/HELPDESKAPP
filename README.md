# HELPDESKAPP

Nice — Lesson 3 coming right up. Below I give a concrete, copy-paste-ready tech-stack spec, exact version pins, and the local / dev / prod parity setup you should use. I include ready-to-use files you can drop into your repo: Dockerfile, docker-compose.yml (dev), requirements.txt, and .env.example, plus clear guidance and snippets for settings/base.py, settings/dev.py, and settings/prod.py. Follow these and your local environment will closely mirror staging/production while keeping safe differences where necessary.

Lesson 3 — Tech stack, versions, and local/dev/prod parity

1) Chosen stack & pinned versions (recommended)

Language: Python 3.11 (use 3.11.x).

Django: 5.2.* (LTS target).

Django REST Framework: >=3.15,<4.0.

Database: PostgreSQL 15 (server) / psycopg2-binary client.

Cache / Broker: Redis 7.

Background jobs: Celery 5.2.*.

Web server (prod): Gunicorn 20.1.*.

Static serving (prod): Whitenoise 6.* (if using Gunicorn + S3 optional).

Task scheduler: Celery Beat (comes with Celery).

Frontend basics: Vanilla HTML/CSS/JS; FullCalendar 6.x for calendar UI.

Containerization: Docker (Engine) + docker-compose v2 format.

Lint / Formatting: black, isort, ruff, flake8 (dev tools).

CI/CD: GitHub Actions (ubuntu-latest runners).


(These choices balance modern Python features, long-term Django support, and ecosystem stability.)


---

2) Repo layout (important context)

helpdesk/
├── .github/
│   └── workflows/
├── docker/
│   └── nginx/           # optional nginx conf for dev/prod
├── config/              # django project root (django-admin startproject helpdesk_project)
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   └── wsgi.py
├── apps/
│   ├── accounts/
│   ├── tickets/
│   ├── calendar/
│   └── ...              # other apps
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md


---

3) requirements.txt (example pins)

Create requirements.txt at project root:

# Core
Django==5.2
djangorestframework>=3.15,<4.0
psycopg2-binary==2.9.7
gunicorn==20.1.0
whitenoise==6.4.0

# Celery + broker
celery==5.2.10
redis==4.5.5

# Auth / Social (optional)
django-allauth==0.58.0
dj-rest-auth==3.2.3

# Utilities
python-dotenv==1.0.0
django-environ==0.10.0
django-filter==23.1
drf-spectacular==1.1.0

# Dev / QA
pytest==7.4.0
pytest-django==4.7.0
black==24.4.0
isort==5.12.0
ruff==0.28.0

> You can pip-compile or pin further as you stabilize; CI should run tests across pinned set.




---

4) .env.example (copy into .env locally and set secrets)

Create .env.example (DO NOT commit .env containing real secrets):

# .env.example
# General
DJANGO_SECRET_KEY=replace-me-with-a-secure-random-string
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DJANGO_DEBUG=True

# Database - use DATABASE_URL in 12factor style
DATABASE_URL=postgres://helpdesk:helpdesk_password@db:5432/helpdesk_db

# Redis / Celery
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2

# Email (dev: console backend)
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
DEFAULT_FROM_EMAIL=helpdesk@example.com

# Static & Media
MEDIA_URL=/media/
MEDIA_ROOT=/vol/web/media
STATIC_URL=/static/
STATIC_ROOT=/vol/web/static

# Production overrides (example)
SENTRY_DSN=


---

5) Dockerfile (production-oriented, but works for dev builds)

Create Dockerfile at project root:

# Dockerfile
FROM python:3.11-slim

# create non-root user
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN apt-get update && apt-get install -y \
    build-essential \
    libpq-dev \
    curl \
 && rm -rf /var/lib/apt/lists/*

# create app directory
WORKDIR /app

# copy requirements separately for caching
COPY requirements.txt /app/requirements.txt
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r /app/requirements.txt

# Copy code
COPY . /app

# create a non-root user and chown files
RUN useradd -m appuser && chown -R appuser /app
USER appuser

# collectstatic will be run during entrypoint/CI/CD
ENV PORT 8000
EXPOSE 8000

# default command for local development; in prod we run via gunicorn in kubernetes/systemd
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "3"]

Notes:

In production you might build static assets in CI then run collectstatic during image build or entrypoint.

Use a non-root user for security.



---

6) docker-compose.yml for local development

Create a docker-compose.yml at project root to spin up DB, redis, web, worker, and beat:

version: "3.9"
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: helpdesk_db
      POSTGRES_USER: helpdesk
      POSTGRES_PASSWORD: helpdesk_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U helpdesk"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  web:
    build:
      context: .
      dockerfile: Dockerfile
    command: >
      sh -c "python manage.py migrate --noinput &&
             python manage.py collectstatic --noinput &&
             gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 2"
    volumes:
      - .:/app:rw
    env_file:
      - ./.env
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  worker:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A config.celery_app worker --loglevel=info
    env_file:
      - ./.env
    volumes:
      - .:/app:rw
    depends_on:
      - db
      - redis

  beat:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A config.celery_app beat --loglevel=info
    env_file:
      - ./.env
    volumes:
      - .:/app:rw
    depends_on:
      - db
      - redis

volumes:
  postgres_data:
  redis_data:

Notes:

volumes allow edit/reload behaviour locally. In CI/prod you won’t mount source code like this.

env_file points to .env (copy from .env.example and set secrets).



---

7) Django settings architecture & example snippets

Goal: keep parity but separate dev/prod concerns. Use settings/base.py for common settings; dev.py and prod.py import base and override.

config/settings/base.py (important snippets)

# base.py
import os
from pathlib import Path
import environ

ROOT_DIR = Path(__file__).resolve().parent.parent.parent
env = environ.Env()
environ.Env.read_env(ROOT_DIR / ".env")

SECRET_KEY = env("DJANGO_SECRET_KEY")
DEBUG = env.bool("DJANGO_DEBUG", default=False)
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=["localhost", "127.0.0.1"])

INSTALLED_APPS = [
    # django core...
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    # third party
    "rest_framework",
    # apps
    "apps.accounts",
    "apps.tickets",
    "apps.calendar",
    "apps.notifications",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    # whitenoise middleware can be placed here (for static files)
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
]

ROOT_URLCONF = "config.urls"
WSGI_APPLICATION = "config.wsgi.application"

# Database: use DATABASE_URL for 12-factor style
DATABASES = {
    "default": env.db("DATABASE_URL", default="postgres://helpdesk:helpdesk_password@db:5432/helpdesk_db")
}

# Caches (Redis)
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL", default="redis://redis:6379/0"),
    }
}

# Celery settings (imported into celery config)
CELERY_BROKER_URL = env("CELERY_BROKER_URL", default="redis://redis:6379/1")
CELERY_RESULT_BACKEND = env("CELERY_RESULT_BACKEND", default="redis://redis:6379/2")

STATIC_URL = "/static/"
STATIC_ROOT = str(ROOT_DIR / "staticfiles")  # for collectstatic
MEDIA_URL = "/media/"
MEDIA_ROOT = str(ROOT_DIR / "media")

# Static files served by whitenoise in simple deployments
STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"

# REST Framework shared settings
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticatedOrReadOnly",
    ],
}

# Timezone
TIME_ZONE = "UTC"
USE_TZ = True

config/settings/dev.py

from .base import *

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]

# Dev email backend
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"

# Use local filebased cache if needed
CACHES["default"]["LOCATION"] = "redis://redis:6379/0"

config/settings/prod.py

from .base import *
import dj_database_url
from pathlib import Path

DEBUG = False

# Enforce host(s) via env
ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=[])

# Security hardening
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Email - production provider example (use env vars)
EMAIL_BACKEND = "django.core.mail.backends.smtp.EmailBackend"
EMAIL_HOST = env("EMAIL_HOST", default="")
EMAIL_PORT = env.int("EMAIL_PORT", default=587)
EMAIL_HOST_USER = env("EMAIL_HOST_USER", default="")
EMAIL_HOST_PASSWORD = env("EMAIL_HOST_PASSWORD", default="")
EMAIL_USE_TLS = True

# Database: expect DATABASE_URL from env (managed DB)
DATABASES["default"] = dj_database_url.parse(env("DATABASE_URL"), conn_max_age=600)

# Staticfiles served by CDN, but whitenoise can serve as fallback
STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"

How to choose which to load at runtime: set DJANGO_SETTINGS_MODULE=config.settings.dev for local dev and config.settings.prod for production.


---

8) Celery quick config snippet (config/celery_app.py)

# config/celery_app.py
import os
from celery import Celery
from django.conf import settings

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")  # change in prod

app = Celery("config")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()

When deploying to prod, make sure DJANGO_SETTINGS_MODULE is config.settings.prod.


---

9) Local / Dev / Prod parity rules (principles + specific differences)

Parity principle: keep local dev behavior as similar to prod as practical (same DB server type, same broker, same static handling), but allow safety & developer ergonomics differences.

What to keep identical (parity):

Use PostgreSQL in dev (docker-compose) — same major version as prod (Postgres 15).

Use Redis in dev for cache and Celery broker.

Use the same DATABASE_URL / REDIS_URL env pattern so code reads same env vars.

Same Django INSTALLED_APPS and middleware ordering where possible.

Same REST API behavior and error formats.

Use same package versions (pin in requirements.txt) across environments.


What to differ (safety/dev ergonomics):

DEBUG=True in dev (but use dev tools to inspect errors); DEBUG=False in prod.

Email backend: dev uses console backend; prod uses SMTP or SendGrid.

Static file handling: dev may use runserver + staticfiles; prod uses collectstatic + whitenoise or CDN (S3 + CloudFront).

Secrets / keys: store sensitive values in environment variables or secret manager; never commit .env.

Allowed hosts: loosen in dev (localhost), locked in prod.

Logging level: DEBUG in dev, INFO/WARN in prod; prod should integrate with Sentry or centralized logging.


Operational parity:

Use the same entrypoint commands for migrations & collectstatic in CI/CD (so migration issues surface early).

Run the same tests in CI as you run locally (pytest).

Use Docker both locally and in CI to ensure runtime parity.



---

10) Commands to bootstrap locally (copy/paste)

1. Create .env from example:



cp .env.example .env
# edit .env with secure DJANGO_SECRET_KEY before running

2. Build and run dev stack:



docker compose up --build
# or
docker-compose up --build

3. Create a superuser (if web container is running and migrations finished):



docker compose exec web python manage.py createsuperuser

4. Run Celery worker/beat (already in docker-compose as separate services) - logs viewable via docker compose logs -f worker etc.




---

11) Notes on CI/CD parity

CI should use the same Dockerfile and requirements.txt to run tests. This avoids “works locally, fails in CI” caused by mismatched packaging.

Use GitHub Actions matrix for lint/test steps but run the integration test suite in a container that spins up Postgres & Redis (services) mirroring docker-compose.

In CI you should set DJANGO_SETTINGS_MODULE=config.settings.test (optional) and point DB to service host.
