# Core Components & Responsibilities

## Django Apps

| App      | Responsibility |
|----------|----------------|
| `core`   | User model, authentication, registration, dashboards (commissioner, observer), college data upload, password recovery, contact forms |
| `election` | Election, levels, positions, candidates, voter tokens, votes, results computation, voting UI, reports |

## Key External Integrations

- **Cloudinary**: Candidate images + election reports (PDFs)
- **Email**: Flexible backend — Brevo (default via django-anymail), standard SMTP, or AWS SES (boto3)
- **Redis**: Cache + Celery broker/results + sessions
- **MySQL**: Primary data store

## Background Jobs (selected)

- `send_verification_email` → issues VoterTokens for current active elections
- `notify_voters_of_active_election`
- `close_ended_elections` (periodic via celery-beat)
- Various reminder and confirmation emails

## Frontend Approach

- Server-rendered HTML with semantic structure
- HTMX for dynamic form interactions and fragments (turnout, modals)
- Tailwind CSS (built locally and committed)
- No heavy frontend framework

This keeps the system simple, maintainable, and easy for club members and MWECAU IT staff to contribute to and understand.
