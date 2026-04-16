# Pat Testing Flow — Full Detail

**KB Type:** HR (ab_)
**Maintained by:** Karen

This article expands on the condensed testing flow summary in `agent.md`. It is the authoritative reference for Pat's full testing procedure.

---

## Full 10-Step Testing Flow

1. **PM spawns FRESH Pat + FRESH Karen** — Karen monitors Pat throughout the entire cycle.
2. **Pat runs existing test suite** — baseline verification to confirm no pre-existing failures before new tests are authored.
3. **Pat authors new tests per IOC spec** — write unit, integration, and regression tests that cover the IOC's requirements and pass criteria.
4. **Pat runs full test suite** — all tests, not just the new ones.
5. **If tests fail and methods are insufficient** — Pat + Carl research new approaches; document findings for future reference.
6. **Pat verifies pass criteria:**
   - Build succeeds: build command per `master_spec.md`
   - Tests pass: test command per `master_spec.md`
   - Lint passes: lint command per `master_spec.md`
   - No regression: full test suite passes without new failures
7. **Pat performs visual verification** — build project, run with Xvfb virtual display, capture screenshots, verify UI renders correctly. See visual testing procedure below.
8. **Pat performs manual verification** per IOC spec requirements.
9. **On failure:** Pat creates or updates `<feature>_fix_plan.md` in `specs/<module>/` documenting what failed, how it failed, and why (if known). Include screenshot evidence for visual bugs. Report to PM.
10. **On success:** Pat updates fix_plan with the solution that worked (if a fix_plan exists) and marks it resolved. Pat spawns Carl so Carl can update KB articles with learned knowledge. Pat reports success to PM. Karen captures any learned behavioral patterns.

---

## Visual Testing Procedure

Run these steps whenever a UI element, panel layout, 3D rendering output, or color-coded behavior needs visual confirmation.

1. **Build project:**
   Use the build command defined in `master_spec.md` to produce the runnable binary or entry point.

2. **Start Xvfb virtual display:**
   ```bash
   Xvfb :99 -screen 0 1280x800x24 &
   ```

3. **Run with virtual display:**
   ```bash
   DISPLAY=:99 <run command per master_spec.md> &
   ```
   Use the run command defined in `master_spec.md`.

4. **Wait for startup and capture data** — wait 3–5 seconds for the render loop to initialize. If testing capture/connection rendering, start capture and wait an additional 5–8 seconds for connections to populate before taking screenshots.

5. **Capture screenshot:**
   ```bash
   DISPLAY=:99 scrot testshots/<category>/<name>.png
   ```
   Screenshot categories: `ui/`, `features/`, `layout/`, `bugs/`

6. **Terminate processes:**
   ```bash
   pkill <process-name per master_spec.md> && pkill Xvfb
   ```

7. **Review screenshot** — verify the expected visual state. For bugs, compare against a known-good reference if available (ImageMagick `compare` command).

See also: project-specific visual testing KB in `kb/<project-name>/` for extended procedures, checklist items, and ImageMagick comparison commands.

---

## Pass Criteria Reference

| Criterion | Verification Command / Method |
|-----------|-------------------------------|
| Build succeeds | Build command per `master_spec.md` |
| Tests pass | Test command per `master_spec.md` |
| Lint passes | Lint command per `master_spec.md` |
| No regression | Full test suite passes |
| Visual verification | Build → run with Xvfb → screenshot → verify |
| Manual verification | Per IOC spec requirements |

---

*See also: `agent.md` Pat section for the summary list.*
