# Bug and Defect Workflow

**Maintained by**: Karen (HR Subagent)

This article documents the correct process for handling bugs and defects discovered at runtime, from visual reports (screenshots), user reports, crashes, or unexpected UI behavior. This workflow is distinct from the standard IOC test-fail cycle.

---

## When to Use This Workflow

Use this workflow any time a bug is discovered **outside of a normal IOC test cycle**, including:
- A screenshot reveals incorrect visual output
- A user or agent reports unexpected behavior at runtime
- A crash or error occurs in a running build
- A visual defect is observed during manual testing not tied to an active IOC

---

## Step-by-Step Process

### Step 1 — PM spawns Pat to create fix_plan FIRST

Before any code is touched, PM spawns Pat to document the bug in a fix_plan file.

- Pat creates `<feature>_fix_plan.md` in `specs/<module>/` (e.g., `specs/capture/capture_fix_plan.md`)
- **No code changes are made at this stage**
- The fix_plan must capture:
  - What was observed (description, screenshot reference if applicable)
  - Steps to reproduce (if known)
  - Suspected cause (if known)
  - Scope of impact

### Step 2 — PM spawns Chad (fresh) to implement the fix

- PM spawns a **fresh Chad** with the fix_plan as context
- Chad reads the fix_plan and implements the fix
- Chad must NOT be a reused/existing Chad instance (avoids context compaction)
- Chad reports completion to PM when done

### Step 3 — PM spawns Pat to test and verify

- Pat tests the fix against the reported bug
- Pat updates the fix_plan with test results

### Step 4a — On Pass

- Pat marks the fix_plan as resolved
- Pat spawns Carl and communicates the solution
- Carl updates relevant KB articles with learned knowledge
- PM updates session.md

### Step 4b — On Fail

- Pat updates the fix_plan with:
  - What was tried in this attempt
  - What did not work
  - Full history of all attempts (never delete prior attempt history)
- PM reassigns a fresh Chad with the updated fix_plan as context
- Repeat from Step 2

---

## fix_plan Naming Convention

| Scope | File Name | Example |
|-------|-----------|---------|
| UI bugs | `<specific_feature>_fix_plan.md` | `toolbar_filter_label_overlap_fix_plan.md` |
| Capture bugs | `<specific_feature>_fix_plan.md` | `capture_packet_drop_fix_plan.md` |
| Rendering bugs | `<specific_feature>_fix_plan.md` | `star_glow_rendering_fix_plan.md` |

- fix_plan files are stored in **`specs/<module>/`** — never in the project root
- **One bug per file**: Each fix_plan covers exactly one bug. Never combine multiple bugs in a single fix_plan file.
- **Descriptive filename**: `<feature>` must describe the specific bug, not the IOC or module generically. Bad: `fix_plan.md`, `ui_fix_plan.md`. Good: `toolbar_filter_label_overlap_fix_plan.md`.

---

## Rules Summary

| Rule | Details |
|------|---------|
| fix_plan before code | Pat must create fix_plan BEFORE Chad writes any fix code |
| PM does not write code | PM must never make code changes directly — all code changes go through Chad |
| Fresh Chad per fix | Each fix assignment uses a fresh Chad spawn |
| Capture history | fix_plan must retain history of all attempts, even failed ones |
| Karen spawned immediately | When a new correct behavior is identified, PM spawns Karen in the same session — not after the fact |
| agent.md updated same session | New correct behaviors must be captured in agent.md within the same session they are learned |

---

## Why This Matters

Skipping the fix_plan and having the PM write code directly leads to:
- No audit trail of what was tried and why
- Loss of institutional knowledge
- Future agents repeating failed approaches
- PM role drift (PM is a coordinator, not an implementer)

The fix_plan is a lightweight document that takes minutes to create and prevents hours of repeated debugging.

---

*Last updated: 2026-03-20 by Carl*
