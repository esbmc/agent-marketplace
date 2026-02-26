---
name: verify
description: Verify a source file with ESBMC for bugs and safety violations
arguments:
  - name: file
    description: The source file to verify
    required: true
  - name: checks
    description: Additional checks to enable (memory, overflow, concurrent, all)
    required: false
---

# ESBMC Verification Command

Verify the specified source file using ESBMC bounded model checker.

## Instructions

1. Check that ESBMC is installed by running `which esbmc`. If not found, stop and tell the user:
   - Download pre-built binaries from https://github.com/esbmc/esbmc/releases
   - Or build from source: https://github.com/esbmc/esbmc
   - Ensure `esbmc` is in the PATH: `export PATH=$PATH:/path/to/esbmc/bin`

2. Determine the file type from the extension:
   - `.c` → C file
   - `.cpp`, `.cc`, `.cxx` → C++ file
   - `.py` → Python file (auto-detected by extension)
   - `.sol` → Solidity file (use `--sol`)
   - `.cu` → CUDA file

3. Discover loop structure by running:
   ```bash
   esbmc <file> --show-loops
   ```
   Examine the output to identify loop IDs and decide on an unwinding strategy:
   - **If the file has no loops**, proceed directly with `--incremental-bmc`.
   - **If loops are present and bounds are apparent from the source**, use `--unwindset <L1:N,L2:M,...>` with the specific loop IDs and counts shown by `--show-loops`.
   - **If loops are present but bounds are unknown**, use `--incremental-bmc`.

4. Build the ESBMC command. Always use `--boolector` as the SMT solver. Add the
   appropriate checks from the `checks` argument:
   - `memory` → add `--memory-leak-check`
   - `overflow` → add `--overflow-check --unsigned-overflow-check`
   - `concurrent` → add `--deadlock-check --data-races-check --context-bound 2`
   - `all` → add all of the above

   **With `--incremental-bmc` (unknown loop bounds):**
   ```bash
   esbmc <file> --boolector --incremental-bmc --timeout 120s [checks]
   ```

   **With `--unwindset` (known loop bounds):**
   ```bash
   esbmc <file> --boolector --unwindset <L1:N,...> --timeout 60s [checks]
   ```

5. Run the command using the Bash tool.

6. Interpret the results:
   - **VERIFICATION SUCCESSFUL** → All checked properties hold within bounds
   - **VERIFICATION FAILED** → Bug found, examine counterexample trace
   - **UNKNOWN/TIMEOUT** → Verification inconclusive

7. If verification fails, provide:
   - Summary of the violation type
   - Key parts of the counterexample trace
   - Suggestions for fixing the issue

## Examples

User: `/verify src/parser.c`
→ Run loop discovery, then incremental-bmc or unwindset as appropriate

User: `/verify src/memory.c memory`
→ Run with memory leak checking added to the base command

User: `/verify threaded.c concurrent`
→ Run with concurrency checks

User: `/verify contract.sol --contract MyContract`
→ Run Solidity verification
