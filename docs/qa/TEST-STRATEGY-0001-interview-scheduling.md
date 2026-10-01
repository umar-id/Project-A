# TEST-STRATEGY-0001: Interview Scheduling — Test Strategy & DoR Vectors

- **Status:** Draft
- **Date:** 2026-10-01
- **Owner:** Zeeshan (SQA)
- **Ticket:** [#6](https://github.com/umar-id/Project-A/issues/6)
- **Depends on:** [ADR-0001](../adr/ADR-0001-calendar-integration.md) / [#2](https://github.com/umar-id/Project-A/issues/2)
- **Feeds:** [#7](https://github.com/umar-id/Project-A/issues/7) E2E smoke (after #3–#5 happy paths)
- **Epic:** [#1](https://github.com/umar-id/Project-A/issues/1)

## 1. Goal

Lock DoR test vectors against ADR-0001 contracts so #3–#5 are testable before merge, and give #7 a stable smoke matrix. Vectors are provisional until ADR is committed.

## 2. Scope

| In scope | Out of scope |
| --- | --- |
| OAuth, freeBusy, slots, book/reschedule/cancel API vectors | Full UI automation |
| SPOF negatives, idempotency, audit, optimistic lock (409) | Phase-2 push sync and load soak |
| Deterministic mocked-Google fixtures | Real Google OAuth in CI |

## 3. Test levels

| Level | Owner / when | Notes |
| --- | --- | --- |
| Unit | Dev (#3–#5) | Slot math (DST/TZ), backoff, idempotency |
| Contract / integration | SQA + Dev | CalendarClient/services with Google mocked |
| E2E smoke | #7 | OAuth → slots → book → reschedule → cancel |

## 4. Fixtures & seams (DoR)

Stories are not Ready until these exist: injectable CalendarClient; TokenStore test double with encryption stub and revoked_at; injectable clock/TZ (Asia/Karachi plus a DST zone); inspectable idempotency store; fixed busy intervals for empty, blocked, overlapping and edge-window cases.

## 5. Vector matrix

### 5.1 OAuth / TokenStore (#3)

- O-01: start returns 200 and redirectUrl.
- O-02: valid callback stores encrypted refresh token, issues session, and leaks no plaintext token.
- O-03: bad/missing code returns 4xx with no TokenStore write.
- O-04: state mismatch returns 4xx with no token write.
- O-05: invalid_grant sets revoked_at and returns 401 REAUTH_REQUIRED.
- O-06: stored scopes include calendar.events and calendar.readonly (or ADR minimum).

### 5.2 CalendarClient (#3)

- C-01: freeBusy normalizes busy intervals.
- C-02: 403 rateLimitExceeded/429 retries at most three times with backoff+jitter, then returns 503 CALENDAR_RATE_LIMITED.
- C-03: persistent Google 5xx circuits/returns 503 with no partial write.
- C-04: createEvent returns eventId and etag; expired access token refreshes transparently.

### 5.3 Availability (#4)

- A-01: clear 45-minute window returns sorted slots inside working hours.
- A-02: no slot overlaps a busy block; multi-interviewer slots are mutually free.
- A-03: Asia/Karachi and DST fixtures have correct local boundaries.
- A-04: duration beyond remaining window yields empty/truncated slots without overrun; invalid body returns 4xx.

### 5.4 Book / reschedule / cancel (#5)

- I-01: book with Idempotency-Key returns interviewId/calendarEventId, audit row and etag.
- I-02: same key/body replays same ids and performs one Calendar create; changed body conflicts.
- I-03: double-book recheck returns 409 SLOT_TAKEN with no orphan event.
- I-04: PATCH reschedule updates Calendar and audit; DELETE cancels event/status; unknown id returns 404.
- I-05: revoked token returns 401 REAUTH_REQUIRED and creates no event.

### 5.5 NFR spot-checks (ADR §6)

- Slots p95 <2s for ≤5 calendars/14-day window; book p95 <3s including Google round-trip.
- Scan/review confirms no refresh tokens in repo, logs or error payloads.
- Mutate logs include user_id/interview_id and metrics cover refresh failures, 403/429 and conflicts.

## 6. DoR checklist

- [ ] Acceptance criteria map to vector IDs (or explicit waiver)
- [ ] Mock seam and fixtures listed
- [ ] Error codes match ADR: 401 REAUTH_REQUIRED, 503 CALENDAR_RATE_LIMITED, 409 SLOT_TAKEN
- [ ] Idempotency-Key specified for book/reschedule
- [ ] Audit fields asserted; no live Google required for merge CI

## 7. #7 E2E smoke

After #3–#5 are green: OAuth start/callback (sealed account), availability returns a slot, book creates visible event, PATCH updates time/etag, DELETE removes event/status canceled. Stop #7 until all five are green in staging or recorded fixtures.

## 8. Risks

| Risk | Mitigation |
| --- | --- |
| ADR drift | Re-diff this doc when ADR changes |
| Flaky live Google | Mocks in CI; live only in sealed smoke |
| Double-book under load | Concurrent race test (I-03) |

## 9. Approval

- [x] QA draft (Zeeshan)
- [ ] PM (Adil) — commit path
- [ ] SA ack (Akhter)
- [ ] Dev ack (Ali)

**Suggested path:** `docs/qa/TEST-STRATEGY-0001-interview-scheduling.md` (closes / advances #6)
