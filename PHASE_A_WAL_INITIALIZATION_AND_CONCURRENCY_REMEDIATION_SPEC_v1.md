# PHASE_A_WAL_INITIALIZATION_AND_CONCURRENCY_REMEDIATION_SPEC_v1

**Project:** GoldBot v8.2.5 · PHASE A
**Document type:** Technical Remediation Specification (design only — **no code**)
**Date:** 2026-11-26
**Baseline under specification:** `GoldBot_v8.2.5_PHASE_A_TARGETED_REMEDIATION_2026-11-25.zip`
SHA-256 `937517037894c99916e6b30d4b5ff3a63530e9bf9931f3b10616384f52d1758d`
**Source tree referenced:** `GoldBot_Source_v8.2.5` (299-entry `SOURCE-MANIFEST.txt`, problems: 0)
**Scope owner:** project owner (the only party that may authorize implementation)

> **Status vocabulary for this document.**
> **Fact** = observed directly in the delivered source or in a recorded run.
> **Inference** = a conclusion drawn from Facts, stated as such.
> **Proposal** = a design decision this document recommends; not implemented.
> **Assumption** = an unverified premise the design relies on.
> **Unknown** = not established by any evidence available.
> **NOT RUN / NOT VERIFIED** = explicitly not executed — never "passed".

---

## 1. Executive Summary

This document specifies — and only specifies — the remediation design for three findings raised by the
independent audit of 2026-11-26:

| ID | Finding class | One-line statement | Blast radius |
| --- | --- | --- | --- |
| **F-1** | Availability / Startup defect (pre-existing) | `PRAGMA journal_mode=WAL` can fail with `database is locked` when two processes initialize a **brand-new** SQLite file at the same time. Three adapters perform this initialization independently. | Startup of CLI, report, web and Live paths; no data corruption observed |
| **F-2** | Verification gap | The A7 multi-process test does not deterministically prove the absence of a Lost Update, and cannot detect a faithful re-introduction of the pre-fix whole-row write. | Test credibility only; **no evidence that the current implementation is defective** |
| **F-3** | Test portability gap | Three test sites hard-code `multiprocessing.get_context("fork")`, which Windows does not support; the target deployment platform is Windows VPS. | Test executability on the production platform; no source defect |

**Headline conclusions (each labelled):**

* **Fact (F-1):** the exact string `PRAGMA journal_mode=WAL` appears in exactly three source locations:
  `sqlite_store.py:160`, `sqlite_operation_ledger.py:153`, `sqlite_state_store.py:33`. No fourth location exists.
* **Fact (F-1):** `SqliteJournal.__init__` has no retry, no verification of the resulting journal mode, and does not close the connection on failure.
* **Fact (F-1):** `SqliteRiskStateStore` sets **no** `busy_timeout` at all.
* **Fact (F-2):** `claim_for_send` and `mark_submit_attempt` do **not** use `_in_immediate`; only `transition`,
  `_bump_duplicate` and `_normalise_digest` do. This corrects an imprecision in the audit brief (see §2.4).
* **Fact (F-2):** a faithful re-introduction of the pre-fix whole-row write reproduced failures in only
  3 of 40 A7 runs (7.5%) — the test is not a reliable detector.
* **Fact (F-3):** `get_context("fork")` appears at `test_phase_a_audit_multiprocess.py:66`, `:112` and
  `test_fix05_counter_semantics.py:181`.
* **Proposal:** one shared initialization helper used by all three adapters, deadline-bounded retry with
  jittered exponential backoff, explicit WAL verification, connection closed on every failure path, and a
  new fail-closed exception type.
* **Proposal:** deterministic interleaving tests built on a forced pause (no sleeps), a test-only connection
  proxy (zero production change), and a mutation harness that must fail 100% of runs.
* **Proposal:** `spawn` start method everywhere, pickle-safe module-level workers, an explicit sys.path
  preparation step, and a watchdog so a dead worker produces a diagnosable error instead of a hang.
* **Nothing in this document has been implemented. No test has been run in this round.**

**Recommended authorization sequence** (§12): Gate-1 (F-1 helper + adapter wiring) → Gate-2 (F-2 deterministic
harness) → Gate-3 (F-3 portability rewrite) → Gate-4 (independent verification). F-3 may be executed before
F-2 if Windows execution is the nearer milestone; F-2 and F-3 touch the same test files and should not be
implemented concurrently by different people (§9.4).

---

## 2. Audit Evidence and Scope

### 2.1 Evidence sources

| # | Source | Nature | Used for |
| --- | --- | --- | --- |
| E-1 | Owner-provided audit of 2026-11-26 (F-1/F-2/F-3 evidence table, 120/40/40 process-pairs) | Owner-supplied fact | Failure rates per adapter |
| E-2 | Independent audit of 2026-11-26 (`INDEPENDENT_VERIFICATION_2026-11-26.md`, `/home/user/evidence/`) | Independent measurement | Root cause, scope split (fresh vs existing WAL) |
| E-3 | Delivered source tree (this document's own `grep`/`sed` reads) | Fact | All line numbers in this document |
| E-4 | `SOURCE_DIFF_FULL.diff` from the delivery | Fact | Proves `sqlite_store.py` is unchanged by the Targeted Remediation |

### 2.2 F-1 — measured failure rates

| Adapter | Process pairs | Pairs with ≥1 failure | Rate | Source |
| --- | --- | --- | --- | --- |
| `SqliteJournal` | 120 | 56 | **46.7%** | E-1 |
| `SqliteRiskStateStore` | 40 | 22 | **55.0%** | E-1 |
| `SqliteOperationLedger` (path-based) | 40 | 23 | **57.5%** | E-1 |
| `SqliteJournal` (independent re-measure, fresh DB) | 250 | 11 | **4.4%** | E-2 |
| `SqliteJournal` (independent re-measure, DB already in WAL) | 250 | **0** | **0.0%** | E-2 |
| Bare-SQL stepwise probe (which statement fails) | 150 | 13 | **8.7% — 13/13 at `PRAGMA journal_mode=WAL`** | E-2 |

**Inference:** the two independent measurements disagree by an order of magnitude (46.7% vs 4.4%).
This is expected and not a contradiction: the rate depends on how tightly the two constructors are
aligned in time. E-1's harness starts both constructors as close together as possible; E-2's harness
passes through a barrier with more work on either side. **The rate is a function of the harness, not a
constant of the product.** This has a direct consequence for the acceptance criteria in §7: the *residual
rate we must exclude* drives the sample size, and we must choose the sample size for the smallest rate we
care about (1%), not for the largest one observed.

**Fact (root cause, E-2):** the failing statement is the **journal-mode transition itself**, not the
subsequent schema script. A stepwise probe that isolates each statement (`connect` → `journal_mode=WAL` →
`synchronous=FULL` → `busy_timeout` → `executescript`) failed 13/150 times, and **every** failure was on
`journal_mode=WAL`.

**Fact (root cause, E-2):** installing `PRAGMA busy_timeout` *before* the WAL transition does **not** fix it
(8/200 failures with the reordering versus 5/200 without). The busy handler is therefore not the mechanism
that is missing at the transition.

**Inference:** SQLite must be told the busy policy in a way that survives the mode transition, or the
transition itself must be retried by the caller. The Evidence supports the second option and does not
disprove the first. §3.3 adopts "retry the transition, then verify" and records the alternative as a
rejected option with its reason.

### 2.3 F-1 — scope of exposure (blast radius)

| Entry point | Takes `LiveInstanceLock`? | Opened file(s) | Concurrent exposure |
| --- | --- | --- | --- |
| `run_live` (`runtime/live.py:198`) | **Yes** — decorator `_with_local_live_lock` at `:175`, journal built inside `build_components()` at `:341` | journal `db_path` + `state_db_path(db_path)` (`live.py:228`) | Protected against a second *Live* process; **not** protected against CLI/web |
| `cmd_ack`, `cmd_status`, `cmd_override`, `cmd_health`, `cmd_level`, `cmd_approve`, `cmd_onboarding` (`runtime/cli.py:389,454,489,572,616,736,744`) | **No** | risk-state DB (7 construction sites), some also the journal | **Yes** |
| `cmd_report` (`cli.py:167`), `cmd_signals` (`:794`), `cmd_demo/backtest` (`_run`, `:176`) | **No** | journal | **Yes** |
| `web/app.py:210,249,503,526` (journal) and `:200` (risk state) | **No** | journal, risk state | **Yes** |
| `build_components` (`runtime/bootstrap.py:117` journal, `:247` ledger, and risk state) | Inherited from caller | journal + ledger + state | **Yes** when the caller is a CLI/web path |

**Fact:** `SqliteJournal(` is constructed at **58 sites across `src/`, `tests/`** (12 in `src/`: 1 bootstrap,
7 CLI, 4 web). There is no single factory.

**Inference:** F-1 is *not* solved by the Live lock. The Live lock reduces, but does not remove, the window;
and the CLI/report path is precisely the operator's diagnostic tool during an incident, i.e. the moment a
second process is most likely.

**Fact:** no test in the suite asserts that the journal mode is actually `wal` after initialization
(`grep -rn "journal_mode" tests/` → no match). Requirement 7 therefore has **zero existing coverage**.

### 2.4 F-2 — what the current test actually does, and a correction to the brief

**Fact:** in `tests/test_phase_a_audit_multiprocess.py` both workers construct their journal **before**
reaching the barrier:

```
36:        journal = SqliteJournal(db_path)
37:        ledger = SqliteOperationLedger(journal=journal, clock=None)
39:        go.wait(timeout=60)
...
220:        journal = SqliteJournal(db_path)
221:        ledger = SqliteOperationLedger(journal=journal, clock=clock)
...
257:        go.wait(timeout=60)
```

**Inference:** the test synchronizes only the *claim*, never the *initialization*. It therefore cannot,
by construction, detect F-1. This is consistent with E-2's observation that the A7 file passes 40/40 in a
loop while a separate fresh-DB harness fails ~4% of iterations.

**Correction to the audit brief (Fact):** the brief states that `_bump_duplicate`, `transition`,
`claim_for_send` and `mark_submit_attempt` all use targeted SQL **and** `BEGIN IMMEDIATE`. That is true for
the first two and for `_normalise_digest`, but **not** for `claim_for_send` and `mark_submit_attempt`:

| Method | Line | Targeted SQL | `_in_immediate` (BEGIN IMMEDIATE) | Concurrency safety argument |
| --- | --- | --- | --- | --- |
| `_bump_duplicate` | `sqlite_operation_ledger.py:503` | `UPDATE … duplicate_requests = duplicate_requests + 1` | **yes** (`:521`) | read-modify-write serialized |
| `_normalise_digest` | `:525` | conditional `UPDATE … WHERE op_id=? AND payload_hash=?` | **yes** (`:551`) | guarded write + re-read |
| `transition` | `:446` | full-row `UPDATE` from `_apply_transition` | **yes** (`:498`) | read-modify-write serialized |
| `claim_for_send` | `:562` | `UPDATE … SET state=?, claim_attempts=claim_attempts+1 WHERE op_id=? AND state IN (?,?)` | **no** (`:568` lock + commit) | single conditional statement is atomic by itself; `rowcount` is the CAS witness |
| `mark_submit_attempt` | `:594` | `UPDATE … submit_call_markers = submit_call_markers + 1 WHERE op_id=?` | **no** (`:601` lock + commit) | single incrementing statement is atomic by itself |

**Inference:** this correction does not create a defect — the two non-`_in_immediate` methods issue a single
statement whose increments are computed by SQLite relative to the stored value, and both are wrapped in the
instance lock plus an explicit commit. But the **specification and the tests must state the distinction**,
because a future refactor that adds a second statement to either method would silently lose the
serialization. This is recorded as design item **F-2-D7** (§4.2) and as an Out-of-Scope finding **OOS-2**.

**Fact (mutation result, E-2):** replacing `_bump_duplicate` with the pre-fix whole-row write from a stale
snapshot produced failures in **3 of 40** A7 runs (7.5%). The failures were
`IllegalTransition('illegal order transition APPROVED -> ACKED')` — i.e. the FSM refused the illegal
transition rather than a second order being sent. **Inference:** the safety property held (no double
submission), but the *test* is an unreliable detector (7.5% sensitivity).

**Fact (mutation result, E-2):** removing `_in_immediate` from `transition` produced **0 failures in 60**
A7 runs.

**Inference:** the A7 test can be green on a materially weakened implementation. F-2 is therefore a genuine
verification gap, and the brief's framing ("no evidence that the current implementation is defective") is
the correct one.

### 2.5 F-3 — exact sites

| File | Line | Call |
| --- | --- | --- |
| `tests/test_phase_a_audit_multiprocess.py` | **66** | `ctx = mp.get_context("fork")` (AU-22) |
| `tests/test_phase_a_audit_multiprocess.py` | **112** | `ctx = mp.get_context("fork")` (AU-23) |
| `tests/test_fix05_counter_semantics.py` | **181** | `ctx = mp.get_context("fork")` (crash-evidence test) |

**Fact:** `tests/` contains 86 test modules and **no `tests/__init__.py`**.

**Fact:** `conftest.py` inserts `<root>/src` into `sys.path` for the parent pytest process only. A `spawn`
child does not import `conftest.py`.

**Fact:** Python 3.13 on this Linux host reports `get_all_start_methods() == ['fork','spawn','forkserver']`.

**Fact:** `psutil` 7.2.2 was importable in this workspace, but it is **not** a project dependency
(`requirements.txt` contains only `pydantic`). Any design that needs `psutil` adds a dependency; §5.6 avoids it.

**Fact:** `pytest-timeout` and `pytest-xdist` are **not** installed. There is no global watchdog; a hung
worker hangs the suite.

**Unknown:** whether a `spawn`ed child under pytest inherits a `sys.path` that can import `goldbot`. The
child does receive the parent's `sys.path` through the spawn preparation data, so this is *likely* to work —
but the specification does not rely on it; §5.3 adds an explicit module-level path preparation so the
outcome does not depend on the answer.

### 2.6 Integrity evidence carried forward

| Check | Result | Status |
| --- | --- | --- |
| Delivery ZIP SHA-256 | `9375170378…52d1758d` | **Fact — verified 2026-11-26** |
| `SHA256SUMS.txt` | 316/316 OK (re-verified after all audit runs) | **Fact** |
| `SOURCE-MANIFEST.txt` | files: 299 \| problems: 0 | **Fact** |
| `sqlite_store.py` SHA-256 | `a74e06db297c9f9ee8d6b6d8ef6942daa6528a70cbacad44db5a0f1fd29cfcba` | **Fact — matches the brief** |
| `sqlite_store.py` present in `SOURCE_DIFF_FULL.diff` | 0 occurrences | **Fact — F-1 is pre-existing** |
| `domain/fsm/order.py` SHA-256 | `bf4154c6…f1774d0f` | **Fact — untouched** |
| Full suite on delivered tree | `1224 passed, 6 skipped, 3 warnings` | **Fact — 13 runs** |

---

## 3. F-1 Technical Root Cause and Remediation Design

### 3.1 Current code, verbatim (three constructors)

**A. `src/goldbot/adapters/sqlite_store.py:158–168`**

```python
158:  self._conn = sqlite3.connect(self.path, timeout=10.0, check_same_thread=False)
160:  self._conn.execute("PRAGMA journal_mode=WAL")
162:  self._conn.execute("PRAGMA synchronous=FULL")
165:  self._conn.execute("PRAGMA busy_timeout=10000")
166:  self._conn.executescript(SCHEMA)
167:  self._migrate_trades()
168:  self._migrate_signals()
169:  self._conn.commit()
```

**B. `src/goldbot/adapters/sqlite_operation_ledger.py:147–160`** (path-based branch only)

```python
147:  self.path = Path(path)
150:  self._conn = sqlite3.connect(self.path, timeout=10.0)
152:  self._conn.row_factory = sqlite3.Row
153:  self._conn.execute("PRAGMA journal_mode=WAL")
154:  self._conn.execute("PRAGMA synchronous=FULL")
155:  self._conn.execute("PRAGMA busy_timeout=10000")
156:  self._owns_connection = True
160:  self.migrate()          # may BEGIN IMMEDIATE (line 209) and take a backup (line 218)
```

**C. `src/goldbot/adapters/sqlite_state_store.py:30–37`**

```python
30:  self.path.parent.mkdir(parents=True, exist_ok=True)
31:  self._conn = sqlite3.connect(self.path, timeout=10.0)
32:  self._conn.row_factory = sqlite3.Row
33:  self._conn.execute("PRAGMA journal_mode=WAL")
35:  self._conn.execute("PRAGMA synchronous=FULL")
36:  self._conn.executescript(SCHEMA)
37:  self._conn.commit()
```

### 3.2 Defect inventory (each item is a Fact about the current code)

| ID | Defect | Location | Consequence |
| --- | --- | --- | --- |
| **D-1** | The `journal_mode=WAL` transition is attempted exactly once, with no retry | A:160, B:153, C:33 | Any momentary lock → immediate constructor failure |
| **D-2** | `PRAGMA busy_timeout` is set **after** the transition | A:165, B:155 | The value cannot help the statement that needs it (confirmed: reordering to before does not fix it either — §2.2) |
| **D-3** | `SqliteRiskStateStore` sets **no** `busy_timeout` at all | C (absent) | Any `SQLITE_BUSY` on a write is immediate; inconsistent policy between adapters |
| **D-4** | The *resulting* journal mode is never read back | A, B, C | A silent fallback to `delete` mode would go unnoticed; requirement 7 unmet |
| **D-5** | No `try/except` around initialization; the connection is not closed on failure | A:158–169, B:147–160, C:31–37 | Leaked file handle on every failed construction (observable as an unreleased connection object; on Windows a leaked handle can also block file deletion) |
| **D-6** | The error surfaced to the operator is a bare `sqlite3.OperationalError` | A, B, C | Not diagnosable: no path, no elapsed time, no attempt count, no guidance |
| **D-7** | Three independent copies of the same policy | A, B, C | Divergence already visible: D-3 exists in one file only (requirement 10) |
| **D-8** | The ledger's shared-connection branch (`journal is not None`, `:141–146`) inherits the journal's policy and runs `migrate()` on it | B:141–160 | Acceptable by design, but the spec must state that the shared branch performs **no** WAL transition of its own and must not be given one (§3.6) |

### 3.3 Design decision F-1-D1 — a single shared initializer

**Proposal:** add one new module, `src/goldbot/adapters/sqlite_init.py`, exposing exactly one public function:

```python
def open_sqlite_connection(
    path: str | Path,
    *,
    deadline_s: float = 30.0,
    busy_timeout_ms: int = 10_000,
    row_factory: bool = True,
) -> sqlite3.Connection
```

Contract:

1. `path.parent.mkdir(parents=True, exist_ok=True)` — same as today, no path change.
2. `conn = sqlite3.connect(path, timeout=<min(deadline_s, 10.0)>)`.
3. `conn.row_factory = sqlite3.Row` (configurable; the state store and ledger both want it).
4. **Retry loop until `deadline_s`:** attempt `PRAGMA journal_mode=WAL`; on `OperationalError` whose error
   is busy/locked, sleep `backoff(n)` and retry; on any other `OperationalError`, re-raise immediately
   (do not mask a genuinely broken file — this preserves AU-35's behaviour, see §3.5).
5. **Verify:** `PRAGMA journal_mode` must return `wal` (case-insensitive). If it returns anything else, raise
   `SqliteInitializationError` — a `PRAGMA` that did not take effect is a failure, not a warning.
6. `PRAGMA synchronous=FULL`; `PRAGMA busy_timeout=<busy_timeout_ms>`.
7. Return the connection. **On any exception in steps 2–6, `conn.close()` must be called before re-raising**
   (D-5).
8. Never delete, move, recreate or rename a database file; never switch to `:memory:`; never fall back to a
   different path (requirement 9).

**Why a shared helper and not three local patches (requirement 1):**

* The three call sites must obey *one* policy (requirement 10). Three copies already diverged once (D-3).
* The concurrency logic is subtle (deadline, backoff, verification, close-on-error). It is testable once at
  the helper level *and* once per adapter; duplicating it triples both the code and the evidence burden.
* The helper is the natural place to attach the observability required by §8 (attempt count, elapsed).

**Alternatives considered and rejected:**

| Alternative | Why rejected |
| --- | --- |
| Only raise `busy_timeout` / reorder the PRAGMAs | Disproved by measurement (§2.2): 8/200 with the reordering vs 5/200 without. Busy-handler ordering is not the mechanism. |
| Open in rollback mode and never use WAL | Changes durability/performance characteristics and contradicts the existing design (`synchronous=FULL` + WAL is load-bearing for crash evidence). Out of scope. |
| Serialize *all* access through one process / one connection | Architectural change to the ownership model — explicitly out of scope. |
| A cross-process startup lock file (like `LiveInstanceLock`) | Adds a second locking mechanism with its own failure modes (stale lock, LF/`msvcrt` divergence, recovery), and it would have to be acquired by all 58 construction sites. Retrying the one statement that actually fails is strictly smaller. |
| Retry inside each adapter | Violates requirement 10 and re-creates D-7. |

### 3.4 Design decision F-1-D2 — busy/locked handling, backoff and deadline (requirements 2, 3, 4)

**Which errors are retryable:**

| Python exception | `sqlite_errorcode` | Action |
| --- | --- | --- |
| `sqlite3.OperationalError` mentioning `database is locked` | `SQLITE_BUSY` (5) | **Retry** until deadline |
| `sqlite3.OperationalError` mentioning `database table is locked` | `SQLITE_LOCKED` (6) | **Retry** until deadline (defensive; not observed in F-1) |
| `sqlite3.OperationalError` with any other code (e.g. `file is not a database`, `unable to open`) | other | **Re-raise immediately** — retrying cannot help and would delay an honest failure |
| Any other exception | — | Re-raise immediately (after closing) |

**Implementation note (Fact):** `sqlite3.Error.sqlite_errorcode` exists in Python 3.11+, so 3.13 satisfies it.
The helper must **also** match the message text as a fallback, because `sqlite_errorcode` is not populated for
every path. Both conditions are cheap and the message check is what the audit actually observed.

**Backoff schedule (Proposal):**

```
delay(n) = min(250 ms, 10 ms * 2**(n-1)) * uniform(0.8, 1.2)     n = 1, 2, 3, …
```

**Justification, with numbers:**

* The lock is held by the *other* process only for the duration of its own WAL transition — a handful of page
  writes. E-2's probes show sub-100 ms stalls in every successful contention case. A 10 ms first step means
  the single-process (uncontended) path never sleeps at all, and a contended path wakes within one or two
  steps.
* The 250 ms cap bounds CPU churn: at the cap, a full 30 s deadline costs at most ~120 wake-ups, which is
  negligible for a startup path.
* The ±20% jitter prevents two processes from retrying in lockstep — the classic thundering-herd pattern that
  would keep re-colliding. This matters precisely because both processes start within microseconds of each
  other (E-1's harness).
* No total-attempt cap: the *deadline* is the contract (requirement 4 is "bounded", not "count-bounded"). The
  attempt count is recorded in the error so the operator can see it.

**Deadline default 30 s. Justification:**

* It must be far above the worst honest WAL transition (sub-second) — 30 s is two orders of magnitude of
  headroom.
* It must be **below the harness timeouts of the A7 test** (`go.wait(timeout=60)`, `results.get(timeout=60)`,
  `proc.join(timeout=120)`). If the deadline were, say, 120 s, a genuine deadlock would surface as a *test*
  timeout (indistinguishable from a harness bug — see the 60.10 s evidence in `A7_FAIL_30.log`) instead of as
  the new explicit error. **30 s < 60 s preserves diagnosability.**
* It is above the existing `sqlite3.connect(timeout=10.0)` so it does not change any existing failure mode.
* For `run_live` the whole start-up is already gated by a non-blocking lock; 30 s of retry inside the lock is
  acceptable and far cheaper than a failed start.

### 3.5 Design decision F-1-D3 — fail-closed error type (requirement 5)

**Proposal:** new exception `SqliteInitializationError(GoldbotError)` in `sqlite_init.py`, carrying:
`path`, `attempts`, `elapsed_s`, `deadline_s`, `last_error` (the original exception).

**Fact — a test-compatibility constraint that must not be broken:** `test_phase_a_audit_fsm_and_live.py`
AU-35 (`:224`) asserts, for a corrupted file:

```python
assert "database" in repr(excinfo.value).lower() or "ledger" in repr(excinfo.value).lower()
```

The current pass is accidental: `sqlite3.DatabaseError("file is encrypted or is not a database")` happens to
contain the word *database*. **Any new error message must contain the words "database" or "ledger"**, or AU-35
starts failing — and weakening AU-35 is forbidden by the project rules.

**Proposal:** message format

```
SQLite initialization failed for database {path}: {reason} after {attempts} attempt(s)
in {elapsed_s:.2f}s (deadline {deadline_s:.1f}s)
```

This contains "database" and names the path, the attempts and the deadline (requirement 5: transparent and
diagnosable). It also satisfies AU-35 without touching it.

**Adapter-level mapping (Proposal):**

| Adapter | Raises to its callers |
| --- | --- |
| `SqliteJournal` | `SqliteInitializationError` (new; currently `sqlite3.DatabaseError`/`OperationalError`) |
| `SqliteRiskStateStore` | `SqliteInitializationError` |
| `SqliteOperationLedger` (path branch) | `LedgerUnavailable(f"…") from SqliteInitializationError` — the ledger's public contract is already `LedgerUnavailable` (`ports/ledger.py:77`), and the chained cause preserves the detail |

**Inference:** for the ledger this keeps the port contract intact while making the cause visible in a
traceback. **Assumption A-4:** no caller does `except sqlite3.Error`, which would no longer catch the failure
at the journal level. *This must be checked during implementation (§13 Q-4); it is not claimed as verified.*

### 3.6 Per-adapter wiring (requirement 10 — all three, and only where correct)

| Adapter | Change | Must **not** change |
| --- | --- | --- |
| `SqliteJournal.__init__` (`sqlite_store.py:153–169`) | Replace lines 158–165 with `self._conn = open_sqlite_connection(self.path, row_factory=True)`, keep `executescript(SCHEMA)`, `_migrate_trades()`, `_migrate_signals()`, `commit()`. The migration steps run **after** the verified WAL transition, so `ALTER TABLE` (a write) meets an already-armed busy handler. | `SCHEMA` text, migration semantics, `check_same_thread=False`, `path.parent.mkdir` |
| `SqliteOperationLedger.__init__` path branch (`:147–160`) | Replace 151–155 with the helper (no `row_factory` duplication), set `_owns_connection = True`, keep `self.migrate()` at 160. | The shared-journal branch (`:141–146`) must **not** call the helper: it does not own the connection, and a second WAL transition on someone else's connection is exactly the kind of side effect requirement 9 forbids. `migrate()` keeps its own `BEGIN IMMEDIATE` + serialized version re-check (`:209`), which is already correct and must not be replaced. |
| `SqliteRiskStateStore.__init__` (`:27–37`) | Replace 31–36 with the helper — this **adds** the missing `busy_timeout` (D-3). | `SCHEMA`, `key` semantics, `load/save/clear` |

**Note on ordering risk (Fact):** `SqliteJournal` performs `executescript(SCHEMA)` at 166. `executescript`
issues an implicit `COMMIT` before running. Moving the busy handler earlier (via the helper) is what makes
that write safe under contention; today it is already after `busy_timeout=10000`, so no regression is expected.

### 3.7 Behaviour matrix required by requirement 8

| Scenario | Initial file state | Expected result after remediation | Test |
| --- | --- | --- | --- |
| S1 | absent (brand-new) | WAL verified, schema created, no error | T-F1-01 |
| S2 | exists, **rollback** mode, with data | transition to WAL succeeds, **row data unchanged** | T-F1-02 |
| S3 | exists, already WAL | transition is a no-op, no error, data unchanged | T-F1-03 |
| S4 | absent, **two processes** start at once | both succeed; 0 transient errors over N runs | T-F1-04, T-F1-05 |
| S5 | absent, 4 processes at once | all succeed; exactly one file set (`.db`, `-wal`, `-shm`) | T-F1-06 |
| S6 | exists, foreign connection holds `BEGIN EXCLUSIVE` beyond the deadline | fails closed with `SqliteInitializationError` within deadline + 2 s; connection closed; **no** file deleted or recreated | T-F1-07, T-F1-08 |
| S7 | same, lock released after ~1 short interval | succeeds via retry (proves the loop retries rather than merely failing fast) | T-F1-09 |
| S8 | corrupted file (not a database) | fails **immediately** (no retry storm), error contains "database", AU-35 still passes | T-F1-10, AU-35 |
| S9 | any | directory listing before/after identical except `-wal`/`-shm` | T-F1-11 |

### 3.8 Requirements-trace for §3

| Requirement | Design item | Where |
| --- | --- | --- |
| 1 shared initializer + reason | F-1-D1 | §3.3 |
| 2 BUSY/LOCKED handling | F-1-D2 | §3.4 |
| 3 bounded backoff + deadline | F-1-D2 | §3.4 |
| 4 no unbounded retry | F-1-D2 | §3.4 |
| 5 fail closed + clear error | F-1-D3 | §3.5 |
| 6 connection closed on constructor failure | F-1-D1 step 7 | §3.3, T-F1-08 |
| 7 verify WAL actually set | F-1-D1 step 5 | §3.3, T-F1-01 |
| 8 new/existing/WAL/concurrent behaviour | behaviour matrix | §3.7 |
| 9 no parallel DB / no deletion / no path change | F-1-D1 step 8 | §3.3, T-F1-11 |
| 10 same policy for all three adapters | F-1-D1 + wiring | §3.3, §3.6 |

---

## 4. F-2 Deterministic Concurrency Test Design

### 4.1 The requirement, restated precisely

A test is **deterministic** if, for a given source tree, it produces the same verdict on every run, on every
supported platform, with no dependence on scheduling. A test is a **valid detector** of a mutation if it
fails on the mutated tree. F-2 requires both properties for the Lost-Update property.

**Fact from E-2 that shapes the design:** the current A7 harness achieves 7.5% sensitivity against a faithful
mutant, and 0% against `_in_immediate` removal. **Inference:** running the same probabilistic recipe more
often is not a design; the interleaving must be *forced*.

### 4.2 The forcing mechanism (F-2-D1)

**Principle:** do not try to make two processes "collide". Instead, **stop** the code between its read and its
write, perform the competing write while it is stopped, then let it continue. The only way to do that without
touching production code is to intercept the connection's `execute` at the test boundary.

**Proposal — a test-only connection proxy (no source change):**

```python
class PausingConnection:                 # lives in tests/, never in src/
    def __init__(self, real, on_before_write): ...
    def execute(self, sql, params=()):
        if self._on_before_write and is_target_statement(sql):
            self._on_before_write(sql, params)      # may block on an Event
        return self._real.execute(sql, params)
    def __getattr__(self, name):  return getattr(self._real, name)
```

Installed with `monkeypatch.setattr(adapter, "_conn", PausingConnection(adapter._conn, hook))`. It **does not
alter any SQL** — it only delays a specific statement. Therefore the code under test is the real code.

**Sequence for the canonical test (T-F2-01):**

| Step | Actor | Action | Synchronization |
| --- | --- | --- | --- |
| 1 | Main | Create ledger `L1` on a fresh DB; `record()` an ENTRY → `APPROVED` | — (only actor) |
| 2 | Main | Install the pause hook on `L1` for the statement `UPDATE operation_ledger SET … duplicate_requests …` | — |
| 3 | **Worker thread** (or spawned process) | Calls `L1.record(<same intent>)` → enters `_bump_duplicate` → takes `BEGIN IMMEDIATE` → reads the row → **hits the hook** | Worker signals `reached` (Event A) |
| 4 | Main | Waits for Event A (deterministic; no sleep). Then, through an **independent connection** `L2`, performs `claim_for_send(op)` → state becomes `SUBMITTING`, `claim_attempts = 1`; commit. | Main signals `released` (Event B) |
| 5 | Worker | Hook returns; the guarded UPDATE executes | — |
| 6 | Main | Joins worker with an explicit timeout; then reopens the DB read-only via plain `sqlite3` and asserts | — |

**Why this interleaving is forced, not lucky:** the worker physically cannot execute the UPDATE before
step 4 completes, because it is blocked inside the hook on Event B. The advance in step 4 therefore happens
*strictly after* the read in step 3 and *strictly before* the write in step 5. There is no scheduling that can
change that order, on any OS.

**Wait — is `BEGIN IMMEDIATE` already held by the worker at step 3, which would block the main writer?**
Yes: `_bump_duplicate` calls `_in_immediate`, so the worker holds a **RESERVED** lock. `L2`'s write would then
wait on `busy_timeout` (10 s) and, worse, the test would deadlock against itself. **This is the critical
design constraint** and it must be stated plainly:

| Option | Mechanism | Verdict |
| --- | --- | --- |
| **(a)** Hook fires **before** `BEGIN IMMEDIATE` — i.e. pause between "we decided this is a duplicate" and "we take the write lock" | The pause is on `execute("BEGIN IMMEDIATE")`; the competing write completes; the worker then takes its lock and reads *fresh* state | **Cannot detect the mutant**: the mutant also re-reads fresh state after the lock, so it would not revert anything. **Rejected.** |
| **(b)** Hook fires **inside** the transaction, after the row read, and the competing writer uses a **plain `SELECT`-free direct `UPDATE`** with `busy_timeout=0` | The competing write fails with BUSY — not a reproduction of the defect | **Rejected.** |
| **(c)** Hook fires inside the transaction; the competing "advance" is performed by **directly manipulating the same connection** (i.e. the proxy's hook executes the advance SQL on the *same* connection, as if another actor had committed it) | Deterministic, no lock conflict, and it changes the row exactly as a competing process would have: the worker's in-memory snapshot is now stale, and the subsequent write lands on a row whose state differs from the snapshot | **Accepted — this is the discriminator.** |
| **(d)** Two real processes, and the pause is a file-lock handshake with the *other* process doing a real commit | Realistic, but the `BEGIN IMMEDIATE` deadlock of (a) applies unless the pause is before the transaction, which defeats the purpose | **Rejected for the deterministic test; retained as the (non-deterministic) stress lane.** |

**Proposal:** adopt **(c)** as the deterministic discriminator, and be **explicit in the test's docstring** that
it simulates a committed competing write rather than performing one across processes. This is honest: the
property under test is *"the write must not be based on a stale read"*, and (c) reproduces exactly the
database state a competing commit would have produced, at exactly the point that matters.

**Complementary real-process test (T-F2-04):** keep one two-process test that does the *whole* competition
for real (as A7 does today), but **move construction after the barrier and add the F-1 helper**; this becomes
the regression net for "the deterministic test's model matches reality". Its verdict may remain
probabilistic — and it is labelled as such (§7, "class P").

### 4.3 Test catalogue

Every test below names its **goal, initial state, actors, exact order, sync points, expected result, mutation,
and why the mutation is certainly detected.** All IDs are unique and stable.

#### T-F2-01 — stale duplicate bump cannot revert a claim *(class D)*
* **Goal:** a duplicate-request counter bump based on a stale read must not restore a sendable state.
* **Initial state:** fresh file; one ENTRY op in `APPROVED`, `claim_attempts=0`, `duplicate_requests=0`.
* **Actors:** 1 ledger instance `L1`; 1 worker thread; 1 raw `sqlite3` connection for the advance and the
  final assertions.
* **Order:** record → install pause on the `duplicate_requests` UPDATE → worker enters `record()` (duplicate
  path) → **pause** → main advances `state='SUBMITTING'`, `claim_attempts=1` on the raw connection and
  commits → release → worker's UPDATE executes → join.
* **Sync points:** two `threading.Event` objects; the worker signals *reached* immediately before its write,
  main signals *released* immediately after its commit.
* **Expected (fixed code):** final `state == 'SUBMITTING'`, `claim_attempts == 1`, and the operation is not
  claimable: a subsequent `L.claim_for_send(op)` returns `False`. `duplicate_requests` is either 1 (guarded
  update succeeded) or 0 (guarded update detected a changed row and raised `LedgerUnavailable`); **both are
  acceptable** and the test asserts the *invariant*, not a specific counter — see §4.4 on why.
* **Mutation (M-1):** replace `_bump_duplicate` with `self._update(self._apply_bump(existing))` (the pre-fix
  whole-row write).
* **Why certainly detected:** under M-1 the UPDATE is unconditional and writes `state='APPROVED'` over
  `'SUBMITTING'` with probability 1, because the stale snapshot is *guaranteed* by construction, not by luck.
  The final-state assertion then fails deterministically. (E-2 measured the same mutation at 7.5% under the
  probabilistic harness; under this harness it is 100%.)

#### T-F2-02 — stale transition cannot overwrite a newer state *(class D)*
* **Goal:** `transition()` must not write a whole row derived from a snapshot that another actor has advanced
  (specifically: `APPROVED→EXPIRED` must not overwrite `UNKNOWN`).
* **Initial state:** one op in `APPROVED`.
* **Actors:** `L1`, worker thread, raw connection.
* **Order:** pause the `UPDATE` inside `transition()` **after** its internal read; main sets `state='UNKNOWN'`
  and `last_error='restart_before_confirmation'`; release.
* **Expected:** final state `UNKNOWN`; `EXPIRED` must never appear; `claim_for_send` returns `False`.
* **Mutation (M-2):** make `transition`'s UPDATE unconditional on state (the delivered code is unconditional
  on state but is serialized by `_in_immediate`; M-2 removes the serialization *and* the pause makes the
  staleness real).
* **Why certainly detected:** identical argument to T-F2-01 — the pause is inside the write window.

#### T-F2-03 — a guarded digest normalisation cannot rewrite a changed hash *(class D)*
* **Goal:** the `WHERE op_id=? AND payload_hash=?` guard must reject a normalisation whose hash precondition
  no longer holds.
* **Initial state:** one op stored with a v1 hash; `digest_scheme='v1'`.
* **Order:** pause before the normalisation UPDATE; main rewrites `payload_hash` to a different value on the
  raw connection; release.
* **Expected:** `LedgerUnavailable` is raised to the caller of `record()`; the stored hash is the one the
  *other* actor wrote; `digest_scheme` is unchanged.
* **Mutation (M-3):** drop `AND payload_hash=?` from the guard.
* **Why certainly detected:** with M-3 the UPDATE succeeds and overwrites the other actor's hash — the
  assertion on the stored hash fails with probability 1.

#### T-F2-04 — the same competition across two real processes *(class P, complementary)*
* **Goal:** confirm the deterministic model is faithful, and keep the real-process net.
* **Actors:** 2 spawned processes, **construction after the barrier** (this is the F-1/F-2 overlap; see §5.7).
* **Expected:** exactly one `OperationClaimed` event; exactly one `claim_attempts` increment; exactly one
  submission line; final state `SUBMITTING`; `integrity_check == 'ok'`.
* **Labelled:** probabilistic. It is **not** the acceptance witness for F-2 (§7 states which tests are the
  witnesses). It runs N times and its pass criterion is stated as an *observed* rate, not as proof.

#### T-F2-05 — the safety consequence, asserted directly *(class D)*
* **Goal:** after any interleaving from T-F2-01/02, no code path may license a resend.
* **Expected:** `claim_for_send(op)` is `False`; `recover_incomplete()` does not move `UNKNOWN` to anything
  sendable; `RESOLVED_ABSENT` is unreachable without a complete query (already covered by FIX-03 tests).
* **Why this exists:** it converts "the counters look right" into "the money property holds".

#### T-F2-06 — mutation harness *(class M, meta-test)*
* **Goal:** prove, mechanically, that T-F2-01/02/03 fail on their mutants.
* **Design:** a test module that (a) copies the source tree to a tmp dir, (b) applies a scripted text mutation,
  (c) runs the three deterministic tests in a subprocess against that copy, (d) asserts **non-zero exit and
  the expected failing test IDs**.
* **Expected (fixed tree):** the harness itself passes — i.e. every mutant is caught, **5 of 5 runs** (it is
  deterministic, so one run would suffice; 5 is cheap insurance against flakiness in the *harness*).
* **Why this matters:** it turns "we believe the test would catch it" into a repeatable machine verdict, and it
  prevents a future refactor from silently decoupling the tests from the property.

#### T-F2-07 — the `_in_immediate`-removal mutation must be caught *(class M)*
* **Goal:** close the specific hole E-2 measured (0/60 detection).
* **Design:** the M-2 mutation in T-F2-02 is *precisely* the removal of serialization; T-F2-07 asserts that
  M-2 is in the mutation harness's required-catch list.
* **Expected:** caught, deterministically.

### 4.4 What the deterministic test must **not** do (and why)

| Anti-pattern | Why it is refused |
| --- | --- |
| `time.sleep(...)` to "widen the window" | Non-deterministic by construction; violates the brief. |
| Repeating a race N times and requiring "most runs pass" | A probabilistic gate cannot distinguish 1% from 0%; E-2 measured exactly this failure mode. |
| Asserting an exact `duplicate_requests` value | The delivered contract allows the guarded path either to succeed (+1) or to detect the change and raise. Pinning one of the two would over-specify the implementation and make the test brittle against a legitimate refactor. **The invariant is the assertion; the counter is reported, not mandated.** |
| Using an in-process `threading.Lock` as the "other actor" | It would prove nothing about cross-process safety; the raw `sqlite3` connection is the correct stand-in because it is a genuinely independent connection to the same file, exactly like a second process. |
| Deleting or editing the existing A7 file | Forbidden. A7 stays as-is; F-2 **adds** tests. Any change to A7 (moving construction after the barrier) is a *strengthening* and is proposed separately as **F-2-D6** with its own gate (§5.7). |

### 4.5 Design decision F-2-D7 — document the atomicity classes

**Proposal:** add a docstring block to `sqlite_operation_ledger.py` (comment only, no behaviour change) that
tabulates §2.4's table: which methods are `_in_immediate` read-modify-write, and which are single-statement
atomic. Rationale: the audit brief itself reproduced the imprecision, which is evidence that the distinction is
not discoverable from the code. **This is a source change (comments only) and therefore still needs Gate-2.**

---

## 5. F-3 Windows Portability Design

### 5.1 Exact files and lines in scope

| File | Line(s) | Change |
| --- | --- | --- |
| `tests/test_phase_a_audit_multiprocess.py` | 66, 112 | `mp.get_context("fork")` → a shared `get_mp_context()` returning `spawn` on every platform |
| `tests/test_fix05_counter_semantics.py` | 181 | same |
| `tests/test_phase_a_audit_multiprocess.py` | 36–37, 220–221 / 39, 257 | move journal/ledger construction **after** `go.wait()` (strengthening; separate gate) |
| new `tests/mp_support.py` (no `__init__.py`; importable because pytest inserts the test file's directory on `sys.path`) | — | context factory, worker-exit watchdog, safe teardown |

### 5.2 Design decision F-3-D1 — `spawn` everywhere, not "fork on Linux, spawn on Windows"

**Proposal:** one start method, `spawn`, on both platforms.

**Justification:**
* A test that runs a different mechanism on the developer's machine than on the target machine is not evidence
  about the target. Windows VPS is the stated deployment platform; `spawn` is the only common denominator.
* `fork` **inherits the parent's open SQLite connections**, which is undefined behaviour for SQLite and is a
  latent source of false passes. `spawn` forces a clean process, which is also what the product does in
  production.
* Cost: `spawn` is slower (interpreter re-import per worker, ~0.3–0.8 s). Measured cost matters for the
  acceptance sample size (§7) and is accounted for there.

**Fact that constrains this choice:** under `spawn`, the worker function is re-imported **by module name**. With
no `tests/__init__.py`, the module name is the bare file stem (e.g. `test_phase_a_audit_multiprocess`), so the
child must be able to import that file. pytest's import mode (default `prepend` with `rootdir` insertion) makes
this work for the parent; the child receives the parent's `sys.path` in the spawn preparation data, so it
*should* also work. **Assumption A-3: it does.** §5.3 removes the dependency on the answer.

### 5.3 Design decision F-3-D2 — worker contract (picklability and imports)

Rules every worker in the in-scope files must satisfy:

1. **Module-level function**, never a closure, lambda, bound method, or local class. (Today: all three workers
   already satisfy this — `_claim_worker`, `_coordinator_worker`, `_crash_worker`. **Fact.**)
2. **Arguments picklable and minimal:** `str` paths, `str` stages, Manager proxies, `int`s. `pathlib.Path` is
   picklable but `str` is used for uniformity. (Today: satisfied. **Fact.**)
3. **No inheritance of parent state.** Nothing may rely on a module-level mutable that the parent mutated after
   import — under `spawn` the child re-imports the module and sees the pristine value.
   **Fact:** the in-scope files use module-level *constants* (`SCOPE`, `INTENT`, `DIGEST`) — safe, because they
   are immutable and identical on both sides.
4. **An explicit import guard at the top of each test module that spawns:**

   ```python
   import sys
   from pathlib import Path
   _SRC = Path(__file__).resolve().parents[1] / "src"
   if str(_SRC) not in sys.path:
       sys.path.insert(0, str(_SRC))
   ```

   This executes in the parent *and* in every spawned child (the child re-imports the module), making the
   `goldbot` import independent of `conftest.py` and of `pytest.ini`'s `pythonpath`. **This is the concrete
   mitigation for A-3 and is mandatory.**
5. **`if __name__ == "__main__":`** — **Fact/correction:** under pytest it is **not** required, because the
   module is imported by the pytest entry point, not executed as `__main__`. The commonly cited rule applies to
   scripts that start processes at module import time. **Proposal:** add the guard anyway in any standalone
   reproduction script shipped under `tools/`, and state in the docstring *why* it is not needed under pytest —
   so a future reviewer does not "fix" a non-problem.

### 5.4 Design decision F-3-D3 — synchronization primitives that survive `spawn`

| Primitive | Under `spawn` | Verdict |
| --- | --- | --- |
| `multiprocessing.Barrier` (plain) | **Not shareable** across `spawn` (no handle inheritance) | **Forbidden** |
| `Manager().Barrier(n)` | Proxy — picklable, works under `spawn` | **Required** (this is what the code already uses — **Fact**, `test_phase_a_audit_multiprocess.py:68,114`) |
| `Manager().Queue()` | Proxy — picklable | **Required** (already used) |
| `multiprocessing.Event()` (plain) | Not shareable under `spawn` | **Forbidden** for cross-process tests; fine for **threads** (T-F2-01..03 use threads + `threading.Event`) |
| `Manager().Event()` | Proxy | Allowed where a cross-process event is genuinely needed |

**Inference:** because the in-scope tests already use `Manager().Barrier` and `Manager().Queue`, switching to
`spawn` is a one-line change per site *plus* the import guard. **This is the cheapest version of F-3 and should
be stated as such** — it is not a rewrite.

### 5.5 Design decision F-3-D4 — determinism of the *harness*, per-platform

Under `spawn` the child re-imports the test module. Any module-level code in that module runs in the child. The
in-scope modules must therefore have **no module-level side effects** (no file creation, no DB opening, no clock
reads). **Fact:** currently satisfied. **Proposal:** add an assertion-style test (T-F3-06) that imports each
in-scope module in a subprocess and asserts it performed no I/O in a fresh tmp cwd — cheap insurance against a
future edit.

### 5.6 Design decision F-3-D5 — bounded teardown, no hang, no dependency

**Facts:** no `pytest-timeout`; no `psutil` in `requirements.txt`; a dead worker currently manifests as a
multi-second stall and, in E-2's evidence, a 60.10 s red run whose message was `BrokenBarrierError` — i.e. the
harness consumed the failure signal instead of reporting the cause.

**Proposal — `tests/mp_support.py`:**

| Function | Behaviour |
| --- | --- |
| `get_mp_context()` | returns `mp.get_context("spawn")`; single source of truth |
| `run_workers(ctx, target, args_list, *, join_timeout, result_timeout)` | starts all, joins each with `join_timeout`, and **on timeout calls `terminate()` then `join(5)` then `kill()`**; returns `[(exitcode, payload_or_None)]`; raises a diagnostic `HarnessError` naming which worker died and with what exit code |
| `finish(manager)` | `manager.shutdown()` in a `finally:` — today the managers are not explicitly shut down; **Fact** |
| `assert_clean_exit(results)` | fails with `exitcode` and captured `repr` when a worker reports `error` |

**Thresholds (Proposal, justified):**
* `join_timeout = 120 s` — unchanged from today (Fact: `proc.join(timeout=120)`), high enough for `spawn`
  import cost on a loaded CI box, low enough to fail within a CI job.
* `result_timeout = 60 s` — unchanged (Fact: `results.get(timeout=60)`).
* **The ordering constraint that matters:** F-1's deadline (30 s) must stay **below** `result_timeout` (60 s),
  so a lock-timeout surfaces as `SqliteInitializationError`, not as a queue timeout. §3.4 records this.
* Terminate→kill escalation is **not a sleep loop**: `join(5)` is a bounded wait on a process that has been
  sent SIGTERM; if it is still alive, `kill()` is unconditional. This terminates in ≤5 s or not at all
  (and `kill()` cannot be ignored).

**No `psutil`.** Everything above is stdlib (`Process.terminate`, `Process.kill`, `Process.exitcode`). This keeps
`requirements.txt` unchanged, which matters because the project deliberately keeps it at one line.

### 5.7 F-2 / F-3 overlap — the recommended single change to A7

**Fact:** moving `SqliteJournal(...)`/`SqliteOperationLedger(...)` from before the barrier to after it would
convert A7 from a claim-only test into a **claim + initialization** test, i.e. it would start detecting F-1.

**Proposal F-2-D6:** move construction after `go.wait()` in AU-22 (`:36–37`) and AU-23 (`:220–221`), and keep
every existing assertion unchanged. This **strengthens** the test (it now exercises strictly more), so it does
not violate the "never weaken" rule.
**Caveat (Fact → Inference):** once the F-1 helper is in place, this change is safe. **Implementing it *before*
F-1** would make A7 flaky at ~4–45% — which would look like a regression introduced by the test change. **The
ordering in §10 is therefore not optional: F-1 first.**

### 5.8 Windows-specific risks that this spec does **not** resolve

| Risk | Status |
| --- | --- |
| Behaviour of `PRAGMA journal_mode=WAL` under Windows file locking (the real F-1 rate on Windows) | **Unknown / NOT RUN.** All measurements are Linux. The design is platform-neutral (retry + verify), but the *rate* and the *deadline adequacy* on Windows are unverified. |
| `msvcrt` vs `fcntl` in `LiveInstanceLock` (`live_instance_lock.py:44–62`) | **Out of scope for this spec** (not a test-portability issue). Recorded as OOS-6. |
| Antivirus/indexer holding the DB file open on a Windows VPS | **Unknown.** Would manifest as a longer-lived BUSY; the 30 s deadline is the mitigation and the failure is explicit. |
| `spawn` cost on a small Windows VPS (2 vCPU class) | **Unknown.** Affects the wall-clock of the acceptance run (§7.2), not its verdict. |
| Path-length / unicode path handling on Windows | **Unknown**, unchanged by this spec. |

---

## 6. Source File Impact Matrix

| File | Change type | Lines (current) | Requirement source | Risk | Gate |
| --- | --- | --- | --- | --- | --- |
| `src/goldbot/adapters/sqlite_init.py` | **NEW** | — (~120 lines expected) | F-1-D1/D2/D3 | Low — additive | 1 |
| `src/goldbot/adapters/sqlite_store.py` | modify | 153–169 | F-1 §3.6 | **Medium** — 58 construction sites depend on it; hash `a74e06db…` will change | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | modify | 147–160 (path branch only) | F-1 §3.6 | Medium — ledger is on the order path | 1 |
| `src/goldbot/adapters/sqlite_state_store.py` | modify | 27–37 | F-1 §3.6 (adds missing `busy_timeout`) | Low — single-row store | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | comment-only | docstring near `:440` | F-2-D7 | None | 2 |
| `tests/test_f1_wal_initialization.py` | **NEW** | — | F-1 acceptance | Low | 1 |
| `tests/test_f1_wal_initialization_slow.py` | **NEW** (N=300 lane, env-gated like `GOLDBOT_SOAK`) | — | F-1 acceptance §7 | Low | 1 |
| `tests/test_f2_deterministic_lost_update.py` | **NEW** | — | F-2 | Low | 2 |
| `tests/test_f2_mutation_harness.py` | **NEW** | — | F-2-D6 / T-F2-06 | Medium — copies and mutates source in tmp | 2 |
| `tests/mp_support.py` | **NEW** | — | F-3 | Low | 3 |
| `tests/test_phase_a_audit_multiprocess.py` | modify (strengthen) | 36–37, 66, 112, 220–221 | F-3-D1, F-2-D6 | **Medium** — an existing audit test | 3 (after 1) |
| `tests/test_fix05_counter_semantics.py` | modify | 181 (+import guard) | F-3-D1 | Low | 3 |
| `SOURCE-MANIFEST.txt` | regenerate | — | project rule | — | 4 |
| `docs/adr/` | **NEW ADR** | — | decision record for §3.3/§3.4 | None | 1 |

**Explicitly NOT touched (non-goals, §9):** `domain/fsm/order.py`, `ports/ledger.py` contracts, `app/order_coordinator.py`,
`adapters/mt5_broker.py`, `runtime/*`, `web/*`, any schema/DDL, any `magic`/risk/strategy file, `requirements.txt`.

**Fact to carry into implementation:** after Gate-1, `sqlite_store.py`'s SHA-256 will no longer be
`a74e06db…`. That hash is the audit's proof that F-1 was pre-existing; the implementation report must record
both the old and new hashes so the "pre-existing" claim remains auditable.

---

## 7. Test Matrix and Acceptance Criteria

### 7.1 Test classes

| Class | Meaning | May serve as acceptance witness? |
| --- | --- | --- |
| **D** — Deterministic | Forced interleaving; same verdict every run; mutation sensitivity 100% | **Yes** |
| **P** — Probabilistic | Race repeated N times; verdict depends on scheduling | **No** — reported as a rate, never as proof |
| **S** — Stress/sampling | N repetitions to bound a *rate* | Yes, for F-1 (with §7.2's power argument) |
| **M** — Meta/mutation | Proves another test detects a mutation | **Yes**, for the mutation claim |
| **X** — Not runnable here | Windows/PowerShell/MT5 | **No** — carries `NOT RUN` |

### 7.2 Sample size — why 300, and why 40 is not enough

Let *p* be the residual failure probability of one two-process initialization. A run of *n* iterations detects
nothing with probability (1−p)ⁿ. To claim "residual rate ≤ 1% at 95% confidence" we need (1−0.01)ⁿ ≤ 0.05,
i.e. **n ≥ ln(0.05)/ln(0.99) ≈ 298**, so **n = 300**, and **zero failures in 300** is the acceptance condition.

| n | Smallest residual rate detectable with 95% power | Verdict |
| --- | --- | --- |
| 10 | ~26% | useless |
| 40 | **~7.2%** | **insufficient** — E-1 measured 46.7%, but E-2 measured 4.4%; a 4.4% regression would be missed |
| 100 | ~3.0% | marginal |
| **300** | **~1.0%** | **acceptance lane** |

**Cost (Proposal):** with `spawn` at ~0.5–0.8 s per iteration pair, 300 iterations ≈ 150–240 s per adapter per
scenario; three adapters × one primary scenario ≈ **8–12 minutes**. That is too slow for the default test lane
(the whole suite currently runs in ~21 s), so the design splits the lanes:

| Lane | Trigger | n | Purpose | Gate |
| --- | --- | --- | --- | --- |
| `fast` (always on) | default pytest run | 40 per adapter, 1 scenario | smoke; catches gross regressions | — |
| `acceptance` (opt-in) | `GOLDBOT_WAL_ACCEPTANCE=1` | **300** per adapter, S4; plus 100 for S5 (4 processes) | the F-1 acceptance witness | Gate-4 |
| `deadline` (always on) | default | 3 | deterministic fail-closed proof (S6/S7) | — |

**Why the fast lane is not allowed to be the witness:** §7.2's table. The document states this so nobody later
cites a 40-iteration pass as proof.

### 7.3 F-1 test matrix

| ID | Class | Scenario | Setup | Assertions | Acceptance |
| --- | --- | --- | --- | --- | --- |
| T-F1-01 | D | S1 new DB, single process, ×3 adapters | tmp dir, absent file | no exception; `PRAGMA journal_mode` == `wal`; expected tables exist | 3/3 pass |
| T-F1-02 | D | S2 existing rollback-mode DB with 100 rows | pre-seeded raw DB; checksum of rows | mode becomes `wal`; row count 100; column values byte-identical | pass |
| T-F1-03 | D | S3 existing WAL DB | pre-seeded with WAL already on | no exception; no data change; no rebase | pass |
| T-F1-04 | S | S4 two processes, new DB | `spawn`, Manager barrier, construction **after** barrier | all constructors return; final mode `wal`; exactly 1 table set | **0 failures / 300** (acceptance lane) |
| T-F1-05 | S | S4 same, **without** the fix | — | (mutation/negative control) | ≥1 failure in 300 — proves the lane is sensitive |
| T-F1-06 | S | S5 four processes | as T-F1-04 | as T-F1-04 | **0 / 100** |
| T-F1-07 | D | S6 lock beyond deadline | foreign conn holds `BEGIN EXCLUSIVE` on a rollback-mode file for `deadline+10s` | `SqliteInitializationError` raised; `elapsed ≤ deadline + 2s`; message contains path + attempts + deadline | pass |
| T-F1-08 | D | S6, connection leak | patch `sqlite3.connect` in the helper's module to record objects | the recorded connection raises `ProgrammingError` ("closed") — i.e. it was closed; **0 leaked / 100 failing constructions** | pass |
| T-F1-09 | D | S7 short lock | foreign conn holds the lock for ~0.3 s | constructor succeeds; attempts recorded ≥2 (proves retry, not luck) | pass |
| T-F1-10 | D | S8 corrupted file | bytes `b"not a sqlite database"*64` | fails **fast** (`elapsed < 2 s`, attempts == 1); message contains "database"; **AU-35 unchanged and still passing** | pass |
| T-F1-11 | D | S9 no stray files | snapshot `set(os.listdir(tmp))` before/after | removed/added set ⊆ {`<name>-wal`, `<name>-shm`}; no new `.db`; no deletion | pass |
| T-F1-12 | D | requirement 7, mode verification | patch `PRAGMA journal_mode` to return `"delete"` | `SqliteInitializationError` raised — a PRAGMA that did not take effect is a failure | pass |
| T-F1-13 | D | ledger shared-connection branch | `SqliteOperationLedger(journal=J)` | the helper is **not** called; journal's mode untouched; `migrate()` still serialized | pass |
| T-F1-14 | X | Windows | — | — | **NOT RUN** |

### 7.4 F-2 test matrix

| ID | Class | Property | Mutation it must catch | Sensitivity |
| --- | --- | --- | --- | --- |
| T-F2-01 | D | stale bump cannot revert a claim | M-1 whole-row write | **100%** (forced) |
| T-F2-02 | D | stale transition cannot overwrite a newer state | M-2 unserialized transition | **100%** |
| T-F2-03 | D | guarded normalisation rejects a changed hash | M-3 dropped `payload_hash` guard | **100%** |
| T-F2-04 | P | real two-process claim | — | reported as rate only |
| T-F2-05 | D | no resend is ever licensed | — | 100% |
| T-F2-06 | M | the harness catches M-1/M-2/M-3 | itself | 5/5 runs must fail the mutant |
| T-F2-07 | M | `_in_immediate` removal is caught | M-2 | included in T-F2-06's required list |

**Acceptance for F-2:** T-F2-01/02/03/05 pass on the unmutated tree **and** T-F2-06 fails on every listed
mutant, in 5 of 5 runs, on both a Linux run and a Windows run.
**Explicitly not claimed:** that T-F2-04 passes "usually" means anything. It is reported with its N and its
failure count.

### 7.5 F-3 test matrix

| ID | Class | Assertion | Platform |
| --- | --- | --- | --- |
| T-F3-01 | D | `get_mp_context().get_start_method() == "spawn"` on both platforms | Linux ✅ / Windows **NOT RUN** |
| T-F3-02 | D | every worker target is module-level and pickles: `pickle.dumps(target)` succeeds | both |
| T-F3-03 | D | a worker that raises produces a diagnosable failure, not a hang (exit code surfaced) | both |
| T-F3-04 | D | a worker that hangs is terminated: `run_workers(..., join_timeout=2)` returns/raises within ≤ 2+5+ε s | both |
| T-F3-05 | D | no plain `Barrier`/`Event` is used in a multiprocess test (static grep + a Manager-proxy assertion) | both |
| T-F3-06 | D | importing each in-scope test module in a subprocess performs no I/O (module-level purity) | both |
| T-F3-07 | D | `Manager` is shut down in `finally` (no dangling process at test end; check `mp.active_children()` empty) | both |
| T-F3-08 | X | the full three test files under `spawn` on Windows | **NOT RUN** |

**Acceptance for F-3:** T-F3-01..07 pass on Linux, **and** the three files (AU-22, AU-23, crash test) pass
under `spawn` on Linux 20 consecutive runs with 0 failures; **Windows execution remains NOT RUN in this
document and must be executed on the target Windows VPS before F-3 can be called complete.**

---

## 8. Failure Handling and Observability

| Condition | Detected by | Behaviour | Operator sees | Test |
| --- | --- | --- | --- | --- |
| Transient BUSY (< deadline) | retry loop | retries with jittered backoff; succeeds | nothing | T-F1-09 |
| Persistent BUSY (> deadline) | deadline | **fail closed**; connection closed; no fallback, no path change | `SqliteInitializationError` with path, attempts, elapsed, deadline, chained cause | T-F1-07/T-F1-08 |
| WAL not actually applied | post-transition verify | fail closed | same error, reason `journal_mode is 'delete', not 'wal'` | T-F1-12 |
| Corrupt / non-DB file | immediate re-raise | fail fast (< 2 s), no retry storm | error containing "database"; AU-35 still green | T-F1-10 |
| Worker dies | `run_workers` | terminate→join(5)→kill; raise `HarnessError` naming worker + exit code | test failure that says *which* worker and *why* | T-F3-03 |
| Worker hangs | `join_timeout` | same; test fails in bounded time | ditto | T-F3-04 |
| Deterministic interleaving not reached | Event `wait(timeout=…)` in the *test* | test fails with "hook never fired" — distinguishes "code path changed" from "assertion failed" | explicit message | T-F2-01 |
| Mutation not caught | mutation harness | harness fails | says which mutant escaped | T-F2-06 |

**Proposal — one log line per initialization (INFO, structured):** `path`, `attempts`, `elapsed_ms`,
`journal_mode_after`, `created` (bool: file absent before). Rationale: F-1 is a *rate*; without this line the
operator has no way to learn that their deployment is contending. This uses the project's existing structured
logging convention (`log.warning(..., extra={"fields": {...}})` as seen in `runtime/live.py:361`) — **Fact that
the convention exists**; the exact logger name is an implementation detail (§13 Q-6).

**Proposal — an explicit `migration_report()["journal_mode"]` key** is **NOT** proposed: `migration_report()` is
a ledger-schema artefact and overloading it with connection state would couple two concerns. Recorded as Q-7.

---

## 9. Risks and Non-Goals

### 9.1 Risks of the proposed F-1 change

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| The new exception type breaks an existing `except sqlite3.Error` | **Unknown — A-4** | Check every `except` on the journal path during Gate-1 (§13 Q-4); keep the chain (`from exc`) so tracebacks stay informative |
| 30 s deadline is too short on a loaded Windows VPS | Medium | Deadline is a parameter with a documented default; T-F1-07 pins the *contract* (must fail at deadline), not the value; Q-3 asks the owner to confirm 30 s |
| Retry masks a genuine permanent lock and turns a fast failure into a 30 s hang | Medium | Only busy/locked codes retry; corrupt/unopenable fail fast (T-F1-10); attempts are logged |
| Adds one more file to the adapter layer | Low | It replaces three divergent copies; net complexity decreases |
| `sqlite_store.py` hash changes, weakening the "pre-existing" narrative | Low | Record old+new hashes in the implementation report (§6) |

### 9.2 Risks of F-2

| Risk | Mitigation |
| --- | --- |
| The proxy's "simulated competing write" is judged unrealistic | Documented explicitly in the test docstring; complemented by the real-process T-F2-04; the property being asserted is stale-read safety, which the simulation reproduces exactly |
| Pausing inside a transaction holds a RESERVED lock for the duration of the pause | The pause is bounded by two Events in the same process (milliseconds); the *test* asserts the pause fired; there is no sleep |
| The mutation harness copies and mutates source — slow, and a path bug could mutate the wrong tree | It must copy to `tmp_path` and **assert** it is not operating under the repository root; add to §13 Q-5 |

### 9.3 Risks of F-3

| Risk | Mitigation |
| --- | --- |
| `spawn` child cannot import the test module or `goldbot` | The mandatory module-level sys.path guard (§5.3 rule 4) makes it independent of pytest's path handling |
| Test wall-clock grows 3–5× | Accept; the acceptance lane is opt-in; the fast lane stays small |
| Moving construction after the barrier makes A7 flaky **if implemented before F-1** | §10 enforces F-1 → F-3 ordering; this is the single most important sequencing constraint in this document |

### 9.4 Non-goals (explicitly out of scope for any implementation of this spec)

Order Coordinator architecture · trading/strategy/indicator logic · broker/MT5 behaviour · schema or migration
changes · database path or name changes · deletion/recreation of any real database · MT5/broker connection ·
demo or real orders · VPS configuration · deployment to Windows VPS · operational soak · PHASE B · PHASE C ·
any production-readiness claim · `requirements.txt` changes · `LiveInstanceLock` semantics.

**One schema-adjacent proposal that is explicitly NOT authorized:** none. This spec proposes no DDL change at
all. If implementation finds one is needed, it must stop and request a separate gate (requirement 7 of the
brief's out-of-scope list).

---

## 10. Implementation Sequence

| Step | Content | Gate | Blocking constraint |
| --- | --- | --- | --- |
| 0 | Owner reads this spec and answers §13 Q-1…Q-7 | **Gate-0** | nothing proceeds without it |
| 1 | `sqlite_init.py` + unit tests T-F1-01/02/03/07/08/10/11/12 | 1 | — |
| 2 | Wire the three adapters (§3.6); full suite must stay `1224 passed, 6 skipped` | 1 | **No test may be edited in this step** |
| 3 | Add `test_f1_wal_initialization.py` (T-F1-04..09,13) + slow lane | 1 | T-F1-05 negative control must fail on the pre-fix tree |
| 4 | Run the acceptance lane (300×3) on Linux; record the exact command and output | 1→4 | — |
| 5 | F-3 `mp_support.py`, `spawn`, import guard, teardown | 3 | **Step 2 must be merged first** |
| 6 | F-2-D6: move construction after the barrier in AU-22/AU-23 | 3 | **Only after step 2** (else A7 becomes flaky) |
| 7 | F-2 deterministic tests + mutation harness | 2 | New files only |
| 8 | F-2-D7 docstring; regenerate `SOURCE-MANIFEST.txt`; ADR | 2 | — |
| 9 | Independent verification (fresh extraction of the new ZIP, hash, manifest, full suite) | 4 | — |
| 10 | Windows execution of T-F3-08 + T-F1-14 | 4 | **Requires the Windows VPS; NOT RUN otherwise** |

**Hard sequencing rules:**
* **F-1 before F-3's A7 edit.** Reversing this makes an existing audit test intermittent and would be
  indistinguishable from a regression.
* **No test file is edited in steps 1–3.** The F-1 evidence must exist before any test harness changes, or the
  "before" measurement is lost.
* **One ZIP per gate**, not one ZIP at the end, so each gate is independently verifiable.

---

## 11. Independent Verification Plan

The verifier must be able to reproduce every claim without trusting this document.

| # | Check | Command / method | Expected |
| --- | --- | --- | --- |
| V-1 | Delivery integrity | `sha256sum <zip>`; `sha256sum -c SHA256SUMS.txt` | matches the gate's recorded hash; 100% OK |
| V-2 | Manifest integrity | `SOURCE-MANIFEST.txt` recomputation | `files: <n> \| problems: 0` |
| V-3 | No weakened tests | `git`-free diff of the test tree against the previous ZIP; assert **no removed assertion** in pre-existing files | only the disclosed strengthenings |
| V-4 | Baseline suite | `GOLDBOT_SOAK=0 python3 -m pytest -q` | `1224 + <new tests> passed, 6 skipped` |
| V-5 | F-1 acceptance | acceptance lane, n=300, 3 adapters | 0 failures; the command and raw log are attached |
| V-6 | F-1 negative control | run the **same** lane against the pre-fix tree | **>0 failures** — otherwise the lane proves nothing |
| V-7 | F-1 deadline | T-F1-07 | error at deadline+≤2 s; connection closed |
| V-8 | F-2 determinism | T-F2-01..03 run 20 times consecutively | **20/20 identical verdict** (not "usually") |
| V-9 | F-2 mutation | T-F2-06 harness | every listed mutant caught, 5/5 |
| V-10 | F-3 start method | T-F3-01 + grep for `"fork"` | zero occurrences in the in-scope files |
| V-11 | F-3 hang safety | T-F3-04 | bounded termination |
| V-12 | Windows | T-F1-14, T-F3-08 | **NOT RUN** unless a Windows VPS is provided; must be stated as such |

**Rule:** V-6 is mandatory. An acceptance lane that is not shown to be *sensitive* to the defect is not
evidence of its absence. E-2's data proves this is a real risk: the same A7 file was 0/40 red on a tree that
was later shown to contain a reproducible 4.4% startup failure.

---

## 12. Explicit Authorization Gates

| Gate | Authorizes | Does **not** authorize | Evidence required to close |
| --- | --- | --- | --- |
| **Gate-0** | Nothing. Reading and commenting on this spec. | any edit | owner's answers to §13 |
| **Gate-1** | `sqlite_init.py` + the three adapter wirings + F-1 tests + F-1 DDL-free changes | F-2, F-3, any test edit, any schema change | §§7.3 all green + V-5 + V-6 |
| **Gate-2** | F-2 test files + mutation harness + comment-only docstring | any behavioural source change | §§7.4 green including T-F2-06 5/5 |
| **Gate-3** | `mp_support.py`, `spawn`, import guards, A7 construction move | any new assertion removal | §§7.5 green on Linux (20/20 for determinism) |
| **Gate-4** | Independent verification of the resulting ZIP | any further change | §11 V-1…V-11 |
| **Gate-5** | Windows execution and any Windows-specific adjustment | anything else | V-12 with a real Windows VPS |

**Standing rule:** the delivery ZIP is never edited in place; extract → hash → verify → work on an independent
copy. A new ZIP is produced per gate, and every gate's ZIP hash is recorded in the report.

---

## 13. Traceability Matrix

Columns: **ID · Finding · Source Location · Required Change · Test ID · Acceptance Criteria · Evidence**

### 13.1 F-1

| ID | Finding | Source Location | Required Change | Test ID | Acceptance Criteria | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F1-R1** | F-1 | `sqlite_store.py:160` | single WAL transition via `open_sqlite_connection()` | T-F1-01, T-F1-04 | 0 failures / 300 two-process pairs; 3/3 single-process | E-1 rate 56/120, E-2 rate 11/250 |
| | | `sqlite_operation_ledger.py:153` | same | T-F1-01, T-F1-04, T-F1-13 | as above; shared branch untouched | E-1 23/40 |
| | | `sqlite_state_store.py:33` | same **plus** adding the absent `busy_timeout` (D-3) | T-F1-01, T-F1-04 | as above | E-1 22/40 |
| **F1-R2** | F-1 | `sqlite_store.py:165`, `sqlite_operation_ledger.py:155`, `sqlite_state_store.py` (absent) | `busy_timeout` applied inside the helper, after the transition, uniformly | T-F1-13 (policy parity) | all three adapters report the same `busy_timeout` value | §2.2 (reordering alone insufficient) |
| **F1-R3** | F-1 | same three sites | bounded backoff + deadline (`delay = min(250ms, 10ms·2ⁿ⁻¹)·jitter`, deadline 30 s) | T-F1-07, T-F1-09 | S7 succeeds with ≥2 attempts; S6 fails at deadline+≤2 s | §3.4 justification |
| **F1-R4** | F-1 | same three sites | no unbounded retry (deadline is the only bound) | T-F1-07 | elapsed ≤ deadline + 2 s | §3.4 |
| **F1-R5** | F-1 | `sqlite_store.py:158–169`, `sqlite_operation_ledger.py:151–160`, `sqlite_state_store.py:31–37` | fail closed with `SqliteInitializationError` (path, attempts, elapsed, deadline, cause); message contains "database" | T-F1-07, T-F1-10 | AU-35 still passes unmodified; message fields present | AU-35 at `test_phase_a_audit_fsm_and_live.py:251` |
| **F1-R6** | F-1 | same three constructors | close the connection on every failure path | T-F1-08 | 0 leaks / 100 failing constructions | D-5 |
| **F1-R7** | F-1 | same three constructors | verify `PRAGMA journal_mode == 'wal'` after the transition | T-F1-01, T-F1-12 | 100% of successful inits; forced-fallback raises | no existing test (Fact) |
| **F1-R8** | F-1 | same three constructors | defined behaviour for new/existing/WAL/concurrent | T-F1-01…T-F1-06 | §3.7 matrix fully covered | §3.7 |
| **F1-R9** | F-1 | same three constructors | no parallel DB, no deletion, no path change | T-F1-11 | added/removed files ⊆ {`-wal`,`-shm`} | requirement 9 |
| **F1-R10** | F-1 | all three | one policy, one helper | T-F1-13 + code review | the string `PRAGMA journal_mode=WAL` appears **once**, inside the helper | §2.2 gap; D-3/D-7 |
| **F1-R11** | F-1 | §2.3 blast radius | document that Live lock is not the remedy | — (doc) | §2.3 present in the report | E-2 |
| **F1-R12** | F-1 | `sqlite_operation_ledger.py:141–146` | the shared-journal branch must **not** invoke the helper | T-F1-13 | journal mode untouched; no second transition | **Proposal** (no existing coverage) |

### 13.2 F-2

| ID | Finding | Source Location | Required Change | Test ID | Acceptance Criteria | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F2-R1** | F-2 | `tests/test_phase_a_audit_multiprocess.py:36–37, 220–221` | add deterministic interleaving tests (new file) | T-F2-01 | 20/20 identical verdict | E-2 mutation 3/40 |
| **F2-R2** | F-2 | `sqlite_operation_ledger.py:503–523` (`_bump_duplicate`) | prove stale reads cannot revert state | T-F2-01, T-F2-05 | mutant M-1 caught 5/5 | §2.4 |
| **F2-R3** | F-2 | `:446–501` (`transition`) | prove stale transitions cannot overwrite | T-F2-02 | mutant M-2 caught 5/5 | E-2 0/60 |
| **F2-R4** | F-2 | `:525–560` (`_normalise_digest`) | prove the hash guard rejects | T-F2-03 | mutant M-3 caught 5/5 | §2.4 |
| **F2-R5** | F-2 | `:562–592` (`claim_for_send`), `:594–614` (`mark_submit_attempt`) | document that these are single-statement atomic, **not** `_in_immediate` | T-F2-07 + docstring | docstring present; no behaviour change | **Fact correction** §2.4 |
| **F2-R6** | F-2 | new `tests/test_f2_mutation_harness.py` | mechanical mutation detection | T-F2-06 | 5/5 runs catch every mutant | §4.3 |
| **F2-R7** | F-2 | existing A7 file | keep A7 unchanged except the disclosed strengthening | T-F2-04 (P) | no assertion removed; 20/20 green after F-1 | §5.7 |

### 13.3 F-3

| ID | Finding | Source Location | Required Change | Test ID | Acceptance Criteria | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F3-R1** | F-3 | `test_phase_a_audit_multiprocess.py:66, 112` | `spawn` via shared context factory | T-F3-01 | zero `"fork"` occurrences | grep Fact §2.5 |
| **F3-R2** | F-3 | `test_fix05_counter_semantics.py:181` | same | T-F3-01 | as above | grep Fact |
| **F3-R3** | F-3 | all three workers | module-level + picklable + no parent state | T-F3-02 | `pickle.dumps` succeeds for every target | already true (Fact) |
| **F3-R4** | F-3 | in-scope test modules | module-level `sys.path` guard for `goldbot` | T-F3-06 | child imports succeed; module I/O-free | A-3 |
| **F3-R5** | F-3 | new `tests/mp_support.py` | bounded teardown: terminate→join(5)→kill; `Manager.shutdown()` in `finally` | T-F3-04, T-F3-07 | termination ≤ 7 s; `mp.active_children()` empty | no `pytest-timeout` (Fact) |
| **F3-R6** | F-3 | in-scope tests | diagnosable failure on worker death | T-F3-03 | failure names the worker + exit code | `A7_FAIL_30.log` (60.10 s signal loss) |
| **F3-R7** | F-3 | in-scope tests | Manager proxies only (no plain `Barrier`/`Event` across processes) | T-F3-05 | static + runtime assertion | §5.4 |
| **F3-R8** | F-3 | — | Windows execution | T-F3-08, T-F1-14 | **NOT RUN** until a Windows VPS exists | §5.8 |

### 13.4 Out-of-Scope Findings (recorded, not addressed)

| ID | Finding | Location | Why out of scope | Suggested owner decision |
| --- | --- | --- | --- | --- |
| **OOS-1** | `SqliteRiskStateStore` has no `busy_timeout` at all | `sqlite_state_store.py:31–36` | *Partially* inside F-1 (D-3) — fixing it is authorized **only** as part of F1-R2 | — |
| **OOS-2** | Atomicity class of `claim_for_send`/`mark_submit_attempt` is undocumented and was mis-stated in the audit brief | `:562`, `:594` | Doc-only (F-2-D7) | Gate-2 |
| **OOS-3** | 58 `SqliteJournal(` construction sites, no factory | `src/` ×12, `tests/` ×46 | A factory is an architectural change | consider for PHASE C |
| **OOS-4** | `check_same_thread=False` on the journal connection | `sqlite_store.py:158` | Thread-safety of shared connection is not assessed here | PHASE C |
| **OOS-5** | CLI/web open the same DB as a running Live process without any lock | §2.3 table | Ownership model — out of scope | PHASE C / Cross-VPS decision |
| **OOS-6** | `LiveInstanceLock` uses `msvcrt` vs `fcntl`; no Windows test of the lock exists | `live_instance_lock.py:44–62` | Not F-1/F-2/F-3 | Windows validation plan |
| **OOS-7** | No test anywhere asserts `journal_mode == 'wal'` | whole suite | **Inside** F-1 (T-F1-01/12) | Gate-1 |
| **OOS-8** | `migrate()` backup path (`:218`) contended badly enough in the past to stall startup (~20 s, per the handover §5) | `sqlite_operation_ledger.py:185–220` | The handover records it as already mitigated (backup only when a ledger exists) — **not re-verified here** | note for Gate-1 review |

---

## 14. Open Questions / Assumptions

### 14.1 Assumptions (unverified premises the design relies on)

| ID | Assumption | Impact if false | How to check |
| --- | --- | --- | --- |
| **A-1** | The observed F-1 rate on Linux (4.4%–46.7%) is representative of the *mechanism*, not of the *platform rate* | The 300-iteration sample may be over/under-powered on Windows | Run T-F1-14 on Windows; adjust n if the platform rate is much lower |
| **A-2** | `sqlite3.Error.sqlite_errorcode` is populated on the busy path in Python 3.13 | We fall back to message matching (already specified) | Covered by T-F1-07's message assertion |
| **A-3** | A `spawn` child under pytest can import the test module by name and re-imports it cleanly | F-3 needs the explicit sys.path guard (already mandatory) | T-F3-06 |
| **A-4** | No caller on the journal path catches `sqlite3.Error` in a way that the new exception would escape | A start could crash with an uncaught new exception where it previously printed a message | Static review of every `except` on the journal path during Gate-1 (Q-4) |
| **A-5** | The deterministic proxy (option c) is accepted as a *faithful* stand-in for a competing commit | F-2's acceptance witness would be weaker than claimed | Review of the test docstring + T-F2-04 as the complementary real-process net |
| **A-6** | The 30 s deadline is acceptable for operator-facing CLI paths | A slow report becomes a 30 s hang instead of an instant error | Q-3 |

### 14.2 Open questions for the owner (each blocks a gate)

| ID | Question | Default if unanswered | Blocks |
| --- | --- | --- | --- |
| **Q-1** | Is Gate-1 authorized as specified (new module + three adapter edits)? | no work begins | Gate-0 |
| **Q-2** | Should the acceptance lane be **300** iterations (8–12 min) or a smaller number with a weaker statistical claim? | 300; no weaker claim allowed | Gate-1 close |
| **Q-3** | Is a **30 s** initialization deadline acceptable for CLI paths, or should CLI use a shorter one (e.g. 5 s) while Live uses 30 s? | single 30 s default | Gate-1 |
| **Q-4** | May the implementation change the exception type on the journal path (requiring the A-4 review), or must it keep `sqlite3.OperationalError` and only enrich the message? | keep the new type, subject to the A-4 review | Gate-1 |
| **Q-5** | Is a **mutation harness that copies and textually patches source** acceptable in the repo's test suite, or should it live under `tools/` and run only in acceptance? | `tools/`, acceptance-only | Gate-2 |
| **Q-6** | Which logger name/level should the initialization INFO line use? | `goldbot.adapters.sqlite_init`, INFO | Gate-1 |
| **Q-7** | Should connection state be exposed through `migration_report()`, or kept entirely out of it? | kept out (§8) | Gate-2 |

### 14.3 Explicitly NOT established by this document

* Whether the **current** implementation contains a Lost Update. **No evidence was found that it does**; F-2 is
  a verification gap, and this document does not upgrade it to a defect.
* Whether F-1 occurs on **Windows**, or at what rate.
* Whether the 30 s deadline is sufficient **in production**.
* Whether any test listed here **passes** — **no test was run while writing this spec.**
* Any statement about MT5, the broker, demo execution, soak behaviour, Cross-VPS fencing or production readiness.

---

**End of specification.**

```
STATUS: SPECIFICATION ONLY — NO IMPLEMENTATION, NO TEST EXECUTION, NO DELIVERY CHANGE
AWAITING OWNER REVIEW AND EXPLICIT GATE-0 AUTHORIZATION
```
