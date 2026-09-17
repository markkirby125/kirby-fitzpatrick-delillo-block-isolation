# Delillo Block Isolation — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [How to Write a Great Sentence | Writing Tips from an English Professor](https://www.youtube.com/watch?v=vFsQgIQFtwM)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: A Unit Can Only Be Judged Off the Page

The lecture's technique is a craft practice that reads like a magic trick and is actually a measurement instrument. William Fitzpatrick's account of how a great sentence gets made contains no scene of a writer improving a sentence *in place*. The sentence is lifted out of the paragraph onto a clean surface — a blank page, a fresh sheet in the typewriter — and it is rewritten there, repeatedly, until it stands. Don DeLillo is the exemplar Fitzpatrick returns to: the sentence being worked is the only sentence in view.

Why the blank page is not a stylistic preference but a necessity:

> **A sentence read inside a paragraph is read as the paragraph's sentence, not as its own.**

In context, the neighbouring sentences do enormous unpaid work. They supply the antecedent of *it*, they carry the pacing, they hold the emotional register, they have already established what is being discussed — so a sentence with a broken subject-verb core, a dangling referent, or no weight of its own *passes* while the paragraph props it up. Remove the props and the failure becomes audible. This is why the sentence has to be seen alone: **isolation is the only condition under which the sentence's own load-bearing structure is the thing being judged.** Fitzpatrick draws the consequence sharply — a sentence that only functions because the previous sentence set it up is not a sentence, it is a fragment of a paragraph wearing a period.

Software has the identical failure, and it is the dominant failure of refactoring on dense legacy code. A function read inside a 1,400-line file is read as *the file's* function. It silently inherits:

- the module's imports and module-level singletons (`CONFIG`, `_CACHE`, `logger`),
- the file's formatting and naming conventions, which the model will imitate rather than question,
- the sibling helpers that appear to explain it,
- and, most dangerously, **the assumption that the surrounding code already resolves its ambiguities.** `state`, `ctx`, `item`, `now`, `DEFAULT_LIMITS` all look defined, because you saw their definitions (or something close enough) 300 lines up.

In a clean buffer those names are undefined, and the parser says so. That is the entire mechanism: isolation converts **hidden coupling** into **a compile error**. Ambient reads stop being a style opinion and become a list of names you must either name in the signature or consciously stub. DeLillo's blank page and a scratch file are the same device — they reduce the visible surface until only the load-bearing parts remain, and then they let the unit fail visibly.

There is a second, quieter gain: **isolation creates the control group.** Refactoring in place means the reference implementation is being edited at the same moment it is being compared against; any behavioral difference is unattributable. Pulling the block into a buffer and leaving the file untouched means the original remains runnable, unmodified, and executable as an oracle. The buffer is the experiment; the file is the control.

### The Nine Load-Bearing Definitions

1. **Block** — the single unit under judgment: one function, one method, one hot path, or one pure core plus its explicit type contract. **Exactly one unit per buffer.** Two functions in one scratchpad are two sentences on one page, and the noise is back.
2. **Closure** — the set of names the block needs from outside itself: parameters, injected ports, and every **ambient read**. The closure is a *fact to be discovered by enumeration*, never an assumption made by memory.
3. **Ambient read** — any name the block resolves from its surroundings instead of its signature: module globals, singletons, `this`/receiver state, env vars, the clock, the RNG, the filesystem, the network. Every ambient read is a hidden parameter; the count of them is the block's **noise debt at runtime**, the reason it cannot be tested as written.
4. **Buffer** (scratchpad) — a new file with a **real extension** in a scratch location, containing the block plus the minimum harness — nothing else. Real extension matters: the buffer must still be parsed, type-checked, and linted by the same toolchain, or you have isolated the block from the compiler too.
5. **Judgment view** — everything visible while you edit the block. The protocol's whole point is to make the judgment view *equal* to the block; any line in the view that is neither travelling with the block nor executing it is noise.
6. **Harness** — the minimal driver that executes the block: recorded inputs against asserted outputs, or a replay of captured production traces. Stub only the I/O boundary; keep the **types real** (see **Chomsky Vase-First Scaffolding**).
7. **Baseline** — the block's pre-refactor behavior, captured *before* the first edit as fixtures, golden output, or a replayed trace, with content hashes. Recorded, not remembered.
8. **Noise budget** — the number of non-block lines allowed in the judgment view. Default: **zero** outside the closure and the harness. Every visible line must be justified as closure (it travels) or harness (it executes).
9. **Re-insertion** — the mechanical move of the block back into its file. The re-insertion diff is a **move plus wiring only**. Any semantic change is a separate commit and therefore a separate decision.

### The Failure and the Fix, Visualized

```text
BEFORE — IN-PLACE REFACTOR  (judgment view = the whole file)

 legacy/pipeline.py                                         1,412 lines
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ import json, time, os, hashlib                        ← noise: 38 lines  │
 │ CONFIG = load_yaml("conf/prod.yaml")   ← ambient read, invisible off-page│
 │ def _helper_a(...)                     ← 210 lines of unrelated sibling  │
 │                                                                          │
 │ def reconcile(batch, state):          ← THE BLOCK (96 lines, entangled)  │
 │     ... reads CONFIG, state, time.time(), locks a global ...             │
 │                                                                          │
 │ def _helper_b(...)                     ← block's reader must hold this    │
 │ if __name__ == "__main__":             ← 200 lines of CLI argparse noise  │
 └──────────────────────────────────────────────────────────────────────────┘

  Judgment contaminated:  "state looks defined"  (it is — 300 lines up)
                          "CONFIG looks fine"     (it is — in prod only)
  Experiment contaminated: the reference implementation is being edited
```

```text
AFTER — DeLILLO BLOCK ISOLATION  (judgment view = block + closure + harness)

 scratch/isolated_reconcile.py
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ from .types import Batch, LedgerState, RetryLimits, ReconcileResult      │
 │                                                                          │
 │ def reconcile(batch: Batch, state: LedgerState, *,                     │
 │               clock: Callable[[], float],                                │
 │               limits: RetryLimits,                                       │
 │               ledger: LedgerPort) -> ReconcileResult:                    │
 │     "the block, behavior-identical, zero ambient reads"                  │
 │                                                                          │
 │ # ── harness ────────────────────────────────────────────────────────────│
 │ if __name__ == "__main__":                                               │
 │     assert reconcile(*golden_01) == expect_01                            │
 └──────────────────────────────────────────────────────────────────────────┘

 legacy/pipeline.py         ← UNTOUCHED, still runnable: the control group
 acceptance test            ← rg -n "pipeline" scratch/  →  0 hits
```

---

## 2. Core Transformation Protocols

### Protocol 1 — Name the block before you open a buffer

Write one sentence: *"The block under judgment is `<symbol>`, and its job is `<one clause>`."* If the sentence needs an "and", or the symbol is a class with eleven methods, you have not found a block — you have found a file. Candidate blocks: the function named in the incident, the function the PR touches, the method whose p99 the profiler flags. **Failure mode:** opening a scratchpad first and pulling in "everything relevant", which reproduces the file in a new location and isolates nothing.

### Protocol 2 — Discover the closure mechanically, never from memory

Three steps, in order, with no editing in between:

1. **Enumerate identifiers in the block** — `rg -nO '\b[A-Za-z_][A-Za-z0-9_]*\b' <file>` scoped to the symbol's line range, or the language server's scope view.
2. **Classify each name** — *local* (declared in the block; travels free), *closure* (needs to travel), *builtin/stdlib* (travels as an import), or *ambient* (needs Protocol 4).
3. **Emit the closure table** before writing any code:

| Name | Origin in source | Resolution in buffer | Confirmed by |
|---|---|---|---|
| `state` | parameter | parameter (typed) | signature |
| `CONFIG.retry_ceiling` | module global | `limits: RetryLimits` port | `rg -n "retry_ceiling"` |
| `time.time()` | clock ambient | `clock: Callable[[], float]` | grep of call sites |
| `_CACHE` | module singleton | `ledger: LedgerPort` | grep of mutations |
| `json.dumps` | stdlib | import | — |

**Threshold:** a closure wider than **6–8 travelling names** is a signal, not a detail — the block is doing more than one job, or it is a thin wrapper over the file. Split it and isolate the real core. **Failure mode:** skipping the table and discovering the closure via a chain of `NameError`s, which is slow, incomplete, and leaves the model guessing at signatures.

### Protocol 3 — Write the buffer from zero; never copy-then-delete

Open a new file and type the block into it. Do **not** copy the whole file and delete down to the target — deleting leaves the file's shape behind: its import block, its comment banners, its formatting, its trailing `if __name__` idiom, and with them the conventions you were trying to escape. The buffer starts empty so that every line that appears does so because the closure or the harness demanded it. This is the mechanical content of "clean page".

### Protocol 4 — Convert every ambient read into a named parameter or a port

Ambient reads are the reason the block cannot be judged. Convert them by class, not ad hoc:

| Ambient species | Example | Clean conversion |
|---|---|---|
| Module constant / config | `CONFIG.retry_ceiling` | `limits: RetryLimits` parameter |
| Mutable module singleton | `_CACHE` | `cache: CachePort` parameter, injected |
| Clock | `time.time()`, `datetime.now()` | `clock: Callable[[], float]` parameter |
| Randomness | `random.random()` | `rng: Random` parameter, seeded in harness |
| Environment | `os.environ["STAGE"]` | `stage: Stage` parameter, resolved once at the edge |
| Receiver state | `self.connection` | explicit argument; the block becomes a free function or a pure method on data |
| Filesystem / network | `open(path)`, `requests.post(...)` | port interface; real HTTP in the harness only if the network *is* the block |
| Logging / metrics | `logger.info(...)` | keep, but as an injected sink so assertions can read it |

**Rule:** an ambient read that survives isolation must appear in the closure table with a written justification. There is no "it's fine, it's a constant". **Failure mode:** converting ambient reads *silently while refactoring* — the conversion is then unreviewable, and a behavior change (e.g. who owns the retry budget) ships inside a "move".

### Protocol 5 — Stub only the boundary; keep the types and the core real

A scaffold where every collaborator returns an empty value is not an isolation, it is a mock — it cannot execute the block's logic, so it cannot tell you anything. Stub the far side of the I/O boundary (a port that records calls and returns canned data), keep the data types **real and imported**, and keep the branchy core **fully present**. The types are the block's grammar: structure is what survives when payload is stubbed (**Chomsky Vase-First Scaffolding** covers the inverse ordering).

### Protocol 6 — Capture the baseline before the first keystroke

Record the block's current behavior as fixtures plus hashes: `golden/*.json` with a `baseline_sha` per case. If no test exists and production traces are available, replay a **read-only** window of real inputs and record the outputs. The baseline is the only thing that makes "behavior-identical" a checkable claim instead of an assertion of good faith, and it is why the scratch rewrite must never be promoted to truth on its own authority ([Read-Only Vault Isolation](../../kirby-fitzpatrick-read-only-vault-isolation/SKILL.md)). **Failure mode:** refactoring first and "recording what it does now" afterwards, which certifies whatever bugs the refactor introduced.

### Protocol 7 — Edit only inside the buffer while the isolation window is open

No edits to the source file, no edits to its helpers, no opportunistic fixes. The file is the control group and must stay executable and unchanged. Enforce it mechanically:

```bash
git diff --stat -- legacy/pipeline.py     # must print nothing during the window
```

**Failure mode:** "while I'm here" edits to the file — at which point the baseline no longer corresponds to the control, and the comparison is void.

### Protocol 8 — Run the isolation test; it is the acceptance gate

```bash
# 1. no references back into the source module
rg -n "$SOURCE_MODULE" scratch/            # expected: 0 hits
# 2. the buffer compiles and lints standalone
cargo check --manifest-path scratch/Cargo.toml   # or: mypy scratch/isolated_block.py
# 3. the harness executes the block and matches the baseline
python scratch/isolated_reconcile.py       # expected: all golden cases pass
```

All three must hold. A block that "works" only when the file is importable has not been isolated — it has been copied.

### Protocol 9 — One block per buffer, and one decision per block

If you need a second function to judge, make a second buffer. If the buffer grows a second responsibility (it now also normalizes and persists), you have merged two sentences onto one page and reintroduced the paragraph.

### Protocol 10 — Measure in the buffer, with numbers and labels

In isolation the block finally *can* be measured: wall time over the golden cases, allocation counts, cyclomatic complexity, line count of the core versus the wiring. Report each number with its conditions and a status prefix — `Measured:` (with the window and the harness revision), `Inferred:` (with the model), `Unverified:`. Numbers smuggled in from an unisolated run are noise wearing a decimal point ([Calibrated Technical Tone](../../kirby-fitzpatrick-calibrated-technical-tone/SKILL.md)).

### Protocol 11 — Re-insert mechanically: move plus wiring, nothing else

The re-insertion diff must decompose into exactly two kinds of hunk:

```text
move    the block's body, byte-identical in behavior, into the file
wire    the call sites: arguments that were ambient reads become explicit
```

Review it with `git diff --find-renames` and read the *non-move* lines: if any of them changes a decision (a limit, an ordering, an error path), it belongs in a separate commit with its own rationale and its own test. Isolation is not authorization to also fix things ([Substance-First Refactoring](../../kirby-fitzpatrick-substance-first-refactoring/SKILL.md)).

### Protocol 12 — Retire the buffer deliberately; never let it become a second truth

The buffer has one of exactly three fates: **promoted** (it becomes the implementation, and the file's copy is deleted in the same commit — one owner), **preserved as evidence** (kept as a test fixture or a dated scratch artifact that is clearly non-normative), or **deleted** once re-insertion is verified. What is forbidden is the fourth: a scratch copy that stays around and is now a plausible-looking answer to "what does this actually do?" Two copies of a block is a drift generator with a timer on it.

### Protocol 13 — The STOP Signals

```text
STOP — you cannot name the block in one clause
STOP — the closure table has no row for a name you know the block reads
STOP — the closure exceeds 6–8 travelling names  (split first)
STOP — the buffer imports the source module to compile
STOP — you are editing the source file during the isolation window
STOP — the block needs the file's surrounding line numbers to be understood
STOP — the scratchpad contains a second function, or a second responsibility
STOP — you cannot say what the baseline hash is for the current behavior
STOP — the re-insertion diff contains a line that changes a decision
```

### Conversion Table: Anti-Pattern → Clean Replacement

| Anti-pattern (judgment view = the file) | What the noise hides | Clean replacement |
|---|---|---|
| "Open the file, refactor the function in place" | Ambient coupling; the block was never independently readable | Extract closure → write `scratch/isolated_<symbol>.<ext>` → edit only there |
| Copy the file, delete down to the target | The file's conventions, imports, and banners still shape the block | Start from an empty buffer; every line must earn its place |
| `CONFIG`, `_CACHE`, `self.x` used inside the block | Undeclared inputs; untestable in place | Named parameters or ports; each in the closure table |
| `time.time()` / `random()` inline | Non-reproducible behavior | `clock` / `rng` parameters, seeded by the harness |
| "It's a constant, leave it ambient" | An unowned dependency and an untestable branch | Port or parameter, or a written justification row |
| Mock everything; assert on nothing | The block never executed, so nothing was measured | Stub the I/O boundary only; keep types and core real |
| Record the baseline after refactoring | The refactor certifies its own bugs | Baseline fixtures + `baseline_sha` before the first keystroke |
| Judge readability "as it appears in the file" | Neighbours supply the block's referents | Judge the block alone, at the buffer, with the harness green |
| Two or three functions in one scratchpad | The paragraph is back; attention is split | One block per buffer, one responsibility per block |
| "While I'm here, also fix the retry limit" | A semantic change inside a move | Isolation only; separate commit, separate test |
| Re-insert by retyping from memory | Silent divergence from the judged artifact | Mechanical move of the buffer's text + a wiring-only diff |
| Scratchpad kept "just in case" after merge | Two sources of truth; drift on a timer | Promote (delete the old copy), preserve as non-normative evidence, or delete |
| Comparisons run while the file is mid-edit | No control group; differences are unattributable | File untouched during the window; `git diff --stat` empty |

---

## 3. Engineering Application Scenarios

### Scenario A — Code Reviews (reviewer-side): review the block, not the file

A reviewer reading a 900-line file's diff is reading the diff as *the file's* diff, exactly the contamination the lecture warns about. The DeLillo move in review is to **isolate the hunk under discussion into a buffer of its own** — extract the changed function with its closure, stub the boundary, run the reviewer's own pathological case — and then write the comment about the isolated block. The verdict becomes testable, and the reviewer's evidence travels with the comment instead of being asserted.

```text
ANTI-PATTERN — Review issued from inside the file
────────────────────────────────────────────────────────────────────────────────
"Looking at reconcile here in pipeline.py, it's a bit hard to follow with
everything else going on in this file. I think the retry logic might loop
forever if the ledger keeps failing? Not sure, hard to tell from the diff.
Also, is state supposed to be mutated here?"
```

```text
PATTERN — Review issued against the isolated block + harness evidence
────────────────────────────────────────────────────────────────────────────────
🔴 Blocking — unbounded retry. Isolated `reconcile` at
   scratch/isolated_reconcile.py (closure: 6 names, 0 ambient reads);
   `clock` stubbed to a constant and `ledger` failing 100%:

     python scratch/isolated_reconcile.py --case always-fail
     → 4,096 calls in 1.2 s, no exit condition     [Measured, harness @ a1c9f2e]

   `pipeline.py:141` sets `limits: RetryLimits(ceiling=None)`. Cap at 5
   attempts + jitter, or make the ceiling a required field on the type.

🟡 Non-blocking — `state` is both parameter and return path (two owners,
   one object). Isolation run shows the returned result is discarded in 3
   of 4 golden cases; prefer `ReconcileResult` as the sole output.

⚪ Nit — in the isolated block, `json.dumps` only participates in the
   logger port call; drop it from the closure if the sink is injected.

Checked and clear: separation of the batch keying (pure, 0 ambient reads,
baseline holds for golden_01..golden_14), so nothing to do there.
```

The last line is the reviewer's negative result, which is Reader's Draft material precisely because it prevents the author from re-doing the work ([Decoupled Reader Delivery](../../kirby-fitzpatrick-decoupled-reader-delivery/SKILL.md)).

### Scenario B — PR Descriptions (author-side): publish the isolation record, not the journey

A PR body is read by people bisecting at 03:00 and by you in eight months with none of your present state. What the reader needs is not the story of how you extracted the function — it is **the block-level before/after and the evidence that the move was behavior-preserving**. The isolation record is exactly that evidence, and it is cheap to publish because isolation produced it mechanically.

```markdown
## Isolation evidence — `reconcile`

Block under judgment: `pipeline.reconcile` (96 lines → 41 lines of core + ports).

### Closure (every ambient read resolved)
| Name             | Was                     | Now                          |
|------------------|-------------------------|------------------------------|
| retry ceiling    | `CONFIG.retry_ceiling`  | `limits: RetryLimits`        |
| `_CACHE`         | module singleton        | `ledger: LedgerPort`         |
| `time.time()`    | inline                  | `clock: Callable[[], float]` |
| `STAGE`          | `os.environ`            | `stage: Stage` (edge-resolved) |

Ambient reads remaining in the block: 0.`  `7 names travel, 4 converted.`  `
Harness: `scratch/isolated_reconcile.py` — 14 golden cases, stubbed ledger.

### Acceptance
* `rg -n "pipeline" scratch/` → 0 hits (block compiles with the file absent)
* `python scratch/isolated_reconcile.py` → 14/14 green against
  `golden/*.json @ baseline_sha 8f21ac`  [Measured, 2025-06-04]

### Re-insertion diff
`git diff --find-renames` → 1 move hunk (body byte-identical) + 4 wiring
hunks at the call sites. Non-move lines changing a decision: 0.
The retry-ceiling change is PR #4412, deliberately separate.
```

The PR body states the isolation boundary, the acceptance test, and the mechanical character of the re-insertion; it does **not** narrate the extraction, list the `NameError`s, or reprint the file. The scratchpad's fate is declared in the same body (promoted / fixture / deleted), which is what keeps it from becoming a second source of truth.

### Scenario C — Architecture RFCs and ADRs: the isolation appendix is the evidence

An RFC that asks reviewers to accept a mechanism while the mechanism is embedded in a service — interleaved with the service's auth, metrics, and persistence noise — is asking them to judge a sentence inside a paragraph. The isolation buffer is the appendix that lets them judge the unit, and it simultaneously supplies the "Measured" numbers that replace the "Estimated" hand-waving.

```markdown
## Status        Proposed
## Context       The v2 ledger writer duplicates postings during mid-batch
                restarts: 1,204 duplicates on 2026-01-14.
## Decision — key the write on `(tenant_id, batch_seq)`, written in-txn

### Isolation appendix (non-normative for the decision; normative for the numbers)
The hot path was extracted to `scratch/isolated_apply_batch.py` with its
closure resolved (2 ambient reads converted: clock, ledger port). The block
below executes standalone — 0 references to the service — against the
replayed incident window:

    def apply_batch(batch, ledger: LedgerPort, *, clock) -> ApplyResult:
        # identical structure; ambient reads: 0
    ...
    python scratch/isolated_apply_batch.py --replay 2026-01-14T14:00/15:00
    → replayed 1,204 duplicate postings → 0     [Measured, replay @ 3d1b77c]

## Estimated     dual-write adds ~38 % p99 write latency for 7 days (+/- 9 %)
                [Inferred — not measured in isolation; the port stub serializes]
## Consequences  reports lag ≤ 1 day for 7 days; v2 schema frozen meanwhile
## Rollback — `LEDGER_PRIMARY=v2`; re-insertion is reverted as a single move
## Blocking — 340 contract fixtures must pass on the isolated block first
## Rejected — per-writer locking on v2 (deadlocks at 12k rps in the shadow run)
```

Three things the isolation appendix enforces in an RFC: (1) reviewers judge the mechanism **alone**, with the closure table showing that nothing ambient is doing hidden work; (2) the claims that *can* be measured in isolation are labelled `Measured` with a revision, and the ones that cannot are labelled `Inferred` with their model — no decimal point is borrowed from an unisolated run; (3) the rollback story is honest, because a mechanical re-insertion has a mechanical inverse. The ADR does not ship the buffer — it ships the block and retires the scratchpad, per Protocol 12.

---

## 4. Verification Checklist

- [ ] **The block was named in one clause before the buffer existed, and one buffer held one block.** The judgment view equalled the block plus its closure plus its harness — nothing else. I verified by reading the buffer top to bottom and pointing at every line that is neither travelling with the block nor executing it: that count is 0, and no second function or second responsibility is present.
- [ ] **The closure was discovered by enumeration and published as a table, not remembered.** Every identifier in the block is classified (local / closure / stdlib / ambient); every ambient read was converted to a named parameter or port, or carries a written justification row; travelling names ≤ 6–8. `rg -n "$SOURCE_MODULE" scratch/` returns **0 hits**, and the buffer compiles, lints, and runs with the source file absent from the import path.
- [ ] **The baseline predates the first edit, and the file was untouched throughout the window.** Fixtures/golden outputs carry `baseline_sha`, and the harness reproduces them after the refactor; `git diff --stat -- <source file>` printed nothing for the whole isolation window, so the control group is intact and every behavioral difference is attributable to the block.
- [ ] **Re-insertion is a move plus wiring, and the buffer's fate is decided.** `git diff --find-renames` shows the block body as a move with behavior-identical content; every non-move line is wiring, and the count of non-move lines that change a decision is **0** (those changes live in their own commit). The buffer is promoted (old copy deleted), preserved as explicitly non-normative evidence, or deleted — never left as a parallel implementation.
- [ ] **Every claim measured in isolation carries its conditions and a status prefix.** Each number cites the harness revision and the case set (`Measured:`), or states its model (`Inferred:`) or its absence (`Unverified:`); no measurement taken while the source file was mid-edit appears anywhere in the review, PR, or RFC; and the delivered review/PR/RFC says what the block *does* and what is *left to do*, with no narrative of the extraction itself.