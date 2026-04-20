# Pat Testing Flow — Full Detail

**KB Type:** HR (ab_)
**Maintained by:** Karen

This article expands on the condensed testing flow summary in `agent.md`. It is the authoritative reference for Pat's full testing procedure for the Chalk project.

---

## Full 10-Step Testing Flow

1. **PM spawns FRESH Pat + FRESH Karen** — Karen monitors Pat throughout the entire cycle.
2. **Pat runs existing test suite** — baseline verification to confirm no pre-existing failures before new tests are authored.
3. **Pat authors new tests per IOC spec** — write unit, integration, and regression tests that cover the IOC's requirements and pass criteria.
4. **Pat runs full test suite** — all tests, not just the new ones.
5. **If tests fail and methods are insufficient** — Pat + Carl research new approaches; document findings for future reference.
6. **Pat verifies pass criteria:**
   - Tests pass: `.venv/bin/python -m pytest`
   - Type checking passes: `python -m mypy src/`
   - No regression: full test suite passes without new failures
   - Visual verification (where applicable): see Visual Testing Procedure below
   - Manual verification per IOC spec
7. **Pat performs visual verification** — launch app with offscreen platform or real display, capture screenshots, verify UI renders correctly. See visual testing procedure below.
8. **Pat performs manual verification** per IOC spec requirements.
9. **On failure:** Pat creates or updates `<feature>_fix_plan.md` in `specs/<module>/` documenting what failed, how it failed, and why (if known). Include screenshot evidence for visual bugs. Report to PM.
10. **On success:** Pat updates fix_plan with the solution that worked (if a fix_plan exists) and marks it resolved. Pat spawns Carl so Carl can update KB articles with learned knowledge. Pat reports success to PM. Karen captures any learned behavioral patterns.

---

## Test Runner Commands

**Critical:** always use the venv Python, not the system Python. PyQt6 and pytest-qt are installed in `.venv/`, not on the system interpreter.

| Task | Command |
|------|---------|
| Run all tests | `.venv/bin/python -m pytest` |
| Run one test (verbose) | `.venv/bin/python -m pytest tests/path/to/test_file.py::TestClass::test_method -v` |
| Run with coverage | `.venv/bin/python -m pytest --cov` |
| Type checking | `python -m mypy src/` (system Python — mypy is NOT in venv) |

Running `python -m pytest` (system Python) instead of `.venv/bin/python -m pytest` will fail with `ModuleNotFoundError: No module named 'PyQt6'` because the system Python does not have PyQt6 installed (PEP 668 prevents pip install to system Python on Ubuntu 23.04+).

See `kb/chalk/testing_setup_kb.md` for venv setup instructions.

---

## Visual Testing Procedure

Run these steps whenever a UI element, panel layout, or color-coded behavior needs visual confirmation.

1. **Ensure the venv is set up** (see `testing_setup_kb.md`).

2. **Start Xvfb virtual display:**
   ```bash
   Xvfb :99 -screen 0 1280x800x24 &
   ```

3. **Run the app with virtual display:**
   ```bash
   DISPLAY=:99 .venv/bin/python -m src
   ```

4. **Capture screenshot:**
   ```bash
   DISPLAY=:99 scrot testshots/<category>/<name>.png
   ```
   Screenshot categories: `ui/`, `toolbar/`, `dialogs/`, `bugs/`

5. **Terminate processes:**
   ```bash
   pkill -f "python -m src"; pkill Xvfb
   ```

6. **Review screenshot** — verify the expected visual state. For bugs, compare against a known-good reference if available (ImageMagick `compare` command).

For headless automated tests, pytest-qt uses `QT_QPA_PLATFORM=offscreen` automatically — no Xvfb needed for test runs. Xvfb is only for full-app visual inspection.

---

## Pass Criteria Reference

| Criterion | Verification Command / Method |
|-----------|-------------------------------|
| Tests pass | `.venv/bin/python -m pytest` |
| Type checking passes | `python -m mypy src/` |
| No regression | Full test suite passes |
| Visual verification | Launch app → screenshot → verify (where applicable) |
| Manual verification | Per IOC spec requirements |

---

*See also: `agent.md` Pat section for the summary list. `kb/chalk/testing_setup_kb.md` for venv setup.*
