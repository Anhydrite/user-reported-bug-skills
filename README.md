# user-reported-bug

OpenCode skill that enforces red-test-first when you report a bug. Instead of jumping to a fix, the agent reproduces the bug, writes a failing test that captures the symptom, logs it in a ledger, and only then proposes a fix.

## Install

Clone this repo directly into your OpenCode skills directory:

```bash
git clone https://github.com/Anhydrite/user-reported-bug-skills.git ~/.config/opencode/skills/user-reported-bug
```

Restart OpenCode. The skill triggers automatically when you report a bug (e.g. "j'ai un bug", "this is broken", "ça ne marche pas").

## What it enforces

- **Reproduce first.** The agent must see the bug fail before writing any test.
- **Red test before fix.** A failing test that asserts expected behavior is written and confirmed red.
- **USER_FOUND tag.** Every bug gets a comment block and a row in the ledger (`docs/testing/user-found-bugs.md`).
- **Fix only after the test exists.** The fix is verified by the red test turning green.
- **Guard test stays.** The test remains in the suite to catch regressions.

## Workflow

1. **Acknowledge & reproduce** — agent reproduces the bug in a controlled way
2. **Write the red test** — asserts expected behavior, fails because bug is unfixed
3. **Confirm red** — test is run, must fail
4. **Tag & ledger** — `// USER_FOUND` comment + entry in `docs/testing/user-found-bugs.md`
5. **Fix** — minimal fix, red test turns green, no regressions
6. **Update ledger** — status becomes FIXED with commit hash
