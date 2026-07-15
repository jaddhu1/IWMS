# IWMS — Data Model & Resource Schema v1.0
**Status: FROZEN — Source of Truth for Implementation**
**Derived from Architecture Freeze v1.0.**
Storage model: TPY `User.saveData/getData` (per-worker) and `Bot.saveData/getData` (global/shared), confirmed to support nested objects natively (no manual JSON serialization needed).

---

## Conventions
- **ID format:** `{TYPE}-{YEAR}-{6-digit-sequence}` (e.g., `WK-2026-000154`, `TX-2026-000483`, `APL-2026-000203`). Globally unique across the system, supports future migration/multi-bot scenarios without collision risk.
- `created_at` / `updated_at` on every record (ISO timestamp).
- Fields marked **[Versioned]** retain full history via snapshot-on-change, never overwritten silently.
- Fields marked **[Never Delete]** are append-only / permanent.

---

## 1. Worker
**Key:** `User.saveData("worker_profile", {...})` (per-worker, own record)

| Field | Type | Notes |
|---|---|---|
| worker_id | string | Permanent, e.g. `WK-2026-000154` |
| telegram_id | string | Internal use only, never shown as primary identity |
| name, username | string | |
| language | string | Changeable via Settings anytime |
| level | enum | Bronze/Silver/Gold/Platinum — performance-based, independent of payment |
| status | enum | New/Applying/Training/VerificationPending/Active/Paused/Suspended/Removed/AppealPending/Deleted |
| stage | string | Current onboarding/journey stage (drives Resume) |
| resume_point | object | Granular position within current stage (Golden Rule 3) |
| current_week_id | string | Reference to current Season/Week |
| instagram_accounts | array[ref] | → Instagram Account IDs |
| assigned_profile_photos | ref [Versioned] | |
| assigned_channel_link | ref [Versioned] | |
| assigned_training_version | string | Locked per Golden Rule 5 (State Version) |
| slot_ids | array[ref] | Selected slots |
| current_work_id | ref/null | Direct pointer to active Work assignment — avoids scanning Work collection |
| slot_change_frequency_limit | int/null | Configurable, default unlimited |
| exit_reason | string/null | |
| removal_type | enum/null | Voluntary / SevereViolation |

---

## 2. Instagram Account
**Key:** `Bot.saveData("ig_account:" + account_id, {...})`

| Field | Type | Notes |
|---|---|---|
| account_id | string | |
| worker_id | ref | |
| link | string | Format-validated only |
| status | enum | Active/Replaced/UnderReview |
| replacement_count | int | |
| replacement_history | array [Never Delete] | {old_link, reason, timestamp, admin} |

---

## 3. Work (Assignment)
**Key:** `Bot.saveData("work:" + work_id, {...})`

| Field | Type | Notes |
|---|---|---|
| work_id | string | |
| worker_id | ref | |
| type | enum | FirstWork / RegularSlot |
| reel_ref, caption_ref | string | |
| assigned_at, deadline_at | timestamp | FirstWork: +3h default; RegularSlot: next_slot − 30min |
| status | enum | Pending/Submitted/Missed/UnderReview |
| screenshots | array[ref] | → Screenshot IDs |

---

## 4. Screenshot Submission
**Key:** nested under Work, or `Bot.saveData("submission:" + work_id, {...})`

| Field | Type | Notes |
|---|---|---|
| account_id | ref | Which IG account this screenshot is for |
| file_id | string | Telegram file_id (bot-scoped — see Migration notes) |
| submitted_at | timestamp | |
| edit_window_expires_at | timestamp | last-screenshot-time + 10min, capped at deadline |
| locked | bool | true after edit window closes |

---

## 5. Review Record
**Key:** `Bot.saveData("review:" + review_id, {...})`

| Field | Type | Notes |
|---|---|---|
| review_id | string | |
| work_id | ref | |
| worker_id | ref | |
| queue_position | int | FIFO |
| per_account_decisions | array | {account_id, decision: Approve/Reject, category, severity, note} |
| reviewing_admin | ref/null | Soft indicator only |
| final_outcome | enum | Approved/Rejected/UnderInvestigation |
| investigation_id | ref/null | |

---

## 6. Investigation
**Key:** `Bot.saveData("investigation:" + investigation_id, {...})`

| Field | Type | Notes |
|---|---|---|
| investigation_id | string | |
| worker_id, review_id | ref | |
| mode | enum | Standard/Restricted (Review Policy-driven) |
| result | enum/null | FalseAlarm/RedoWork/PolicyViolation/SevereFraud |
| mapped_outcome | enum | auto-derived from result (Golden Rule 8) |

---

## 7. Wallet Ledger Entry [Never Delete, Append-Only]
**Key:** `Bot.saveData("ledger:" + worker_id, [...])` (array of entries) or per-entry keys

| Field | Type | Notes |
|---|---|---|
| entry_id | string | e.g. `TX-2026-000483` |
| worker_id | ref | |
| type | enum | RewardCredit/WithdrawLock/Processing/Withdrawn/Available/Forfeited |
| amount | decimal | |
| related_request_id | ref/null | → Withdrawal Request |
| related_appeal_id | ref/null | For forfeit-reversal traceability |
| reason | string | |
| timestamp | timestamp | |

*Current balance = sum of entries — the ledger remains the source of truth.*

### 7.1 Wallet Summary (Cache)
**Key:** `Bot.saveData("wallet_summary:" + worker_id, {...})`

| Field | Type |
|---|---|
| available, locked, processing | decimal |
| last_ledger_entry_id | ref |
| last_reconciled_at | timestamp |

**Golden Rule: Ledger = Truth. Summary = Cache.** Every ledger write must update this cache in the same operation (best-effort, given no confirmed atomic multi-write support in TPY — see Verification Sprint). Because a partial-write failure could desync cache from ledger, the cache **must never be trusted blindly for high-stakes decisions** (e.g., final payment execution) — those should recompute from the ledger directly, or at minimum re-verify against `last_ledger_entry_id` before acting. The cache exists purely to make Dashboard/Workspace summary display fast; it is **not** an accounting record. Periodic reconciliation (cache-sum vs ledger-sum) should feed the same Health Check mismatch-detection flow defined for post-restore reconciliation (Architecture Freeze §2, Golden Rule 11).

---

## 8. Withdrawal Request (Payroll Workflow)
**Key:** `Bot.saveData("withdrawal:" + request_id, {...})`

| Field | Type | Notes |
|---|---|---|
| request_id | string | e.g. `WR-2026-000001` — one immutable lifecycle |
| worker_id | ref | |
| amount_requested, fee, net_amount | decimal | |
| payment_method | enum | UPI/Bank |
| payment_details_ref | ref [Versioned] | Reusable across requests |
| workflow_status | enum | PendingReview/Approved/ReadyForPayroll/Processing/Paid/Cancelled/Rejected/Closed |
| reject_or_cancel_reason | string/null | |
| proof_ref, transaction_ref | string/null | Optional |
| owning_admin | ref/null | |

---

## 9. Compensation Policy [Versioned]
**Key:** `Bot.saveData("comp_policy:" + version, {...})`

| Field | Type |
|---|---|
| version | string |
| max_weekly_earning, max_weekly_slots, max_weekly_join_target | number |
| work_weight, join_weight | percent |
| processing_fee | percent/flat |
| minimum_withdrawal | decimal |
| weekly_payday | day-of-week |
| effective_from_week_id | ref |

---

## 10. Weekly Snapshot [Never Delete]
**Key:** `Bot.saveData("snapshot:" + worker_id + ":" + week_id, {...})`

| Field | Type |
|---|---|
| week_id, worker_id | ref |
| comp_policy_version | ref |
| approved_work, rejected_work | int |
| previous_members, current_members, weekly_joins | int |
| channel_replaced_this_week | bool |
| work_score, join_score, performance_score | decimal |
| wallet_credit | decimal |

---

## 11. Appeal
**Key:** `Bot.saveData("appeal:" + appeal_id, {...})`

| Field | Type | Notes |
|---|---|---|
| appeal_id | string | e.g. `APL-2026-000203` |
| worker_id | ref | |
| category | enum | WrongRemoval/WrongPenalty/WrongForfeit/WrongReviewDecision/Other |
| explanation | text | |
| evidence_refs | array | Optional |
| status | enum | Draft/Submitted/UnderReview/Approved/Rejected/Cancelled |
| resume_point | object | Draft-stage resume support |
| employment_decision | enum/null | Restored/NotRestored |
| financial_decision | enum/null | Restore/PartialRestore/NoRestore |
| reviewing_admin | ref | Default Super Admin, delegable |

---

## 12. Support Ticket
**Key:** `Bot.saveData("ticket:" + ticket_id, {...})`

| Field | Type | Notes |
|---|---|---|
| ticket_id | string | |
| worker_id | ref | |
| category | string | Worker-chosen |
| message, evidence_refs | text/array | |
| status | enum | Draft/Submitted/Open/Assigned/AdminReply/WaitingForWorker/Resolved/Closed/Reopened |
| owner_admin | ref | Responsibility, not exclusive access |
| close_reason | enum/null | Resolved/ClosedByWorker/ClosedByAdmin/AutoClosedNoResponse |
| reopen_window_expires_at | timestamp/null | Default 7 days post-close |

---

## 13. Bug Report
**Key:** `Bot.saveData("bug:" + bug_id, {...})`
*(Extends Support Ticket schema — same FSM/fields, plus:)*

| Field | Type |
|---|---|
| affected_area | enum (Required) |
| severity | enum — admin-assigned, internal only |
| resolution_type | enum | UserError/Duplicate/CannotReproduce/ConfirmedBug/Fixed |

---

## 14. Timeline Event [Never Delete]
**Key:** `Bot.saveData("timeline:" + worker_id, [...])`

| Field | Type |
|---|---|
| event_id, worker_id | ref |
| event_type | string |
| payload | object |
| timestamp | timestamp |

---

## 15. Audit Record [Never Delete]
**Key:** `Bot.saveData("audit:" + record_id, {...})`

| Field | Type |
|---|---|
| record_id | string |
| actor_admin | ref |
| action_type | string |
| target_ref | ref (worker/ticket/appeal/etc.) |
| before_state, after_state | object |
| timestamp | timestamp |

---

## 16. Notification
**Key:** `User.saveData("notifications", [...])` (per-worker)

| Field | Type |
|---|---|
| notification_id | string |
| type | enum | Silent/Normal/Urgent |
| message | text |
| status | enum | Unread/Read/Archived |
| created_at | timestamp |

---

## 17. Workspace Context (Admin-side, ephemeral but resumable)
**Key:** `User.saveData("admin_nav_stack", [...], user=admin_id)`

| Field | Type |
|---|---|
| stack | array (max depth 20, FIFO evict) |
| each_entry | {screen, filter, sort, current_item, scroll_offset} |

---

## 18. Review Policy [Versioned]
**Key:** `Bot.saveData("review_policy:" + version, {...})`

| Field | Type |
|---|---|
| categories | array {name, group, severity} |
| investigation_mode_default | enum |
| display_order | array |

---

## v1.1 Backlog (Not Blocking v1.0 Freeze)
- Add `schema_version` field to major resources (Worker, Wallet, Review, Appeal, Ticket) for future migration ease.
- Add `deleted_at` / `deleted_by` soft-delete fields to Worker, Ticket, Appeal — not mandatory for v1.0, reserved for future soft-delete implementation.

---

## Open Notes for Implementation Phase
- Exact `saveData` size/nesting limits for large arrays (e.g., Timeline, Ledger) — flagged in Verification Sprint, still pending confirmation of long-term growth behavior.
- Whether Ledger/Timeline should be single growing arrays per worker or paginated by week/month — recommend **paginate by week** once real data volume is known, to avoid unbounded single-key growth.
