# IWMS — Project Vision v1.0

## Objective
Build a Telegram-based Workforce Management System (IWMS) that manages a distributed workforce of Instagram content-posting workers — from onboarding through work assignment, review, payroll, and removal/appeal — with maximum worker simplicity and maximum administrative automation.

## Scope (V1)
- Platform: Telegram, built on Telebot Creator (TPY).
- Workforce type: Instagram reel-posting workers only.
- Single organization (no multi-tenant support in V1).

## Mission
- **Zero worker confusion** — every screen shows exactly one clear next action.
- **~90% administrative automation** — admins intervene only where genuine human judgment is required (review decisions, disciplinary actions, appeals).
- **Complete history preservation** — nothing important is ever silently lost or overwritten.
- **Configuration over hardcoding** — business rules (fees, weights, thresholds, reasons) are versioned policy data, not code.
- **Scalable to 1000+ active workers, 10+ administrators** without architectural redesign.

## Long-Term Direction (Not V1 Scope)
Architecture is intentionally designed so it *could* generalize to other workforce types (YouTube, Facebook, freelance, support, moderation, data entry) later — but V1 delivery is never slowed down to accommodate this. Any V1 decision that had to choose between "generic" and "Instagram-specific" chose specific, with the tradeoff explicitly flagged (see Architecture Freeze v1.0, §14 Out of Scope).

## Source of Truth
This Vision, together with `02_Architecture_Freeze_v1.0.md`, `03_Data_Model_v1.0.md`, and `04_Data_Dictionary_v1.0.md`, constitutes the complete and final product specification for V1. Nothing in this document is open for reinterpretation during implementation — see `07_IWMS_v1.0_LOCK.md`.
