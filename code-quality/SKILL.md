---
name: code-quality
description: >-
  Raise the quality of existing code against this repo's actual patterns:
  coupling, complexity, API shape, error handling, tests, and deletion of dead
  weight. Use when the user asks for a quality pass, refactor, cleanup, smell
  review, simplify this, make this maintainable, or runs /code-quality.
---

# Code Quality

You improve code that other people will edit under pressure. Quality is measured against this repository, not a blog post about clean code.

## Hard rules

- Open neighboring files before judging. Local consistency beats a generic idiom.
- Do not reformat, rename for taste, or "modernize" APIs that are already uniform in the package.
- Do not introduce an abstraction used once. Do not keep a wrong abstraction used twice.
- Do not change behavior in a quality pass unless you found a defect and said so.
- Do not add comments that restate the code. Comments are for non-obvious constraints (law, protocol, race, why not).
- If the file is generated or vendor-owned, leave it.

## Intake

1. Confirm scope: file, directory, PR diff, or "the code I just wrote".
2. Detect the language, test runner, linter, and module boundaries the repo already uses.
3. Read 2–3 peer modules that do a similar job. Those are the style guide.
4. Run the existing linter/formatter/typecheck only if they are already wired. Do not add a toolchain.

If the user said "quality" after a feature change, stay inside that change. Do not boil the rest of the module.

## What quality means here

Evaluate in this order. Stop when the next item would be taste.

### 1. Behavior is obvious from the names and the types

- A reader can predict side effects from the function name and signature.
- Booleans that encode modes (`isAdminOrNewUserAndNotRestricted`) become named types or small functions.
- Return types do not lie: `null` vs thrown vs `Result` follows the surrounding module, one style per layer.
- Public functions do not require reading the body to know the units, mutability, or lifetime of arguments.

Finding form: "A caller cannot know X from the signature; they have to read N."

### 2. Coupling is honest

- A module that imports half the app is not "shared utilities"; it is a knot. Split along existing domain lines, not along technical layers unless the repo is already layered that way.
- New dependencies go in the direction the repo already uses (e.g. UI → domain → data). Do not invert that for convenience.
- Pass the value you need, not the god object "for later".
- Duplication of 5 similar lines is cheaper than a wrong helper. Duplication of a business rule in two services is a defect.

### 3. Control flow is boring

- Flatten: early return over nested `if`. One level of callback/Promise depth unless the language forces more.
- No hidden `else` after `return`. No flags that only exist to remember which branch you were in.
- Loops that build maps/sets should look like the rest of the codebase (`for`, `reduce`, or comprehension — pick the local one).
- Regex, parser, or state machine logic gets a name and tests; it does not sit inline in a request handler.

### 4. Errors and edges are designed

- Fail at the boundary (HTTP, queue, CLI) with a type/code the rest of the system already uses.
- Do not catch-and-log-and-continue on a path that leaves data half-written.
- Timeouts, cancellation, and retries are explicit where I/O happens. Silent infinite wait is a quality bug.
- User input is validated once at the edge and trusted inward, or validated at the persistence boundary if that is the local pattern. Not both with different rules.

### 5. Size and deletion

- A function that does two jobs (orchestrate + compute, I/O + format, authz + query) should be split if each job already exists as a pattern nearby.
- Dead parameters, unused exports, commented-out blocks, and feature-flagged-off code with no owner get deleted — after you confirm they have no dynamic/reflective callers.
- "Flexibility" parameters that are always the same value are removed.

### 6. Tests that pin behavior

- Quality work on logic without a test is incomplete if the package already tests this layer.
- Add or adjust tests for the behavior you extracted, not for the private helpers you created.
- Delete tests that only assert mocks were called with the same structure as the implementation.

## Refactor method

1. Write or identify a characterization test if behavior is non-trivial and uncovered.
2. Change structure in small steps that keep tests green. Do not mix rename, move, and behavior fix in one step unless the file is tiny.
3. Keep the public surface stable unless the user asked to change the API.
4. Run the narrowest relevant tests, then the package tests if the change crosses files.

If you cannot run tests, say so and limit yourself to transformations you can reason about locally.

## Output

When reviewing (no edits asked):

```markdown
## Baseline
<what "good" looks like in this repo — 2–3 citations of existing files>

## Findings

### <id>: <title>
- Where: `path:line`
- Smell: <coupling | naming | control-flow | errors | dead-code | tests | api-shape>
- Cost: <who pays later, and when>
- Change: <smallest edit>
- Risk of the change: <behavior / callers>

## Leave alone
<what looks rough but matches the module, or is not worth the churn>
```

When editing, apply only findings you would take yourself in this PR. Summarize after:

```markdown
## What changed
## Behavior unchanged
## Tests run
```

## Do not

- Apply a new architecture (hexagonal, clean, "use cases") to a file that does not already live in one.
- Replace working code with a trendy API (`effect`, new state library, custom Result type) the repo does not use.
- Optimize without a profile or an algorithmic defect.
- Leave the file "halfway nicer". Finish the scoped pass or do not start it.
