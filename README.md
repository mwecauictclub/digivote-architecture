# MWECAU DigiVote — Architecture Documentation

MWECAU DigiVote is the official electronic voting platform for Mwenge Catholic University (MWECAU) student elections. This repository provides the **public architecture documentation** for the system.

It is designed as a clean, standalone resource for the MWECAU IT Department, university staff, student contributors, and other institutions interested in digital election systems. The documentation covers high-level design, key processes, technology choices, and visual diagrams — without exposing application source code or operational secrets.

**Key Highlights**
- Mermaid diagrams for system overview, data flows, security model, and deployment
- Documentation on MySQL database, flexible email backends (Brevo / SMTP / AWS SES), caching, and scalability
- Focus on secure, auditable, one-person-one-vote election processes

> All diagrams are written in [Mermaid](https://mermaid.js.org/) and render natively on GitHub.

## Table of Contents

- [System Overview](#system-overview)
- [Core Components](#core-components)
- [Cache Layer (Redis)](#cache-layer-redis)
- [Background Processing (Celery)](#background-processing-celery)
- [Documentation Structure](#documentation-structure)
- [Email Options](#email-options)
- [User Roles & Permissions](#user-roles--permissions)
- [Voter Journey](#voter-journey)
- [Secure Voting Process](#secure-voting-process)
- [Election Lifecycle](#election-lifecycle)
- [Deployment Architecture](#deployment-architecture)
- [Data Model Overview](#data-model-overview)
- [Security & Integrity](#security--integrity)
- [Request Lifecycle](#request-lifecycle)

---

## System Overview

```mermaid
flowchart TB
    subgraph Users["👥 Users"]
        Voter["Voters &\nCandidates"]
        ClassLeaders[Class Leaders]
        Commissioners[Commissioners]
        Observers[Observers]
        Public[Public]
    end

    subgraph Edge["🌐 Edge / CDN"]
        Cloudflare["Cloudflare\nProxy + TLS"]
    end

    subgraph App["🗳️ DigiVote Application"]
        direction TB
        Nginx["nginx\nStatic + Reverse Proxy"]
        Gunicorn["Gunicorn\n(or uvicorn)"]
        Django["Django 5.2\n+ DRF + HTMX"]
    end

    subgraph Services["⚙️ Services"]
        direction TB
        MySQL[(MySQL)]
        Redis["(Redis\nCache + Broker)"]
        Celery["Celery Workers\n+ Beat"]
        Cloudinary["Cloudinary\n(Media)"]
        EmailSvc["Email\n(Brevo / SMTP / AWS)"]
    end

    Public --> Cloudflare
    Voter --> Cloudflare
    ClassLeaders --> Cloudflare
    Commissioners --> Cloudflare
    Observers --> Cloudflare

    Cloudflare --> Nginx
    Nginx --> Gunicorn
    Gunicorn --> Django

    Django --> MySQL
    Django --> Redis
    Django --> Celery
    Celery --> Redis
    Celery --> MySQL
    Celery --> EmailSvc
    Django --> Cloudinary

    classDef user fill:#dbeafe,stroke:#1e40af
    classDef edge fill:#e0e7ff,stroke:#3730a3
    classDef app fill:#fef3c7,stroke:#854d0e
    classDef data fill:#dcfce7,stroke:#166534
    class Voter,ClassLeaders,Commissioners,Observers,Public user
    class Cloudflare edge
    class Nginx,Gunicorn,Django app
    class MySQL,Redis,Celery,Cloudinary,EmailSvc data
```

---

## Core Components

| Layer              | Technology                          | Purpose |
|--------------------|-------------------------------------|-------|
| Frontend           | Django Templates + HTMX + Tailwind  | Responsive UI, partial updates |
| Backend            | Django 5.2 + Django REST Framework  | Core logic, auth, APIs |
| Authentication     | Custom RegistrationNumberBackend    | Login by university reg. number |
| Sessions           | Cached sessions (Redis)             | Fast, scalable session storage |
| Database           | MySQL (primary)                     | All persistent data |
| Cache & Broker     | Redis (django-redis + Celery)       | Caching + async messaging |
| Task Queue         | Celery + django-celery-beat         | Email, notifications, scheduled jobs |
| Media Storage      | Cloudinary                          | Candidate photos, PDF reports |
| Email              | Brevo / SMTP / AWS SES + Celery     | Transactional emails |
| Web Server         | nginx + Gunicorn                    | Production serving |
| Proxy / TLS        | Cloudflare (in front of nginx)      | DDoS protection, TLS termination |
| Deployment         | GitHub Actions + SSH to VPS         | Automated deploys |

---

## Documentation Structure

```
.
├── README.md                 # High-level overview + key diagrams
├── docs/
│   ├── authentication.md
│   ├── contributing.md
│   ├── database-mysql.md
│   ├── email-configuration.md
│   ├── email-options.md
│   ├── monitoring-and-logging.md
│   ├── scalability.md
│   ├── security-model.md
│   └── why-mysql.md
├── diagrams/                 # Standalone Mermaid diagrams
├── CACHE-LAYER.md
├── COMPONENTS.md
├── DEPLOYMENT.md
├── FLOWS.md
└── SECURITY.md
```

All documentation is written to be suitable for public sharing with the MWECAU IT department and other institutions. No secrets or internal operational details are included.

---

## Cache Layer (Redis)

Redis is central to performance and correctness.

### Cache Usage Map

```mermaid
flowchart LR
    subgraph DjangoApp["Django Application"]
        Sessions[Session Engine]
        LoginRate["Login Brute-force\nProtection"]
        PwdReset["Password Reset\nOTP + Questions"]
        Turnout["Live Turnout\nFragment"]
        Results["Election Results\nComputation"]
    end

    subgraph Redis["Redis (Shared)"]
        direction TB
        DB0["DB 0\nCelery Broker + Results"]
        DB1["DB 1\nDjango Cache + Sessions"]
        CacheKeys["digivote:* keys\nwith prefix"]
    end

    Sessions --> DB1
    LoginRate --> DB1
    PwdReset --> DB1
    Turnout --> DB1
    Results --> DB1

    CeleryWorkers --> DB0

    style Redis fill:#fee2e2,stroke:#991b1b
```

### Caching Strategies

| Component                | Key Pattern                     | TTL (typical)      | Invalidation |
|--------------------------|---------------------------------|--------------------|--------------|
| Login attempts / lockout | `login_attempts:REG`, `login_lockout:REG` | 15 min            | On success or expiry |
| Password reset OTP       | `pwd_reset_otp:{user_id}`       | 10 min             | On use / expiry |
| Password reset questions | `pwd_reset_q_set:REG`           | 10 min             | On success |
| Election turnout         | `election_turnout:{id}`         | 10 seconds         | Time-based |
| Election results         | `election_results:{id}`         | 30s (active) / 1h (ended) | On vote submit + election end |
| General Django cache     | `digivote:*`                    | 5 minutes default  | Manual or timeout |

**Important design decisions:**
- Results cache uses short TTL during active voting to reduce load while still protecting final tallies behind the `results_published` flag.
- Brute-force protection and password reset state live **only in cache** (ephemeral by design).
- Sessions are stored in Redis for easy horizontal scaling.

---

## Background Processing (Celery)

```mermaid
flowchart TB
    subgraph Producers["Task Producers"]
        Views[Web Views]
        Admin[Admin Actions]
        Beat["Celery Beat\n(scheduled)"]
    end

    subgraph Queues["Redis Queues"]
        EmailQ[email_queue]
        NotifQ[notification_queue]
    end

    subgraph Workers["Celery Workers"]
        EmailWorker["Email Worker\n(lower concurrency)"]
        NotifWorker[Notification Worker]
    end

    subgraph Effects["Side Effects"]
        EmailSvc[Email (Brevo/SMTP/AWS)]
        DB[(MySQL)]
        Cache[(Redis Cache)]
    end

    Views --> EmailQ
    Views --> NotifQ
    Admin --> NotifQ
    Beat --> NotifQ

    EmailQ --> EmailWorker --> EmailSvc
    NotifQ --> NotifWorker --> DB
    NotifQ --> NotifWorker --> Cache

    NotifWorker -->|token issuance| DB
    NotifWorker -->|result invalidation| Cache
```

**Key tasks include:**
- User verification + voter token issuance on email verification
- Password reset emails
- Election activation / reminders / end notifications
- Close ended elections (periodic)
- Bulk college data processing (upload batches)
- Vote confirmation emails (optional)

Queues separate heavy notification fan-out from regular email.

---

## User Roles & Permissions

See detailed matrix in the [Roles diagram](#user-roles--permissions).

---

## Voter Journey

```mermaid
flowchart TD
    Start[Discover election] --> Register

    subgraph Registration
        ClassUpload["Class leader uploads\ncollege data"]
        Reg1[Submit reg. number]
        Reg2["Complete profile +\nemail + password"]
        Created["Account created\n(unverified)"]
    end

    subgraph Verification
        Verify[Commissioner verifies]
        Tokens["Tokens generated for\nactive elections"]
    end

    subgraph Voting
        Active[Active election]
        Choose["Choose candidates\nper level"]
        Submit["Atomic submit\nusing VoterToken"]
        Recorded["Vote recorded\nToken consumed"]
    end

    subgraph Results
        Close[Election ends]
        Publish[Commissioner publishes]
        View["Public results +\nPDF report"]
    end

    Register --> Reg1 --> Reg2 --> Created --> Verify --> Tokens
    Tokens --> Active --> Choose --> Submit --> Recorded
    Recorded --> Close --> Publish --> View
```

---

## Secure Voting Process

```mermaid
sequenceDiagram
    autonumber
    actor Voter
    participant UI
    participant Server
    participant Token as VoterToken
    participant Votes as Vote

    Voter->>UI: Load vote page
    UI->>Server: GET /elections/{id}/vote/
    Server-->>UI: Eligible levels + unused tokens

    loop Per election level
        Voter->>UI: Select candidates
        Voter->>UI: Submit level votes
        UI->>Server: POST submit

        Server->>Token: mark_as_used() — atomic UPDATE WHERE is_used=false
        alt Already used
            Token-->>Server: 0 rows
            Server-->>UI: Error — already voted
        else Success
            Token-->>Server: Success
            loop For each position voted
                Server->>Votes: INSERT Vote (token, candidate...)
            end
            Server-->>UI: Votes recorded
            Note over Server,Votes: Voter identity never exposed in public tallies
        end
    end
```

**Guarantees:**
- One token per (user, election, level)
- Atomic consumption prevents double-voting even under concurrency
- Votes linked to token, not user, in result views

---

## Election Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Configured: Define levels, positions, candidates

    Configured --> Ready: Set dates + activate

    Ready --> Active: Voting window opens
    Active --> Voting: Voters cast votes

    Voting --> Ended: End date / manual close
    Active --> Ended: Force close

    Ended --> Tallied: Internal tally ready

    Tallied --> Published: Commissioner publishes results
    Published --> Archived: Generate signed PDF report

    Archived --> [*]

    note right of Active
        - Tokens usable
        - Results sealed
        - Turnout visible to monitors
    end note

    note right of Published
        - Full counts public
        - PDF downloadable
        - Immutable
    end note
```

---

## Deployment Architecture

Current production setup (2026):

```mermaid
flowchart TB
    subgraph Internet["Internet / Users"]
        CF["Cloudflare\nA record + Proxy"]
    end

    subgraph VPS["VPS 159.65.119.182"]
        direction TB
        Nginx[nginx :443 + :81]
        Gunicorn["Gunicorn\n3 workers\nunix socket"]
        DjangoApp["Django App\n(src/)"]
        MySQL["(MySQL\ndigivote_db)"]
        Redis["(Redis\nlocal)"]
        CeleryWorker[Celery worker]
        CeleryBeat[Celery beat]
    end

    subgraph External["External Services"]
        Cloudinary[Cloudinary]
        EmailSvc[Email]
    end

    CF -->|HTTPS| Nginx
    Nginx --> Gunicorn
    Gunicorn --> DjangoApp

    DjangoApp --> MySQL
    DjangoApp --> Redis
    CeleryWorker --> Redis
    CeleryWorker --> MySQL
    CeleryWorker --> EmailSvc
    DjangoApp --> Cloudinary

    subgraph GitHub["GitHub"]
        Actions[Actions Workflow]
    end

    Actions -->|SSH deploy key| VPS

    classDef infra fill:#fef3c7,stroke:#854d0e
    classDef data fill:#dcfce7,stroke:#166534
    class Nginx,Gunicorn,DjangoApp,CeleryWorker,CeleryBeat,Actions infra
    class MySQL,Redis data
```

**Key points:**
- Single VPS (1 vCPU / 1 GB currently)
- Two access paths: direct IP:81 (HTTP) + subdomain via Cloudflare
- Cloudinary for user-uploaded media (candidate images + reports)
- GitHub Actions uses SSH to deploy (fetch + pip + migrate + collectstatic + restart)
- Future direction mentioned in code: multi-instance with shared Redis + DB

---

## Data Model Overview

```mermaid
classDiagram
    class User {
        registration_number PK
        role
        is_verified
        voter_id UUID
    }

    class CollegeData {
        registration_number
        is_used
        uploaded_by
    }

    class Election {
        title
        start_date, end_date
        is_active, has_ended
        results_published
    }

    class ElectionLevel {
        type (president|course|state)
    }

    class Position {
        title
        gender_restriction
    }

    class Candidate {
        user
        bio
        running_mate
        vote_count
    }

    class VoterToken {
        user
        election
        election_level
        token UUID
        is_used
    }

    class Vote {
        token
        candidate
        election_level
        position
        timestamp
    }

    User "1" --> "*" CollegeData : used to create
    Election "1" *-- "*" ElectionLevel
    ElectionLevel "1" *-- "*" Position
    Position "1" --> "*" Candidate
    Election "1" --> "*" VoterToken
    User "1" --> "*" VoterToken
    VoterToken "1" --> "0..*" Vote
    Candidate "1" --> "*" Vote
```

**Critical invariant:** A `Vote` is always tied to a `VoterToken`. Public results never surface the original user.

---

## Security & Integrity

- **Vote privacy**: Votes stored with token reference. User identity stripped in public views and reports.
- **Atomic voting**: `VoterToken.mark_as_used()` uses conditional update.
- **Sealed results**: `results_published` flag + `has_ended` gate all tallies.
- **Rate limiting + cache lockouts**: Login, password reset, and contact forms.
- **Role enforcement**: Multiple layers (decorators, permissions classes, template guards).
- **Token-based auth support**: JWT available via DRF for future mobile / API clients.
- **Audit artifacts**: CollegeDataUploadBatch, ElectionReport, logs.

---

## Request Lifecycle

```mermaid
sequenceDiagram
    participant Browser
    participant CF as Cloudflare
    participant Nginx
    participant Gunicorn
    participant Django
    participant Redis
    participant DB as MySQL

    Browser->>CF: Request (HTTPS)
    CF->>Nginx: Forward (with X-Forwarded-Proto)
    Nginx->>Gunicorn: unix socket
    Gunicorn->>Django: WSGI call

    Django->>Redis: Check session / cache
    Django->>DB: ORM queries

    alt Cache hit
        Redis-->>Django: Cached data
    else Miss
        DB-->>Django: Data
        Django->>Redis: Store in cache
    end

    Django-->>Gunicorn: Response
    Gunicorn-->>Nginx
    Nginx-->>CF
    CF-->>Browser
```

HTMX requests follow the same path but return fragments.

---

## How to Use This Documentation

- All diagrams are self-contained Mermaid.
- To edit: copy into https://mermaid.live
- To publish: this entire folder/repo can be hosted via GitHub Pages or simply linked from the main project.

## Repository Purpose

This is the **public face** of MWECAU DigiVote architecture. Implementation details live in the application repository.

Maintained by the MWECAU ICT Club.

## License

This documentation is released under the MIT License (see [LICENSE](LICENSE)).

---

*Last updated: 2026*
