---
name: esbmc-verification
description: This skill should be used when the user asks to "verify code", "run ESBMC", "model check", "check for bugs", "find memory leaks", "detect buffer overflow", "find undefined behavior", "check for race conditions", "detect deadlocks", "prove correctness", "add verification intrinsics", "add nondet values", "add type annotations", "add preconditions", "make code verifiable", "add loop invariant", "write loop invariant", "use __ESBMC_loop_invariant", "verify loop", "prove loop correct", "add function contract", "write function contract", "use __ESBMC_requires", "use __ESBMC_ensures", "use __ESBMC_assigns", "enforce contract", "replace call with contract", "modular verification", "compositional verification", "frame specification", "precondition postcondition", or mentions bounded model checking, SMT solving, formal methods, safety properties, loop invariants, or function contracts. Provides guidance for verifying C, C++, Python, Solidity, CUDA, and Java/Kotlin programs with ESBMC and adding verification annotations.
version: 1.0.0
---

# ESBMC Verification Skill

ESBMC (Efficient SMT-based Context-Bounded Model Checker) detects bugs or proves their absence in C, C++, CUDA, CHERI-C, Python, Solidity, Java, and Kotlin programs.

## Prerequisites Check

Before running any ESBMC command, verify that ESBMC is installed by running `which esbmc` or `esbmc --version`. If ESBMC is not found, inform the user and provide installation instructions:

- **Pre-built binaries**: Download from [ESBMC releases](https://github.com/esbmc/esbmc/releases)
- **Build from source**: Clone https://github.com/esbmc/esbmc and follow the build instructions in its README
- **After installing**, ensure `esbmc` is in the PATH: `export PATH=$PATH:/path/to/esbmc/bin`

Do not proceed with verification commands until ESBMC is confirmed available.

## Verification Pipeline

Source Code → Frontend Parser → GOTO Program → Symbolic Execution (SSA) → SMT Formula → Solver → Result

## Quick Start

```bash
# C file with default checks
esbmc file.c

# C++ file
esbmc file.cpp

# Python (requires type annotations)
esbmc file.py

# Solidity contract
esbmc --sol contract.sol --contract ContractName

# Common safety checks
esbmc file.c --memory-leak-check --overflow-check --unwind 10 --timeout 60s

# Concurrency verification
esbmc file.c --deadlock-check --data-races-check --context-bound 2
```

## Supported Languages

| Language | Command | Notes |
|----------|---------|-------|
| C | `esbmc file.c` | Default, Clang frontend |
| C++ | `esbmc file.cpp` | C++11-20 via `--std` |
| Python | `esbmc file.py` | Requires type annotations, Python 3.10+ |
| Solidity | `esbmc --sol file.sol --contract Name` | Smart contracts |
| CUDA | `esbmc file.cu` | GPU kernel verification |
| Java/Kotlin | `esbmc file.jimple` | Requires Soot conversion from .class |

For language-specific options and examples, see `references/language-specific.md`.

## Safety Properties

### Default Checks (Always On)
- **Array bounds**: Out-of-bounds array access
- **Division by zero**: Integer and floating-point
- **Pointer safety**: Null dereference, invalid pointer arithmetic
- **Assertions**: User-defined `assert()` statements

### Optional Checks (Enable Explicitly)
- `--overflow-check` / `--unsigned-overflow-check`: Integer overflow
- `--memory-leak-check`: Memory leaks
- `--deadlock-check` / `--data-races-check`: Concurrency safety
- `--nan-check`: Floating-point NaN
- `--ub-shift-check`: Undefined shift behavior
- `--is-instance-check`: Enable runtime isinstance assertions for annotated Python code

### Disable Specific Checks
`--no-bounds-check`, `--no-pointer-check`, `--no-div-by-zero-check`, `--no-assertions`

For the full CLI reference, see `references/cli-options.md`.

## Loop Invariants and Function Contracts

These features extend ESBMC beyond the reach of plain BMC — use them when BMC hits its scaling limits, not as a general replacement. Both are under active development; encourage users to try them.

### When to recommend each

| Situation | Recommend |
|-----------|-----------|
| Loop with small or known bound | `--unwind N` — simpler, counterexamples are confirmed bugs |
| Loop bound large / unbounded, BMC times out | `--loop-invariant --ir` |
| Function called once or twice | Plain BMC / k-induction |
| Function called many times, BMC times out | `--enforce-contract` + `--replace-call-with-contract` |

### What to tell the user before they start

- These are **over-approximations** — a verification failure may mean the abstraction needs refinement, not that the program has a real bug.
- **Iteration is expected and normal.** Guide the user through the refinement cycle rather than treating the first failure as a dead end.
- `VERIFICATION SUCCESSFUL` with a correct abstraction is a sound, trustworthy result.

For the refinement workflow, see `references/loop-invariants.md` and `references/function-contracts.md`.

## Loop Handling

K-induction is an alternative strategy that bypasses loop discovery entirely — see the Strategy Selection section below. If you are using BMC instead, always start by discovering loops before choosing an unwind strategy. Never guess a bound with `--unwind N`.

```bash
# Step 1: discover all loops and their IDs
esbmc file.c --show-loops
```

Decision tree based on the output:
- **No loops** → no unwind flag needed; run without any unwind option
- **Loops with apparent bounds** (e.g., `for (i = 0; i < N; i++)`) → use per-loop bounds derived from the source:
  ```bash
  esbmc file.c --unwindset L1:N,L2:M
  ```
- **Loops with unknown or dynamic bounds** → use incremental BMC:
  ```bash
  esbmc file.c --incremental-bmc
  ```

## Strategy Selection

Choose a high-level path before running any checks:

**Path A — k-Induction (try first for proving correctness)**
- No loop discovery needed.
- Detect CPU count: `nproc` on Linux, `sysctl -n hw.ncpu` on macOS.
- >4 CPUs → use `--k-induction-parallel`
- ≤4 CPUs → use `--k-induction`
- If k-induction succeeds → property is proved (unbounded result).
- If k-induction times out or returns UNKNOWN → fall back to Path B.

**Path B — BMC (bounded, loop-aware)**
- Run `--show-loops`, then choose `--unwindset` or `--incremental-bmc` per the Loop Handling decision tree above.

## Verification Strategies

| Goal | Strategy | Command |
|------|----------|---------|
| Loops with known bounds | BMC with unwindset | `--unwindset L1:N,...` |
| Unknown loop bounds | Incremental BMC | `--incremental-bmc` |
| Prove correctness (≤4 CPUs) | k-Induction | `--k-induction` |
| Prove correctness (>4 CPUs) | k-Induction parallel | `--k-induction-parallel` |
| All violations | Multi-property | `--multi-property` |
| Large programs | Incremental SMT | `--smt-during-symex` |
| Concurrent code | Context-bounded | `--context-bound 3` |

For detailed descriptions of the strategies and their configurations, see `references/verification-strategies.md`.

## Solver Selection

Always detect available solvers via `--list-solvers` rather than assuming one is present. Use the best available solver by priority: **Boolector → Bitwuzla → Z3**.

```bash
esbmc --list-solvers      # Detect available solvers
esbmc file.c --boolector  # Boolector (highest priority)
esbmc file.c --bitwuzla   # Bitwuzla (second priority)
esbmc file.c --z3         # Z3 (fallback)
```

If none of these solvers are available, ESBMC was built without solver support and verification cannot proceed.

## ESBMC Intrinsics

Use these in the source code to guide verification.

### Quick Reference

| Purpose | C/C++ | Python |
|---------|-------|--------|
| Symbolic int | `__ESBMC_nondet_int()` | `nondet_int()` |
| Symbolic uint | `__ESBMC_nondet_uint()` | N/A |
| Symbolic bool | `__ESBMC_nondet_bool()` | `nondet_bool()` |
| Symbolic float | `__ESBMC_nondet_float()` | `nondet_float()` |
| Symbolic string | N/A | `nondet_str()` |
| Symbolic list | N/A | `nondet_list()` |
| Symbolic dictionary | N/A | `nondet_dict()` |
| Assumption | `__ESBMC_assume(cond)` | `assume(cond)` |
| Assertion | `__ESBMC_assert(cond, msg)` | `esbmc_assert(cond, msg)` |
| Atomic section | `__ESBMC_atomic_begin/end()` | N/A |

### Basic Usage

```c
int x = __ESBMC_nondet_int();       // Symbolic input
__ESBMC_assume(x > 0 && x < 100);   // Constrain input
__ESBMC_assert(result >= 0, "msg");  // Verify property
```

```python
x: int = nondet_int()
__ESBMC_assume(x > 0 and x < 100)
assert result >= 0, "msg"
```

For the step-by-step process of adding intrinsics to code, see `references/adding-intrinsics.md`.
For the full intrinsics API, see `references/intrinsics.md`.

## Output and Debugging

```bash
esbmc file.c                          # Counterexample shown on failure
esbmc file.c --witness-output w.graphml # Generate witness file
esbmc file.c --show-vcc               # Show verification conditions
esbmc file.c --generate-testcase      # Generate test from counterexample
esbmc file.py --generate-pytest-testcase            # Generate pytest
```

## Resource Limits

```bash
esbmc file.c --timeout 300s   # Time limit
esbmc file.c --memlimit 4g    # Memory limit
```

## Common Workflows

### Bug Hunting
```bash
# Step 1: discover loops
esbmc file.c --show-loops
# Step 2a: known bounds
esbmc file.c --boolector --unwindset L1:N --timeout 60s
# Step 2b: unknown bounds
esbmc file.c --boolector --incremental-bmc --timeout 120s
```

### Proving Correctness
```bash
# Step 1: detect CPUs
nproc   # Linux; use `sysctl -n hw.ncpu` on macOS

# Step 2a: >4 CPUs
esbmc file.c --boolector --k-induction-parallel --overflow-check

# Step 2b: ≤4 CPUs
esbmc file.c --boolector --k-induction --overflow-check

# Step 3: if k-induction times out or returns UNKNOWN, fall back to BMC
esbmc file.c --show-loops   # then use --unwindset or --incremental-bmc
```

### Memory Safety Audit
```bash
esbmc file.c --boolector --unwindset L1:N --memory-leak-check
```

### Concurrency Verification
```bash
esbmc threaded.c --boolector --deadlock-check --data-races-check --context-bound 2
```

For guidance on fixing verification failures, see `references/fixing-failures.md`.

## Additional Resources

### Reference Files
- **`references/cli-options.md`** — Complete CLI reference
- **`references/verification-strategies.md`** — Detailed strategy guide
- **`references/language-specific.md`** — Language-specific features and options
- **`references/intrinsics.md`** — Full ESBMC intrinsics API
- **`references/adding-intrinsics.md`** — Step-by-step guide for annotating code
- **`references/fixing-failures.md`** — Diagnosing and fixing verification failures
- **`references/loop-invariants.md`** — Loop invariant syntax, CLI flag, design guidelines, and debugging
- **`references/function-contracts.md`** — Function contract clauses, enforce/replace modes, when to use each, and examples

### Example Files
- **`examples/memory-check.c`** — Memory safety verification (C)
- **`examples/overflow-check.c`** — Integer overflow detection (C)
- **`examples/concurrent.c`** — Concurrency verification (C)
- **`examples/cpp-verify.cpp`** — C++ verification (classes, STL, RAII)
- **`examples/python-verify.py`** — Python verification

### Scripts
- **`scripts/quick-verify.sh`** — Quick verification wrapper
- **`scripts/full-audit.sh`** — Comprehensive security audit
