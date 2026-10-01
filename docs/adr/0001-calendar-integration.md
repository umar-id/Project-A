# ADR 0001: Interview Scheduling — Google Calendar Integration

- **Status:** Proposed (authored from Solution Architect provisional contracts; posted by PM for #2)
- **Date:** 2026-10-01
- **Issue:** https://github.com/umar-id/Project-A/issues/2
- **Owner:** Solution Architect (Akhter)

## Context

Project A schedules interviews using Google Calendar for availability, booking, reschedule, and cancel. We need clear service boundaries, OAuth/token handling, Calendar API usage, failure modes, and SPOFs before implementation (#3–#5) and test design (#6–#7).

## Decision

Build a small **Scheduling service** that owns OAuth tokens, availability computation, and interview lifecycle. Google Calendar is an external system of record for busy times and events; our DB is system of record for interview metadata and audit.

### Service boundaries

| Component | Responsibility |
|-----------|----------------|
| Auth / OAuth | Consent, callback, encrypted refresh-token store, re-consent on `invalid_grant` |
| Calendar client | Typed wrapper: listCalendars, freeBusy, create/update/delete events |
| Availability | Slot calc from busy intervals + working hours + duration + TZ |
| Interviews API | Book / reschedule / cancel with idempotency + audit |

### OAuth / tokens

- `POST /oauth/google/start` → redirect URL
- `GET /oauth/google/callback?code=` → stores refresh token encrypted at rest; returns session
- Token store fields: `user_id`, `refresh_token_enc`, `scopes`, `expiry`, `revoked_at`
- Client refreshes access tokens transparently; on `invalid_grant` → force re-consent
- Secrets never in repo or plaintext env; use secret manager / KMS

### Calendar client (#3)

- `listCalendars(userId)`
- `freeBusy(userId, calendarIds[], timeMin, timeMax)` → busy intervals
- Event CRUD for book/reschedule/cancel
- Scopes: prefer minimal set covering freeBusy + event CRUD (`calendar.events` / readonly as needed)

### Availability (#4)

- `POST /availability/slots`  
  `{ interviewerIds[], durationMin, window: {start,end}, workingHours, tz }`  
  → `{ slots: [{start,end}] }` sorted, tz-aware (DST covered in tests)

### Book / reschedule / cancel (#5)

- `POST /interviews` `{ slot, attendees[], metadata }` → `{ interviewId, calendarEventId }`
- `PATCH /interviews/:id` reschedule (idempotent on `Idempotency-Key`)
- `DELETE /interviews/:id` cancel + sync Calendar
- Audit: `created_by`, `updated_at`, `calendar_etag`, `status`
- Optimistic locking on slot commit to reduce double-book races

### SPOFs & mitigations

| Risk | Mitigation |
|------|------------|
| Google token revoke | Detect `invalid_grant`; force re-consent; degrade gracefully |
| Webhook / push lag | Prefer freeBusy at query time for booking; treat push as cache invalidate |
| Rate-limit 403s | Backoff + jitter; queue; surface retryable errors |
| Double-book race | Idempotency keys + optimistic lock on slot / etag |

## Consequences

- Ali can scaffold #3 against these contracts once this ADR is accepted.
- Zeeshan locks #6 vectors to these shapes.
- Full sequence diagrams can follow in `docs/adr/0001-sequences.md` without blocking scaffolding.

## Alternatives considered

| Option | Pros | Cons | Why not |
|--------|------|------|---------|
| Calendar-only (no DB) | Simple | Weak audit / idempotency | Rejected |
| Sync-all events into DB | Fast queries | Sync complexity, drift | Deferred |
