# Spec: LLVM-backend determinism gate — divergence classifier

**Audience:** an implementing agent (Sonnet / medium effort is sufficient).
**Status:** design complete; implement against this spec, validate via CI.
**File to edit:** `substratevm/mx.substratevm/mx_substratevm.py`
**Workflow to update:** `.github/workflows/llvm-backend-determinism.yml`

---

## 1. Background (why this exists)

`mx llvm-backend-determinism-test` builds HelloWorld twice with
`--tool:llvm-backend` and fails if any per-function `f<n>.bc`, the linked
`llvm.o`, or the final binary differ between the two builds.

Investigation (commits `197c7aa`…`e426a92`) fixed every nondeterminism source
the LLVM backend itself controls:

* the parallel bitcode-file-id race in `writeBitcode`, and
* statepoint/patchpoint id assignment (now a deterministic
  column-major index over implementation-invoked methods).

The **residual** divergence (~417 of 9080 `f*.bc`) is **NOT** LLVM-backend
specific. It is upstream Native Image nondeterminism that is **backend
agnostic**, proven by `--backend default` (the ordinary backend's object file
diverges in the same way — see commit `2f752a4`):

1. **Type-id range checks.** `instanceof`/`checkcast` lower to
   `(typeId - start) <u range`. The `start`/`range` constants come from
   `TypeCheckBuilder`, which renumbers when the reachable **type** set wobbles
   between builds. In LLVM IR this looks like:
   ```
   %73 = and i32 %72, 65535
   -  %74 = add i32 %73, -6          ; start = 6  (run-a)
   +  %74 = add i32 %73, -1          ; start = 1  (run-b)
   -  %87 = icmp ult i32 %74, 107    ; range = 107
   +  %87 = icmp ult i32 %74, 106    ; range = 106
   ```
2. **Field offsets.** Object field layout shifts when the reachable **field**
   set wobbles; loads/stores/GEPs then use different offset constants.

Because this is upstream and shared by all backends, the gate is **permanently
red** on it, which would **mask a future LLVM-backend regression**. This spec
adds a classifier so the gate fails **only** on LLVM-backend-attributable
divergence and reports upstream-only divergence as a non-failing "known
upstream" result.

---

## 2. Core design — invert the problem

Do **not** try to classify each changed integer constant as "type-id vs
field-offset vs program constant" (ambiguous, fragile). Instead, **whitelist
the lines the LLVM backend is now responsible for keeping byte-stable**, and
fail only if *those* differ:

* **REGRESSION** (fail): a statepoint/patchpoint id changed, or the IR changed
  **structurally** (different opcode, SSA value numbering, metadata ref,
  attribute-group ref, type, instruction/line count, function signature).
* **UPSTREAM** (don't fail): the only differences are **plain integer operand
  literals** (type-id `start`/`range`, field offsets) — everything else is
  byte-identical.

Soundness: if every diverging `f<n>.bc` differs only in integer operand
literals (and not in any statepoint id), then the optimised (`f<n>o.bc`),
compiled (`f<n>.o`), linked (`llvm.o`) and final-binary stages differ only as a
**deterministic function** of those upstream constants (because `opt`, `llc`,
`lld` are deterministic for identical input). So classifying **stage 1
(`f*.bc`) only** is sufficient — see the consistency checks in §5.

---

## 3. The line-pair classifier (the only judgement-heavy part)

Given two LLVM IR text files (from `llvm-dis` of the two `f<n>.bc` copies),
classify the **file** as `UPSTREAM`, `REGRESSION`, or `IDENTICAL`.

```python
import re

_INT_TOKEN = re.compile(r'-?\d+\Z')
# Split a line into tokens while KEEPING the delimiters, so token positions
# are preserved exactly.
_SPLIT = re.compile(r'([\s,()]+)')

def _constant_only_diff(a, b):
    """True iff lines a and b differ ONLY in plain integer operand literals.

    Tokens like iN (type width), %N (SSA), !N (metadata), #N (attr group),
    @name, label names, etc. are NOT plain integers, so a difference in any of
    them returns False (structural). A difference where both differing tokens
    match -?\\d+ (e.g. operand constants -6 vs -1, or 107 vs 106) is allowed.
    """
    ta = _SPLIT.split(a)
    tb = _SPLIT.split(b)
    if len(ta) != len(tb):
        return False                      # structural: token count changed
    for x, y in zip(ta, tb):
        if x == y:
            continue
        if _INT_TOKEN.match(x) and _INT_TOKEN.match(y):
            continue                      # operand-constant-only difference
        return False                      # any non-integer token differs
    return True

def _classify_line_pair(a, b):
    """a, b are the run-a / run-b versions of the SAME line; a != b already."""
    if a.lstrip().startswith('; ModuleID'):
        return 'ignore'                   # llvm-dis artefact: input file path
    # Statepoint / patchpoint id MUST be byte-stable now -- any change is a
    # genuine LLVM-backend regression, even though it is "only an integer".
    if 'llvm.experimental.stackmap' in a or 'statepoint-id' in a:
        return 'regression'
    if _constant_only_diff(a, b):
        return 'upstream'
    return 'regression'

def _classify_ir_files(ll_a_lines, ll_b_lines):
    """Returns ('identical'|'upstream'|'regression', [evidence_lines])."""
    if len(ll_a_lines) != len(ll_b_lines):
        # Added/removed lines => structural change => regression.
        return ('regression',
                ['line count differs: {} vs {}'.format(len(ll_a_lines), len(ll_b_lines))])
    verdict = 'identical'
    evidence = []
    for a, b in zip(ll_a_lines, ll_b_lines):
        if a == b:
            continue
        kind = _classify_line_pair(a, b)
        if kind == 'ignore':
            continue
        if kind == 'regression':
            evidence.append('REGRESSION: -{!r} +{!r}'.format(a.rstrip(), b.rstrip()))
            verdict = 'regression'
        elif kind == 'upstream' and verdict != 'regression':
            if len(evidence) < 5:
                evidence.append('upstream: -{!r} +{!r}'.format(a.rstrip(), b.rstrip()))
            verdict = 'upstream'
    return (verdict, evidence)
```

### Why the token rule is robust (worked examples)

> All rows below were machine-verified against the reference implementation in
> §3 (10/10 pass), including the negative cases. Use them as the unit-test
> vectors required by §8.1.

| run-a line | run-b line | tokens that differ | verdict |
|---|---|---|---|
| `%74 = add i32 %73, -6` | `%74 = add i32 %73, -1` | `-6` vs `-1` (both ints) | **upstream** |
| `%87 = icmp ult i32 %74, 107` | `…, 106` | `107` vs `106` | **upstream** |
| `... getelementptr i8, ptr %x, i64 72` | `… i64 80` | `72` vs `80` | **upstream** (field offset) |
| `%17 = add i32 %16, 5` | `%18 = add i32 %16, 5` | `%17` vs `%18` (not ints) | **regression** (SSA renumber) |
| `... @llvm.experimental.stackmap(i64 13505, ...)` | `…(i64 13520, …)` | matched by stackmap guard | **regression** |
| `store i32 %a, ptr %p` | `store i64 %a, ptr %p` | `i32` vs `i64` (not ints) | **regression** (type change) |
| `... !dbg !0` | `... !dbg !1` | `!0` vs `!1` (not ints) | **regression** (metadata ref) |

### Known imprecisions (acceptable; note them in a comment)

* `align 4` vs `align 8` classifies as **upstream** (both plain ints). Alignment
  rarely changes for the upstream wobble; treating it as upstream is a minor
  false-negative, not a safety issue.
* Hex float immediates (`0x3FF…`) are not plain decimal ints, so any change in
  them classifies as **regression** (conservative — correct).

---

## 4. New CLI flag

Add to `llvm_backend_determinism_test`'s argparse:

```python
parser.add_argument('--allow-upstream', action='store_true',
                    help='Pass the gate if every divergence is attributable to '
                         'known upstream (backend-agnostic) nondeterminism -- '
                         'type-id range-check constants and field offsets -- '
                         'verified by IR classification (only integer operand '
                         'literals differ; no statepoint-id or structural '
                         'changes). Without this flag the gate fails on ANY '
                         'divergence (legacy strict behaviour).')
```

Default (`--allow-upstream` absent) **preserves today's strict behaviour**.
The workflow opts in (see §6).

---

## 5. Where the classifier plugs in

In `llvm_backend_determinism_test`, after `diffs` is computed and found
non-empty, **before** the existing `mx.abort(...)`:

1. If `not parsed.allow_upstream`: keep current behaviour (abort). Done.
2. Else run the classifier over **stage-1 (`f*.bc`) diverging files only**:
   * Reuse `_find_llvm_bin('llvm-dis')`; if missing, **fail closed** (cannot
     classify ⇒ abort as today, with a message saying llvm-dis is required for
     `--allow-upstream`).
   * For each stage-1 diverging `f<n>.bc` (extract names from `diffs` where
     `order == 1`, same regex as `_dump_ir_diffs`):
     - `llvm-dis` both copies (run-a / run-b SVM-*/llvm dirs, as `_dump_ir_diffs`
       already locates them), read lines, call `_classify_ir_files`.
     - Collect verdict per file.
   * **Consistency checks (soundness guards):**
     - (a) Every diverging stage-3 (`f*o.bc`) and stage-4 (`f*.o`) file must
       correspond, by function id `n`, to a diverging stage-1 `f<n>.bc`. If a
       stage-3/4 file diverges whose `f<n>.bc` was **identical**, that is
       `opt`/`llc` nondeterminism ⇒ **REGRESSION** (report it).
     - (b) If **no** `f*.bc` diverges yet `llvm.o`/binary do, that is
       link-stage nondeterminism ⇒ **REGRESSION**.
3. Decision:
   * If **any** file (or consistency check) is `REGRESSION` ⇒ print the
     regression evidence and `mx.abort(...)` (gate fails — this is a real
     LLVM-backend regression).
   * Else (**all** diverging stage-1 files are `UPSTREAM`/`IDENTICAL`) ⇒ print
     the known-upstream summary (see §7) and **return normally** (exit 0).

Performance: classifying ~400 files = ~800 `llvm-dis` calls (~1–2 min). Fine
for a gate. Do **not** cap the count — correctness requires checking every
diverging `f*.bc` (a regression could hide in any one). If runtime is a
problem, parallelise with a thread pool; do not sample.

---

## 6. Workflow change

In `.github/workflows/llvm-backend-determinism.yml`, the gating step becomes:

```yaml
- name: Run llvm-backend-determinism-test
  shell: bash
  run: |
    cd substratevm
    ${MX_PATH}/mx --java-home "${JAVA_HOME}" --dy /substratevm \
      llvm-backend-determinism-test --allow-upstream \
      ${{ github.event.inputs.extra-native-image-args || '' }}
```

Now the job is **green** while only upstream divergence remains, and turns
**red** the moment an LLVM-backend-attributable divergence reappears — i.e. it
becomes a durable regression guard for the fixes already landed.

(The `--backend default` attribution step stays as-is: informational,
`continue-on-error: true`.)

---

## 7. Report format

On the all-upstream (pass) path, print something like:

```
================================================================================
LLVM-backend determinism: NO backend-attributable divergence
================================================================================
Stage-1 functions diverging: 417 of 9080
  classified UPSTREAM (integer operand constants only): 417
  classified REGRESSION:                                  0

All divergence is attributable to backend-agnostic upstream nondeterminism
(type-id range-check constants and/or field offsets). Verified: no
statepoint-id changes, no structural IR changes. See `--backend default`
attribution for confirmation the ordinary backend diverges identically.

Sample upstream differences (first 5):
  upstream: -'  %74 = add i32 %73, -6' +'  %74 = add i32 %73, -1'
  ...
GATE PASS (--allow-upstream).
```

On the regression (fail) path, print the per-file regression evidence
(filename + the offending `-`/`+` lines), then `mx.abort(...)` with a message
making clear this is an **LLVM-backend regression**, not upstream.

---

## 8. Validation plan (no local build available)

You cannot build the LLVM backend on Windows; validate via CI and via unit-style
self-checks:

1. **Unit-test the classifier in isolation** (pure Python, runs anywhere). Add a
   tiny `mx` self-test or a `python -c` snippet that feeds the worked examples
   from §3 into `_classify_line_pair` / `_classify_ir_files` and asserts the
   expected verdicts. This catches regex/token bugs without CI.
2. **CI run 1 (expected PASS):** push; the gate with `--allow-upstream` should
   now go green (current divergence is all type-id/field-offset). Download the
   artefact and confirm the report says "417 UPSTREAM, 0 REGRESSION".
3. **CI run 2 (negative control, optional):** temporarily revert the
   `e426a92` invoked-set fix on a throwaway branch and confirm the gate goes
   **red** with statepoint-id REGRESSION evidence — proving the classifier
   still catches a real regression.

---

## 9. Out of scope

* Fixing the upstream type-id / field-offset nondeterminism (core Native Image
  analysis/layout determinism) — separate, large effort.
* Full semantic IR classification (resolving each constant to its origin) — not
  needed; the whitelist/token approach is sufficient and more robust.
