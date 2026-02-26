---
name: audit
description: Perform a comprehensive security audit on a source file using ESBMC
arguments:
  - name: file
    description: The source file to audit
    required: true
---

# ESBMC Security Audit Command

Perform a comprehensive security audit using multiple ESBMC verification passes run in parallel.

## Instructions

1. Check that ESBMC is installed by running `which esbmc`. If not found, stop and tell the user:
   - Download pre-built binaries from https://github.com/esbmc/esbmc/releases
   - Or build from source: https://github.com/esbmc/esbmc
   - Ensure `esbmc` is in the PATH: `export PATH=$PATH:/path/to/esbmc/bin`

2. Read the source file to understand its structure and detect:
   - Language (C, C++, Python, Solidity)
   - Presence of concurrency (pthreads, std::thread, threading module)
   - Complexity indicators (loops, recursion, dynamic allocation)

3. Discover loop structure:
   ```bash
   esbmc <file> --show-loops
   ```
   From the output, determine the unwinding strategy for all subsequent passes:
   - **Known loop bounds**: use `--unwindset <L1:N,...>` (derived from source + loop IDs)
   - **Unknown loop bounds**: use `--incremental-bmc`

   Let `<UNWIND>` denote whichever flag was chosen above. Always use `--boolector` as
   the SMT solver in every pass.

4. Run the following passes **in parallel** (issue all Bash tool calls in a single
   response so they execute concurrently):

   **Pass A: Default properties** (pointer safety, bounds, division by zero)
   ```bash
   esbmc <file> --boolector <UNWIND> --timeout 60s
   ```

   **Pass B: Memory safety**
   ```bash
   esbmc <file> --boolector <UNWIND> --memory-leak-check --timeout 60s
   ```

   **Pass C: Integer safety**
   ```bash
   esbmc <file> --boolector <UNWIND> --overflow-check --unsigned-overflow-check --timeout 60s
   ```

   **Pass D: Undefined-behavior shifts**
   ```bash
   esbmc <file> --boolector <UNWIND> --ub-shift-check --timeout 60s
   ```

   **Pass E: Concurrency** (only if the file contains threads/mutexes)
   ```bash
   esbmc <file> --boolector <UNWIND> --deadlock-check --data-races-check --context-bound 2 --timeout 60s
   ```

5. After all passes complete, compile results into a summary report:
   - Result of each pass (PASSED / FAILED / UNKNOWN)
   - For any FAILED pass: violation type and key counterexample trace
   - Overall issue count by category
   - Recommendations for fixes

## Output Format

```
ESBMC Security Audit Report
===========================
File: <filename>
Language: <language>
Concurrency: <yes/no>
Solver: Boolector
Unwind strategy: <incremental-bmc | unwindset ...>

Results:
--------
[Pass A: Default properties]
  ✓ PASSED / ✗ FAILED: <details>

[Pass B: Memory safety]
  ✓ PASSED / ✗ FAILED: <details>

[Pass C: Integer safety]
  ✓ PASSED / ✗ FAILED: <details>

[Pass D: UB shifts]
  ✓ PASSED / ✗ FAILED: <details>

[Pass E: Concurrency]
  ✓ PASSED / ✗ FAILED / — SKIPPED: <details>

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
