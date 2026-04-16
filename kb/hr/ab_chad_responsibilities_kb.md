# Chad Responsibilities — Full Detail

**KB Type:** HR (ab_)
**Maintained by:** Karen

This article expands on the condensed responsibility list in `agent.md`. Each item here is the authoritative full-detail version.

---

## 1. Implement Current IOC

Read the relevant specification document to understand requirements. The spec is in `specs/<module>/<module>_spec.md`; the IOC detail is in `specs/<module>/<name>_ioc.md`. Write code to fully implement the assigned IOC. Follow project coding standards and best practices (Rust idioms, module structure, error handling). Do not implement anything beyond the scope of the current IOC.

## 2. Implement Fix Plans

If a `fix_plan.md` exists for the IOC, implement the fix as described in the plan. If previous fix attempts have already failed, review the full fix_plan history to understand what was tried and what did not work before attempting a new approach. Never repeat a documented failed approach without a clear reason it might succeed under changed conditions.

## 3. Use Knowledge Base

Before implementing any IOC, query Carl for relevant KB articles. At minimum, consult:
- The **usage/patterns KB** for the languages involved (e.g., `html5_kb.md`, `css_kb.md`, `javascript_kb.md`)
- The **best practices KB** for the languages involved (e.g., `html5_best_practices_kb.md`, `css_best_practices_kb.md`, `javascript_best_practices_kb.md`)
- The **security KB** (`web_security_kb.md`) for any IOC that handles dynamic data, user input, URLs, or external resources — run through the security checklist at the bottom of that article before marking the IOC complete
- Project-specific KBs relevant to the IOC (e.g., `content_system_kb.md` before implementing any gallery/content IOC)

Carl maintains `kb/helpdesk.md` as a searchable index. Use it.

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
