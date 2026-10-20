# Bookstore

A Django project, focused on backend development: authentication, a book catalog, permissions, search, and a Stripe payment demonstration. The application runs with Docker and PostgreSQL.

## Features

- Custom user model and Django admin integration.
- Email-based signup and login using django-allauth.
- Book listings, optional cover images, and reviews managed through the admin.
- UUID-based book URLs.
- Login-required catalog and book details restricted by `books.special_status`.
- Case-insensitive title search using Django `Q` objects.
- Stripe test checkout demonstrating a $39 USD purchase for access to all book details.
- Bootstrap 4 forms with django-crispy-forms, WhiteNoise static files, and Django Debug Toolbar.

## Technology stack

The repository specifies Python 3.7, Django 2.2.7, and PostgreSQL 11. Python dependencies are declared in `Pipfile`, with a lockfile in `Pipfile.lock`. Docker Compose runs Gunicorn and a separate database service.

## Local setup

### Prerequisites

- Docker with Docker Compose (`docker compose`).
- A browser.
- Your own Stripe test API keys to try checkout.

### Configure the services

Before starting, update `docker-compose.yml`:

1. Replace the web service's checked-in `SECRET_KEY` with your own random value.
2. Replace `STRIPE_TEST_PUBLISHABLE_KEY` and `STRIPE_TEST_SECRET_KEY` with your own test keys, without angle brackets, to enable checkout.
3. Add this environment setting to the `db` service so a new PostgreSQL database initializes with the password expected by Django:

   ```yaml
   db:
     image: postgres:11
     environment:
       POSTGRES_PASSWORD: postgres
     volumes:
       - postgres_data:/var/lib/postgresql/data/
   ```

Django defaults to database `postgres`, user `postgres`, password `postgres`, and host `db` on port `5432`. The `db.sqlite3` file is not used by the configured application. For an existing database volume, the initialization setting does not change its password.

### Build and start

Run from the project directory:

```bash
docker compose build
docker compose up -d db
docker compose logs db
```

Wait until PostgreSQL reports that it is ready to accept connections, then run:

```bash
docker compose run --rm web python manage.py migrate
docker compose run --rm web python manage.py createsuperuser
docker compose run --rm web python manage.py collectstatic --noinput
docker compose up -d web
```

Open [localhost:8000](http://localhost:8000). Log in to the [admin](http://localhost:8000/anything-but-admin/) with your superuser account and add books and reviews; no catalog seed data is provided.

For a regular account to view book details, grant the **Can read all books** permission through the admin. Authentication emails are printed to the console rather than sent through an email provider.

Useful commands:

```bash
# Follow application logs
docker compose logs -f web

# Run Django's development server instead of Gunicorn
# Stop the existing web service first to free port 8000
docker compose stop web
docker compose run --rm --service-ports web python manage.py runserver 0.0.0.0:8000

# Stop services while preserving the database volume
docker compose down
```

## Routes

| Path | Purpose |
| --- | --- |
| `/` | Home page |
| `/about/` | About page |
| `/accounts/signup/` | Sign up |
| `/accounts/login/` | Log in |
| `/books/` | Book catalog; login required |
| `/books/<uuid>` | Book details and reviews; login and permission required |
| `/books/search/?q=title` | Title search |
| `/orders/` | Stripe test checkout |
| `/orders/charge/` | Checkout form submission |
| `/anything-but-admin/` | Django admin |

## Configuration

| Variable | Behavior |
| --- | --- |
| `SECRET_KEY` | Required Django secret key; settings supply no default. |
| `DEBUG` | Integer flag, `0` or `1`; defaults to `0`. Local Compose sets `1`. |
| `ENVIRONMENT` | Defaults to `development`; `production` enables HTTPS redirects and additional security settings. |
| `DATABASE_URL` | Overrides corresponding default database connection settings through dj-database-url when supplied. |
| `STRIPE_TEST_PUBLISHABLE_KEY` | Public key used by the checkout template. |
| `STRIPE_TEST_SECRET_KEY` | Secret key used by the charge view. |
| `STRIPE_LIVE_PUBLISHABLE_KEY`, `STRIPE_LIVE_SECRET_KEY` | Read by settings, but not used by the current checkout views. |

## Tests

With the database service running and local configuration in place:

```bash
docker compose run --rm web python manage.py check
docker compose run --rm web python manage.py test
```

Tests cover the home and about pages, custom users, signup, book models, and catalog access permissions. The orders app currently has no payment tests.

## Project structure

```text
bookstore_project/  Django settings, root URLs, and WSGI entry point
books/              Book and review models, catalog views, search, and tests
users/              Custom user model, forms, admin, and tests
pages/              Home and about pages
orders/             Stripe checkout and charge views
templates/          Shared and app-specific HTML templates
static/             Source CSS, JavaScript, and images
staticfiles/        Collected static assets
media/              Uploaded book covers
```

## Known limitations and deployment

This repository contains a historical learning implementation and requires updates before production use:

- The runtime and dependencies are pinned to older versions. The Dockerfile installs an unpinned Pipenv, so rebuilding may require a Pipenv version compatible with Python 3.7.
- The charge view grants book-access permission before confirming a successful payment and lacks an explicit login guard. Use it only as a test demonstration until this flow is corrected.
- Checkout always uses test keys, including when `ENVIRONMENT=production`. The production Compose file supplies only live-key variables, so it does not configure the current checkout correctly.
- Search matches titles only; its two `Q` conditions are identical.
- Compose files contain hard-coded secret values. Replace them and move deployment secrets into environment-managed configuration.

`docker-compose-prod.yml` and `heroku.yml` preserve the original deployment configuration. The previously documented Heroku hostname is `guarded-meadow-23348.herokuapp.com`; its availability has not been verified. A new deployment needs its own host configuration, database credentials, static/media handling, and migrations. The Heroku release command currently collects static files but does not run migrations.

