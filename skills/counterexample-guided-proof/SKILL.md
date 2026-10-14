---
name: counterexample-guided-proof
description: Use when Verus (or another verifier) rejects a loop invariant and the fix is not obvious. Extract the failing loop, find concrete counterexamples (Z3, cvc5, or the verifier's own counterexample tools), plug them back in to confirm they trigger the error, then fix the invariant and check the counterexamples are blocked.
---

# Counterexample-Guided Proof Repair

Don't guess a fix from the error message. Find a concrete state that breaks the
invariant, confirm it with the verifier, and change the invariant so that state is
ruled out. The running example below shows what each step looks like.

```rust
fn count_up(n: u32) -> (r: u32)
    requires n < 1000,
    ensures r == n,
{
    let mut i: u32 = 0;
    let mut j: u32 = 0;
    while i < n
        invariant i <= n, j <= 1000,
    {
        i = i + 1;
        j = j + 1;
    }
    j
}
// error: invariant not satisfied at end of loop body:  j <= 1000
```

## 1. Locate the invariant

With several errors, fix one per round. ExVerus goes in this order:
preconditions / index bounds / overflow, then "invariant not satisfied before
loop", then "at end of loop body", then assertions and postconditions. Errors that
aren't about a loop (type errors, termination, ...) don't need counterexamples:
fix them directly.

Pull the failing loop out into a harness that runs the body once: what holds
before (loop condition + invariants), the body, and what must hold after.

```rust
fn count_up_step(n: u32, i: u32, j: u32) {
    // before: condition + invariants
    assume(i < n && i <= n && j <= 1000);
    let i = i + 1;
    let j = j + 1;
    // after: invariants
    assert(i <= n);
    assert(j <= 1000);
}
```

## 2. Solve (e.g. with Z3)

Ask for states where "before" holds but "after" fails:

```python
import z3
n, i, j = z3.Ints("n i j")
s = z3.Solver()
for v in (n, i, j): s.add(0 <= v, v < 2**32)            # u32
s.add(i < n, i <= n, j <= 1000)                         # before
s.add(z3.Not(z3.And(i + 1 <= n, j + 1 <= 1000)))       # after fails
# enumerate a few models with blocking clauses
# -> {n: 1, i: 0, j: 1000}, {n: 2, i: 1, j: 1000}, ...
```

Z3 is not the only option. Pick what fits the task:

- **cvc5** (`pip install cvc5`): `from cvc5.pythonic import *` takes the same
  code as above. Try it when Z3 returns `unknown` or times out, especially with
  quantifiers or strings.
- **The verifier's own counterexamples**, when it gives concrete values you can
  act on. Run them yourself instead of writing a query:
  Lean `plausible` (random testing; the Plausible library, included with Mathlib),
  `bv_decide` (prints an assignment for bitvector goals), and `omega`, which only
  prints constraints, so treat its output as a hint;
  Dafny counterexample extraction (`--extract-counterexample`);
  SPARK `gnatprove --counterexamples=on`; Why3 counterexamples;
  Isabelle Nitpick / Quickcheck; Rocq QuickChick.
- **Running the real code** (tests, or Kani's concrete playback for Rust) gives
  reachable states only. That helps in step 4.

Whatever the source, still do step 3: confirm each counterexample before using it.

## 3. Get the counterexample

Plug the values into the harness (replace the `assume` with `let n = 1; let i = 0;
let j = 1000;`) and run the verifier. It is a real counterexample if the "before"
checks pass and an "after" check fails: here `assert(j <= 1000)` fails.
If a "before" check fails, or nothing fails, discard it.

## 4. Analyze

Can a real run reach `{n: 1, i: 0, j: 1000}`? No: `j == i` always holds in this
loop. An unreachable counterexample means the invariant is **too weak**:
strengthen it. A reachable one means the invariant is **wrong**: weaken or fix it.

## 5. Fix and re-verify

Write a few candidate fixes, not just one. For each, plug this round's
counterexamples into the new harness. Here, with `j == i` added, `{n: 1, i: 0, j: 1000}`
now fails a "before" check (`j == i` is false), so it is **blocked**. A
counterexample for a wrong invariant counts as fixed when nothing fails on it.

Pick a candidate that makes the whole file verify. Otherwise pick the one that blocks
the most counterexamples (if tied, the one with fewest verifier errors) and start
the next round from it.

Still failing? Repeat. Round 2 finds `{n: 1001, i: 1000, j: 1000}`. The loop
can't see `requires n < 1000`, so add `n < 1000` to the invariant. The file verifies.

## Budget: your call

ExVerus used fixed settings: 10 counterexamples per round, 5 candidate fixes per
round, and at most 10 rounds. Treat these as reference points and choose for the
task at hand: more counterexamples when the pattern is unclear, more candidates
when fixes keep failing, and stop early when rounds stop making progress.

## Record usage

Keep a count of how often you use each step, and write it to `cex_usage.json`
next to the file you are repairing when you finish:

```json
{"rounds": 2, "solver_queries": {"z3": 2}, "counterexamples_found": 8,
 "counterexamples_confirmed": 5, "candidate_fixes": 6, "verified": true}
```

## Rules

- Don't change executable code or `requires`/`ensures`. Don't add `assume`,
  `admit`, or similar to the real file (the harness is scratch).
- If no counterexample checks out but the invariants look right, the verifier
  needs help instead: a lemma, an `assert`, `reveal`, or a trigger.
