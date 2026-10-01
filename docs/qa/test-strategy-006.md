# Test Strategy & DoR Vectors — Interview Scheduling (#6)

- **Issue:** https://github.com/umar-id/Project-A/issues/6
- **Owner:** SQA Engineer (Zeeshan)
- **Aligned to:** ADR 0001 provisional contracts (revise when ADR updates)

## Risk focus

| Area | Risk | Severity |
|------|------|----------|
| OAuth / token refresh / revoke | Auth break, bad secret handling | Critical |
| freeBusy → slots (TZ/DST) | Wrong interview times | Critical |
| Book / reschedule / cancel | Double-book, orphan events | Critical |
| Rate-limit / webhook lag | Flaky UX, false free slots | High |

## DoR test vectors (lock to API shapes)

### OAuth (#3)
- Happy: start → callback → encrypted token stored; listCalendars works
- Refresh: expired access token auto-refreshes
- Negative: `invalid_grant` forces re-consent; no plaintext secrets in logs

### Availability (#4)
- Happy: busy gaps yield sorted slots in requested tz
- DST / cross-midnight working hours
- Empty window / fully busy → empty slots

### Book / reschedule / cancel (#5)
- Happy: create event + attendees; PATCH reschedule updates Calendar; DELETE cancels
- Idempotency-Key replay returns same interviewId
- Race: two concurrent books on same slot → one wins, other conflict

### SPOFs
- Rate-limit 403 → retryable error surface
- Stale freeBusy vs concurrent book → conflict path

## #7 dependency

E2E smoke for book/reschedule/cancel waits on #3–#5 happy paths + CI sandbox Calendar credentials.
