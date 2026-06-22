# Security & Integrity Model (Public View)

## Core Guarantees

1. **One person, one vote**
   - Enforced by `VoterToken` per `(user, election, election_level)`
   - Token can be consumed only once via atomic database update

2. **Vote anonymity**
   - Public results and PDF reports contain only `Vote` records linked to tokens and candidates
   - No voter identity is exposed in result views or exports

3. **Sealed results**
   - Vote tallies are never shown until both `has_ended=True` **and** `results_published=True`
   - Even commissioners and observers see only turnout data before publication

4. **Tamper resistance**
   - All vote writes go through validated forms + model constraints
   - Double-vote attempts are rejected at the token layer

## Defense in Depth

- Brute-force protection on login and password reset (Redis-backed)
- Rate limiting at DRF level
- CSRF + session security
- Role-based access control on all privileged views
- Input validation and model-level `clean()` / `full_clean()`

## Known Trade-offs (Public)

- Single Redis instance is both cache and broker (simplifies operations)
- Current deployment is single-VPS (future plans include multi-instance)
- Media stored on Cloudinary (chosen for multi-server compatibility)

This document describes the **public security model**. Implementation details and internal controls live in the application repository.
