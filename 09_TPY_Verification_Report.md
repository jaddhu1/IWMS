# IWMS — TPY Verification Report v1.0

Results from the pre-implementation feasibility verification pass. This report exists so Codex (or any implementer) never has to assume TPY behavior — it either checks here, or treats the item as unverified and flags it.

| # | Item | Status | Notes |
|---|---|---|---|
| 1 | `User.getData` / `saveData` | ✅ Verified | Per-worker key-value storage confirmed working |
| 2 | Nested object storage | ✅ Verified | Native dict/object storage confirmed — no manual JSON serialization needed |
| 3 | Large object storage | ✅ Verified | Confirmed working at tested scale |
| 4 | `Bot.runCommandAfter` | ✅ Verified | Delayed single-command scheduling confirmed; returns job with `id` |
| 5 | `Bot.cancelScheduledTask` | ✅ Verified | Confirmed cancels a previously scheduled job |
| 6 | `while` loop restriction | ✅ Verified | Confirmed restricted — avoid unbounded loops in command code |
| 7 | Shared/global data concurrent write (Review Queue, Payroll Queue, Worker Resource under simultaneous multi-admin access) | ⏳ Pending | Could not be tested with a single account — requires 2+ concurrent admin sessions or a shared-resource load test. **Treated as an accepted V1 risk** (Architecture Freeze v1.0, Golden Rule 16) until tested under real operational load. |
| 8 | 100 scheduled-commands-per-user limit under realistic load | ⏳ Pending | Requires load testing with many simultaneous reminders/deadlines per worker. Not blocking — flagged for monitoring during early rollout. |

**Confirmed separately (platform fact, not a test):** Telegram `file_id` is bot-token-scoped and does not transfer across different bots — this is a Telegram platform limitation, not a TPY limitation, and is treated as a permanent constraint (see Migration handling, Architecture Freeze v1.0 §12).

---

## Status
6 of 8 items verified. 2 pending items are **not blockers** — they are deferred to implementation-time monitoring, per the explicit decision to pause the Verification Sprint and proceed (documented in the planning history). If either pending item later reveals a hard limitation, only the affected implementation strategy changes — not the frozen Architecture.
