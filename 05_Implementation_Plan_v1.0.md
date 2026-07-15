# IWMS — Implementation Plan v1.0
**Status: FROZEN — Implementation Blueprint**
Blueprint converting Architecture Freeze v1.0 + Data Model v1.0 into an actual Telebot Creator (TPY) build.

---

## 1. Folder / Command Organization Strategy
TPY has no real filesystem — organization happens via **command naming + folders (if TPY's editor supports folder grouping, as seen in the reference bot export)**. Proposed folder grouping, mirroring Architecture modules:

```
/onboarding      → language, intro, apply, ig_collect, training_setup
/work             → first_work, slot_select, work_assign, screenshot_submit
/review           → review_queue, investigation
/payroll          → wallet, withdrawal, payroll_workspace
/appeal           → appeal_draft, appeal_review
/support          → support_ticket, bug_report
/admin            → dashboard, worker_workspace, search
/system           → reminders, health_check, migration, .env
```

---

## 2. Command Naming Convention
`{module}_{action}[_{qualifier}]`, all lowercase, underscore-separated (matches the pattern observed in the reference bot code).

Examples: `onboarding_apply_now`, `work_submit_screenshot`, `review_approve_account`, `payroll_approve_withdrawal`, `admin_open_worker`, `support_reopen_ticket`.

**Rule:** every command name must make its module obvious from the prefix alone — supports the folder grouping above and makes Search/debugging easier at scale.

---

## 3. Resource Loading Strategy
- Per-worker data → `User.getData/saveData` (own record) or `user=` param (admin cross-access), per Data Model v1.0.
- Shared/global data (queues, policies, snapshots) → `Bot.getData/saveData`.
- **No resource is loaded "just in case."** Every command loads only the specific keys it needs (e.g., `review_approve_account` loads the one Review record + relevant Ledger append — not the entire Worker object).
- Large collections (Review Queue, Ready-for-Payroll list) are stored as **arrays under a single shared key** per Verification Sprint findings (confirmed working for moderate sizes); if Verification Sprint's pending "shared concurrent write" and "100-scheduler" tests later reveal limits, this section is revisited without touching Architecture v1.0.

---

## 4. Scheduler Strategy (`runCommandAfter` / `cancelScheduledTask`)
- Every scheduled task stores its returned `job_id` against the entity it belongs to (e.g., Work record stores `deadline_job_id`; Ticket stores `reminder_job_id`) — enabling clean cancellation on early completion (debounce pattern, confirmed working per Verification Sprint).
- **Reminder Engine is a single reusable command set**, parameterized by entity type, not duplicated per module (Onboarding, Support, Bug Report all call the same underlying reminder scheduler with different message templates and intervals).
- Per-worker scheduled-task count is tracked informally during early rollout (manual spot-check) until the "100 scheduled commands per user" boundary is verified at real load — flagged, not blocking.

---

## 5. Concurrency Handling Strategy (Given No Confirmed Atomic Writes)
- All shared-queue mutations (Review Queue, Payroll Ready list, Ticket ownership) follow **read → modify → write** with no lock, per accepted V1 risk (Golden Rule 16).
- **Mitigation in place for V1:** soft "currently viewing" indicators (Review Queue, Support), single-owner convention (tickets), and — most importantly — **all financial mutations are ledger-append-only**, so even a lost/overwritten *summary* update never loses the underlying truth (Data Model §7/§7.1).
- Formal system-level locking is explicitly Out of Scope for V1 (Architecture Freeze §14).

---

## 6. Error Handling Standard
- Every command wrapped in `try/except`, matching the pattern already confirmed in the reference bot code.
- On exception: (1) send a generic, non-alarming message to the user ("Something went wrong, please try again — our team has been notified"), (2) log full error detail (command, user, timestamp, exception) to the Log Channel workspace, (3) never expose raw error text/stack traces to workers or non-Super-Admins.
- Financial commands (Wallet, Withdrawal, Payroll) get an **extra rule**: on any exception during a multi-step financial mutation, the command must write a partial-failure marker to the Audit Record *before* re-raising/exiting — feeding directly into the Golden-Rule-11 reconciliation flow, so no failure is ever silent.

---

## 7. Logging Standard
Three tiers, mapped to existing Workspace Architecture:
- **Technical Log Channel** — every command execution error (from §6).
- **Audit Trail** — every admin action with business consequence (approvals, rejections, removals, payments) — per Data Model §15, never deleted.
- **Timeline** — every worker-facing business event (per Data Model §14), never deleted.

**Rule:** a bug fix or feature must never rely on Technical Logs for business history — Technical Logs are ephemeral/debugging-only; Audit + Timeline are the permanent record.

---

## 8. Shared Utility Libraries (Internal, Reused Across Modules)
- `reminder_engine` — used by Onboarding, Support, Bug Report.
- `resume_engine` — resolves current stage + resume_point → correct screen, used by every `/start`-equivalent entry point (Golden Rules 1–4).
- `workspace_context` — admin navigation stack push/pop/validate, used by every admin-side module (§8.4 of Architecture Freeze).
- `ledger_writer` — the only code path permitted to append Wallet Ledger entries (never write balance fields directly) — single choke point enforcing Golden Rule "Ledger, Not Balance."
- `policy_reader` — fetches the currently-effective versioned Policy (Compensation/Review) for a given week, enforcing "mid-week changes apply next week only."
- `notification_dispatcher` — routes a message through the correct Type (Silent/Normal/Urgent) and any attached Reminder rule.

---

## 9. Development Phases

**Phase 1 — Worker Core Loop**
Onboarding → Instagram Collection → Training/Setup → First Work → Slot Selection → Regular Work → Screenshot Submission.
*Goal: a worker can join and complete work end-to-end, no review/payment yet (review auto-approved as a stub).*

**Phase 2 — Review & Investigation**
Review Queue (admin), per-account decisions, rejection categories, Investigation flow.
*Goal: real review replaces the Phase 1 stub.*

**Phase 3 — Wallet, Payroll, Compensation**
Ledger, Weekly Snapshot, Compensation Policy, Withdrawal Request, Payroll Workspace.
*Goal: worker earns and gets paid end-to-end.*

**Phase 4 — Removal, Appeal**
Pause/Remove/Delete, Appeal FSM, Employment/Financial independent decisions.

**Phase 5 — Support, Bug Report**
Shared FSM, Reminder Engine reuse, Ownership.

**Phase 6 — Admin Dashboard & Worker Workspace**
Priority-driven Dashboard, permission-scoped views, Workspace Context navigation, Search.
*(Built last, deliberately — every module it surfaces must already exist.)*

**Phase 7 — Recovery & Migration**
Health Check reconciliation, Backup/Restore, `file_id` migration checklist tooling.

**Cross-cutting, integrated throughout (not a separate phase):** Golden Rules compliance, Notification dispatch, Timeline/Audit writes.

---

## 10. Definition of Done (Per Phase)
A phase is not "done" when the happy path works — it is done when:
1. The relevant Architecture Freeze §Stress-Test-equivalent scenario passes manually.
2. Every enum used is present in the Data Dictionary.
3. Every financial or state-changing action writes to Timeline/Audit as specified.
4. Error handling (§6) and logging (§7) are in place, not deferred.

---

## Status: OFFICIALLY LOCKED ✅
Implementation Plan v1.0 is now binding alongside Architecture Freeze v1.0, Data Model & Resource Schema v1.0, and Data Dictionary v1.0. Any structural change during coding (folder layout, naming convention, phase order) is a versioned revision (v1.1+), not a silent edit.

**Planning Phase: COMPLETE.**
**Next action: Phase 1 (Worker Core Loop) — production TPY coding begins.**
