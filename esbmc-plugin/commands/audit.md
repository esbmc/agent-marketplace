---
name: audit
description: Perform a comprehensive security audit on a source file using ESBMC
arguments:
  - name: file
    description: The source file to audit
    required: true
---

# ESBMC Security Audit Command

Perform a comprehensive security audit using multiple ESBMC verification passes run concurrently.

## Instructions

1. Check that ESBMC is installed by running `which esbmc`. If not found, stop and tell the user:
   - Download pre-built binaries from https://github.com/esbmc/esbmc/releases
   - Or build from source: https://github.com/esbmc/esbmc
   - Ensure `esbmc` is in the PATH: `export PATH=$PATH:/path/to/esbmc/bin`

2. Read the source file to understand its structure and detect:
   - Language (C, C++, Python, Solidity)
   - Presence of concurrency (pthreads, std::thread, threading module)
   - Complexity indicators (loops, recursion, dynamic allocation)

3. **Solver detection** — run `esbmc --list-solvers` and select the best available solver using this priority order: Boolector → Bitwuzla → Z3. Use the corresponding flag: `--boolector`, `--bitwuzla`, or `--z3`. Store this as `<solver-flag>`. If none of these solvers are listed as available, stop and tell the user that ESBMC is built without solver support, and direct them to build ESBMC with solver support: https://github.com/esbmc/esbmc

4. **Loop discovery** — run `esbmc <file> --show-loops` to list all loops. Choose the unwind strategy:
   - No loops found → no unwind flag needed
   - Loops with bounds apparent from the source → use `--unwindset L1:N,...` with per-loop bounds derived from the source
   - Loops with unknown or dynamic bounds → use `--incremental-bmc`

   Store the resulting unwind flag(s) as `<unwind-strategy>`.

5. **Spawn parallel subagents** using the Task tool (subagent_type: general-purpose), one per pass. Launch all subagents in a single message so they execute concurrently. Each subagent receives: the file path, `<solver-flag>`, `<unwind-strategy>`, and its specific checks.

   | Subagent | Checks | Command |
   |----------|--------|---------|
   | A: Default properties | _(none — covers bounds, ptr safety, div-by-zero)_ | `esbmc <file> <solver-flag> <unwind-strategy> --timeout 60s` |
   | B: Memory safety | `--memory-leak-check` | `esbmc <file> <solver-flag> <unwind-strategy> --memory-leak-check --timeout 60s` |
   | C: Integer safety | `--overflow-check --unsigned-overflow-check` | `esbmc <file> <solver-flag> <unwind-strategy> --overflow-check --unsigned-overflow-check --timeout 60s` |
   | D: UB shifts | `--ub-shift-check` | `esbmc <file> <solver-flag> <unwind-strategy> --ub-shift-check --timeout 60s` |
   | E: Concurrency _(only if threads detected in step 2)_ | `--deadlock-check --data-races-check --context-bound 2` | `esbmc <file> <solver-flag> <unwind-strategy> --deadlock-check --data-races-check --context-bound 2 --timeout 60s` |

6. Collect results from all subagents and compile into a structured report:

## Output Format

```
ESBMC Security Audit Report
===========================
File: <filename>
Language: <language>
Concurrency: <yes/no>
Solver: <solver used>
Unwind strategy: <strategy>

Results:
--------
[A: Default Properties]
  ✓ PASSED / ✗ FAILED: <details>

[B: Memory Safety]
  ✓ PASSED / ✗ FAILED: <details>

[C: Integer Safety]
  ✓ PASSED / ✗ FAILED: <details>

[D: UB Shifts]
  ✓ PASSED / ✗ FAILED: <details>

[E: Concurrency]
  ✓ PASSED / ✗ FAILED / N/A: <details>

Summary:
--------
Total Issues: N
- Default properties: X issues
- Memory: Y issues
- Integer: Z issues
- UB shifts: W issues
- Concurrency: V issues

Recommendations:
----------------
1. <recommendation>
2. <recommendation>
```
