# Pulse — Notes

- **Live:** _TBD (Vercel URL)_
- **Repo:** _TBD (GitHub URL)_

Short, per-phase notes on what I changed and why. Assumptions and trade-offs are
called out inline.

---

## Phase 1 — Make it run

_Format: **Symptom** — root cause (file:line) · how I found it · fix._

- **Chat messages never arrived** — the sender's own bubble showed, but the
  other side got nothing. `sendChat` sent `{ t: "msg" }` while the data-channel
  handler only accepts `t === "chat"`, so every message was silently dropped
  ([lib/webrtc.ts:129](lib/webrtc.ts:129) vs [lib/webrtc.ts:76](lib/webrtc.ts:76)).
  · Found by tracing the send and receive paths in `PeerSession`.
  · Fixed by sending `t: "chat"`.
- **Ghost dots stayed on the map** — users who crashed or lost connection
  never disappeared. The poll heartbeat ran `updateMany({ where: {} })`, so any
  single poll refreshed *every* row's `lastSeen` and the stale reaper never
  found anything ([app/api/poll/route.ts:26](app/api/poll/route.ts:26)).
  · Found by reading the heartbeat/reap logic against the README's example
  report. A clean tab close hides it (`sendBeacon` → `/api/leave`), so it only
  shows when a user leaves without a clean leave.
  · Fixed by scoping the heartbeat to the caller: `where: { id }`.
- **Users stuck "busy" after a chat** — once a chat ended, both dots stayed
  dimmed and every new request to either user was auto-declined. `accept` set
  `busy` on both users, but only `decline` cleared it — not `end` — even though
  the comment says "decline/end"
  ([app/api/signal/route.ts:79](app/api/signal/route.ts:79)).
  · Found by reproducing (connect → End → reconnect → "Request declined") and
  comparing the comment with the code.
  · Fixed by clearing `busy` on `end` too.
- **Chat stuck on "Connecting…"** — the request was accepted but the data
  channel never opened. ICE candidates that arrive before the offer/answer are
  queued, but the queue was flushed *before* `setRemoteDescription`
  ([lib/webrtc.ts:107](lib/webrtc.ts:107)). `addIceCandidate` fails without a
  remote description and the error was swallowed, so the peers never learned
  each other's addresses. Polling delivers signals in 1.5 s batches (and the
  fire-and-forget POSTs can land out of order), so candidates almost always
  arrive in the same batch as, or before, the SDP.
  · Found when a two-window test hung on "Connecting…"; traced the signal
  handling order in `PeerSession.handleSignal`.
  · Fixed by flushing queued candidates right after `setRemoteDescription`.

**Verify:**
- Chat: connect two windows; messages appear on both sides.
- Ghost dots: with one window open, add a user that never polls
  (`POST /api/join` with a test id via curl). Its dot disappears within ~15–17 s.
- Busy: connect, End, then request the same user again — they get the prompt,
  and both dots are back to full brightness.
- Connecting: after Accept, both panels switch to "Connected" within a few
  seconds, every time.

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
- Stayed on Prisma 7.x; did not adopt the 8.0 release candidate the CLI suggests
  (a major, pre-release upgrade is out of scope and risky for the deliverable).
- `end` frees both ids it names unconditionally; the server doesn't track who is
  connected to whom, so partner-aware clearing is deferred (Task 7 / Phase 3).
