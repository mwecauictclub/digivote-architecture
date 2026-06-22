# Deployment Architecture

## Current Production Topology (2026)

- **Primary domain**: https://digivote.xenohuru.com (Cloudflare proxied)
- **Fallback**: http://159.65.119.182:81/
- **VPS**: 159.65.119.182 (DigitalOcean or similar)
- **Application path**: `/var/www/mwecau_digivote`
- **WSGI server**: Gunicorn (3 workers) via unix socket
- **Web server**: nginx
- **Database**: MySQL (`digivote_db`)
- **Cache/Broker**: Redis (local on same VPS)
- **Media**: Cloudinary
- **Email**: Brevo / SMTP / AWS SES (via Celery + Anymail or Django email backends)

## Deployment Pipeline

```mermaid
flowchart LR
    Dev[Developer pushes] --> GH[GitHub]
    GH --> Action[deploy.yml workflow]
    Action -->|SSH| VPS
    VPS -->|git fetch/reset| Code
    Code -->|pip install| Deps
    Deps -->|migrate + collectstatic| Django
    Django --> Restart[gunicorn-digivote restart]
```

The workflow runs on push to the active deploy branch (currently `init-dep` / `feature/general-election-2026` etc.).

## Important Details

- Two `.env` files exist; the one inside `src/` takes precedence.
- `git reset --hard` is used (untracked files like stray migrations can cause issues).
- Static files are pre-built (Tailwind) and committed.
- No Node.js is required on the VPS.

## Future / Planned

- Multiple web application servers behind a load balancer
- Dedicated Celery worker machine(s)
- Shared Redis and MySQL remain the coordination layer
- Possibly switch to uvicorn + ASGI for higher concurrency

See `CICD.md` in the main application repository for the full operational runbook.

This public documentation focuses only on the high-level architecture suitable for sharing with university IT departments.
