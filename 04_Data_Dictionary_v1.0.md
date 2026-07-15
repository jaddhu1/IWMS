# IWMS — Data Dictionary v1.0
Exact meaning of every enum used in Data Model & Resource Schema v1.0. One meaning per state — no ambiguity.

---

### worker.status
| Value | Meaning |
|---|---|
| New | Just `/start`ed, no progress yet |
| Applying | Introduction seen, Apply Now not yet tapped, or mid Instagram-collection |
| Training | In Training/Setup combined step |
| VerificationPending | Setup screenshot submitted, background verification not yet resolved |
| Active | Normal working state — eligible for work assignment |
| Paused | Admin-triggered temporary hold; wallet frozen by default; data intact |
| Suspended | Reserved for policy-driven temporary restriction distinct from manual Pause (e.g., Restricted Investigation Mode) |
| Removed | Voluntary or Severe-Violation removal — see `worker.removal_type` |
| AppealPending | Removed, appeal filed, awaiting decision |
| Deleted | Permanent delete executed — irreversible |

### worker.removal_type
| Value | Meaning |
|---|---|
| Voluntary | Non-fraud removal — wallet Available balance preserved |
| SevereViolation | Fraud/severe policy breach — wallet Available balance forfeited |

### work.status
| Value | Meaning |
|---|---|
| Pending | Assigned, deadline not yet reached, not fully submitted |
| Submitted | All required screenshots locked (edit window closed) |
| Missed | Deadline passed without full submission, or a submitted account was Rejected |
| UnderReview | In Review Queue, decision not yet made |

### review.final_outcome
| Value | Meaning |
|---|---|
| Approved | All accounts approved — proceeds to Reward Engine |
| Rejected | At least one account rejected (Minor/Major severity) — worker sees "Rejected," full redo required |
| UnderInvestigation | At least one account flagged Critical severity — routed to Investigation, worker sees "Under Review" |

### review.per_account_decisions[].severity (internal only — never shown to worker)
| Value | Meaning |
|---|---|
| Minor | Quality issue (blurry/cropped/incomplete) — same worker-facing recovery as Major |
| Major | Assignment issue (wrong account/content/missing) — same worker-facing recovery as Minor |
| Critical | Suspected fraud/policy violation — never a simple reject, always routes to Investigation |

### investigation.mode
| Value | Meaning |
|---|---|
| Standard | Worker keeps receiving new work; only the flagged submission is held (default) |
| Restricted | Worker's new-work eligibility paused entirely until investigation resolves |

### investigation.result → investigation.mapped_outcome
| Result | Automatic Mapped Outcome |
|---|---|
| FalseAlarm | Approve the original submission |
| RedoWork | Reject → standard redo path |
| PolicyViolation | Remove (Voluntary type — wallet-safe) |
| SevereFraud | Remove (SevereViolation type — wallet forfeit) |

### ledger entry.type (Wallet Ledger — append-only, immutable)
| Value | Meaning |
|---|---|
| RewardCredit | Payroll calculation credited earnings |
| WithdrawLock | Withdrawal requested — funds moved conceptually Available→Locked |
| Processing | Payment execution started (post payroll-day approval) |
| Withdrawn | **Only truly irreversible state** — payment confirmed successful |
| Available | Funds released back (admin cancellation, payment failure, or Appeal reversal) |
| Forfeited | Disciplinary — terminal under normal operation; reversible **only** via Approved/Partially-Approved Appeal |

### withdrawal.workflow_status (Payroll Workflow — independent from ledger state)
| Value | Meaning |
|---|---|
| PendingReview | Request created, awaiting admin verification (any day of week) |
| Approved | Admin verified request details as correct |
| ReadyForPayroll | Approved and waiting for scheduled payroll day execution |
| Processing | Payroll day — payment execution in progress |
| Paid | Payment execution confirmed successful (ledger entry: Withdrawn) |
| Cancelled | Administrative cancellation (e.g., wrong UPI) — before payment execution; ledger reverts to Available |
| Rejected | Denied at PendingReview stage, mandatory reason given |
| Closed | Terminal state for both Cancelled and Rejected requests — lifecycle ends, a new request must be created for retry |

### appeal.status
| Value | Meaning |
|---|---|
| Draft | Being composed, resume-supported, not yet submitted |
| Submitted | Sent, not yet picked up for review |
| UnderReview | Admin actively reviewing |
| Approved | Employment and/or Financial decision granted (see below — independent) |
| Rejected | Appeal denied, no change to prior removal/forfeiture |
| Cancelled | Worker withdrew the appeal — does not forfeit right to file a new one (unless policy restricts) |

### appeal.employment_decision
| Value | Meaning |
|---|---|
| Restored | Worker resumes previous *valid* state (Golden Rule 9 — subject to eligibility check) |
| NotRestored | Removal stands |

### appeal.financial_decision (independent of employment_decision)
| Value | Meaning |
|---|---|
| Restore | Full forfeited balance returned to Available |
| PartialRestore | Portion returned — e.g., removal reversed but a specific penalty upheld |
| NoRestore | Forfeiture stands regardless of employment outcome |

### ticket.status (Support & Bug Report — shared FSM)
| Value | Meaning |
|---|---|
| Draft | Being composed, resume-supported |
| Submitted | Sent, not yet opened by admin |
| Open | In queue, unassigned |
| Assigned | Owner admin set |
| AdminReply | Admin has responded, awaiting worker |
| WaitingForWorker | Same as above — explicit wait state driving Reminder Engine |
| Resolved | Issue addressed, pending worker/admin close confirmation |
| Closed | Terminal — see `close_reason` for how it got there |
| Reopened | Worker reopened within policy window (default 7 days) — continues same issue |

### ticket.close_reason
| Value | Meaning |
|---|---|
| Resolved | Issue genuinely fixed/answered |
| ClosedByWorker | Worker manually closed |
| ClosedByAdmin | Admin manually closed |
| AutoClosedNoResponse | Reminder Engine exhausted (24h/72h/7d) with no worker reply |

### bug_report.resolution_type
| Value | Meaning |
|---|---|
| UserError | Not a bug — worker misunderstanding |
| Duplicate | Already reported elsewhere |
| CannotReproduce | Admin unable to replicate |
| ConfirmedBug | Verified genuine defect |
| Fixed | Confirmed bug, resolved |

### notification.type
| Value | Meaning |
|---|---|
| Silent | Dashboard/Notification-Center only, no push |
| Normal | Pushed, non-urgent (general updates) |
| Urgent | Pushed, immediate worker action implied |

### notification.status
| Value | Meaning |
|---|---|
| Unread | Not yet seen |
| Read | Seen |
| Archived | Explicitly filed away — never deleted |

---

## Rule for Future Enum Additions
Any new enum value added during implementation must be added to this dictionary in the same change — an enum value with no dictionary entry is treated as **not yet valid for use**.
