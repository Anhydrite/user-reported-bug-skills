---
name: user-reported-bug
description: Enforce RED-TEST-FIRST when the user reports a bug. Auto-loads on phrases like "j'ai un bug", "ça ne marche pas", "I have a bug", "c'est cassé", "ça plante", "ça échoue", "this is broken", "ça foire", "ça bug", "broken", "ça merde", or any user statement that an action the system should do, doesn't do (or does wrong). Forces a strict workflow: (1) reproduce the bug, (2) write a RED test that captures the symptom, (3) tag it USER_FOUND, (4) add it to docs/testing/user-found-bugs.md, (5) ONLY THEN propose a fix. Reference case: the 2026-06-06 QEMU bug which had existed undetected because the salve didn't include QEMU personas.
---

# User-Reported Bug Workflow (RED-TEST-FIRST)

When the user reports a bug, follow this exact sequence. **Do not skip steps, do not reorder them.** Each step produces evidence. The red test is non-negotiable.

## Why this skill exists

On 2026-06-06 the user reported "I can't make QEMU VMs". Investigation showed:

- A test suite existed with 24 personas and a salve of 4
- The salve was passing
- The QEMU bug had been latent for some time
- The salve never executed a QEMU boot end-to-end
- The user was, in effect, a 25th persona with unique visibility

**The lesson:** automated tests cannot replace human-in-the-loop bug reports. When a user reports a bug, treat it as a critical signal. Convert it to a permanent guard test BEFORE fixing.

## The 5 principles (from docs/testing.md)

These principles are derived from the QEMU case and are non-negotiable:

1. **Test = proof of existence, not coverage.** A passing test is evidence that ONE path didn't fail. Always check the coverage matrix.
2. **Symptom > path.** Test the symptom (e.g., "VM should reach state=running"), not the path (e.g., "TAP creation works"). One symptom test catches many path bugs.
3. **Persona = vector of risk.** Each persona covers a risk class. Stratify the salve to guarantee coverage.
4. **Async bug = its own class.** For async actions, the test must wait for terminal state and assert it strongly. Half-success is failure.
5. **The user is a persona.** Real user bug reports are a unique signal. Convert them to guard tests.

## The workflow

### Step 1: Acknowledge and reproduce

Don't immediately jump to fixing. First, acknowledge the bug, then **reproduce it yourself** in a controlled way. This gives you:
- A concrete understanding of the failure mode
- The exact error message (which becomes the test assertion)
- A way to validate the fix later

Use a sub-agent or quick script. If you can't reproduce it, ask the user for more details. Do NOT proceed without reproduction.

### Step 2: Write the RED test

Create a test file (or add to an existing one) that asserts the **expected** behavior. The test should fail (red) because the bug is unfixed.

Template:

```go
// USER_FOUND: <one-line description of the bug>
// Date: <YYYY-MM-DD>
// Reporter: <name/handle>
// Reference: docs/testing/user-found-bugs.md#<entry-number>
//
// <2-3 lines on what the user reported and how to reproduce.>
// This test asserts the EXPECTED behavior. It will fail (red) until
// the bug is fixed. After the fix, this test stays in the suite as
// a permanent guard.

func TestUserFound_<NNN>_<ShortName>(t *testing.T) {
    // Setup: connect to API, login, cleanup prior state
    // ...
    
    // Reproduce: do the action that triggers the bug
    // ...
    
    // Assertion: the action should reach a healthy state
    if state != "expected_state" {
        t.Errorf("BUG REPRODUCED: ... (got %q, want %q)", state, "expected_state")
    }
}
```

**Test placement rules:**

- If the bug is in a specific feature: add to `tests/personas/<feature>_test.go`
- If the bug is a cross-cutting invariant (like "all backends should boot"): add to `tests/personas/boot_invariant_test.go` or create a new `<invariant>_test.go`
- Use the same patterns as the QEMU red tests in `tests/personas/qemu_bug_test.go` (model file)

### Step 3: Run the test, confirm RED

Run the test in isolation:

```bash
go test -count=1 -timeout=120s -v -run "TestUserFound_<NNN>" ./tests/personas/
```

**The test MUST fail.** If it passes, either:
- The bug is not reproducible (go back to Step 1)
- Your test doesn't actually capture the symptom (rewrite it)
- The bug was already fixed (in which case, you're done — but document what happened)

### Step 4: Tag the test and update the ledger

In the test file, add the `// USER_FOUND` comment block (see template above).

Update `docs/testing/user-found-bugs.md` with a new row in the entries table:

```markdown
| <next-number> | <YYYY-MM-DD> | <reporter> | <symptom> | `<test-file-path>` | 🔴 OPEN — fix pending |
```

### Step 5: NOW propose a fix

Only after steps 1-4 are complete. Use systematic-debugging skill to:
- Characterize the bug's root cause
- Propose a minimal fix
- Verify the fix makes the red test pass (green)
- Verify no other tests regressed
- Run a stratified salve to verify coverage is still good

### Step 6: Update the ledger

After the fix:
- Update the entry: status becomes "✅ FIXED — <commit-hash>"
- Add the root cause section if useful for future readers

## What NOT to do

❌ **Fix before the red test exists.** The fix without a test is a hypothesis, not a solution. The next regression will be invisible.

❌ **Skip the reproduction step.** You can't write a red test without understanding the symptom. "I think the bug is X" is not enough — you must SEE it fail.

❌ **Add a test that always passes.** A test that doesn't fail for a buggy system is worse than no test — it gives false confidence.

❌ **Remove the red test after fixing.** The test is now a guard. It should fail if the bug regresses. Removing it removes the protection.

❌ **Mark the bug fixed without re-running the red test.** The fix is unverified. Run the test, see it pass, then mark it fixed.

❌ **Skip the ledger.** The ledger is the institutional memory. Future devs (and future AIs) need to know which bugs were real, when, and how they were fixed.

## Reference materials

- `docs/testing.md` — Section "Testing Principles" (5 principles)
- `docs/testing/user-found-bugs.md` — Ledger of user-reported bugs
- `tests/personas/qemu_bug_test.go` — Model red test (QEMU case)
- `tests/personas/boot_invariant_test.go` — Symptom-based invariant test
- `tests/personas/stratified.go` — Stratified salve (coverage guarantee)

## Quick checklist (for the AI)

When the user reports a bug, work through this list:

- [ ] Did I acknowledge the bug without rushing to fix?
- [ ] Did I reproduce it with a concrete reproduction (curl, script, sub-agent)?
- [ ] Did I capture the exact error message (verbatim)?
- [ ] Did I write a red test that ASSERTS EXPECTED behavior?
- [ ] Did I run the test and confirm it FAILS (red)?
- [ ] Did I add the `// USER_FOUND` comment block?
- [ ] Did I add a row to `docs/testing/user-found-bugs.md`?
- [ ] Did I check the coverage matrix to see if the bug would have been caught by existing tests? (If yes, why wasn't it? If no, update the matrix.)
- [ ] Did I defer the fix until the user approves proceeding?

If all boxes are checked, you've followed the workflow correctly. If any box is unchecked, stop and complete it.
