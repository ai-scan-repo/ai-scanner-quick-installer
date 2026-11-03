# Changelog

This file records notable changes to the installer, including who made them
and why. New entries should be added at the top, directly below this
introduction.

## 2026-11-03 — Aviel

- **Changed:** The GitHub **Check access** request now includes the installer
  repository currently entered in the wizard. The API checks that submitted
  value before falling back to the saved value.
- **Why:** The wizard previously sent only the GitHub owner and token. The
  backend therefore checked the last saved installer repository, which could
  differ from the value currently entered (for example,
  `ai-scan-repo/ai-scanner-quick-installer`). This could make the access check
  misleading and leave the user unsure which repository would be used.
- **Files changed:** `ui/src/App.tsx`, `ui/src/api.ts`, `installer/app.py`,
  `ui/src/Help.tsx`, and `tests/test_backend.py`.
- **Verification:** Added a regression test that submits a new repository while
  an older repository is saved, and checks that the submitted repository is
  used. The test could not be run in the available environment because
  `pytest` is not installed.

## Adding a changelog entry

1. Add the new entry immediately below this introduction and above older
   entries, so the latest change appears first.
2. Use a heading with the date and contributor, in this format:
   `## YYYY-MM-DD — Name`.
3. Briefly describe what changed and why the change was made. Include relevant
   verification or known limitations when useful.
4. Keep entries focused on notable changes; do not record every small edit.
