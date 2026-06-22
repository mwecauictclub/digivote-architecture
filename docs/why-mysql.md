# Why MySQL

This document captures the rationale for using MySQL as the primary database for MWECAU DigiVote.

## Context

Early versions of the platform were prototyped with SQLite and later documented with PostgreSQL in some materials.

After evaluating operational reality at Mwenge Catholic University and similar institutions:

- MySQL / MariaDB has stronger local operational expertise.
- Easier integration with existing university infrastructure and backup procedures.
- Good Django support.
- Sufficient feature set for this workload (elections are not high-frequency OLTP).

## Decision

**Adopt MySQL (utf8mb4) as the supported production database.**

PostgreSQL remains usable for development or special deployments if desired, but MySQL is the documented and tested target.

## Consequences

- All new documentation, deployment guides, and example configurations use MySQL.
- Migration scripts and environment variables are MySQL-oriented.
- Connection pooling and timeout settings are tuned for typical MySQL behavior.
