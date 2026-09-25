---
name: bug-finder
description: >-
  Find the actual defect from a symptom: reproduce, form competing hypotheses,
  disprove them with evidence, and isolate root cause before proposing a fix.
  Use when something is wrong, a test fails, a user reports a bug, logs show an
  error, a regression appeared, or the user runs /bug-finder or "why is this broken".
---

# Bug Finder

You are hunting a specific defect. You do not "take a look around". You do not fix until you can name the line that makes the symptom inevitable.

## Hard rules

- No hypothesis, no edit. The first output after intake is a short hypothesis list, not a patch.
- Do not shotgun: no drive-by refactors, no dependency upgrades, no "while I'm here" null checks.
- Do not treat a crash as the root cause. Walk to the first wrong state.
- Do not add `try/catch`, a retry, or a default value to hide the symptom unless the user asked for a mitigation and you said the cause is still open.
- If you cannot reproduce and cannot get a trace, say what evidence you need. Do not invent a cause from the file names.

## Intake

Collect, in order, whatever exists. Do not restart the user through a form if they already pasted it.

1. **Symptom** — exact output, status, screenshot, or user quote. "It doesn't work" is not a symptom.
2. **Expected** — what should have happened, with the input that should produce it.
3. **Since when** — last good revision if known (`git log` / bisect later).
4. **Surface** — URL, job name, test name, query, CLI args.
5. **Evidence on disk** — stack trace, failing test, log line, screenshot, metrics, video.

If the workspace already has a failing test or a red command in the terminal, that is the reproduction. Start there.

## Reproduce

Prefer a command you can re-run.

1. Run the existing failing test or the smallest script that hits the path.
2. If there is no test, write one that fails for the right reason (the symptom), or a single curl/REPL/SQL invocation.
3. Record: command, input, observed output, exit code.
4. If it does not fail locally, say so. Next: environment drift (data, flags, clock, permissions, version). Do not "fix" a bug you cannot see unless the user wants a code-inspection pass only.

Code-inspection-only mode (user said "don't run" or there is no runtime): treat every claim as a hypothesis and mark it `unconfirmed`.

## Hypothesize

Write 2–4 hypotheses. Each must be falsifiable.

```text
H1: <claim about a specific function/state>
    disprove if: <observation>
H2: ...
```

Good: "H1: `invoice.total` is cents but `formatMoney` assumes dollars — disprove if a logged total of 1099 renders as 1099.00 not 10.99."
Bad: "H1: maybe a race somewhere."

Rank by how cheap they are to check, not by how clever they sound.

## Disprove

For each hypothesis, gather one piece of evidence. Tools, in preferred order:

1. Read the exact function on the stack / in the failing assertion.
2. Print or log the intermediate value if reading is ambiguous (then remove the probe).
3. `git log -L` / `git blame` on the suspect line; `git bisect` if it is a regression and the range is known.
4. Search callers and the data that feeds them. The bug is often one layer up from the throw.

Discard hypotheses you disproved. Do not keep them in the final report.

When two remain, design the cheapest observation that splits them.

## Root cause

You may name a root cause only when all of these are true:

- You can point at the code or data that is wrong (`path:line` or row/id).
- You can explain the mechanism in causal order (input → wrong state → symptom).
- A fix at that place would make the reproduction pass without hiding other failures.

If the first wrong state is bad data from the past, the root cause is the writer (or the missing invariant), not the reader that finally threw.

## Fix (only after root cause, and only if asked or clearly implied)

1. Change the cause, not the symptom.
2. Add or adjust a test that fails without the fix and passes with it. The test should express the user-visible rule, not the internal helper.
3. Check the same invariant at sibling call sites. Fix those in the same change if they are the same bug. Do not "harden" unrelated paths.
4. Re-run the reproduction and the nearest test package.

Do not silently change API contracts to make a test pass.

## Output

```markdown
## Symptom
<observed vs expected, including command if you ran one>

## Root cause
<mechanism, `path:line`, why this input hits it>

## Evidence
- <what you ran or read, and what it showed>
- <hypotheses discarded, and why>

## Fix
<patch approach, or "not applied — cause unconfirmed">

## Residual
<what could still be wrong: other platforms, old rows, races you could not prove>
```

If you applied a fix, add the test command and result.

## Special cases

- **Flake** — do not "fix" by raising a timeout. Find the shared state, clock, or unordered set. If you cannot, say it is unproven and isolate the test.
- **Heisenbug / race** — add synchronization or make the order explicit in production code; a sleep is not a fix.
- **Cannot repro** — inspect recent diffs on the path, list the top 2 remaining hypotheses, and the exact probe the user (or you, next) should run.
- **Multiple bugs** — finish the one that matches the reported symptom first. File the others as separate findings, do not entangle the patch.

## Do not

- Blame the user, the framework, or "eventual consistency" without a trace.
- Paste a theory of the whole architecture.
- Close with "this should work now" without the reproduction result.
