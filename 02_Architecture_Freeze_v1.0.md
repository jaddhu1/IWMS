# Instagram Workforce Management System (IWMS)
# Architecture Freeze v1.0

**Status:** FROZEN — Source of Truth for Implementation
**Scope:** Product Behaviour, State Machines, Admin/Worker UX, Recovery Architecture
**Precedes:** TPY Verification Sprint → Implementation Planning → Production Coding
**Base Reference:** Supersedes/extends the original 40-document GitHub architecture repository with concrete behavioural decisions made during this planning phase.

---

## 0. How to Read This Document

This document contains **only final, locked decisions**. It does not contain discussion, alternatives considered, or rationale in most cases (rationale exists in the planning conversation history if needed). Every rule here is binding unless a future, explicitly-versioned revision changes it. If implementation ever contradicts this document, the document wins — implementation must be corrected, or this document must be formally revised first.

---

## 1. Vision & Scope

**Product:** Telegram-based Workforce Management System. First implementation manages Instagram reel-posting workers. Architecture is intentionally generalizable to future workforce types (YouTube, Facebook, Freelance, Support, Moderation, Data Entry) without core redesign — but V1 scope is Instagram-only, on Telegram, built on Telebot Creator (TPY).

**Success Criteria:** 1000+ active workers, 10+ administrators, minimal worker confusion (to be made measurable during UX metrics definition), ~90% administrative automation, full history preservation, migration without worker restart.

**Core Philosophy:** Worker First · Automation First · Configuration over Hardcoding · History Never Lost · Everything Versioned · Everything Searchable · Everything Traceable · Modular Architecture · Simple for Workers · Powerful for Administrators.

**Non-Negotiable Rules:**
- Never lose worker progress.
- Never overwrite important history.
- Never hardcode business rules.
- Never duplicate important data.
- Every important action must be traceable.
- Every module remains independent.
- Every workflow should be automatable.
- Migration should never require worker restart.
- Speculative generality must never slow down V1 delivery — when a decision must choose between "generic" and "Instagram-specific," and generality would slow V1, choose specific and flag it as a known future-refactor item.

**Terminology note:** Backend resource remains named "Season"; worker-facing display label is "Week" (e.g., "Week 3"). A Season/Week = one business week, defined by a Global Calendar Week (see §6).

---

## 1.5 Architecture Principles

These are distinct from the Golden Rules (§2). Golden Rules govern *specific behavioral mechanics* (state transitions, resume logic, navigation). Architecture Principles govern the *underlying philosophy* that every design decision in this document was measured against.

- **Modular First** — every module (Worker, Wallet, Review, Support, etc.) has a single responsibility and communicates with others only by reference, never by direct coupling.
- **Configuration over Hardcoding** — business rules (slot counts, fees, weights, reasons, thresholds) live in versioned Policy objects, not code.
- **State Machines over Flags** — worker/admin/ticket/appeal progress is modeled as explicit states with defined transitions, never as loose boolean flags.
- **Event Driven** — automation reacts to triggers (submission complete, deadline passed, investigation resolved) rather than being manually polled or sequenced by admins.
- **History Never Deleted** — Timeline, Audit, Payment History, Appeal History are permanent and append-only.
- **Ledger, Not Balance** — anything financial is derived from an immutable transaction log, never stored as a single mutable field.
- **Worker Simplicity over Admin Convenience** — when a design choice trades off between reducing worker confusion and reducing admin effort, worker simplicity wins by default.
- **Admin Efficiency over UI Complexity** — for the admin side, task-oriented, priority-driven, permission-scoped design is preferred over exhaustive information display.

---

## 2. Golden Rules (Universal, Cross-Cutting)

These apply to every module, worker-side and admin-side, without exception unless explicitly overridden by a more specific locked rule.

1. **Resume, Never Reset.** `/start` (or any re-entry) always resumes current state; it never restarts a process. Behavior is state-driven per current stage (New User → Language Selection, mid-Training → Training Resume, Active → Dashboard, Paused → Pause Screen, Removed → Removed Status Screen, Permanently Deleted → Fresh registration only if policy allows).
2. **State-Driven, Not Command-Driven.** Commands, menus, and buttons only reflect the current state; they never independently determine or reset progress.
3. **Every Multi-Step Process Stores an Exact Resume Point.** Not just the stage name — the precise position within it (e.g., "Training – Video 3", "Instagram Setup – Account 2 of 3", "Withdrawal – Waiting for IFSC Code").
4. **Resume Happens on the Next User Interaction.** The bot cannot proactively resume a worker/admin after an interruption (crash, restart) — resume is triggered reactively by their next message/photo/button/callback, whichever comes first.
5. **State Version with Force-Update Priority.** Default: a worker who started a versioned resource (e.g., Training) completes that same version even if a newer version is released mid-progress. **Exception (priority overrides default):** Admin may explicitly flag a version as a "Critical Force Update," which interrupts in-progress workers with a visible reason ("Important policy update — please review before continuing") before allowing continuation. Priority order: Critical Force Update > State Version > Resume Point.
6. **Atomic State Transitions (Intent-Level).** A state changes only after its required action successfully completes. (Note: this is an *intent-level* rule, not a database-transaction guarantee — TPY has no confirmed atomic multi-write transaction support. Multi-write transitions require an implementation-phase recovery/verification pattern; see §11.)
7. **Setup Verification Affects Future Eligibility, Never Work Already in Progress.** Any setup/account verification (Training setup screenshots, Instagram account replacement, etc.) that later fails never cancels work currently in progress. It only blocks the *next* work assignment until corrected. This rule is universal across First Work, regular slot work, and account-replacement scenarios.
8. **Administrative Actions Are Selected Once; Consequences Apply Automatically by Policy.** Admins pick a single categorized decision (e.g., an Investigation Result); the system automatically maps it to the correct downstream consequence (approval, removal type, wallet treatment) via policy — admins are never asked to redundantly re-decide the same consequence.
9. **Resume Previous *Valid* State (refinement of Rule 1).** Restoring a worker/admin after interruption or reinstatement is never a blind restore — it is always subject to an eligibility check first (e.g., pending setup verification must resolve before resuming normal work flow).
10. **Financial Transactions Never Auto-Resolve After Interruption.** Any financial state left ambiguous by a crash/restart (e.g., stuck in "Processing") requires explicit human reconciliation. The system must never assume success or failure.
11. **Audit Ledger Is the Source of Truth; Backup Is System-State Restore, Not Truth.** After any restore-from-backup, the system must reconcile restored state against the immutable audit ledger and flag mismatches for admin resolution.
12. **Backup Frequency Is Configurable; Financial Correctness Must Never Depend on It.** (Direct consequence of Rule 11.)
13. **Every Waiting State Must Define Its Own Timeout Policy.** No workflow may leave a human waiting indefinitely without a defined reminder/escalation/auto-resolution path.
14. **No Administrator Sees Information or Actions They Are Not Authorized to Act On.** Applies to Dashboard cards, Critical Alerts, Worker Workspace sections/actions. Exception: Super Admin sees/acts on everything.
15. **Global Workspaces Locate Workers; Worker Workspace Executes Work.** All multi-worker queues (Review Queue, Support Queue, Payroll Queue, Search) are entry points only. All actual work on a specific worker happens inside that Worker's Workspace, and "Back" always returns to the exact originating queue position.
16. **Ownership Defines Responsibility, Not Exclusive Access.** A single named owner is accountable for a work item (e.g., a support ticket), but other authorized admins may still view/act on it; the system provides soft awareness (viewing indicators, last-activity), never a hard lock. (Noted future item: system-level locking may be introduced if operational conflicts become frequent — see §11.)
17. **Resource-Defined Validity.** Workspace Context restoration never assumes data is still valid — each resource type defines its own validity check (e.g., "does this worker still exist," "is this review still pending") and its own fallback/recovery strategy if invalid. Three possible outcomes: VALID (restore), PARTIALLY VALID (restore with adjustment), INVALID (fallback).
18. **Every Unique System-Generated Identifier Must Be Searchable.** Any new module introducing a new ID type must be added to the Search System automatically as a design requirement.
19. **Search Opens the Smallest Relevant Workspace.** A search result never dumps the full Worker Workspace by default — it opens directly into the smallest scoped context needed (e.g., a specific Appeal, a specific Transaction), with contextual navigation (Previous/Next, Worker Overview shortcut) available from there.

---

## 3. Worker Journey — Full State Machine

### 3.1 Discovery, Language, Introduction
- New user `/start` → Language selection → Work Introduction → **Apply Now** button (not a form, not a separate approval-wait stage — a pure intent-confirmation trigger).
- Language changeable anytime via Settings. Already-sent messages are never edited/deleted; only future interactions use the new language. If the current step has language-specific assets (e.g., mid-Training), worker gets a **"Reopen Current Step"** option in the new language.
- If worker views Introduction but does not tap Apply Now: reminder sequence (Day 1 / Day 3 / Day 7, "Normal" notification type), then stops permanently — no further reminders. This funnel (Introduced but not Applied) must be visible to admins as a **Lead Management** view (Abandoned Applications), not silently discarded.
- Multiple `/start` presses are idempotent — same screen, no duplicate registration, no duplicate resources, no duplicate messages.
- Bot-block-then-unblock: on next `/start`, Resume (not restart) — Golden Rule 1.

### 3.2 Instagram Accounts Collection
- Number of required accounts = **one global default** (admin-configurable, same for all workers), shown to worker immediately after Apply Now.
- Accounts collected sequentially, one link at a time.
- Validation: **format check only** (does it look like a valid Instagram URL). No live/deep Instagram verification.
- Worker may retry an unlimited number of times on format failure.
- Post-hoc correction (e.g., admin later notices account is private/fake): admin manually initiates a change request with the worker — not automatic.

### 3.3 Training / Setup
Sequence (worker-controlled checkpoints, no forced wizard mid-sequence within the combined step):
```
[Combined Setup Step: Setup Video + Photos(assigned) + Bio Link(assigned) + Bio Caption(assigned) + Audio]
        ↓
[Setup Screenshot Submit — non-blocking, background verification]
        ↓
[Next Step button]
        ↓
Upload Tutorial
        ↓
[Start Work button]
        ↓
First Work
```
- Profile Photos and Channel (Telegram invite) Link are assigned **during this Training step** (not immediately after Instagram collection).
- Setup Screenshot count = number of Instagram accounts (one screenshot per account).
- **Setup verification is non-blocking** — worker proceeds immediately to Upload Tutorial and First Work without waiting. Verification happens in the background.
- If verification fails at any point: worker gets an immediate, specific-reason notification ("Issue found: [reason]. Please fix and resubmit."). Per Golden Rule 7, **work already in progress is never cancelled** — only the *next* work assignment is blocked until the worker fixes and resubmits successfully.
- Resume Point for this combined step: worker returns to the full combined screen on re-entry (granular per-item tracking not required here, since it is one logical step).

### 3.4 First Work (Slot-Independent)
- Triggered by "Start Work" button. No slot selection has occurred yet.
- Deadline: 3 hours (admin-configurable default).
- On success: submission → Review Queue → **Slot Selection unlocks**.
- On miss (deadline passes without completion): **no penalty.** Worker receives a notification ("You haven't completed your first work yet") plus an **"Ab Start Karein" ("Start Now")** button, available at any time, no restriction. Tapping it issues a **new** reel/caption (old reel/caption message is auto-deleted from the worker's chat).

### 3.5 Slot Selection
- Occurs only after First Work is completed successfully.
- Bot displays admin-configured min/max slot range; worker selects within that range.
- Slots are fixed at a **2-hour minimum gap**, enforced system-wide (including Extra Work requests — the system checks and either allows or suggests the next valid time; it does not simply reject).
- The bot **prevents** selection of slots violating the 2-hour gap by disabling conflicting options — not by showing an error after the fact.
- Slot changes: currently unlimited frequency, but implemented as a configurable setting (`slot_change_frequency_limit`, default: unlimited) for future scale-driven restriction.
- No "last slot of the day" exception exists — slots are configured on a continuous 24/7, 2-hour-gap cycle, so "next slot" always exists.

### 3.6 Regular Work Assignment (Slot-Based)
- **Entry:** Slot time reached, worker Active, Instagram accounts Active/Verified, no other pending work.
- **Deadline:** Next slot start minus 30 minutes (clean, no exceptions, due to 2-hour continuous cycle).
- **Auto-delete rule (universal):** the reel/caption work-message is deleted from the worker's chat in exactly two cases: (1) when the deadline passes (Missed), a new reel replaces it; (2) the moment the worker finishes submitting all required screenshots (Success) — chat stays clean either way. Submitted screenshots and backend Work/Review records are **never** deleted, only the display message.
- **On Missed (no submission at all):** Status → Missed. Worker gets an **immediate** per-slot notification. Worker may retry at the **next available slot**, valid only until end of that calendar day (11:59 PM). Multiple missed slots in one day generate **one combined "Missed Work" flag** at day-end (11:59 PM), not one per miss. Admin receives a **day-summary notification** at day-end (total missed slots per worker), not a per-miss alert.
- **On Rejected (submission existed, admin rejected any account's screenshot during review):** Entire submission becomes invalid → treated as Missed for that slot → next available slot offered, **but with no same-day restriction** (since review delay is not the worker's fault — review has no SLA, see §4).

### 3.7 Screenshot Submission
- Upload order: **any order** (not sequential/enforced).
- Bot validation: **photo-type check only** ("is this a photo"); content correctness is admin's job at review time.
- Progress: **auto-progress** — no confirm button between accounts.
- **Shared edit window:** a single 10-minute edit window for the entire submission, starting **after the last screenshot is submitted**. Within this window, worker may replace any screenshot. After the window closes, submission is locked ("submitted"), no further edits.
- If the deadline is less than 10 minutes away when the last screenshot lands, the edit window **shrinks to the deadline** (never extends past it).
- Duplicate-screenshot detection via image hashing was considered and **explicitly rejected** (TPY capability unconfirmed; replaced by the 10-minute edit window as the simpler, sufficient control).
- On success: submission locks → Review Queue → work-message auto-deletes.
- On deadline passing without full completion: Missed (per §3.6). Screenshots that *did* arrive before the deadline are still saved and go to Review Queue; the missing account(s) show as "Missing."

---

## 4. Review Queue & Investigation Workflow (Admin-Side)

- Queue order: **FIFO.**
- Reel/original assignment is **not** shown during review (screenshot alone is sufficient proof).
- Review granularity: **per-account** — admin reviews and decides (Approve/Reject) each account's screenshot individually, one at a time, within a submission.
- Navigation: manual **"Next"** button after all accounts in a submission are decided (no auto-advance).
- Concurrent-admin visibility: **soft indicator only** ("👀 [Admin Name] currently reviewing this") — no hard lock (see Golden Rule 16 and §11 open item).
- **No SLA on review** — purely manual admin discipline, by design decision.

### 4.1 Rejection Structure
- **Step 1 — Predefined category** (single tap): organized into three groups —
  - *Quality Issues:* Blurry Screenshot, Cropped Screenshot, Incomplete Screenshot
  - *Assignment Issues:* Wrong Instagram Account, Wrong Content, Missing Account
  - *Policy Issues:* Fake Screenshot, Edited Screenshot, Rule Violation
  - *Other:* free-text (mandatory when selected)
- **Step 2 — Optional note** (admin can add clarifying free text; not mandatory unless "Other" was chosen).
- Categories/severities/display order are **configurable via a Review Policy** (same pattern as Compensation Policy) — not hardcoded, to support future non-Instagram work types without redesign.
- Reasons feed a **rejection analytics dashboard** (e.g., "Wrong Account: 48 this week") for identifying systemic worker/training issues.

### 4.2 Severity (Internal-Only — Never Shown to Worker)
- Every predefined rejection reason maps to a **Severity**: Minor, Major, or Critical.
- **Worker always sees only one of three statuses: Approved / Rejected / Under Review.** Severity, category detail, and any fraud-investigation context are never disclosed to the worker.
- **Minor and Major rejections use the exact same worker-facing recovery path** (§3.6 — entire work invalid, redo required at next slot). There is deliberately **no** separate "quick reshoot" path for Minor — this was considered and explicitly rejected to preserve the "one mistake → one recovery path" principle (Worker Simplicity).
- Severity exists purely for: Analytics, Audit, Admin Dashboard, and driving Investigation Mode (below).
- **Critical severity does not reject the submission — it triggers Investigation** instead (see §4.3).

### 4.3 Investigation Workflow (Critical Severity)
```
Review → Critical Flag → Investigation Queue → Investigation Result → automatic policy-mapped outcome
```
- Worker sees a neutral **"Under Review"** message only — investigation, fraud, or any specific reasoning is **never** disclosed to the worker (evidence-tampering / panic risk).
- **Investigation Mode is Review-Policy-configurable:**
  - **Standard (default):** worker continues to receive new work normally; only the flagged submission itself is held.
  - **Restricted:** worker's new-work eligibility is paused entirely until investigation resolves. (Reserved for high-risk/future client-sensitive scenarios; Instagram V1 will typically use Standard.)
- **Investigation Result options and their automatic policy mapping (Golden Rule 8):**

| Investigation Result | Automatic Outcome |
|---|---|
| False Alarm | Approve |
| Redo Work | Reject → Redo (standard recovery path) |
| Policy Violation | Standard Remove (wallet-safe — see §7) |
| Severe Fraud | Severe Remove (wallet forfeit — see §7) |

Admin selects the Investigation Result **once**; the system applies the correct downstream Removal type and Wallet treatment automatically — admin is never asked to redundantly re-choose "Standard vs Severe."

---

## 5. Payroll, Wallet & Compensation System

### 5.1 Core Philosophy
Reward and Payment are separate concepts, bridged by a Wallet. Flow:
```
Review Approved → Performance Calculation → Reward Engine → Worker Wallet
→ Weekly Payroll → Withdrawal Request → Payroll Workspace → Payment → History & Audit
```

### 5.2 Wallet Ledger — Independent State Machine (Accounting)
- **Append-only ledger.** Balance is *derived* from a full transaction log, never stored/mutated as a single overwritable field — this is mandatory given TPY's lack of confirmed atomic transaction support, and prevents silent race-condition data loss.
```
Available → Locked → Processing → Withdrawn (FINAL — only truly irreversible state)
                   ↘ Available (Administrative Cancellation / Payment Failure)
                   ↘ Forfeited (Disciplinary — terminal under normal operation)
```
- **"Failed" is not a ledger state — it is a workflow outcome** that always resolves into either `Available` (cancellation/failure) or `Forfeited` (disciplinary).
- **Forfeited is reversible exclusively through an Approved/Partially-Approved Appeal** (see §8). `Withdrawn` remains the only permanently irreversible financial state (Financial Finality Rule, Golden Rule 10/11 family).
- **Financial Finality Rule:** "Money is considered final only after successful payment (Withdrawn). Until then it can be cancelled, rejected, forfeited, or recalculated."

### 5.3 Payroll Workflow — Independent State Machine (Operational)
```
Pending Review → Approved → Ready For Payroll → Processing → Paid
                          ↘ Cancelled → Closed (before payment execution — see below)
Pending Review → Rejected → Closed
```
- **Wallet Ledger and Payroll Workflow are two fully independent state machines.** Workflow status changes (e.g., "Approved," "Ready for Payroll") do **not** move ledger balance out of `Locked` — that only happens at actual payment execution (`Processing → Withdrawn`) or failure/cancellation/forfeiture.
- **Approval is not a financial commitment.** An approved-but-unpaid request can still be Cancelled (administrative — e.g., wrong UPI, duplicate request → ledger reverts `Locked → Available`) or forced into `Forfeited` (disciplinary — e.g., fraud discovered after approval, before payment → ledger `Locked → Forfeited`, Audit records reason).
- **Every withdrawal request has exactly one immutable lifecycle** — a Cancelled/Rejected/Closed request is never reopened or reused. A worker needing to retry (e.g., after fixing UPI details) creates a brand-new request (new Request ID). Full, clean audit trail.

### 5.4 Withdrawal — Weekly Verification Model
- Worker may submit a withdrawal request **any day of the week** (not only on payroll day), provided: payroll is open, minimum withdrawal reached, no existing active request, sufficient Available balance.
- Requested amount moves `Available → Locked` immediately on request.
- **Admin verifies (Approve/Reject) throughout the week**, as requests arrive — not all in a batch on payroll day. Approved requests move to a **"Ready for Payroll"** queue.
- On the scheduled Payroll Day, admin executes only the **payment step** (`Ready for Payroll → Processing → Paid`) for the already-verified batch — dramatically reducing payday-specific workload.
- If Rejected: mandatory reason, worker notified immediately, `Locked → Available`. Worker retains full control over whether/when to submit a new request (e.g., after updating UPI/bank details) — no forced auto-retry.
- Fee model: **worker bears the fee** (Option A) — e.g., ₹1000 requested → ₹1000 debited from wallet → worker receives ₹980 after a 2% fee. Worker always sees Requested / Fee / Final Amount, all three, before confirming.
- Payment Proof and Transaction Reference (UTR/UPI ref/Bank txn ID) are both **optional** for admin to attach — payment remains valid without them; if present, worker may view.
- Payment History and Audit are permanent, never deleted, and store full detail (Request ID, Worker, Amount, Fee, Net, Method, Status, Date, Administrator, optional Proof/Reference).

### 5.5 Compensation Policy (Versioned, Configurable)
- Never hardcode salary values. All of the following are Compensation-Policy-versioned: Maximum Weekly Earning, Maximum Weekly Slots, Maximum Weekly Join Target, Work Weight, Join Weight, Processing Fee, Minimum Withdrawal, Weekly Payday.
- **Every completed payroll week permanently stores the exact policy version used** (immutable Weekly Snapshot). Future policy changes never retroactively affect historical payroll calculations.
- **Mid-week policy changes apply only starting the following week** — never mid-cycle.

### 5.6 Performance Calculation
- **Work Score** = Approved Work ÷ Maximum Weekly Slots.
  - Rejected work contributes 0 (neither positive nor negative).
  - Missed work is **not** separately penalized — its effect is purely via the denominator (no explicit extra subtraction).
  - **Pro-rated for mid-week joiners:** Maximum Weekly Slots is calculated only for the days the worker was actually active that week (not the full week), removing bias against workers who joined mid-week.
- **Join Score** = MAX(0, Current Members − Last Recorded Members), i.e. **floored at zero** — a net-negative week (due to genuine member departures, outside worker control) never produces a negative Join Score or drags down Work Score.
  - Source of truth: Telegram's **live, current** `getChatMemberCount` only (confirmed via Telegram Bot API — there is no persistent "total historical joins via this link" counter available to bots). This also means join-then-immediately-leave fake-growth abuse is naturally self-limiting, since the count is live at calculation time.
  - **Channel Replacement:** when a worker's channel is replaced, a new baseline starts. **The Join Score is entirely excluded (not merely floored) for the week in which replacement occurs** — only Work Score counts that week (100% weight). The Weekly Snapshot must flag `channel_replaced_this_week: true` for transparency. Following weeks resume normal Join Score counting on the new baseline.
- **Final Performance Score** = weighted combination (e.g., Work 70% / Join 30%, both configurable via Compensation Policy) → drives final Wallet credit (capped at Maximum Weekly Earning).

### 5.7 Weekly Continuation
- **Global Calendar Week** (not per-worker rolling) — all workers share the same week start/end, admin-defined. A worker joining mid-week gets a naturally shorter first (partial) week; already handled by Work Score pro-rating (§5.6).
- New week starts **automatically** the moment Payroll Calculation completes — no manual admin trigger.
- Week transition is **entirely backend** — no worker-facing transition message.
- Work/Review pending across a week boundary is attributed to **the week the work belongs to** (submission week), not the week the review happened to complete in.
- **Withdrawal eligibility:** a worker may withdraw as soon as their current period's Payroll Calculation is complete — no additional fixed "N days since joining" restriction. This falls naturally out of the Wallet mechanics (balance only becomes Available post-calculation) and requires no extra flag.

---

## 6. Worker Removal, Pause & Appeal

### 6.1 Pause (Temporary)
- Admin-triggered, with reason. No new work assigned. Wallet/history/progress fully intact.
- Wallet actions **freeze by default** on pause; admin may manually override to process a payment anyway if desired.
- Unpause → Resume exactly where left off (Golden Rule 1).

### 6.2 Remove — Two Distinct Sub-Types (Critical Distinction)
| Type | Trigger | Wallet Impact |
|---|---|---|
| **Voluntary / Performance Removal** | Worker exit request, poor performance, or any non-fraud reason | Available Balance **preserved** — worker gets one last withdrawal opportunity |
| **Severe Violation Removal** | Fraud, fake screenshots, or equivalent serious violation confirmed | Available Balance **forfeited** — no payment |
- Admin does **not** freely choose this at removal time when triggered via Investigation — it is **automatically mapped** from the Investigation Result (§4.3, Golden Rule 8). For non-investigation removals, an explicit toggle ("Forfeit Wallet: Yes/No") is required at removal time — never inferred solely from a free-text Exit Reason.

### 6.3 Permanent Delete
- Triggered by failed Appeal or explicit worker data-deletion request. Irreversible. Only minimum audit record survives.

### 6.4 Appeal
- No time limit to file an appeal.
- Default reviewer: Super Admin, but delegable to any admin.
- **Submission (hybrid, resume-supported):**
```
Draft (resume-supported) → Submitted → Under Review → Approved / Rejected / Cancelled
```
  - Step 1: Category (Wrong Removal / Wrong Penalty / Wrong Wallet Forfeit / Wrong Review Decision / Other).
  - Step 2: Free-text explanation.
  - Step 3 (optional): Evidence upload (screenshot/video/proof).
- **Appeal Pending screen (not a full block):** worker retains limited access — a status screen (Status, Submitted date, View Appeal, Cancel Appeal), not full Dashboard.
- **Cancelling an appeal closes only the current appeal** — it never removes the worker's right to file a new appeal later, unless a business policy explicitly restricts this (future-proofing hook).
- **Employment and Financial decisions on Appeal approval are fully independent:**
  - Employment: Approved → Resume Previous **Valid** State (Golden Rule 9). Rejected → no change.
  - Financial: Restore / Partial Restore / No Restore — decided separately, may not mirror the employment decision (e.g., removal reversed, but a specific late-submission penalty upheld).
- **Appeal History is permanent**, never deleted, visible on the Worker Workspace.
- **Appeal Framework is generic** — designed to support appeals against Removal, Wallet Forfeit, Review Decisions, and Penalties (not hardcoded to Removal only) — future-proofing for the broader Workforce Platform vision.

---

## 7. Notification System

| Type | Push? | Examples |
|---|---|---|
| **Silent** | ❌ | New Training Available, New Announcement, Weekly Stats Ready |
| **Normal** | ✅ | Work Assigned, Support Reply, Payment Approved/Rejected, Weekly Reward Calculated, Training Completed |
| **Urgent** | ✅ | Slot Missed, Work Deadline Near, Setup Rejected, Payment Failed, Critical Policy Update, Account Verification Failed |

- **Reminder is not a fourth notification type — it is an Automation Rule** applied on top of any notification type (defines *when/how often* to repeat, independent of *how important* the message is, which the Type defines).
- Notification Center tracks status: **Unread / Read / Archived** — entries are never deleted (consistent with History/Timeline "never delete" philosophy).

---

## 8. Admin Journey

### 8.1 Admin Mission — 7 Core Objectives
Monitor · Review · Manage · Pay · Support · Maintain · **Grow** (new-worker acquisition / onboarding-funnel tracking, including Abandoned Applications).

### 8.2 Responsibility Classification (3 Categories)
| Category | Nature | Maps to |
|---|---|---|
| **Daily** | Routine, queue-based | Review, Support ticket replies, Grow (abandoned-application check) |
| **Scheduled** | Time-based, predictable | Pay (payday execution), Maintain (periodic health check), Grow (weekly funnel report) |
| **Event-Driven** | Unpredictable, triggered | Manage (removal, account replacement), Monitor (critical failure alerts), Support (appeals), Maintain (migration) |

### 8.3 Admin Daily Workflow — Priority Model (Not a Forced Wizard)
**"Workflow defines priority, not navigation."** Admins navigate freely; the Dashboard only surfaces priority.
- **Level 1 — Critical Alerts (always pinned top, permission-scoped, never count-sortable):** system failures, backup failures, automation stoppage, fraud investigations — but **only shown to admins authorized to act on them** (Golden Rule 14). Super Admin sees all.
- **Level 2 — Operational Tasks (dynamic, sorted by Business Impact Priority, not raw pending count):**
  1. Pending Reviews (system bottleneck — blocks reward/payroll/next cycle)
  2. Critical Appeals (high-priority subset; normal appeals rank lower)
  3. Withdrawal Verification (ongoing weekly clearing)
  4. Support Tickets
  5. Bug Reports
  6. Everything else (analytics, growth, reports) — Level 3, informational only, no action implied.
- **Golden Rule:** *"The Admin Dashboard should answer: 'What requires my attention right now?' — not 'Which menu should I open?'"*
- Dashboard is **task-oriented** (e.g., "25 Reviews Waiting"), never menu-oriented.
- **Attention Panel = "Action Required" only, not general information.** Only items requiring an action appear (Pending Review, Appeal Under Review, Pending Withdrawal Verification, Investigation). Passive information (Locked Balance alone, Paid history, completed reviews) never appears here — that data lives in its normal section instead.
- **Multi-permission admins get one unified, priority-sorted task list** (not separate per-category sections), with color-coded category badges (🔴 Review, 🟠 Payroll, 🔵 Support, etc.) for quick visual parsing.
- Refresh model: **implementation detail, not an architecture constraint** — architecture only requires "Dashboard always reflects latest system state on open/refresh"; exact mechanism (open-triggers-refresh, manual refresh button) is decided at TPY implementation time.

### 8.4 Admin Navigation — Two-Level "Workspace Context" Model
- **Level 1 — Global Workspaces:** Dashboard, Review Queue, Support Queue, Payroll Queue, Search. Purpose: **locate** a worker or multi-worker task.
- **Level 2 — Worker Workspace:** Overview, Wallet, Timeline, Appeals, Support, Instagram, Current Work, Actions. Purpose: **execute** all work related to that one worker.
- **Golden Rule (§2.15): "Global Workspaces locate workers. Worker Workspace completes work."** Once inside a specific worker's context (from any originating queue), all further related actions happen inside that Worker Workspace — the admin does not jump out to unrelated global workspaces mid-task. "Back" always returns to the exact originating global-queue position (filter, sort, scroll, current item — full context, not just screen name).
- **Navigation stack** (renamed "Workspace Context" — stores complete work context, not just page names): max depth **20**, FIFO eviction of oldest entries if exceeded, current context never lost.
- **Back vs Dashboard:** *"Back preserves context. Dashboard intentionally resets navigation context."* Dashboard is always available as a persistent one-tap reset.
- **Cross-session persistence is optional, not mandatory (admin difference from worker):** on re-entry with an existing stale stack, admin is offered **Resume** (restore stack) or **Dismiss** (clear stack, fresh Dashboard) — admin choice, since intentional fresh starts are common for admins.
- **Context expiry is data-based, never time-based.** *"Context expires only when the underlying data becomes invalid, not because time has passed."*
- **Resource-Defined Validity (Golden Rule 17):** each resource type owns its own validity check and recovery fallback:

| Resource | Invalid when | Fallback |
|---|---|---|
| Worker Profile | Worker permanently deleted | Not found → Dashboard/Search |
| Review Queue item | Already processed by another admin | Nearest valid pending item |
| Wallet/Payment History | Never (append-only) | Always restores |
| Support Ticket | N/A — closed tickets are still valid history | Closed → Closed Ticket View |
| Appeal | Already resolved | Pending View → Final Decision View |

### 8.5 Worker Workspace — Structure
```
Worker Summary (ID, Status, Level, Week)
    ↓
Action Required Panel (only actionable items — Pending Review, Appeal Under Review, Pending Verification, Investigation)
    ↓
Overview → Current Work → Instagram → Wallet → Appeals → Support → Timeline → Actions
```
- **Section badges** show *active/actionable* counts only (e.g., "Appeals (2)" = 2 active, not total historical).
- **Actions are hybrid:** Contextual actions live where the decision is made (e.g., "Adjust Balance" inside Wallet section); Global actions (Pause, Remove, Send Message, Search, Dashboard) are persistently available.
- **All actions are permission-scoped** (Golden Rule 14) — an admin never sees an action they cannot execute (e.g., a Support Admin sees Message/Support/Bug-Report actions only, not Wallet/Investigation/Payroll actions).
- **Search is result-oriented and scope-minimal (Golden Rule 19):** e.g., searching an Appeal ID opens directly to that Appeal inside the Worker Workspace (with Previous/Next/Overview shortcuts), not the entire Workspace.
- **Searchable identifiers (expanded, and permanently extensible per Golden Rule 18):** Worker ID, Telegram ID, Username, Name, Link ID, Invite Link, Instagram Username, Work ID, Ticket ID, Appeal ID, Payment Request ID, Wallet Transaction ID, Timeline Event ID, Review ID.

---

## 9. Support System

### 9.1 Worker-Initiated FSM
```
Draft → Submitted → Open → Assigned → Admin Reply ⇄ Waiting for Worker (loop)
    → Resolved → Closed → Reopen (policy-controlled) → Closed Again
Side path: Waiting for Worker → Reminder Engine → Auto Close (No Response)
```
- Submission: worker chooses category → writes message → optional evidence → review → submit.
- Worker cannot DM admin directly — everything routes through Support (or Bug Report).
- **Timeout / Reminder (reused Reminder Engine pattern):** 24h → Reminder #1, 72h → Reminder #2, 7 days → **Auto Close**, with a distinct close-reason: `Auto Closed – No Response` (separate from `Resolved`, `Closed by Worker`, `Closed by Admin`, for clean analytics).
- **Reopen vs New Ticket:** Reopen continues the *same* unresolved issue only. A genuinely new problem always creates a New Ticket. Default reopen window: **7 days** after closure (policy-configurable); after expiry, worker must open a new ticket.
- **Ownership:** every ticket has exactly one responsible **Owner** admin (drives Dashboard task assignment, reminders, SLA responsibility). **Ownership = Responsibility, not Exclusive Access** — any authorized admin may still view/reply (recorded in Timeline as "Reply by Admin B", ownership unchanged); ownership only changes via explicit manual reassignment (Super Admin), never automatically. Soft awareness only (viewing indicator, last-activity) — same concurrency caveat as Review Queue (§11).

---

## 10. Bug Report System

Reuses ~90% of Support System (same FSM, Reminder Engine, Ownership model, ticket lifecycle). Differences:

| Feature | Support | Bug Report |
|---|---|---|
| Category | Worker chooses | Fixed = "Bug" |
| Severity | Not applicable | **Admin-assigned only** (Minor/Major/Critical) — worker never sees severity |
| Evidence | Optional | Optional, but strongly encouraged (screenshot/video upload step, skippable) |
| Affected Area | Optional | **Required** (Registration, Training, Wallet, Work, etc.) |
| Resolution Type | General outcome | User Error / Duplicate / Cannot Reproduce / Confirmed Bug / Fixed |
| FSM / Reminder / Ownership | — | Identical to Support |

---

## 11. Architecture Stress Test — Results Summary

Four end-to-end simulations were run against the fully-locked architecture. All four passed, with the following rules discovered/refined *as a direct result* of the simulations (already incorporated above, listed here for traceability):

| Scenario | Path Tested | Result | New/Refined Rule Produced |
|---|---|---|---|
| **A — Happy Path** | Onboarding → First Work → Slots → Regular Work → Review → Payroll → Withdrawal → Payment | ✅ Pass | Golden Rule 7 (Setup Verification never cancels in-progress work — made universal) |
| **B — Failure Path** (split into Miss Path + Review/Investigation Path) | Miss → Retry → Complete; and separately, Complete → Reject → Critical Investigation → Remove → Appeal → Restore | ✅ Pass | Golden Rule 8 (auto-mapped Investigation consequences); Golden Rule 9 (Resume Previous *Valid* State) |
| **C — Admin Day** | Dashboard → Review Queue → Worker Workspace → (Wallet/Support sections within it) → Back → Dashboard | ✅ Pass | Golden Rule 15 (Global Workspaces locate, Worker Workspace executes — two-level navigation model) |
| **D — Disaster Recovery** | Bot Restart → Resume → Workspace Context → Migration → Backup → Restore | ✅ Pass | Golden Rules 10, 11, 12 (financial reconciliation never automatic; Audit Ledger = truth; backup frequency configurable but never a financial-correctness dependency); Health Check must detect stuck-Processing payments, audit/database mismatches, incomplete workflows, failed scheduled tasks |

**No fundamental architectural contradiction was found in any scenario.** All findings were hidden case-specific assumptions successfully generalized into universal rules — a sign of underlying design consistency, not fragility.

---

## 12. Known Open Items for TPY Verification Sprint (Next Phase — Not Yet Executed)

These are **explicitly deferred to implementation-feasibility verification**, not unresolved architecture. If verification reveals a TPY limitation, **only the implementation strategy changes — this document's architecture does not change** unless a limitation is fundamental enough to require a formal revision.

**Storage**
- [ ] `saveData`/`getData` practical size limits for large/nested worker objects.
- [ ] List/array performance and size limits (e.g., large Review Queues, large Payroll batches).
- [ ] Frequency limits (if any) on repeated `saveData` calls per worker (relevant to granular Resume Points).

**Scheduler**
- [ ] `runCommandAfter` — confirmed 100-scheduled-commands-per-user cap; verify this is never approached under realistic slot/reminder/deadline load per worker.
- [ ] `cancelScheduledTask` reliability under high-frequency debounce/reschedule patterns (e.g., edit-window countdowns).
- [ ] Reminder Engine (Support/Bug Report/Onboarding) — verify total scheduled-task load stays within limits.

**Media**
- [ ] `file_id` cross-bot migration — **confirmed to fail by design** (Telegram `file_id`s are bot-token-scoped). Migration must rely on an admin-facing, bot-generated re-upload checklist — no automatic media transfer is possible.

**Concurrency**
- [ ] Simultaneous `saveData` writes / race-condition behavior on shared keys (Review Queue, Support ticket, Payroll batch). Current architecture position: **accepted as an operational/manually-watched risk in V1** (Golden Rule 16 note: "Future Version: system-level locking may be introduced if operational conflicts become frequent").

**Performance**
- [ ] Large worker dataset behavior (1000+ workers) for Search, Timeline, Dashboard aggregate counts.
- [ ] Audit-ledger-vs-database reconciliation mechanism feasibility (§2 Golden Rule 11) — how mismatch detection is technically implemented after a restore.

**Verification Checklist Format (to be used when the sprint executes):**

| Item | Expected | Result | Fallback Action if Not Verified |
|---|---|---|---|
| *(each item above)* | *(what correct behavior looks like)* | ⏳ Pending | *(redesign/alternative strategy, defined per item at verification time)* |

---

## 13. Module Dependency Diagram

This map shows which modules depend on which others being complete/available — primarily useful for sequencing Implementation Planning (§13/14).

**Core Worker Progression Chain:**
```
Worker (identity)
    ↓
Instagram Accounts
    ↓
Training / Setup
    ↓
First Work
    ↓
Slot System
    ↓
Regular Work Assignment
    ↓
Screenshot Submission
    ↓
Review Queue
    ↓
Reward Engine (Performance Calculation)
    ↓
Wallet (Ledger)
    ↓
Payroll Workflow → Payment
```

**Disciplinary / Investigation Chain:**
```
Review Queue
    ↓
Investigation (Critical Severity)
    ↓
Removal (Voluntary or Severe — policy-mapped)
    ↓
Appeal (Employment + Financial, independent)
    ↓
Wallet (possible Forfeit reversal)
```

**Admin-Facing Convergence Chain:**
```
Support ─┐
Bug Report ─┤
Review Queue ─┤
Payroll Queue ─┼──→ Search ──→ Worker Workspace ←── Dashboard
Appeals ─┤
Investigation ─┘
```
All global/multi-worker workspaces converge on the single **Worker Workspace** as the execution environment (Golden Rule 15). Dashboard is the entry point; Search is an alternate entry point; every queue is a locator, never an executor.

**Cross-cutting dependencies (used by nearly everything, not sequential):**
- Golden Rules (§2) — apply to all modules above.
- Notification System (§7) — triggered by nearly every state transition above.
- Timeline/Audit — written to by nearly every state transition above.
- Workspace Context (§8.4) — governs navigation across every admin-side module.

---

## 14. Out of Scope for V1 (Deliberately Postponed, Not Missed)

The following were explicitly discussed and consciously deferred during planning. Their absence from V1 is a decision, not an oversight — a future implementer should not treat these as forgotten requirements.

- **AI-assisted/automatic Review** (auto-approving/rejecting screenshots without human review).
- **System-level locking for concurrency** (hard locks on Review Queue items, Support tickets, Payroll batches) — V1 uses soft awareness indicators only; accepted as an operational risk, revisit if conflicts become frequent (Golden Rule 16).
- **Multi-company / multi-tenant architecture** — V1 is single-organization.
- **Real-time (push-based) Dashboard updates** — V1 Dashboard refreshes on open/navigation, not via live server push.
- **Automatic cross-bot media migration** — Telegram `file_id`s are bot-scoped; V1 migration relies on an admin-facing manual re-upload checklist, not automated transfer.
- **Cross-platform workforce types** (YouTube, Facebook, freelance, support, moderation, data entry) — architecture is designed to *permit* this later without core redesign, but no such work type is implemented in V1.
- **Advanced/predictive analytics** (beyond the basic rejection-reason and growth reporting already specified).
- **Multi-client support** (e.g., running the same platform for multiple separate businesses/brands simultaneously).
- **Duplicate-screenshot detection via image hashing** — considered and explicitly rejected in favor of the simpler 10-minute edit window; may be revisited only if TPY is later confirmed to support image hashing.
- **Granular multi-role concurrent editing indicators** (separate Owner/Viewer/Editor roles) — V1 uses Owner + single Viewing Indicator only.

---

## 15. Architecture Metrics (Summary)

| Metric | Count |
|---|---|
| Golden Rules | 19 |
| Architecture Principles | 8 |
| Worker-Side State Machines | 7 (Onboarding, Instagram Collection, Training/Setup, First Work, Slot Selection, Regular Work, Screenshot Submission) |
| Shared/Cross-Side State Machines | 4 (Review/Investigation, Payroll Workflow, Wallet Ledger, Appeal) |
| Admin-Side Structural Systems | 4 (Dashboard, Navigation/Workspace Context, Worker Workspace, Admin Responsibility Classification) |
| Support-Family Systems | 2 (Support, Bug Report — ~90% shared) |
| Core Modules Referenced (from original 26-resource architecture) | 26 |
| Stress Test Scenarios Run | 4 |
| Stress Test Scenarios Passed | 4 |
| New Rules Discovered via Stress Testing | 6 |
| Open TPY Verification Items | 5 categories (Storage, Scheduler, Media, Concurrency, Performance) |
| Deliberately Out-of-Scope (V1) Items | 9 |

**Architecture Status: Production-Ready, Pending TPY Verification Sprint.**

---

## 16. Document Status

**This is Architecture Freeze v1.0.** It supersedes ad-hoc discussion as the reference for all subsequent work. Next phases, strictly in order:

```
Architecture Freeze v1.0 ✅ (this document)
        ↓
TPY Verification Sprint (§12 checklist, executed against real TPY)
        ↓
Implementation Planning (folder structure, resource schemas, command list, execution order)
        ↓
Production TPY Coding
```

Any future change to a rule in this document must be an explicit, versioned revision (v1.1, v2.0, etc.) — not a silent edit during implementation. If a new idea or requirement surfaces during the TPY Verification Sprint or Implementation Planning, it goes into a **v1.1 backlog**; v1.0 itself is not edited until formally revised.

---

## 17. Version History

**v1.0 — Initial Architecture Freeze**
- 19 Golden Rules
- 8 Architecture Principles
- Full Worker Journey (7 state machines) + Admin Journey (Dashboard, Navigation/Workspace Context, Worker Workspace)
- Review/Investigation, Payroll/Wallet, Appeal, Support, Bug Report systems fully specified
- 4 Architecture Stress Test scenarios run, 4 passed, 6 new rules discovered
- TPY Verification Sprint checklist defined (not yet executed)
- Explicit Out-of-Scope (V1) list defined

*(Future entries — v1.1, v1.2, etc. — will be logged here as this document is formally revised. v1.0 itself is never silently edited.)*

---

## 18. Terminology Index

| Term | Meaning |
|---|---|
| Week | Worker-facing display name for the backend "Season" resource — one business week |
| Worker Workspace | The single-worker execution environment where all worker-related admin actions are completed |
| Global Workspace | Multi-worker queues/entry points (Dashboard, Review Queue, Support Queue, Payroll Queue, Search) used only to locate a worker or task |
| Workspace Context | The admin-side navigation stack that preserves filters, scroll position, sort order, and current item across "Back" navigation |
| Audit Ledger | The immutable, append-only record treated as the financial and administrative source of truth — takes precedence over restored backups |
| Review Policy | Versioned, configurable set of rejection categories, severities, and display order for the Review Queue |
| Compensation Policy | Versioned, configurable set of payroll parameters (weights, fees, maximums, payday) — never hardcoded |
| Resume Point | The exact, granular position within a multi-step process (not just the stage name) used to restore a worker/admin after interruption |
| Investigation Mode | Review-Policy-configurable setting (Standard/Restricted) controlling whether a worker under Critical-severity investigation keeps receiving new work |
| Ledger State | The 5-state Wallet accounting model (Available/Locked/Processing/Withdrawn/Forfeited) — independent from Payroll Workflow state |
| Payroll Workflow State | The operational review/approval pipeline (Pending/Approved/Ready for Payroll/Processing/Paid/Cancelled/Rejected) — independent from Ledger State |
| Financial Finality Rule | The principle that only "Withdrawn" is truly irreversible; all other financial states remain correctable |
| Severity | Internal-only classification (Minor/Major/Critical) of a rejection reason — never shown to the worker |
| Global Calendar Week | The shared, admin-defined week boundary used for all workers (as opposed to a per-worker rolling week, which was considered and rejected) |

---

## 19. Status: OFFICIALLY LOCKED ✅

**Architecture Freeze v1.0 is now the binding Source of Truth for the Instagram Workforce Management System.**

No further feature discussion will be incorporated into this document until the TPY Verification Sprint is complete. New ideas surfacing during verification or implementation planning are logged to the v1.1 backlog, not merged into v1.0.

**Next immediate deliverable (start of Implementation Planning, not part of this document):** a **Data Model & Resource Schema Document** — concrete object schemas for Worker, Wallet Transaction, Review Record, Appeal Record, Support Ticket, Timeline Event, and all Policy objects (Compensation Policy, Review Policy) — to be produced *after* the TPY Verification Sprint confirms storage feasibility, so the schema is grounded in verified constraints rather than assumptions.
