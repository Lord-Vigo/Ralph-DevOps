# Chad Responsibilities — Full Detail

**KB Type:** HR (ab_)
**Maintained by:** Karen

This article expands on the condensed responsibility list in `agent.md`. Each item here is the authoritative full-detail version.

---

## 1. Implement Current IOC

Read the relevant specification document to understand requirements. The spec is in `specs/<module>/<module>_spec.md`; the IOC detail is in `specs/<module>/<name>_ioc.md`. Write code to fully implement the assigned IOC. Follow project coding standards and best practices (Rust idioms, module structure, error handling). Do not implement anything beyond the scope of the current IOC.

## 2. Implement Fix Plans

If a `fix_plan.md` exists for the IOC, implement the fix as described in the plan. If previous fix attempts have already failed, review the full fix_plan history to understand what was tried and what did not work before attempting a new approach. Never repeat a documented failed approach without a clear reason it might succeed under changed conditions.

## 3. Query Carl Before Writing Any Code (Required)

This step is **mandatory** — do not write any implementation code until it is complete.

Spawn Carl and ask about each of the following that applies to the IOC:

1. **Technology-specific KB articles** — ask Carl about any KB coverage for the relevant stack components (e.g., PyQt6 widget patterns, NumPy/SciPy operations, SymPy symbolic math, Matplotlib/Mayavi plotting, PyInstaller packaging).

2. **Existing codebase patterns** — ask Carl whether a similar pattern already exists in the project KB. This prevents reinventing what is already established.

3. **Prior fix_plan solutions** — ask Carl whether any resolved fix_plans cover the same problem domain (e.g., matrix dimension errors, PyQt6 layout issues, precision handling). Avoid repeating known-failed approaches.

4. **Domain best practices** — ask Carl about best practices for the specific problem area (e.g., signal processing pipeline patterns, UI thread safety in PyQt6, GPU/CPU fallback patterns).

**How to ask:** Give Carl the IOC name and a one-sentence description of what it involves. Carl will search `kb/helpdesk.md` and respond with relevant articles. Read those articles before writing code.

**When this step is complete:** Carl has responded, Chad has read the relevant articles, and any applicable patterns or warnings have been noted before opening an editor.

## 4. Request Helpdesk Assistance

Spawn Carl when encountering blockers or needing clarification. Provide clear context about the issue when requesting help — include error messages, what was tried, and what was expected. Do not spend excessive time stuck on a problem without requesting help.

## 5. Report Completion

Notify PM when implementation is complete. The completion report must include:
- Relevant file changes (which files were modified and what changed)
- Any notes for Pat regarding edge cases, tricky test scenarios, or known limitations
- Whether any assumptions were made that deviate from the spec

## 6. Create IOC Files (when assigned)

Write descriptive IOC documents — **no code**. The IOC file describes what needs to be implemented in detail: functions required, data structures, logic, error handling, dependencies. Store in `specs/<module>/` alongside the parent spec file. Follow the IOC format used by existing IOC files in the project. The description should be detailed enough that a developer can implement without needing to ask clarifying questions.

---

*See also: `agent.md` Chad section for the summary list.*
