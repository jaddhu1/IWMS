# IWMS — Master Implementation Prompt v1.0
**Instructions for Codex (or any implementation agent) before writing any code.**

---

## 0. Mandatory Reading Order
Before generating any command, JSON, or code, read every file in this folder in the order given in `README.md`. Do not skip files. Do not begin implementation until all files are read.

### 0.1 Onboarding Process (Follow in This Order — Do Not Skip Phases)
1. **Read** — read every document completely, no code yet.
2. **Architecture Exam** — without reopening the documents, explain: Worker Journey, Admin Journey, Golden Rules, FSMs, Wallet Architecture, Review System, Payroll, TPY limitations. List anything unclear.
3. **Clarification** — ask all ambiguities in a single batch (not one-by-one). Wait for answers. Once answered, create `10_Clarifications_Log.md` recording every question and its final answer — treat it as binding project documentation, equal in authority to the other 10 files. Never re-ask a logged question unless the human explicitly changes that decision.
4. **Implementation Plan (Codex's own roadmap)** — produce module dependencies, build order, reusable libraries, expected commands, and estimated JSON structure. No code yet.
5. **Implementation Readiness Report** — before writing any TPY code, explicitly confirm: ✅ Documents read, ✅ Architecture understood, ✅ Questions resolved (via `10_Clarifications_Log.md`), ✅ Ready to implement. Only after this report is presented and approved does coding begin.
6. **Coding** — implementation begins, phase-wise per `05_Implementation_Plan_v1.0.md` §9.
7. **Session End Summary** — at the end of every working session: completed modules, pending modules, generated commands, remaining commands, known issues, next step. Never regenerate completed work.
8. **Milestone Freeze** — after each completed milestone, generate `Milestone_Report.md`: completed modules, files changed, commands generated, remaining work, known limitations. This is the resume point for any future session.

---

## 1. What You Are Building
A Telebot Creator (TPY) bot implementing the Instagram Workforce Management System exactly as specified in `01_Project_Vision_v1.0.md` through `05_Implementation_Plan_v1.0.md`. You are not designing the product — the product is already fully designed and frozen (`07_IWMS_v1.0_LOCK.md`). Your job is translation: architecture → working TPY commands.

---

## 2. Coding Rules
1. **No feature invention.** If a behavior is not specified in `02_Architecture_Freeze_v1.0.md` or `03_Data_Model_v1.0.md`, do not invent it — flag it as a question instead of guessing.
2. **No new enum values** without adding them to `04_Data_Dictionary_v1.0.md` in the same change. An enum value absent from the Data Dictionary is not valid for use.
3. **No architecture changes.** If implementation reveals that a locked rule is genuinely infeasible in TPY, stop and flag it — do not silently work around it by changing product behavior. Report it; a human decides the resolution (may become a v1.1 revision).
4. **Follow the command naming convention exactly** (`05_Implementation_Plan_v1.0.md`, §2): `{module}_{action}[_{qualifier}]`, lowercase, underscore-separated.
5. **Follow the folder/grouping structure** in `05_Implementation_Plan_v1.0.md`, §1.
6. **Wallet Ledger is append-only.** No command may ever write directly to a balance field. All financial mutation goes through the single `ledger_writer` utility pattern described in `05_Implementation_Plan_v1.0.md`, §8.
7. **Every state transition writes to Timeline and/or Audit** as specified per resource in `03_Data_Model_v1.0.md`. This is not optional cleanup — it is core behavior.
8. **Every command is wrapped in try/except**, per the Error Handling Standard (`05_Implementation_Plan_v1.0.md`, §6). No bare/unhandled command logic.
9. **Follow Development Phases in order** (`05_Implementation_Plan_v1.0.md`, §9). Do not build Phase 3 (Payroll) logic before Phase 1 (Worker Core Loop) is functional — dependencies matter.
10. **Never assume an undocumented TPY feature exists.** If a needed capability's TPY support is unconfirmed, check `09_TPY_Verification_Report.md` first. If it's marked ⚠ Pending or not listed, flag it rather than assuming it works.

---

## 3. TPY-Specific Rules
- Storage is `User.getData/saveData` (per-worker) and `Bot.getData/saveData` (shared/global) — confirmed to support nested objects and lists natively (see `09_TPY_Verification_Report.md`). Do not manually JSON-serialize.
- Scheduling is `Bot.runCommandAfter(delay, command, ...)` returning a job with an `id`; cancel via `Bot.cancelScheduledTask(job_id)`. There is no job queue, no priority system, no retry manager — only individually-managed delayed single-command executions. Store every returned job_id against the entity it belongs to.
- `while` loops are restricted (per Verification Sprint) — avoid unbounded loops; prefer scheduled re-invocation patterns instead.
- Telegram `file_id` is bot-token-scoped and does not transfer across bots. Never write migration logic that assumes automatic media transfer — always route through the admin-facing re-upload checklist described in Architecture Freeze v1.0.
- No confirmed atomic multi-write transaction support. Multi-step state changes must be written defensively (see Error Handling Standard, §6, financial-command extra rule) so a partial failure is always detectable via Audit, never silent.

---

## 4. Output Rules
- Implement **phase-wise, in the order given in `05_Implementation_Plan_v1.0.md`, §9** (Phase 1 → Phase 7). Do not attempt to generate the entire system in one uninterrupted pass.
- **However, the final deliverable is always a single, consolidated Telebot Creator JSON export** (`{"commands": [...]}`) containing all commands built so far — not one file per phase. After completing each phase, regenerate/update the single JSON export to include that phase's commands alongside all previous phases' commands.
- This ensures that if execution is interrupted mid-way (a real, recurring risk with large generation tasks), completed phases are not lost — the next session can resume from the last consolidated export rather than starting over.
- Final deliverable format matches `08_TelebotCreator_JSON_Sample.json` exactly — same top-level structure (`{"commands": [...]}`, each command object with `command`, `bot`, `admin_only`, `code`, `created_at`, `folder`, `last_updated`, `pinned`).
- Code inside each `code` field must be valid Python as accepted by TPY's command executor — no external imports beyond what TPY's built-in libraries expose.
- When uncertain whether a construct is valid TPY Python, prefer the patterns already confirmed working in the reference bot sample (`08_TelebotCreator_JSON_Sample.json`) over generic Python idioms.

---

## 5. JSON Generation Rules
- One command object per logical action — do not merge unrelated behaviors into a single command "for convenience."
- `folder` field must match the grouping in `05_Implementation_Plan_v1.0.md`, §1.
- `admin_only` must be set accurately per Golden Rule 14 (permission-scoping) — never default to `false` for admin-side commands.
- Every command that can fail must degrade gracefully per §2/§3 above — never leave a command that can silently do nothing on error.

---

## 6. When Something Is Unclear
Stop and ask, citing the specific document and section that is ambiguous or missing, rather than making a silent assumption. This is the single most important rule in this file.

## 7. Hard Stop Conditions
If you encounter any of the following during implementation, **STOP immediately, explain the issue, and wait for approval before continuing** — do not work around it silently:
- A TPY capability you need is not documented or verified in `09_TPY_Verification_Report.md`.
- A requirement from `02_Architecture_Freeze_v1.0.md` or `03_Data_Model_v1.0.md` appears infeasible in TPY.
- Two documents appear to contradict each other.
- A requirement is missing entirely (not specified anywhere).
