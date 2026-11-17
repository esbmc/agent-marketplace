# ESBMC Complete CLI Reference

## Input/Output Options

| Option | Description |
|--------|-------------|
| `--input-file <file>` | Source file names |
| `--preprocess` | Stop after preprocessing |
| `--binary` | Read GOTO program instead of source |
| `--output-goto <file>` | Export generated GOTO program |
| `--cex-output <prefix>` | Save counterexample(s) to file(s) |
| `--file-output <file>` | Redirect all messages to file |
| `--version` | Show ESBMC version |

## Display and Debugging

| Option | Description |
|--------|-------------|
| `--symbol-table-only` | Only show symbol table |
| `--parse-tree-only` | Show parse tree only |
| `--parse-tree-too` | Show parse tree before verification |
| `--goto-functions-only` | Show GOTO program only |
| `--goto-functions-too` | Show GOTO program before verification |
| `--dump-goto-cfg` | Create DOT format CFG files |
| `--ssa-symbol-table` | Show symbol table with SSA |
| `--smt-formula-only` | Show SMT formula only |
| `--smt-formula-too` | Show SMT formula before solving |
| `--smt-model` | Show SMT model if SAT |
| `--show-loops` | Show loops in program |
| `--show-claims` | Only show verification conditions |
| `--show-vcc` | Show verification conditions |
| `--show-stacktrace` | Show stack trace in counterexample |
| `--goto2c` | Translate GOTO to C |
| `--goto2lean` | Translate GOTO to Lean 4 (prototype, scalar fragment) |
| `--color` | Colored output |
| `--quiet` | Suppress unwinding info |

## Trace Output

| Option | Description |
|--------|-------------|
| `--symex-trace` | Print instructions during symex |
| `--ssa-trace` | Print SSA during SMT encoding |
| `--ssa-smt-trace` | Print SMT during encoding |
| `--symex-ssa-trace` | Print SSA during symex |
| `--show-funccall-trace` | Chronological function calls leading to the counterexample |
| `--log-message` | Print LOG with file/line/timestamp |
| `--verbosity <module:N>` | Set log-level for modules |

## Frontend Options (C/C++)

| Option | Description |
|--------|-------------|
| `--std <version>` | Set C/C++ standard (c99, c11, c++11, etc.) |
| `-I, --include <path>` | Add include path |
| `--include-file <file>` | Include file via -include |
| `--nostdinc` | Don't use standard system paths |
| `--idirafter <path>` | Append system include path |
| `-D, --define <macro>` | Define preprocessor macro |
| `-W, --warning <opt>` | Enable/disable warnings |
| `--sysroot <path>` | Set sysroot for frontend |
| `--no-abstracted-cpp-includes` | Don't use C++ operational models |
| `--no-inlining` | Disable function inlining |
| `--full-inlining` | Full function inlining |
| `--little-endian` | Little-endian conversions |
| `--big-endian` | Big-endian conversions |
| `--16, --32, --64` | Set word width (default: 64) |
| `--funsigned-char` | Make char unsigned |
| `--fms-extensions` | Enable MS C extensions |
| `--cheri <mode>` | CHERI-C: hybrid or purecap |
| `--cheri-uncompressed` | Full CHERI capabilities |

## Python Frontend Options

| Option | Description |
|--------|-------------|
| `--python <path>` | Python interpreter binary to use (default: python from $PATH). Note: this is NOT a mode flag — Python frontend is auto-detected by `.py` extension |
| `--override-return-annotation` | Override return type with inferred |
| `--strict-types` | Strict type checking |
| `--nondet-str-length <n>` | Max nondet string length (default: 16) |
| `--python-no-fold` | Disable NumPy constant folding |
| `--python-list-compare-depth <n>` | Max nesting depth for list comparison (default: 4) |
| `--generate-pytest-testcase` | Generate pytest from solution |
| `--pytest-output-dir <dir>` | Directory for generated pytest files (default: `esbmc-pytest`) |
| `--python-typecheck` | Type-check the input with `mypy --strict` and print the report |
| `--python-ignore-assertions` | Treat the program's own `assert` as assumptions, keep exception checks (unlike `--no-assertions`) |
| `--python-check-exceptions <types>` | Report only uncaught exceptions of these types (comma-separated, subclasses included); other exceptions end their path unreported |
| `--python-math-ulps <n>` | Ulps a math/NumPy transcendental result may differ from the C library (default 8; 0 = correctly rounded) |
| `--no-python-typed-lists` | Disable the int64-array storage for all-int lists (typed lists are on by default) |
| `--pytest-values-only` | Record counterexample as a `witness` list instead of a runnable test |

## Solidity Frontend Options

| Option | Description |
|--------|-------------|
| `--sol <path>` | Solidity file names |
| `--solc-bin <path>` | Path to `solc` (default: `$SOLC` or `solc` in `$PATH`) |
| `--contract <name>` | Set contract name |
| `--focus-function <name>` | Verify only the named public/external function; constructor and state initialization still run (needs `--contract` with several contracts) |
| `--no-visibility` | Verify unreachable functions |
| `--unbound` | Model external calls as arbitrary |
| `--bound` | Model inter-contract calls bounded |
| `--reentry-check` | Detect reentrancy |
| `--negating-property <fn>` | Negate assert in function |

## Safety Checks

### Enable Checks

| Option | Description |
|--------|-------------|
| `--overflow-check` | Signed integer overflow |
| `--unsigned-overflow-check` | Unsigned overflow |
| `--memory-leak-check` | Memory leaks |
| `--deadlock-check` | Deadlock detection |
| `--data-races-check` | Data race detection |
| `--data-races-check-only` | Only race checks |
| `--lock-order-check` | Lock ordering |
| `--atomicity-check` | Atomicity at assignments |
| `--nan-check` | Floating-point NaN |
| `--ub-shift-check` | UB on shift operations |
| `--clz-zero-check` | UB on `__builtin_clz` of zero |
| `--uninitialised-vars-check` | Reads of uninitialised locals (CWE-457) |
| `--unchecked-return-value-check` | Unchecked return values of fallible calls (CWE-252) |
| `--excessive-alloc-check[=bytes]` | Allocation sizes above a bound, default 1 MiB (CWE-789). Attach the value with `=`; a space-separated value is read as an input file |
| `--volatile-check` | Volatile variable accesses |
| `--restrict-check` | `restrict` pointer parameters must not alias |
| `--restrict-assume` | Assume entry-function `restrict` parameters do not alias |
| `--unknown-method-args-check` | Pointer arguments passed to unknown calls |
| `--total-stack-limit <bits>` | Bound the combined size of live stack frames |
| `--dead-store-check` | Advisory notes for dead stores (CWE-563), does not flip the verdict |
| `--dead-code-check` | Advisory notes for provably unreachable branch directions (CWE-561). A SUCCESSFUL verdict of this run is not a safety verdict: run it next to a normal verification |
| `--printf-check` | Printf pointer validation |
| `--struct-fields-check` | Struct over-read checks |
| `--is-instance-check` | Runtime isinstance |
| `--error-label <label>` | Check label unreachability |
| `--stack-limit <bits>` | Check stack limit |

### Disable Checks

| Option | Description |
|--------|-------------|
| `--no-assertions` | Ignore user assertions |
| `--no-bounds-check` | Skip array bounds |
| `--no-div-by-zero-check` | Skip div-by-zero |
| `--no-pointer-check` | Skip pointer safety |
| `--no-align-check` | Skip alignment check |
| `--no-unlimited-scanf-check` | Skip scanf overflow |
| `--no-vla-size-check` | Skip VLA size check |
| `--no-fp-conversion-check` | Skip the range check on float-to-integer conversions (rest of `--overflow-check` stays) |
| `--no-fp-overflow-check` | Skip floating-point overflow check (integer overflow checks stay) |
| `--no-library-assertions` | Ignore assertions from ESBMC's operational models, keep user ones (unsupported for Python; can hide real API misuse) |
| `--no-standard-checks` | Disable all default checks |

### Memory Options

| Option | Description |
|--------|-------------|
| `--no-abnormal-memory-leak` | Leaks on normal termination only |
| `--no-reachable-memory-leak` | Exclude reachable from leak check |
| `--force-malloc-success` | Assume malloc succeeds |
| `--force-realloc-success` | Assume realloc succeeds |
| `--malloc-zero-is-null` | malloc(0) returns NULL |
| `--max-symbolic-realloc-copy <n>` | Max symbolic realloc (default: 128) |

## Loop Control

| Option | Description |
|--------|-------------|
| `--unwind <n>` | Unwind all loops n times |
| `--unwindset <L:n,...>` | Unwind specific loops |
| `--unwindsetname <name:idx:n>` | Unwind by function and index |
| `--no-unwinding-assertions` | No unwinding assertions |
| `--partial-loops` | Permit partial loop paths |
| `--goto-unwind` | Unroll at GOTO level |
| `--unlimited-goto-unwind` | Don't unroll bounded loops |
| `--instruction <n>` | Limit executed instructions |

## Verification Strategies

### K-Induction

| Option | Description |
|--------|-------------|
| `--k-induction` | Prove by k-induction |
| `--base-case` | Check base case only |
| `--forward-condition` | Check forward condition |
| `--inductive-step` | Check inductive step |
| `--k-induction-parallel` | Run steps in parallel |
| `--k-step <n>` | k increment (default: 1) |
| `--max-k-step <n>` | Max iterations (default: 50) |
| `--base-k-step <n>` | Start base from n (default: 1) |
| `--unlimited-k-steps` | Max iterations = UINT_MAX |
| `--show-cex` | Print counterexample from step |
| `--goto-contractor` | Contractor-based refinement |

### Loop Invariants

| Option | Description |
|--------|-------------|
| `--loop-invariant` | Loop invariant + k-induction (still unwinds the loop) |
| `--loop-invariant-check` | Havoc abstraction, cost independent of the loop bound |
| `--loop-frame-rule` | Snapshot-havoc-assume frame rule (needs `--loop-invariant-check`) |
| `--synthesise-loop-invariants` | Synthesise invariants for affine counter/accumulator loops; implies `--loop-invariant-check`, `--check-vacuity`, `--multi-property` |
| `--check-vacuity` / `--no-vacuity-check` | Report UNKNOWN on vacuous discharge (default on with loop invariants) |

### Incremental BMC

| Option | Description |
|--------|-------------|
| `--incremental-bmc` | Incremental loop unwinding |
| `--falsification` | Incremental loop unwinding for bug searching (base case only, no forward condition) |
| `--termination` | Incremental assertion verification |

### Multi-Property

| Option | Description |
|--------|-------------|
| `--multi-property` | Verify all claims at bound |
| `--parallel-solving` | Solve VCCs in parallel |
| `--multi-fail-fast <n>` | Stop after n violations |
| `--keep-verified-claims` | Don't skip verified claims |
| `--all-witnesses` | After a failure, enumerate more violating inputs (implies `--multi-property`) |
| `--max-witnesses <n>` | Cap per property (default 16, 0 = unlimited) |
| `--report-unreached-claims` | List claims no path reaches within the bound as vacuously passed |
| `--proof-cache <dir>` | Reuse proofs from earlier runs (needs `--multi-property`) |

### Incremental SMT

| Option | Description |
|--------|-------------|
| `--smt-during-symex` | Incremental SMT solving |
| `--smt-thread-guard` | Check thread guard |
| `--smt-symex-guard` | Check conditional gotos |
| `--smt-symex-assert` | Check assertions during symex |
| `--smt-symex-assume` | Check assumptions during symex |

## Solver Options

| Option | Description |
|--------|-------------|
| `--z3` | Use Z3 solver |
| `--bitwuzla` | Use Bitwuzla solver |
| `--boolector` | Use Boolector solver |
| `--mathsat` | Use MathSAT solver |
| `--cvc4` | Use CVC4 solver |
| `--cvc5` | Use CVC5 solver |
| `--yices` | Use Yices solver |
| `--smtlib` | Use generic SMT-LIB solver |
| `--smtlib-solver-prog <name>` | SMT-LIB solver program |
| `--list-solvers` | List available solvers |
| `--default-solver <solver>` | Override default solver |
| `--bitwuzla-opt <name=value>` | Pass an option to Bitwuzla (repeatable) |
| `--bv` | Bit-vector arithmetic |
| `--ir` | Integer/real arithmetic |
| `--ir-ieee` | Integer/real arithmetic with real-arithmetic enclosures for floats |

### Floating-Point Encoding

| Option | Description |
|--------|-------------|
| `--round-to-nearest` / `--round-to-even` | Rounding mode (default) |
| `--round-to-plus-inf`, `--round-to-minus-inf`, `--round-to-zero` | Other rounding modes |
| `--fp-taylor-terms <n>` | Taylor terms (2 to 12, default 8) for exp/log/pow models |
| `--floatbv` | SMT FP theory (default) |
| `--fixedbv` | Fixed bit-vectors |
| `--fp2bv` | Encode as bit-vectors |

### Z3-Specific

| Option | Description |
|--------|-------------|
| `--z3-debug` | Z3 debug mode |
| `--z3-debug-dump-file <file>` | Z3 debug dump file |
| `--z3-debug-smt-file <file>` | Z3 SMT debug file |

## Analysis Options

### Interval Analysis

| Option | Description |
|--------|-------------|
| `--interval-analysis` | Enable interval analysis |
| `--interval-analysis-dump` | Dump intervals |
| `--interval-analysis-csv-dump <file>` | Dump to CSV |
| `--interval-analysis-wrapped` | Use wrapped intervals |
| `--interval-analysis-arithmetic` | Enable arithmetic |
| `--interval-analysis-bitwise` | Enable bitwise ops |
| `--interval-analysis-modular` | Enable modular arithmetic |
| `--interval-analysis-simplify` | Simplify assertions |
| `--interval-analysis-narrowing` | Enable narrowing |

### Value-Set Analysis

| Option | Description |
|--------|-------------|
| `--show-goto-value-sets` | Show value-sets for GOTO |
| `--show-symex-value-sets` | Show value-sets during symex |
| `--add-symex-value-sets` | Enable and add assumes |

### Concurrency

| Option | Description |
|--------|-------------|
| `--context-bound <n>` | Limit context switches |
| `--state-hashing` | Prune duplicate states |
| `--no-goto-merge` | Don't merge gotos |
| `--no-por` | Disable partial order reduction |
| `--all-runs` | Check all interleavings |
| `--schedule` | Use schedule recording |
| `--incremental-context-bound` | Raise the context bound each round (max `--max-context-bound`, default 20) |
| `--falsify-context-bound <n>` | Search for violations with bounds 1..n first; no proof claimed |
| `--sleep-sets` | Sleep-set pruning (experimental; pair with `--no-por`) |

### Code Coverage

| Option | Description |
|--------|-------------|
| `--assertion-coverage` | Assertion statement coverage |
| `--condition-coverage` | Condition coverage |
| `--branch-coverage` | Branch coverage |
| `--branch-function-coverage` | Branch + function coverage |
| `--k-path-coverage[=N]` | k-path witness coverage |
| `--cov-report-json` | Write `cov-report.json` |
| `--coverage-covered-set <path>` | Persist covered edges across runs (`--branch-coverage`) |

### Function Contracts

| Option | Description |
|--------|-------------|
| `--enforce-contract <fn>` | Check contract (* for all) |
| `--replace-call-with-contract <fn>` | Replace with contract |
| `--enforce-all-contracts` | Enforce every function marked `__ESBMC_contract` |
| `--replace-all-contracts` | Replace calls to every function marked `__ESBMC_contract` |

## Optimization

| Option | Description |
|--------|-------------|
| `--no-slice` | Don't remove unused equations |
| `--no-slice-name <name>` | No slicing for symbol name |
| `--no-slice-id <id>` | No slicing for symbol id |
| `--slice-assumes` | Remove unused assumes |
| `--compact-trace` | Exclude non-user assignments |
| `--simplify-trace` | Simplify trace |
| `--scalarize-structs` | Per-field variables for structs whose address is never taken |
| `--no-simplify` | Don't simplify expressions |
| `--no-propagation` | Disable constant propagation |
| `--gcse` | Common sub-expression elimination |

## Output Generation

| Option | Description |
|--------|-------------|
| `--witness-output <path>` | YAML + GraphML witness |
| `--sarif-output <path\|->` | SARIF 2.1.0 report, results annotated with CWE ids |
| `--witness-output-graphml <path>` | GraphML witness |
| `--witness-output-yaml <path>` | YAML witness |
| `--witness-producer <string>` | Witness producer metadata |
| `--witness-programfile <string>` | Program file metadata |
| `--generate-testcase` | Generate XML testcase |
| `--generate-pytest-testcase` | Generate pytest |
| `--generate-ctest-testcase` | Generate CTest |
| `--generate-html-report` | Generate HTML report |
| `--generate-json-report` | Generate JSON report |
| `--output <file>` | Output SMT-LIB format |

## Entry Point Configuration

| Option | Description |
|--------|-------------|
| `--function <name>` | Main function (default: main) |
| `--class <name>` | Class/namespace name |
| `--claim <n>` | Only check specific claim |
| `--assign-param-nondet` | Assign params to NONDET |

## Resource Limits

| Option | Description |
|--------|-------------|
| `--memlimit <limit>` | Memory limit (100m, 2g) |
| `--timeout <t>` | Time limit (300s, 5m, 1h) |
| `--memstats` | Print memory statistics |
| `--enable-keep-alive` | Keep-alive messages |
| `--keep-alive-interval <s>` | Keep-alive interval |

## Miscellaneous

| Option | Description |
|--------|-------------|
| `--no-library` | Disable abstract C library |
| `--no-arch` | Don't set up architecture |
| `--no-string-literal` | Ignore string literals |
| `--cprover` | CPROVER compatibility |
| `--enable-core-dump` | Enable core dump |
| `--enable-unreachability-intrinsic` | Enable __ESBMC_unreachable() |
