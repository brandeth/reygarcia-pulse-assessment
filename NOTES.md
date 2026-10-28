# Pulse — Notes

- **Live:** _TBD (Vercel URL)_
- **Repo:** _TBD (GitHub URL)_

Short, per-phase notes on what I changed and why. Assumptions and trade-offs are
called out inline.

---

## Phase 1 — Make it run

_Format: **Symptom** — root cause (file:line) · how I found it · fix._

- _TBD_

**Verify:** _TBD_

---

## Phase 2 — Make it good

- _TBD_

---

## Phase 3 — Make it secure

Ranked by impact. Status is updated as fixes land.

| # | Severity | Issue | Impact | Status |
|---|----------|-------|--------|--------|
| 1 | High | No session authentication — the client-chosen session id is the only credential, and `/api/poll` hands every peer's id to every client | Anyone can act as any user (see 2–5) | Open |
| 2 | High | `/api/poll?id=<victim>` drains another user's mailbox | Steal connection requests / SDP / ICE; hijack or break connections | Open |
| 3 | High | `/api/signal` trusts `fromId` | Impersonate users; forged `accept` marks any two users busy | Open |
| 4 | Medium | `/api/leave` and `/api/join` accept any id | Kick anyone off the map; move someone else's dot | Open |
| 5 | Medium | No rate limiting on any endpoint | Mailbox spam, DB load (every poll also runs cleanup deletes) | Open |
| 6 | Low | Signal payloads aren't validated as JSON | Malformed payload throws in the client's `handleSignal` | Open |
| 7 | Low | `poll` `id` unvalidated; ngrok host hardcoded in `allowedDevOrigins` | Hygiene | Open |

**Verify:** _TBD_

---

## Phase 4 — Make it better

- **What:** _TBD_
- **Why:** _TBD_
- **Next with more time:** _TBD_

---

## Assumptions & trade-offs

- STUN-only (no TURN) is kept, per the README — some strict networks won't connect.
