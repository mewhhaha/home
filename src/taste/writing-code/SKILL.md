---
name: writing-code
description: Opinionated standards for writing code that reads well, changes safely, and survives review. Load before writing or modifying any production code. Written to be followed by any capable model (Claude, GPT-5.6+); every rule is stated once, concretely, with the failure mode it prevents.
---

# Writing code

You are writing code that another person will read, review, and modify under
time pressure. Optimize for that reader, not for the moment of writing.

Everything below is one taste, not a menu. The rules reinforce each other:
small diffs make review possible, good names make comments unnecessary, edge
handling at boundaries keeps interiors simple. When two rules seem to
conflict, the precedence order resolves it.

## Precedence

When guidance conflicts, apply in this order:

1. An explicit instruction from the user in this conversation.
2. The surrounding codebase's established convention.
3. This document.

"Established convention" means the pattern appears consistently in the files
you are touching — not a single stray example. Never "fix" the codebase's
style to match this document unless asked.

## Before writing anything

Read the code you are about to change, plus at least one neighboring file
that does something similar. You are looking for four things:

- How errors are handled here (exceptions? result types? logged and ignored?)
- How things are named (verbosity, casing, domain vocabulary)
- What the project already depends on (never add what an existing dep covers)
- Whether the thing you're about to write already exists

New code must be indistinguishable in style from the code around it. If the
file uses long descriptive names and yours are terse, yours are wrong here —
even if they'd be right elsewhere.

If the task is ambiguous, resolve ambiguity by reading more code, not by
guessing or by asking questions the codebase can answer. Ask the user only
when the decision is genuinely theirs: product behavior, irreversible
migrations, tradeoffs with no codebase precedent.

## Scope: a diff is one idea

Change the minimum that accomplishes the task, completely.

- No drive-by refactors, no reformatting lines you didn't otherwise touch, no
  renaming things "while you're here." Each of those is a separate diff.
- "Completely" cuts the other way too: if the fix requires updating a caller,
  a test, and a type, all three belong in this change. Don't leave the
  codebase in a state where your change is half-true.
- If you notice unrelated problems, report them in your summary; don't fix
  them silently.

Test of success: a reviewer can state the entire diff's purpose in one
sentence, and every hunk visibly serves that sentence.

## Naming

Names are the primary documentation. Spend real effort here.

- Name things for their meaning in the domain, not their shape or mechanics.
  `overdueInvoices`, not `filteredList`. `retryDelay`, not `num2`.
- A function name is a promise about behavior. `getUser` must not create
  users. `validate` must not mutate. If you can't name it honestly in a few
  words, the function does too much — split it.
- Banned as names or name-suffixes, because they mean "I didn't decide what
  this is": `data`, `info`, `item`, `manager`, `handler` (outside genuine
  event handlers), `util`, `helper`, `misc`, `process`, `doStuff`.
- Same concept, same word, everywhere. If the codebase says `account`, do not
  introduce `user` for the same entity. If two names exist for one concept,
  you have found a bug in the vocabulary; use the dominant one.
- Length proportional to scope: `i` is fine for a three-line loop; a module
  export gets a name that survives being read with zero context.

## Control flow

The happy path reads top to bottom at one indentation level.

- Handle preconditions and edge cases first with early returns, then write
  the main logic unindented. Nesting depth is a defect count.
- No boolean parameters on public functions — `render(true, false)` is
  unreadable at the call site. Use two functions or a named options object.
- No cleverness that must be decoded. A reader should never need to simulate
  the code in their head to know what it does. If a one-liner needs a
  comment, it should be three lines instead.
- Make the common case fast to *read*, not just to run.

```ts
// No: the logic hides at depth 3
function ship(order: Order) {
  if (order.items.length > 0) {
    if (order.address) {
      // ... 30 lines ...
    } else {
      throw new Error("no address");
    }
  }
}

// Yes: preconditions first, then a flat body
function ship(order: Order) {
  if (order.items.length === 0) return;
  if (!order.address) throw new Error(`order ${order.id} has no address`);
  // ... 30 lines, unindented ...
}
```

## State and data

Push effects to the edges; keep the middle pure.

- Prefer functions that take data and return data. I/O, mutation, clocks,
  randomness, and globals live at the entry points, injected inward — never
  reached for from the middle of business logic.
- Make illegal states unrepresentable. A type that permits an invalid
  combination will eventually hold one. Prefer
  `{ status: "sent", sentAt: Date } | { status: "draft" }` over
  `{ status: string; sentAt?: Date }`.
- Parse, don't validate: convert untrusted input into a trusted, precisely
  typed value once, at the boundary. Past that point, the type is the proof
  and interior code performs no re-checking.
- Don't mutate arguments. Don't share mutable state between functions when a
  return value would do.

## Errors

- Fail loudly, early, and with the evidence attached. An error message must
  contain the values that made it fire: `` `config ${path} missing key
  "region"` ``, never `"invalid config"`. The person reading it is debugging
  at 2am with no context.
- Never swallow an error you can't genuinely handle. `catch (e) {}` and
  `catch (e) { log(e) }` followed by continuing as if it succeeded are the
  worst code you can write: they convert a crash into silent corruption.
  Handle it, enrich-and-rethrow it, or let it propagate.
- Defensive checks belong at trust boundaries (user input, network, disk,
  cross-service calls) — not scattered through interior code that only
  receives already-validated values. An interior null-check for a value that
  can't be null is a lie about the invariants; it hides real bugs.
- Distinguish expected failures (user typo, not-found → typed result, clear
  message) from bugs (violated invariant → throw/assert, crash honestly).

## Comments

A comment states only what the code cannot: the *why*, the constraint, the
external fact.

Write a comment when there is a non-obvious reason, a workaround for a
specific bug (link it), a performance constraint, a spec requirement, or an
invariant the type system can't express. Otherwise don't.

Never write comments that narrate the code (`// increment the counter`),
restate the function name, describe your change relative to the previous
version ("now uses the new API"), or address the reviewer ("this fixes the
issue"). Those are stale the day they merge. If you feel the urge to narrate,
the code is unclear — fix the code.

Delete commented-out code on sight. Version control remembers.

## Functions and abstraction

Duplication is cheaper than the wrong abstraction.

- Don't extract a helper for one caller. Don't build for imagined future
  needs — YAGNI is nearly always right, and speculative generality is the
  most expensive way to be wrong.
- Extract on the *third* occurrence, and only when the copies must change
  together for the same reason. Two similar-looking blocks that evolve
  independently are not duplication; forcing them through one function
  couples unrelated things and breeds boolean parameters.
- If an existing abstraction fights you — you're passing flags to skip parts
  of it, or reaching around it — inline it back into its callers and re-split
  along the real seam. Never grow a wrong abstraction to fit one more case.
- A function does one thing at one level of altitude: either it orchestrates
  (calls a few well-named steps) or it does the work — not both interleaved.

## Dependencies

Every dependency is code you now maintain but can't fix.

- Before adding one: does the stdlib do it? Does an existing project dep do
  it? Is it under ~50 lines to write inline? Any yes means no new dependency.
- A new dependency is justified for genuinely hard domains — crypto, parsing
  standardized formats, timezone math — where a home-grown version would be
  subtly wrong. It is not justified to save 20 lines of glue.
- When you do add one, add exactly one, pinned, and say why in the summary.

## Tests

Tests exist to catch regressions and to document behavior. Judge every test
by: *what bug would this catch, and what does its failure message tell me?*

- Test observable behavior through public interfaces. Never assert on
  internals (private state, call counts of your own code, exact log strings)
  — those tests break on every refactor while catching nothing.
- Each test: one scenario, named as a factual sentence about behavior.
  `expires session after 30 minutes of inactivity`, not `test session 2`.
- Arrange the interesting inputs; assert the meaningful output. Prefer real
  objects; fake only true boundaries (network, clock, disk). Don't mock what
  you own — if constructing your own objects for a test is painful, that
  pain is design feedback.
- Cover the edges the code actually claims to handle: empty, one, many,
  boundary value, malformed input, the error path. Skip permutations that
  exercise the same branch twice.
- A test you had to make pass by copying the implementation's logic into the
  expectation verifies nothing. Compute expected values independently or use
  known-good fixtures.

## Verification is part of writing

Code is not done when it's written; it's done when you've watched it work.

- Run the project's own checks (typecheck, lint, tests) and run or exercise
  the specific path you changed. "It compiles" is not verification.
- Report results honestly and specifically: which commands ran, what passed,
  what you could not verify and why. Never imply verification that didn't
  happen — an unverified change labeled as such is useful; the same change
  labeled "done" is a trap for whoever ships it.
- If tests fail for reasons unrelated to your change, say so with evidence
  (show they fail on the base too); don't silently "fix" them into passing.

## Deleting

The best change is often removal. When a task makes code unreachable, delete
it in full — the function, its tests, its exports, its imports. Dead code is
not "kept just in case"; it's noise every future reader pays for. Prefer the
solution that ends with fewer concepts in the codebase, even at equal line
count.

## Communicating the change

- Summaries lead with what changed and why, in plain sentences a teammate who
  didn't watch you work can follow. State assumptions you made and anything
  you noticed but deliberately didn't touch.
- Commit messages: one line stating the change's intent (what/why), not its
  mechanics — the diff already shows the mechanics.
- Keep it short by *omitting the unimportant*, never by compressing the
  important into fragments, arrows, or jargon.

## Self-review before finishing

Reread your full diff as the reviewer. Confirm each of these; fix, don't
rationalize, any miss:

1. One sentence states the whole diff's purpose, and every hunk serves it.
2. The new code is stylistically indistinguishable from its surroundings.
3. Every name is honest and domain-meaningful; none from the banned list.
4. No comment narrates code or addresses the reviewer; every remaining
   comment states a why the code can't.
5. Errors carry evidence; none are silently swallowed.
6. No abstraction with a single caller; no boolean parameter on a public
   function; no speculative flexibility.
7. No new dependency an existing one or ~50 lines could replace.
8. Tests assert behavior, would actually fail if the code regressed, and
   their names describe the scenario.
9. You ran the checks and the changed path, and your summary states exactly
   what was and wasn't verified.
10. Nothing is left half-migrated, commented out, or "TODO: remove".
