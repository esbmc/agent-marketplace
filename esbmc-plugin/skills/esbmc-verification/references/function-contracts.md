# ESBMC Function Contracts

Function contracts let you specify the **behavioral interface** of a function — what it requires from callers, what it guarantees to them, and which memory locations it is allowed to modify. ESBMC uses contracts in two complementary modes:

1. **Enforce mode** — verify that the function body *satisfies* its own contract.
2. **Replace mode** — replace calls to the function with the contract's abstract semantics, avoiding full symbolic execution of the body.

## When to Recommend Function Contracts

Recommend this feature when a function is called many times and the repeated symbolic execution of its body is causing state space explosion or timeout. For functions called once or twice, plain BMC or k-induction on the whole program is simpler and produces confirmed counterexamples.

```
Function called once or twice  →  suggest plain BMC / k-induction
Function called many times, BMC times out  →  suggest enforce + replace
```

**Enforce and replace must always be used together.** Neither mode alone is sufficient:

- **Enforce** (`--enforce-contract`) proves the function body satisfies its contract in isolation. Combined with `--function`, it cuts the dependency chain from `main`. Because there is no prior execution history, this isolated check has a larger symbolic execution space — enforce by itself is slower, not faster.
- **Replace** (`--replace-call-with-contract`) is where the speed benefit comes from. Every call site is substituted with the contract's `assume` clauses — ESBMC no longer unfolds the function body at each call. In programs where the function is called multiple times, this dramatically shrinks the total state space.

Replace without enforce assumes an unproven contract (unsound). Enforce without replace gives a correctness proof but no speed benefit. Use contracts when the function is called 3 or more times, or when the body is expensive to re-analyze.

This feature is under active development. Encourage the user to experiment — getting a contract right takes iteration, and that is expected behaviour, not a problem.

## What to Set as Expectations

Tell the user upfront:

- Function contracts are an **over-approximation**. A verification failure may mean the contract needs refinement, not that the program has a real bug.
- **Iteration is normal.** The requires, ensures, and assigns clauses may each need several rounds of adjustment.
- **`VERIFICATION SUCCESSFUL` is sound** once the contract is proven by enforce — the result is trustworthy at that point.

## Contract Clauses

### `__ESBMC_requires(expr)` — Precondition

Specifies what must be true when the function is entered. In enforce mode ESBMC **assumes** this; in replace mode it **asserts** it at every call site.

```c
void divide(int a, int b) {
    __ESBMC_requires(b != 0);
    // ...
}
```

### `__ESBMC_ensures(expr)` — Postcondition

Specifies what must be true when the function returns. In enforce mode ESBMC **asserts** this; in replace mode it **assumes** it after the call.

```c
int increment(int x) {
    __ESBMC_requires(x > 0);
    __ESBMC_ensures(__ESBMC_return_value > x);
    return x + 1;
}
```

### `__ESBMC_assigns(targets...)` — Frame Specification

Lists every memory location the function is permitted to modify. ESBMC will **havoc** (assign an arbitrary value to) exactly these locations when replacing calls, and will **check** that no other location is written when enforcing.

```c
void reset_point(Point *p) {
    __ESBMC_requires(p != NULL);
    __ESBMC_assigns(p->x, p->y, p->z);
    __ESBMC_ensures(p->x == 0 && p->y == 0 && p->z == 0);
    p->x = 0; p->y = 0; p->z = 0;
}
```

| Assigns clause | Behaviour during replace |
|----------------|--------------------------|
| `__ESBMC_assigns(a, b)` | Havoc `a` and `b` only |
| `__ESBMC_assigns()` | No havoc — pure function |
| Absent | Conservative: havoc all static globals |

> **Note**: `__ESBMC_assigns` supports up to 5 targets (`a, b, c, d, e`). The macro does not accept `0` as an argument — use `__ESBMC_assigns()` (empty call) to declare a pure function.

### `__ESBMC_return_value` — Postcondition Return Reference

A special symbol available inside `__ESBMC_ensures()` that refers to the value returned by the function.

```c
int abs_val(int x) {
    __ESBMC_ensures(__ESBMC_return_value >= 0);
    __ESBMC_ensures(x >= 0 ? __ESBMC_return_value == x
                           : __ESBMC_return_value == -x);
    return x >= 0 ? x : -x;
}
```

> Use `--ir` when enforcing this function — without it, the bit-vector solver may report a spurious overflow counterexample for `x = INT_MIN`.

### `__ESBMC_old(expr)` — Pre-State Snapshot

Captures the value of an expression **at function entry**, for use inside `__ESBMC_ensures()`. Useful for relating the post-state to the pre-state.

```c
void increment_ptr(int *x) {
    __ESBMC_requires(x != NULL);
    __ESBMC_ensures(*x == __ESBMC_old(*x) + 1);
    *x = *x + 1;
}
```

Multiple `__ESBMC_old()` calls are supported in the same ensures clause:

```c
void swap(int *a, int *b) {
    __ESBMC_assigns(*a, *b);
    __ESBMC_ensures(*a == __ESBMC_old(*b) && *b == __ESBMC_old(*a));
    int tmp = *a; *a = *b; *b = tmp;
}
```

### `__ESBMC_is_fresh(ptr, size)` — Memory Freshness

Verifies that a pointer argument refers to a **freshly allocated** block of at least `size` bytes. Use inside `__ESBMC_ensures()`.

```c
void init_buffer(char **buf, size_t n) {
    __ESBMC_requires(n > 0);
    __ESBMC_ensures(__ESBMC_is_fresh(buf, n));
    *buf = malloc(n);
}
```

## Annotation Style (Optional)

For functions that are entirely defined by their contract, mark them with `__ESBMC_contract`:

```c
__ESBMC_contract
int safe_add(int a, int b) {
    __ESBMC_requires(a >= 0 && b >= 0);
    __ESBMC_requires(a <= INT_MAX - b);          // no overflow
    __ESBMC_ensures(__ESBMC_return_value == a + b);
    __ESBMC_assigns();                            // pure
    return a + b;
}
```

The `__ESBMC_contract` attribute enables batch processing via wildcard flags (see CLI section below).

## CLI Flags

### Enforce Mode

```bash
esbmc file.c --enforce-contract "function_name"
```

Verifies the function body against its own contract:
- Assumes `__ESBMC_requires` at entry.
- Calls the original function body.
- Asserts `__ESBMC_ensures` at exit.
- Checks that only `__ESBMC_assigns` targets are written.

```bash
# Enforce a single function
esbmc file.c --enforce-contract "init_point" --function "init_point"

# Enforce all __ESBMC_contract-annotated functions
esbmc file.c --enforce-all-contracts
```

### Replace Mode

```bash
esbmc file.c --replace-call-with-contract "function_name"
```

At every call site to the named function:
- Asserts `__ESBMC_requires` (caller must satisfy precondition).
- Havocs locations listed in `__ESBMC_assigns`.
- Assumes `__ESBMC_ensures` (caller may rely on postcondition).

The function body is **never symbolically executed** — this drastically reduces state space for recursive, complex, or library functions.

```bash
# Replace calls to one function
esbmc file.c --replace-call-with-contract "malloc_wrapper"

# Replace calls to all __ESBMC_contract-annotated functions
esbmc file.c --replace-all-contracts

# Replace calls to ALL functions that have contracts (wildcard)
esbmc file.c --replace-call-with-contract "*"
```

### Assume Non-Null Pointers Valid

```bash
esbmc file.c --enforce-contract "func" --assume-nonnull-valid
```

In enforce mode, automatically adds `__ESBMC_valid_object` assumptions for every non-null pointer parameter. Useful when the contract does not explicitly require pointer validity.

### Combining Enforce and Replace

A common pattern for **modular verification** of a call hierarchy:

```bash
# Step 1: verify low-level functions against their contracts
esbmc file.c --enforce-contract "low_level_func"

# Step 2: verify high-level function assuming low-level contracts hold
esbmc file.c --replace-call-with-contract "low_level_func" \
             --enforce-contract "high_level_func"
```

## Complete Examples

### Basic Precondition / Postcondition

In replace mode, `assert()` must follow directly from the `ensures` clause — the solver uses the ensures as an `assume` and cannot derive values beyond what the clause states. Use relational postconditions rather than equality to an exact value.

```c
#include <assert.h>

int increment(int x) {
    __ESBMC_requires(x > 0);
    __ESBMC_ensures(__ESBMC_return_value > x);
    return x + 1;
}

int main() {
    int a = 5;
    int result = increment(a);
    assert(result > a);   // follows directly from ensures
    return 0;
}
```

```bash
esbmc main.c --replace-call-with-contract "*"
```

### Struct Mutation with Assigns

```c
#include <stddef.h>
typedef struct { int x, y; } Point;

void zero_point(Point *p) {
    __ESBMC_requires(p != NULL);
    __ESBMC_assigns(p->x, p->y);
    __ESBMC_ensures(p->x == 0 && p->y == 0);
    p->x = 0;
    p->y = 0;
}
```

```bash
# Enforce: check that zero_point really satisfies its contract
esbmc point.c --enforce-contract "zero_point" \
              --function "zero_point" \
              --assume-nonnull-valid
```

### Pre-State Comparison with `__ESBMC_old`

```c
#include <stddef.h>
void push(Stack *s, int val) {
    __ESBMC_requires(s != NULL && s->size < MAX);
    __ESBMC_assigns(s->data[s->size], s->size);
    __ESBMC_ensures(s->size == __ESBMC_old(s->size) + 1);
    __ESBMC_ensures(s->data[__ESBMC_old(s->size)] == val);
    s->data[s->size] = val;
    s->size++;
}
```

### Array Element Assignments

```c
int arr[3] = {0, 0, 0};

void set_element(int i, int v) {
    __ESBMC_requires(i >= 0 && i < 3);
    __ESBMC_assigns(arr[i]);
    __ESBMC_ensures(arr[i] == v);
    arr[i] = v;
}
```

### Hierarchical / Compositional Verification

Enforce the leaf function first, then replace its calls when verifying the caller.

> **Note**: enforce mode does not work well for recursive functions — the recursive calls cause exponential unrolling. Use contracts for non-recursive functions or functions where recursion is bounded and small.

```c
#include <assert.h>

// Level 1: leaf function — enforce its contract independently
int double_val(int x) {
    __ESBMC_requires(x >= 0);
    __ESBMC_ensures(__ESBMC_return_value >= x);
    return x * 2;
}

// Level 2: caller — verified assuming double_val's contract holds
int main() {
    int a = 3, b = 5;
    int ra = double_val(a);
    int rb = double_val(b);
    assert(ra >= a);
    assert(rb >= b);
    return 0;
}
```

```bash
# Step 1: prove double_val satisfies its contract
# --ir avoids spurious overflow counterexamples for arithmetic postconditions
esbmc file.c --enforce-contract "double_val" --function "double_val" --ir

# Step 2: verify main() using the proven contract
esbmc file.c --replace-call-with-contract "double_val"
```

## Verification Modes Comparison

| Aspect | Enforce mode (`--enforce-contract`) | Replace mode (`--replace-call-with-contract`) |
|--------|-------------------------------------|------------------------------------------------|
| What is verified | Function body satisfies its own contract | Callers correctly use the function |
| Precondition | **Assumed** (function does not need to check) | **Asserted** at every call site |
| Postcondition | **Asserted** (function must establish it) | **Assumed** after each call |
| Function body | Symbolically executed in isolation | Not executed — replaced by contract assumes |
| Effect on speed | Slower (no execution history, larger state space) | Faster (avoids re-exploring body at every call site) |
| Role in workflow | Establish safety: prove the contract is valid | Deliver speed: skip repeated body exploration |
| Use together | **Both required** for sound and efficient verification | **Both required** for sound and efficient verification |

## Iterative Refinement and Troubleshooting

Getting contracts right is an iterative process. The typical cycle is:

```
Write contract → enforce (prove contract holds) → replace (verify callers) → read failure → refine contract → repeat
```

A failure at any stage does not automatically mean the program has a bug — it may mean the contract abstraction needs adjustment.

### Failure in enforce mode

**Postcondition assertion failed**
```
[Counterexample] __ESBMC_ensures assertion failed
```
The function body does not establish what the `ensures` clause claims. Either the implementation is wrong, or the postcondition is too strong. Check: does the function actually guarantee this in all cases? If yes, the implementation has a bug. If not, weaken the postcondition.

**Assigns violation**
```
[Counterexample] assigns clause violated
```
The function writes to a memory location not listed in `__ESBMC_assigns`. Either add the missing location to the clause, or remove the write if it is unintended.

### Failure in replace mode

**Precondition assertion failed**
```
[Counterexample] __ESBMC_requires assertion failed
```
A caller passes arguments that violate the precondition. Either add a guard before the call site, or weaken the precondition if it is overly strict.

**Spurious failure after replacement**
If the system-level verification fails at a point unrelated to the contracted function, the `ensures` clause may be too weak — it does not provide enough information for the caller to proceed. Strengthen the postcondition to capture more of what the function actually guarantees.

### Missing assigns clause causes over-havocing

When no `__ESBMC_assigns` clause is present, ESBMC conservatively havocs **all** global/static variables in replace mode. This frequently causes spurious failures in the caller. Always provide an assigns clause. Use `__ESBMC_assigns()` for pure functions that modify nothing.

### Confirming a real bug

A counterexample produced during contract verification **may be an artifact of over-approximation**, not a real program defect. Before reporting a bug:
1. Check whether the counterexample trace makes sense with realistic input values.
2. If possible, reproduce the failure with plain BMC (`--unwind N`) on a smaller scope — a confirmed BMC counterexample is a real bug.
3. If the trace contains unrealistic values (e.g. unconstrained havoced variables), refine the contract first.

