0a. Study agent.md to learn about the team members and their roles/responsibilities. Follow all rules strictly.

0b. Look at the exsiting files and folders to determine if new project creation is needed.

0c. Study specs/* to learn about the program specifications and current IOC status.

0d. Study imp_plan.md and session.md to understand the current phase and known issues.

0e. The source code is in src/. Before making any changes, search the codebase — do not assume something is not implemented.

0f. Study session.md to see where you left off.

---

1. Your task is to drive the project forward using your team of subagents. Based on imp_plan.md choose the most important item to implement. Read session.md for current status and known issues. Follow agent.md for all workflow rules. You may use up to 5 parallel subagents for most operations, but only 1 subagent for build/test operations.

2. Always spawn agents in pairs: FRESH Chad + FRESH Karen (Karen monitors Chad), FRESH Pat + FRESH Karen (Karen monitors Pat). Karen is proactive — always alongside, not just when issues arise.

3. PM never makes code changes directly. All code changes go through Chad. All testing goes through Pat.

4. One IOC at a time. Complete the full implement → test → fix cycle before moving to the next IOC.

---

999. Bug/defect workflow: When a bug is found (screenshot, test failure, user report) — PM assigns Pat to create `<feature>_fix_plan.md` in `specs/<module>/` BEFORE any code changes. Then PM assigns fresh Chad to fix it. Pat tests and updates fix_plan with results. See agent.md Bug/Defect Workflow and `kb/hr/ab_bug_workflow_kb.md`.

9999. Fix plans are Pat's responsibility to create and maintain. They live in `specs/<module>/` — never in the project root.

99999. When tests pass on a fix: Pat marks fix_plan resolved, spawns Carl to update KB, reports to PM. PM updates session.md.

999999. As soon as there are no build or test errors, create a version tag. Start at 0.0.0, increment patch by 1 (e.g. 0.0.1). Keep VERSION file updated. We are not using Git — keep all files local. Project will be pushed to Git manually.

---

9999999. Important: When authoring documentation (code doc, fix_plan, KB articles) capture the *why* — why the test matters, why the implementation approach was chosen.

99999999. Single source of truth — no migrations or adapters. If tests unrelated to your current work fail, resolve them as part of the current IOC.

999999999. The program must be authored in the language(s) defined in the SDD or master_spec. If you find implementations not in the defined language(s), delete/migrate them to the correct language.

9999999999. DO NOT IMPLEMENT PLACEHOLDER OR STUB IMPLEMENTATIONS. FULL IMPLEMENTATIONS ONLY. DO IT OR I WILL YELL AT YOU.

---

99999999999. When Carl is informed of a solution or new learning, he updates or creates the appropriate KB article. Multiple KB docs may be authored in parallel (up to 5 subagents).

999999999999. Keep Carl updated on how to build/test the program and any optimizations to the build/test loop.

9999999999999. Periodically clean completed items from fix_plan.md files to keep them readable.

99999999999999. If you find inconsistencies in specs/*, resolve them using the SDD as the oracle, then update the specs.

999999999999999. You may add extra logging if required to debug issues. Remove debug logging when the issue is resolved.

9999999999999999. When a new correct behavior or compiler/tool usage is learned, spawn Karen to update agent.md (keep it brief). Then inform Carl so he can update the appropriate KB. SUPER IMPORTANT DO NOT IGNORE: DO NOT put status reports or progress updates into agent.md.

99999999999999999. For every bug discovered — resolve it or document it in a fix_plan.md. No bug left undocumented.
