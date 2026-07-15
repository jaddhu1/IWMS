# IWMS v1.0 Clarifications Log

**Status:** FROZEN - Binding Source of Truth for Implementation  
**Created:** 2026-07-15  
**Authority:** Equal to `01_Project_Vision_v1.0.md` through `09_TPY_Verification_Report.md`

These clarifications resolve the Phase 1 Architecture Exam ambiguity batch. They are binding for IWMS v1.0 implementation and must not be re-asked unless the human explicitly changes the decision.

---

## 1. Partial Screenshot Submission

**Question:** How should partial screenshot submissions be treated when at least one screenshot arrives before the deadline but one or more accounts are missing?

**Official Decision:** Partial submissions always enter the Review Queue. Missing accounts are reviewed as "Missing Submission" rejections. "Missed" is used only when absolutely no submission is received before the deadline.

## 2. Final Work Status

**Question:** What final Work status should be stored after review approval?

**Official Decision:** Add a final Work status: `Completed`. After review approval, `work.status = Completed`. Review records remain the source of review history.

## 3. Slot Selection Unlock

**Question:** Does Slot Selection unlock after First Work submission or after First Work review approval?

**Official Decision:** Slot Selection unlocks only after First Work is approved by review. Submission alone does not unlock slots.

## 4. Setup Verification Failure

**Question:** Does setup verification failure change worker status or only future eligibility?

**Official Decision:** Worker remains Active. Setup verification creates an eligibility blocker only. Current work never stops. Only future work assignments are blocked until verification is corrected.

## 5. Instagram Account Status

**Question:** Should a separate `Verified` Instagram Account status be introduced?

**Official Decision:** Active means fully verified and usable. Do not introduce a separate Verified status.

## 6. Extra Work

**Question:** Should Extra Work references be implemented in v1.0?

**Official Decision:** Extra Work is out of scope for IWMS v1.0. Ignore all Extra Work references during implementation.

## 7. Admin Roles

**Question:** What admin role model should v1.0 implement?

**Official Decision:** Implement a permission-based role system. Minimum roles: Super Admin, Review Admin, Payroll Admin, Support Admin, Operations Admin. Permissions must drive visibility and actions. Super Admin always has full access.

## 8. Technical Logs

**Question:** Should Technical Logs be persisted in Bot storage?

**Official Decision:** Technical Logs are Log Channel messages only. Do not persist Technical Logs in Bot storage. Audit Trail and Timeline remain permanent records.

## 9. Environment Configuration

**Question:** Should sample IDs from the reference bot be kept?

**Official Decision:** Never keep sample IDs from the reference bot. Replace every environment-specific value with placeholders.

## 10. Screenshot Storage

**Question:** Should screenshot submissions be nested under Work or stored as separate submission resources?

**Official Decision:** Store submissions nested under the Work record. Do not create a separate submission resource in v1.0.

## 11. Wallet Ledger Storage

**Question:** Should wallet ledger entries be stored as one array per worker or per-entry keys?

**Official Decision:** Store one append-only ledger array per worker. Never mutate historical entries.

## 12. Waiting States

**Question:** Which waiting states have automatic SLA or reminder behavior in v1.0?

**Official Decision:** The following have no automatic SLA in v1.0: Review Queue, Investigation, Setup Verification, Withdrawal Verification, Appeal Review. Only Support and Bug Reports use reminder-based automation.

## 13. Link Resources

**Question:** Should separate Link resources exist?

**Official Decision:** Do not create separate Link resources. Store links as versioned fields only.

## 14. Bug Report affected_area Enum

**Question:** What values are valid for `bug_report.affected_area`?

**Official Decision:** Use: Onboarding, Training, Work Assignment, Screenshot Submission, Review, Wallet, Withdrawal, Payroll, Support, Appeals, Admin Dashboard, Other.

## 15. Permanent Delete

**Question:** What remains after Permanent Delete?

**Official Decision:** Permanent Delete removes operational worker data. The following remain permanently: Minimum Identity, Audit Trail, Wallet Ledger, Financial Records, Appeal History. Operational resources are removed/anonymized according to policy.

