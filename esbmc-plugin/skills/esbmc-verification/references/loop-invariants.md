# ESBMC Loop Invariants

Loop invariants allow ESBMC to verify loops **without bounding them**. Instead of unrolling a loop N times, ESBMC uses the invariant to prove correctness inductively — eliminating the need for `--unwind` and enabling unbounded verification.

## When to Recommend Loop Invariants

Recommend this feature when the user's loop has a large or unbounded iteration count and plain BMC (`--unwind N`) is timing out or impractical. Do not suggest it for loops with small, known bounds — plain BMC is simpler and produces confirmed counterexamples.

```
Loop bound small / known   →  suggest --unwind N
Loop bound large / unknown, BMC times out  →  suggest --loop-invariant --ir
```

This feature is under active development. Encourage the user to try it — iteration and experimentation are expected parts of the workflow, not signs of failure.

## What to Set as Expectations

Tell the user upfront:

- Loop invariant verification is an **over-approximation**. A verification failure may mean the invariant needs adjustment, not that the code has a real bug.
- **Iteration is normal.** The invariant will likely need several rounds of refinement before passing.
- **`VERIFICATION SUCCESSFUL` is sound** — once the invariant is correct, the result is trustworthy.

## How It Works

ESBMC transforms the loop into three proof obligations:

1. **Base case (initialization)**: Assert the invariant holds before the loop starts.
2. **Inductive step (preservation)**: Assume the invariant holds at the top of the loop, execute the body, then assert the invariant holds again.
3. **Termination**: After the inductive step, assume `false` to stop further symbolic execution of the loop (the loop is considered fully summarized).

If all three checks pass, the invariant is proven inductive and any assertion inside or after the loop is verified against the invariant — no unrolling needed.

## Syntax

```c
__ESBMC_loop_invariant(expression);
while (condition) {
    // loop body
}
```

The `__ESBMC_loop_invariant()` macro must be placed **immediately before the loop** it annotates (before the `while`, `for`, or `do` keyword).

### For Loops

The loop variable must be declared **before** the loop so it is visible to the invariant macro. Do not use `for (int i = 0; ...)` — declare `i` separately first.

```c
int i = 0, sum = 0;
__ESBMC_loop_invariant(i >= 0 && i <= n && sum == i * 10);
for (; i < n; i++) {
    sum += 10;
}
```

### Do-While Loops

```c
__ESBMC_loop_invariant(i >= 0);
do {
    i++;
} while (i < 10);
```

## Multiple Invariants

Multiple `__ESBMC_loop_invariant()` calls on the same loop are **combined with AND**:

```c
__ESBMC_loop_invariant(i >= 0 && i <= 5000000);
__ESBMC_loop_invariant(sum == i * 10);
while (i < 5000000) {
    sum += 10;
    i++;
}
```

This is exactly equivalent to:

```c
__ESBMC_loop_invariant(i >= 0 && i <= 5000000 && sum == i * 10);
while (...) { ... }
```

Split invariants across multiple lines for readability when the combined expression is complex.

## Nested Loops

Each loop may have its own independent invariant. Loops without an invariant are still handled (unrolled) according to the `--unwind` setting.

```c
__ESBMC_loop_invariant(i >= 0 && i <= rows);
while (i < rows) {
    j = 0;
    // inner loop has no invariant — unrolled via --unwind
    while (j < cols) {
        matrix[i][j] = 0;
        j++;
    }
    i++;
}
```

```c
// Both loops annotated
__ESBMC_loop_invariant(i >= 0 && i <= rows);
while (i < rows) {
    j = 0;
    __ESBMC_loop_invariant(j >= 0 && j <= cols);
    while (j < cols) {
        matrix[i][j] = 0;
        j++;
    }
    i++;
}
```

## CLI Flag

```bash
esbmc file.c --loop-invariant
```

Enable the loop invariant transformation. Without this flag the `__ESBMC_loop_invariant()` macros are ignored.

### Recommended: Use `--ir` for Effective Invariant Verification

```bash
esbmc file.c --loop-invariant --ir
```

`--ir` switches the solver to **integer/real arithmetic**, treating integers as unbounded mathematical integers rather than fixed-width bit-vectors. This has two effects:

1. **Arithmetic overflow is not checked** — the solver never raises an overflow violation, because integers are modelled as having infinite range.
2. **Significant performance boost** — integer arithmetic is much cheaper for SMT solvers than bit-vector arithmetic.

For loop invariant proofs the primary goal is to establish a relational property over the loop variables (e.g. `sum == i * 10`). Bit-level overflow concerns usually obscure this reasoning and cause spurious counterexamples. Using `--ir` keeps the verification focused on the invariant itself, achieving results comparable to invariant synthesis tools like Code2Inv.

> **Trade-off**: `--ir` over-approximates C integers. If your loop manipulates values near integer bounds and overflow behavior matters, omit `--ir` and handle potential overflow explicitly in the invariant or with `--overflow-check`.

### Combined with Multi-Property

```bash
esbmc file.c --loop-invariant --ir --multi-property
```

Use `--multi-property` when the loop body (or code after the loop) contains multiple assertions — ESBMC will report each property violation separately rather than stopping at the first.

### No `--unwind` Needed for Annotated Loops

For loops that carry an invariant you do **not** need `--unwind`. For any remaining unannotated loops in the same file you still need to provide a bound:

```bash
esbmc file.c --loop-invariant --ir --unwind 10
```

## Complete Example

```c
#include <assert.h>

int main() {
    int i = 0, sum = 0;

    __ESBMC_loop_invariant(i >= 0 && i <= 100 && sum == i * i);
    while (i < 100) {
        sum += 2 * i + 1;   // sum stays equal to i^2 after increment
        i++;
    }

    assert(sum == 10000);   // provable from invariant at loop exit
    return 0;
}
```

Run:
```bash
esbmc main.c --loop-invariant --ir
```

Expected output:
```
VERIFICATION SUCCESSFUL
```

## Invariant Design Guidelines

### What a Good Invariant Must Capture

1. **Bounds on loop variables** — prevents out-of-bounds reasoning.
2. **Relationships between variables** — the "progress" condition that links loop variables to the result.
3. **Strong enough to imply the postcondition** — at loop exit the invariant combined with the negated loop condition must entail what you want to assert.

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Invariant placed inside the loop body | Ignored; must be immediately before the loop keyword | Move above `while`/`for`/`do` |
| Invariant too weak | Cannot prove assertion after loop | Strengthen the relational part |
| Invariant too strong | Base case or inductive step fails | Relax bounds or split into multiple clauses |
| Forgetting loop variable bounds | ESBMC cannot bound the symbolic execution | Always include `i >= 0 && i <= N` style bounds |

### Iterative Refinement Workflow

Getting an invariant right is an iterative process. Follow this cycle:

```
Write invariant → Run ESBMC → Read failure → Adjust invariant → Repeat
```

**Step 1: identify which proof obligation fails**

```bash
esbmc file.c --loop-invariant --ir --multi-property
```

`--multi-property` reports each failed property separately so you can pinpoint which of the three obligations broke.

**Step 2: diagnose by failure type**

| Failure message | What it means | What to do |
|----------------|---------------|------------|
| Base case assertion failed | Invariant is not true before the first iteration | Check the initial values of all variables mentioned in the invariant; adjust bounds or relational terms to match the actual initial state |
| Inductive step assertion failed | The loop body produces a state that violates the invariant | The invariant is too strong or missing a condition — add the missing relationship, or weaken an overly tight bound |
| Assertion after loop failed | Invariant at loop exit does not imply the desired postcondition | Invariant is too weak — strengthen the relational part so that `invariant ∧ ¬condition → postcondition` holds |
| Counterexample shown | May be a real bug, or an artifact of over-approximation | Inspect the counterexample trace; if the values look unrealistic (e.g. astronomically large integers), the invariant is under-constraining — add tighter bounds |

**Step 3: adjust and re-run**

```bash
# After adjusting the invariant, re-run
esbmc file.c --loop-invariant --ir

# If multiple assertions exist, keep --multi-property
esbmc file.c --loop-invariant --ir --multi-property
```

**Step 4: when VERIFICATION SUCCESSFUL**

A passing result is sound — the property genuinely holds under the invariant. But verify that the invariant itself is meaningful: if it is trivially `true`, ESBMC may have passed vacuously without actually proving anything useful.

> **Remember**: a counterexample during invariant verification is not automatically a confirmed bug. Always check whether the counterexample is reproducible without the invariant (e.g. with `--unwind` on a smaller bound) before concluding the program has a real defect.

## Assertions Inside Loop Bodies

Assertions (`assert()`) inside the loop body are verified against the **assumed** invariant at the top of each iteration. This means they are checked independently of the unroll depth.

**Important**: the loop variable must be declared **before** the loop so it is in scope when `__ESBMC_loop_invariant()` is evaluated. Do not use `for (int i = 0; ...)` style declarations — declare the variable outside the loop first.

```c
#include <assert.h>

int main() {
    int i = 0, sum = 0;
    __ESBMC_loop_invariant(i >= 0 && i <= 100 && sum == i * 2);
    while (i < 100) {
        sum += 2;
        assert(sum >= 0);   // follows from invariant: sum == i*2 and i >= 0
        i++;
    }
    return 0;
}
```

```bash
esbmc file.c --loop-invariant --ir
```

## Interaction with k-Induction

`--loop-invariant` and `--k-induction` are different mechanisms:

| | `--loop-invariant` | `--k-induction` |
|-|-------------------|-----------------|
| Invariant source | User-supplied macro | Automatically inferred |
| Completeness | Depends on invariant quality | Automatic but may time out |
| Control | Explicit, predictable | Heuristic |

Use `--loop-invariant` when you know the invariant and want a deterministic, fast proof. Use `--k-induction` when you want ESBMC to search for one automatically.

