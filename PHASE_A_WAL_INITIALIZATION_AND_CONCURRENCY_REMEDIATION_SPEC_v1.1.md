# PHASE_A_WAL_INITIALIZATION_AND_CONCURRENCY_REMEDIATION_SPEC_v1.1

**Project:** GoldBot v8.2.5 · PHASE A
**Document type:** Technical Remediation Specification (**design only — no code**)
**Version:** **v1.1** (supersedes v1, 2026-11-26)
**Date:** 2026-11-26
**Baseline under specification:** `GoldBot_v8.2.5_PHASE_A_TARGETED_REMEDIATION_2026-11-25.zip`
SHA-256 `937517037894c99916e6b30d4b5ff3a63530e9bf9931f3b10616384f52d1758d`
**Source tree referenced:** `GoldBot_Source_v8.2.5` (299-entry `SOURCE-MANIFEST.txt`, problems: 0)
**Scope owner:** project owner — the only party that may authorize implementation.

> **Status vocabulary.**
> **Fact** = read directly from the delivered source, or established by an isolated interpreter probe (§2.7).
> **Inference** = conclusion drawn from Facts, stated as such.
> **Proposal** = a design decision recommended here; **not implemented**.
> **Assumption** = unverified premise the design relies on.
> **Unknown** = not established by any evidence available.
> **NOT RUN / NOT VERIFIED** = explicitly not executed — never "passed".

---

## 0. Change Log — v1 → v1.1

This revision implements six mandatory corrections plus the reporting rules requested by the owner.
Nothing else was changed; sections that remain valid are carried forward verbatim in substance.

| # | Owner's required correction | What changed in v1.1 | Where |
| --- | --- | --- | --- |
| **C-1** | `check_same_thread` | `open_sqlite_connection()` gained an explicit `check_same_thread` parameter. The current per-adapter value is **frozen**: `SqliteJournal` keeps **`False`**, `SqliteOperationLedger` (path branch) keeps the stdlib default **`True`**, `SqliteRiskStateStore` keeps **`True`**. A per-adapter regression test was added (T-F1-15) that asserts the *observable* thread behaviour, not merely the keyword. New requirement row **F1-R13**. | §3.3, §3.6, §7.3, §13.1 |
| **C-2** | Deadline is a true cap | The deadline was redefined as the cap for the **whole initialization operation**, not a retry budget. A monotonic deadline token is created once; **remaining time is recomputed before every attempt**; `sqlite3.connect(timeout=…)` is bounded by the remaining time; `PRAGMA busy_timeout` is **clamped to the remaining time** so a busy wait cannot overrun. `migrate()`'s own `busy_timeout=10000` (line 192) is re-clamped against the same token (new requirement **F1-R16**, new Finding **N-1**). A deterministic overrun test (T-F1-16) and a per-attempt invariant test (T-F1-17) were added. The `deadline + 2 s` acceptance is now shown to be reachable by construction (§3.4.4). | §3.4, §3.8, §7.3 |
| **C-3** | Extended SQLite result codes | Retry classification now extracts the **primary result code** as `sqlite_errorcode & 0xFF` and tests membership in `{5 (SQLITE_BUSY), 6 (SQLITE_LOCKED)}`; extended codes (e.g. `517 = SQLITE_BUSY_SNAPSHOT`, `261 = SQLITE_BUSY_RECOVERY`, `262 = SQLITE_LOCKED_SHAREDCACHE`) therefore classify correctly. **Text matching is demoted to a fallback** used only when the code attribute is absent. Non-retryable codes fail fast with `attempts == 1` (new test T-F1-19). | §3.4.1, §7.3 |
| **C-4** | F-2 simulation honesty | The hook scenario is re-described precisely and renamed **"same-transaction interleaving simulation"**. It is stated explicitly that it does **not** prove multi-process contention and is **not** a "competing committed write". The real multi-process test is kept **separately** and labelled class **P** (never a witness). The specific Mutation **M-2 (removal of `BEGIN IMMEDIATE`)** is now caught **deterministically** by a dedicated **lock-level witness** test (T-F2-03), which is a behavioural discriminator rather than a trace-only assertion. | §4.2–§4.4, §7.4 |
| **C-5** | Exception contract | A four-way taxonomy replaces the single error: `SqliteBusyTimeout`, `SqliteNotADatabase`, `SqliteUnretryableError`, and `SqliteInitializationError` (base), with the ledger adapter mapping to `LedgerUnavailable`. Cause chains are mandatory (`raise … from exc`). Every message keeps the literal token **"database"** so **AU-35 is never weakened**. A full compatibility review of the existing `except` sites was performed and is recorded as a **Gate-1 condition** (§3.6.3). | §3.5, §3.6.3 |
| **C-6** | Gate consistency | Gate boundaries were re-cut (§12). **No gate may claim a Windows result.** Every gate reports **Linux** and **Windows** in two separate columns, with Windows forever `NOT RUN` until Gate-5 has real Windows-host evidence. A gate with outstanding Windows items must be recorded as `PARTIALLY CLOSED — WINDOWS OUTSTANDING`, never `CLOSED`. | §12 |

**Findings discovered while producing v1.1** (recorded, **not** silently folded in):

| ID | Finding | Evidence | Disposition |
| --- | --- | --- | --- |
| **N-1** | `migrate()` unconditionally re-sets `PRAGMA busy_timeout=10000` at `sqlite_operation_ledger.py:192`, after the constructor's own value, and would silently undo a deadline-clamped timeout. | `sed -n '185,225p'` (§2.7) | **In scope** as F1-R16 — must be clamped against the same deadline token. |
| **N-2** | `migrate()` wraps `BEGIN IMMEDIATE` in `except sqlite3.OperationalError: pass` (`:208–213`) with the comment "already inside a transaction". A genuine `SQLITE_BUSY` on that statement is **indistinguishable from** that condition and is silently swallowed, after which DDL proceeds without a lock. | `sqlite_operation_ledger.py:208–213` | **In scope** as F1-R17 — the handler must discriminate by message/code, not by exception class. |
| **N-3** | `_backup()` opens two un-timed connections (`:307`, `:313`) outside any deadline, and runs before the `BEGIN IMMEDIATE`. | `sqlite_operation_ledger.py:300–320` | **Out of scope** (recorded as OOS-9). The handover claims this path was already mitigated; not re-verified here. |
| **N-4** | On a corrupt file, `sqlite3.connect(...)` + `PRAGMA journal_mode=WAL` raises **`DatabaseError`** (code **26 = `SQLITE_NOTADB`**), **not** `OperationalError`. A helper that only catches `OperationalError` would let this escape unclassified. | Interpreter probe (§2.7) | **Fact** — folded into F1-R18's taxonomy and T-F1-10. |

---

## 1. Executive Summary

This document specifies — and only specifies — the remediation design for three findings from the independent
audit of 2026-11-26.

| ID | Class | Statement | Blast radius |
| --- | --- | --- | --- |
| **F-1** | Availability / Startup defect (**pre-existing**) | `PRAGMA journal_mode=WAL` can fail with `database is locked` when two processes initialize a **brand-new** SQLite file simultaneously. Three adapters perform this initialization independently. | CLI, report, web and Live startup; **no corruption observed** |
| **F-2** | Verification gap | The A7 multi-process test does not deterministically prove the absence of a Lost Update, and cannot reliably detect re-introduction of the pre-fix whole-row write. | Test credibility only; **no evidence the current implementation is defective** |
| **F-3** | Test portability gap | Three test sites hard-code `multiprocessing.get_context("fork")`; Windows — the target deployment platform — does not support it. | Test executability on the production platform; no source defect |

**Headline Facts**

* The literal `PRAGMA journal_mode=WAL` appears in exactly **three** source locations: `sqlite_store.py:160`,
  `sqlite_operation_ledger.py:153`, `sqlite_state_store.py:33`. No fourth exists.
* `SqliteRiskStateStore` sets **no** `busy_timeout` at all (`sqlite_state_store.py:31–36`).
* No test anywhere asserts that the journal mode actually became `wal`.
* `claim_for_send` (`:562`) and `mark_submit_attempt` (`:594`) do **not** use `_in_immediate`; only `transition`
  (`:446`), `_bump_duplicate` (`:503`) and `_normalise_digest` (`:525`) do. **This corrects the audit brief.**
* A faithful re-introduction of the pre-fix whole-row write reproduced failures in **3 of 40** A7 runs (7.5%).
* `get_context("fork")` appears at `test_phase_a_audit_multiprocess.py:66`, `:112`, `test_fix05_counter_semantics.py:181`.

**Headline Proposals**

* One shared initializer, deadline-capped, backoff-bounded, WAL-verifying, connection-closing.
* Four-way exception taxonomy with preserved cause chains and AU-35-safe messages.
* Deterministic tests via a **same-transaction interleaving simulation** plus a **lock-level witness** for the
  serialization primitive; real multi-process tests preserved separately and never used as proof.
* `spawn` everywhere; pickle-safe workers; explicit import guard; bounded teardown with no new dependency.
* **Nothing in this document has been implemented. No project test has been executed.**

---

## 2. Audit Evidence and Scope

### 2.1 Evidence sources

| # | Source | Nature | Used for |
| --- | --- | --- | --- |
| **E-1** | Owner-provided audit of 2026-11-26 (F-1/F-2/F-3; process-pair table 120/40/40) | Owner-supplied **Fact** | Failure rates per adapter |
| **E-2** | Independent audit of 2026-11-26 (`INDEPENDENT_VERIFICATION_2026-11-26.md`, `/home/user/evidence/`) | Independent measurement | Root cause; fresh-vs-existing split; mutation sensitivity |
| **E-3** | Delivered source tree, read directly for this document (`grep`/`sed`) | **Fact** | Every line number in this document |
| **E-4** | `SOURCE_DIFF_FULL.diff` from the delivery | **Fact** | Proves `sqlite_store.py` untouched by the Targeted Remediation |
| **E-5** | Isolated interpreter probe of `sqlite3` exception attributes (§2.7) | **Fact** | N-4; extended-code mechanics; message text |

### 2.2 F-1 — measured failure rates

| Adapter | Process pairs | Pairs with ≥1 failure | Rate | Source |
| --- | --- | --- | --- | --- |
| `SqliteJournal` | 120 | 56 | **46.7 %** | E-1 |
| `SqliteRiskStateStore` | 40 | 22 | **55.0 %** | E-1 |
| `SqliteOperationLedger` (path-based) | 40 | 23 | **57.5 %** | E-1 |
| `SqliteJournal` — independent re-measure, **fresh** DB | 250 | 11 | **4.4 %** | E-2 |
| `SqliteJournal` — independent re-measure, DB **already WAL** | 250 | **0** | **0.0 %** | E-2 |
| Bare-SQL stepwise probe (which statement fails) | 150 | 13 | **8.7 % — all 13 at `PRAGMA journal_mode=WAL`** | E-2 |

**Inference:** the two measurements differ by an order of magnitude because the rate is a function of **how
tightly the two constructors are aligned in time**, not a constant of the product. Consequence for §7.2: the
acceptance sample size must be powered for the *smallest* rate we care about (1 %), never for the largest
observed one.

**Fact (root cause):** the failing statement is the **journal-mode transition itself**, not the subsequent schema
script. The stepwise probe isolated `connect → journal_mode=WAL → synchronous → busy_timeout → executescript`;
13/13 failures occurred on `journal_mode=WAL`.

**Fact:** installing `PRAGMA busy_timeout` *before* the transition does **not** fix it (8/200 with the
reordering vs 5/200 without).

**Inference:** the busy handler does not cover the mode transition. Either the policy must be installed in a way
that survives the transition, or the transition must be retried by the caller. The Evidence supports the second
and does not disprove the first; §3.3 records the rejected alternative with its reason.

### 2.3 F-1 — scope of exposure

| Entry point | Takes `LiveInstanceLock`? | Files opened | Concurrent exposure |
| --- | --- | --- | --- |
| `run_live` (`runtime/live.py:198`) | **Yes** (`_with_local_live_lock`, `:175`; journal built inside `build_components()` at `:341`) | journal `db_path` + `state_db_path(db_path)` (`live.py:228`) | Protected against a second **Live** process; **not** against CLI/web |
| `cmd_ack`, `cmd_status`, `cmd_override`, `cmd_health`, `cmd_level`, `cmd_approve`, `cmd_onboarding` (`cli.py:389,454,489,572,616,736,744`) | **No** | risk-state DB (7 sites), some also the journal | **Yes** |
| `cmd_report` (`:167`), `cmd_signals` (`:794`), `_run` for demo/backtest (`:176`), `cmd_serve` (`:275`) | **No** | journal, risk state | **Yes** |
| `web/app.py:210, 249, 503, 526` (journal) and `:200` (risk state) | **No** | journal, risk state | **Yes** |
| `build_components` (`bootstrap.py:117` journal, `:247` ledger) | Inherited | journal + ledger + state | **Yes** when the caller is CLI/web |

**Fact:** `SqliteJournal(` is constructed at **58 sites** in the tree (12 in `src/`: 1 bootstrap, 7 CLI, 4 web;
46 in `tests/`). There is no factory.

**Inference:** F-1 is **not** solved by the Live lock. That lock reduces but does not remove the window, and the
CLI/report path is exactly the operator's diagnostic tool during an incident — the moment a second process is
most likely.

**Fact:** `grep -rn "journal_mode" tests/` returns nothing — requirement 7 has zero existing coverage.

### 2.4 F-2 — what the test actually does, and a correction to the brief

**Fact:** both workers build their journal **before** reaching the barrier:

```
36:  journal = SqliteJournal(db_path)
37:  ledger  = SqliteOperationLedger(journal=journal, clock=None)
39:  go.wait(timeout=60)
…
220: journal = SqliteJournal(db_path)
221: ledger  = SqliteOperationLedger(journal=journal, clock=clock)
257: go.wait(timeout=60)
```

**Inference:** the test synchronizes only the *claim*, never the *initialization*; it therefore cannot detect F-1
by construction.

**Correction to the audit brief (Fact):**

| Method | Line | Targeted SQL | `BEGIN IMMEDIATE`? | Safety argument |
| --- | --- | --- | --- | --- |
| `_bump_duplicate` | `:503` | `UPDATE … duplicate_requests = duplicate_requests + 1` | **yes** (`:521`) | RMW serialized |
| `_normalise_digest` | `:525` | conditional `UPDATE … WHERE op_id=? AND payload_hash=?` | **yes** (`:551`) | guarded write + re-read |
| `transition` | `:446` | full-row `UPDATE` from `_apply_transition` | **yes** (`:498`) | RMW serialized |
| `claim_for_send` | `:562` | `UPDATE … SET state=?, claim_attempts = claim_attempts+1 WHERE op_id=? AND state IN (?,?)` | **no** (`:568` lock + commit) | single conditional statement is atomic; `rowcount` is the CAS witness |
| `mark_submit_attempt` | `:594` | `UPDATE … submit_call_markers = submit_call_markers + 1 WHERE op_id=?` | **no** (`:601` lock + commit) | single incrementing statement is atomic |

**Inference:** no defect is implied — both non-`_in_immediate` methods issue **one** statement whose increments
SQLite computes relative to the stored row. But the distinction must be documented, because a future refactor
adding a second statement to either method would silently lose serialization (F-2-D7, OOS-2).

**Fact (mutation, E-2):** the pre-fix whole-row write in `_bump_duplicate` produced **3 failures in 40** A7 runs
(7.5 %), with `IllegalTransition('illegal order transition APPROVED -> ACKED')` — the FSM refused an illegal
transition; no second order was sent.
**Fact:** removing `_in_immediate` from `transition` produced **0 failures in 60** runs.

**Inference:** the A7 harness can be green on materially weakened code. F-2 is a genuine verification gap and
the brief's framing — no evidence the implementation is defective — is the correct one.

### 2.5 F-3 — exact sites and constraints

| File | Line | Call |
| --- | --- | --- |
| `tests/test_phase_a_audit_multiprocess.py` | **66** | `ctx = mp.get_context("fork")` (AU-22) |
| `tests/test_phase_a_audit_multiprocess.py` | **112** | `ctx = mp.get_context("fork")` (AU-23) |
| `tests/test_fix05_counter_semantics.py` | **181** | `ctx = mp.get_context("fork")` (crash-evidence test) |

* **Fact:** 86 test modules, **no `tests/__init__.py`**.
* **Fact:** `conftest.py` inserts `<root>/src` into `sys.path` **in the parent only**; a `spawn` child does not
  import `conftest.py`.
* **Fact:** this host reports `get_all_start_methods() == ['fork','spawn','forkserver']`.
* **Fact:** `psutil` 7.2.2 happened to be importable in the audit workspace but is **not** a project dependency
  (`requirements.txt` = one line, `pydantic`).
* **Fact:** `pytest-timeout` and `pytest-xdist` are **not** installed; there is no global watchdog.
* **Unknown:** whether a `spawn`ed child under pytest can import the test module by name. §5.3 removes the
  dependency on the answer.

### 2.6 Integrity evidence carried forward

| Check | Result | Status |
| --- | --- | --- |
| Delivery ZIP SHA-256 | `9375170378…52d1758d` | **Fact** |
| `SHA256SUMS.txt` | 316/316 OK (re-verified after every audit run) | **Fact** |
| `SOURCE-MANIFEST.txt` | files: 299 \| problems: 0 | **Fact** |
| `sqlite_store.py` SHA-256 | `a74e06db297c9f9ee8d6b6d8ef6942daa6528a70cbacad44db5a0f1fd29cfcba` | **Fact** — matches the brief |
| `sqlite_store.py` in `SOURCE_DIFF_FULL.diff` | 0 occurrences | **Fact** — F-1 is pre-existing |
| `domain/fsm/order.py` SHA-256 | `bf4154c6…f1774d0f` | **Fact** — untouched |
| Full suite on delivered tree | `1224 passed, 6 skipped, 3 warnings` | **Fact — measured in the prior round, not re-run for v1.1** |

### 2.7 Environment probe and disclosure

> **Disclosure.** Producing v1.1 required exactly one executed probe. It used **only the Python standard
> library** (`sqlite3`, `tempfile`), in a temporary directory, with a synthetic throw-away file. It imported **no
> project code**, opened **no project database**, ran **no project test**, and produced no artefact inside the
> delivery. Its purpose was to establish two Facts that the design depends on and that cannot be established by
> reading.

Results (**Facts**):

| Probe | Result |
| --- | --- |
| Contended write in WAL mode, `timeout=0` | `sqlite3.OperationalError`; `sqlite_errorcode = 5`; `sqlite_errorname = 'SQLITE_BUSY'`; `5 & 0xFF = 5` |
| Corrupt file (`b"not a sqlite database"*64`), `PRAGMA journal_mode=WAL` | **`sqlite3.DatabaseError`**; `sqlite_errorcode = 26`; message `file is not a database` |

**Inferences:** (a) `sqlite_errorcode` is populated in CPython 3.13 and carries the extended code, so the
`& 0xFF` primary extraction in C-3 is sound; (b) the corrupt-file path raises `DatabaseError`, **not**
`OperationalError`, so the helper must catch and classify `DatabaseError` explicitly (N-4); (c) the corrupt-file
message *does* contain the word "database", which is the accidental reason AU-35 currently passes — §3.5.3 keeps
that property deliberately.

**No other execution occurred for this document.**

---

## 3. F-1 Technical Root Cause and Remediation Design

### 3.1 Current code, verbatim

**A. `src/goldbot/adapters/sqlite_store.py:158–169`**

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

**B. `src/goldbot/adapters/sqlite_operation_ledger.py:147–160`** (path branch only)

```python
147:  self.path = Path(path)
151:  self._conn = sqlite3.connect(self.path, timeout=10.0)          # check_same_thread defaults to True
153:  self._conn.execute("PRAGMA journal_mode=WAL")
154:  self._conn.execute("PRAGMA synchronous=FULL")
155:  self._conn.execute("PRAGMA busy_timeout=10000")
156:  self._owns_connection = True
160:  self.migrate()                                                  # :192 sets busy_timeout=10000 again
```

**C. `src/goldbot/adapters/sqlite_state_store.py:30–37`**

```python
31:  self._conn = sqlite3.connect(self.path, timeout=10.0)          # check_same_thread defaults to True
33:  self._conn.execute("PRAGMA journal_mode=WAL")
35:  self._conn.execute("PRAGMA synchronous=FULL")
36:  self._conn.executescript(SCHEMA)
37:  self._conn.commit()                                            # NO busy_timeout anywhere
```

### 3.2 Defect inventory

| ID | Defect | Location | Consequence |
| --- | --- | --- | --- |
| **D-1** | WAL transition attempted exactly once, no retry | A:160, B:153, C:33 | Any momentary lock → constructor failure |
| **D-2** | `busy_timeout` set **after** the transition | A:165, B:155 | Cannot help the statement that needs it (and reordering does not help either — §2.2) |
| **D-3** | `SqliteRiskStateStore` sets **no** `busy_timeout` | C (absent) | Every `SQLITE_BUSY` on a write is immediate; policy diverges |
| **D-4** | Resulting journal mode never read back | A, B, C | A silent fallback to `delete` goes unnoticed; requirement 7 unmet |
| **D-5** | No `try/except`; connection not closed on failure | A:158–169, B:147–160, C:31–37 | Leaked handle on every failed construction; on Windows a leaked handle can also block deletion |
| **D-6** | Operator sees a bare `sqlite3.OperationalError`/`DatabaseError` | A, B, C | No path, no attempt count, no elapsed time, no guidance |
| **D-7** | Three independent copies of one policy | A, B, C | Already diverged (D-3 exists in one file only) |
| **D-8** | Ledger's shared-connection branch (`:141–146`) inherits the journal's policy and runs `migrate()` on it | B:141–160 | Correct by design, but must be stated: the helper is **forbidden** there (§3.6) |
| **D-9** | `migrate()` re-sets `busy_timeout=10000` unconditionally | B:192 | Would silently undo a deadline-clamped value (**N-1**) |
| **D-10** | `migrate()` swallows **any** `OperationalError` on `BEGIN IMMEDIATE` | B:208–213 | A genuine `SQLITE_BUSY` is misread as "already inside a transaction"; DDL then proceeds unlocked (**N-2**) |
| **D-11** | No deadline anywhere in initialization | A, B, C | An operator cannot distinguish "slow" from "stuck"; harnesses time out before the code speaks |

### 3.3 Design decision F-1-D1 — one shared initializer, with the frozen signatures

**Proposal:** one new module `src/goldbot/adapters/sqlite_init.py` exposing:

```python
@dataclass(frozen=True)
class InitDeadline:
    started: float          # time.monotonic() at entry
    budget_s: float
    def remaining(self) -> float: ...
    def remaining_ms(self, *, floor_ms: int = 1) -> int: ...
    def expired(self) -> bool: ...

@dataclass(frozen=True)
class InitReport:
    path: str
    attempts: tuple[Attempt, ...]     # each: index, remaining_ms_before, busy_timeout_ms_used, error_code, error_name
    elapsed_s: float
    journal_mode_after: str
    created: bool

def open_sqlite_connection(
    path: str | Path,
    *,
    deadline: InitDeadline | None = None,
    deadline_s: float = 30.0,
    busy_timeout_ms: int = 10_000,
    row_factory: bool = True,
    check_same_thread: bool = True,        # stdlib default — preserved per adapter (C-1)
) -> tuple[sqlite3.Connection, InitReport]

def clamp_busy_timeout(conn: sqlite3.Connection, deadline: InitDeadline,
                       default_ms: int = 10_000) -> int
```

**Contract, in order:**

1. `path.parent.mkdir(parents=True, exist_ok=True)` — unchanged, no path change.
2. `t0 = time.monotonic()`; `deadline = deadline or InitDeadline(t0, deadline_s)`.
3. `remaining = deadline.remaining()`; if `remaining <= 0` → raise `SqliteBusyTimeout` (elapsed 0).
4. `conn = sqlite3.connect(path, timeout=min(remaining, 10.0), check_same_thread=check_same_thread)`
   — **the connect call cannot outlive the deadline**.
5. `conn.row_factory = sqlite3.Row` when `row_factory` is true.
6. **Retry loop:**
   - before **each** attempt, recompute `remaining = deadline.remaining()`; if `<= 0` → raise `SqliteBusyTimeout`;
   - `conn.execute(f"PRAGMA busy_timeout={deadline.remaining_ms()}")` — **the busy wait can never exceed the
     remaining budget** (C-2);
   - attempt `PRAGMA journal_mode=WAL`;
   - on a **retryable** code (see §3.4.1) → record the attempt, sleep `delay(n)` **only if** the deadline has
     room for it, loop;
   - on a **non-retryable** code → classify (§3.5) and raise immediately with `attempts == 1`;
   - on success → break.
7. **Verify:** `PRAGMA journal_mode` must return `wal` (case-insensitive). Anything else →
   `SqliteInitializationError(reason="journal_mode is '<x>', not 'wal'")`.
8. `PRAGMA synchronous=FULL`.
9. `PRAGMA busy_timeout=min(busy_timeout_ms, deadline.remaining_ms())`.
10. Return `(conn, report)`.

**On any exception in steps 4–9: `conn.close()` must be called before re-raising** (D-5). A helper-local
`try/except BaseException` guarantees this even for `KeyboardInterrupt`.

**Never:** delete, move, recreate, rename or re-base a database file; never switch to `:memory:`; never fall back
to another path (requirement 9).

**Why a shared helper (requirement 1):** the three sites must obey **one** policy, and they already diverged once
(D-3). The concurrency logic is subtle enough (deadline, per-attempt accounting, code classification,
close-on-error) that duplicating it would triple both the code and the evidence burden.

**Alternatives rejected**

| Alternative | Reason |
| --- | --- |
| Only raise / reorder `busy_timeout` | Disproved by measurement (§2.2): 8/200 vs 5/200. |
| Open in rollback mode permanently | Contradicts the load-bearing `WAL + synchronous=FULL` crash-evidence design. |
| Serialize all access through one process | Architectural change to the ownership model — out of scope. |
| A cross-process startup lock file | Adds a second locking mechanism (stale-lock recovery, `msvcrt`/`fcntl` divergence) to be acquired at 58 sites. Retrying the one failing statement is strictly smaller. |
| Retry inside each adapter | Violates requirement 10 and re-creates D-7. |

### 3.4 Deadline semantics (requirements 2, 3, 4)

#### 3.4.1 Retry classification — extended codes first, text only as fallback (C-3)

```
def primary_code(exc: sqlite3.Error) -> int | None:
    raw = getattr(exc, "sqlite_errorcode", None)
    return None if raw is None else (raw & 0xFF)      # 517 -> 5, 261 -> 5, 262 -> 6

RETRYABLE_PRIMARY = {5, 6}        # SQLITE_BUSY, SQLITE_LOCKED
```

| Condition | Classification | Action |
| --- | --- | --- |
| `primary_code(exc) in {5, 6}` — covers BUSY, BUSY_RECOVERY (261), BUSY_SNAPSHOT (517), BUSY_TIMEOUT (773), LOCKED (6), LOCKED_SHAREDCACHE (262) | **Retryable** | retry until deadline |
| code attribute absent (older runtimes) | fallback | retry **only** if the message contains `database is locked` / `database table is locked` (case-insensitive) |
| `primary_code == 26` (`NOTADB`) | **Not retryable** | `SqliteNotADatabase`, `attempts == 1`, `< 2 s` |
| any other code (e.g. 14 `CANTOPEN`, 8 `READONLY`, 3 `PERM`, 11 `CORRUPT`), or `DatabaseError` that is not a code we know | **Not retryable** | `SqliteUnretryableError`, `attempts == 1` |

**Justification for `& 0xFF`:** SQLite defines extended codes as `primary | (N << 8)`; masking is the documented
way to recover the primary code. The probe (§2.7) confirmed the attribute carries the raw value on this
interpreter. **Text matching is explicitly a fallback** and is never the primary discriminator (C-3).

**Justification for fast-failing non-retryables:** retrying `NOTADB`/`CANTOPEN`/`READONLY` cannot succeed, and a
retry storm would turn an honest instant failure into a 30 s hang that looks like a deadlock (requirement 3, and
AU-35's `< 2 s` expectation in §7.3).

#### 3.4.2 Backoff schedule

```
delay(n) = min(250 ms, 10 ms · 2^(n−1)) · uniform(0.8, 1.2)      n = 1, 2, 3, …
```

* The contended party holds the lock only for the duration of its own WAL transition — a handful of page writes;
  E-2 observed sub-100 ms stalls in every successful contention case.
* A 10 ms first step means the uncontended path never sleeps.
* The 250 ms cap bounds wake-ups: at the cap, a full 30 s deadline costs ≤ ~120 wake-ups.
* ±20 % jitter prevents lock-step retries — the thundering-herd pattern that matters precisely because both
  processes start microseconds apart (E-1's harness).
* The bound is a **deadline**, not an attempt count; the attempt log is what the operator sees.

#### 3.4.3 Default deadline: 30 s

| Constraint | Value | Why |
| --- | --- | --- |
| Worst honest WAL transition | sub-second | measured; 30 s is two orders of magnitude of headroom |
| A7 harness waits | `go.wait(60)`, `results.get(60)`, `proc.join(120)` | **deadline (30 s) < 60 s**, so a genuine stall surfaces as `SqliteBusyTimeout`, not as a harness timeout — the exact confusion visible in `A7_FAIL_30.log` (60.10 s) |
| Existing connect timeout | 10 s | the deadline does not tighten any currently-working path |

#### 3.4.4 Why `deadline + 2 s` is now reachable (C-2)

Every blocking call is bounded by the remaining budget:

| Blocking call | Bound applied |
| --- | --- |
| `sqlite3.connect(timeout=…)` | `min(remaining, 10 s)` |
| busy wait inside any statement | `PRAGMA busy_timeout = remaining_ms` (set **before each** attempt) |
| backoff sleep | skipped when `remaining ≤ delay` |
| post-connect PRAGMAs | same clamped `busy_timeout` |

Residual overshoot can therefore only come from **one non-blocking statement's execution** plus interpreter and
scheduler overhead, which is bounded by ~2 s on a 2-vCPU host with a 30 s budget.

**Fact (N-1) that must be closed for this argument to hold:** `migrate()` at `sqlite_operation_ledger.py:192`
re-sets `busy_timeout=10000`, which would escape the clamps for the migration phase. **Proposal F1-R16:** the
ledger passes the same `InitDeadline` into `migrate()` and calls `clamp_busy_timeout(self._conn, deadline)`
instead of the literal 10 000. The **primary** fix remains the helper; the migration is the one place that
re-arms the timeout.

**Scope boundary, stated explicitly:** the deadline caps *the helper's operation* — connect + transition +
PRAGMAs + verification. Schema creation and migration DDL run **after** the helper returns and are covered only
by the re-clamping of `busy_timeout` (F1-R16), not by a hard wall-clock kill. A hard cap over DDL would require
running initialization in a thread with a cancellation protocol, which is a larger change and is **not proposed**
(recorded as Q-8).

### 3.5 Exception contract (C-5)

#### 3.5.1 Taxonomy

```
GoldbotError (domain/errors.py:22)
└── SqliteInitializationError          # base: initialization could not be completed
    ├── SqliteBusyTimeout              # BUSY/LOCKED persisted past the deadline
    ├── SqliteNotADatabase            # SQLITE_NOTADB (26) — corrupt / unreadable
    └── SqliteUnretryableError        # any other non-retryable code
LedgerUnavailable                     # ports/ledger.py:77 — unchanged public contract
```

| Failure | Raised by helper | Adapter surface | `attempts` | `elapsed` | Cause chain |
| --- | --- | --- | --- | --- | --- |
| BUSY/LOCKED beyond deadline | `SqliteBusyTimeout` | journal/state: as-is; ledger: `LedgerUnavailable` | ≥1 (logged) | ≈ deadline | `from last_exc` |
| Corrupt file | `SqliteNotADatabase` | as above | **1** | **< 2 s** | `from DatabaseError` |
| Other non-retryable | `SqliteUnretryableError` | as above | **1** | **< 2 s** | `from exc` |
| WAL not applied | `SqliteInitializationError` | as above | 1 | small | `from None` or cause of the PRAGMA |
| Ledger migration failure | — | `LedgerUnavailable` (existing behaviour, `:341`) | — | — | unchanged |

**Non-negotiable:** every `raise` uses `from exc` (or `from None` only where no cause exists), so
`__cause__` is preserved end-to-end — the ledger's existing `raise LedgerUnavailable(...) from exc` pattern at
`:500/:519/:558` is the model to follow.

#### 3.5.2 Message template

```
SQLite initialization failed for database {path}: {reason} after {attempts} attempt(s)
in {elapsed:.2f}s (deadline {deadline:.1f}s)
```

The literal token **"database"** is mandatory in every subclass (§3.5.3).

#### 3.5.3 AU-35 is preserved by construction, not by luck

**Fact:** `tests/test_phase_a_audit_fsm_and_live.py:251` asserts

```python
assert "database" in repr(excinfo.value).lower() or "ledger" in repr(excinfo.value).lower()
```

Today this passes **accidentally**: the corrupt-file error message is `file is not a database` (probe §2.7).
**Proposal:** the template above makes the token explicit, so AU-35 keeps passing on its original terms.
**AU-35 must not be edited, weakened or skipped.** T-F1-10 re-runs AU-35 unmodified as part of F-1 acceptance.

#### 3.5.4 Compatibility review of existing catch sites (C-5, Gate-1 condition)

**Fact — every `except sqlite3.*` in `src/` is inside a method body, never around a constructor:**

| File:line | Handler | Relationship to the new exception |
| --- | --- | --- |
| `sqlite_operation_ledger.py:193` | `except sqlite3.Error: pass` (around a `busy_timeout` PRAGMA) | unaffected — direct connection call, not the helper |
| `:210` | `except sqlite3.OperationalError: pass` (around `BEGIN IMMEDIATE`) | **must change** — swallows BUSY (**N-2**, F1-R17) |
| `:341, :389, :399, :416, :434, :500, :519, :558, :590, :615, :628` | `except sqlite3.Error` around writes | unaffected — they wrap `self._conn.execute(...)`, not the constructor |
| `order_coordinator.py:244` | `except LedgerUnavailable` | **preserved** because the ledger maps to `LedgerUnavailable` |
| `web/app.py:203, 211, 516, …` | `except Exception` | **preserved** — the new type derives from `Exception`; behaviour is "report the failure", unchanged |

**Gate-1 condition (mandatory, testable by review):** before Gate-1 may close, the implementer must produce a
one-page listing of **every** `except` clause reachable from a journal/ledger/state constructor in `src/` and
state, for each, whether the new type is caught and with what user-visible behaviour. **Assumption A-4** (no
caller relies on `sqlite3.Error` around construction) is only discharged by that review, not by this document.

### 3.6 Per-adapter wiring

#### 3.6.1 What changes, and the frozen values (C-1)

| Adapter | Change | **Frozen — must not change** |
| --- | --- | --- |
| `SqliteJournal` (`sqlite_store.py:153–169`) | replace 158–165 with `self._conn, self._init_report = open_sqlite_connection(self.path, row_factory=True, **check_same_thread=False**)`; keep `executescript(SCHEMA)`, `_migrate_trades()`, `_migrate_signals()`, `commit()` | `SCHEMA`, migration semantics, **`check_same_thread=False`**, `path.parent.mkdir` |
| `SqliteOperationLedger` path branch (`:147–160`) | replace 151–155 with the helper (**`check_same_thread` omitted → stdlib default `True`**); keep `self._owns_connection = True`; keep `self.migrate()` at 160, passing the `InitDeadline` for clamping (F1-R16) and fixing its `except` (F1-R17) | the shared-journal branch (`:141–146`); `migrate()`'s `BEGIN IMMEDIATE` + version re-check; **default `check_same_thread=True`** |
| `SqliteRiskStateStore` (`:27–37`) | replace 31–36 with the helper — **this adds the missing `busy_timeout`** (D-3), **`check_same_thread` omitted → `True`** | `SCHEMA`, `key` semantics, `load/save/clear` |

**Why `check_same_thread` is a real behavioural difference and not a detail (C-1):**

| Value | Observable behaviour |
| --- | --- |
| `False` (journal today) | the connection may be used from a worker thread without `ProgrammingError` |
| `True` (ledger/state today) | using the connection from another thread raises `sqlite3.ProgrammingError` |

Changing either value silently changes thread-safety characteristics of the object. **Both are frozen**, and
T-F1-15 asserts the *observable* behaviour of each, so a future "cleanup" that unifies the keyword fails loudly.

#### 3.6.2 The shared-journal branch

`SqliteOperationLedger.__init__` with `journal is not None` (`:141–146`) **must not** call the helper: it does not
own the connection, and a second WAL transition on a borrowed connection is exactly the side effect requirement 9
forbids. It inherits the journal's already-verified mode.

#### 3.6.3 Migration path requirements (new in v1.1 — N-1, N-2)

| ID | Requirement |
| --- | --- |
| **F1-R16** | `migrate()` must accept the same `InitDeadline` and replace the literal `busy_timeout=10000` at `:192` with `clamp_busy_timeout(self._conn, deadline)`. |
| **F1-R17** | The `except sqlite3.OperationalError: pass` at `:208–213` must become: swallow **only** the "cannot start a transaction within a transaction" condition (matched by message/code), and **re-raise** `SQLITE_BUSY`/`SQLITE_LOCKED` as `LedgerUnavailable` (or `SqliteBusyTimeout` when the deadline has expired). |

**Inference:** without F1-R17, the deadline work is incomplete: a busy `BEGIN IMMEDIATE` would be treated as
"already in a transaction", and the migration would proceed believing it holds a lock it does not hold.

### 3.7 Behaviour matrix (requirement 8)

| Scenario | Initial state | Expected after remediation | Test |
| --- | --- | --- | --- |
| S1 | absent | WAL verified, schema created, no error | T-F1-01 |
| S2 | exists, **rollback** mode, with data | transitions to WAL; **row data unchanged** | T-F1-02 |
| S3 | exists, already WAL | no-op transition; data unchanged | T-F1-03 |
| S4 | absent, **2 processes** at once | both succeed; 0 transient errors over N | T-F1-04, T-F1-05 |
| S5 | absent, **4 processes** at once | all succeed; exactly one file set | T-F1-06 |
| S6 | foreign connector holds `BEGIN EXCLUSIVE` past deadline | `SqliteBusyTimeout` within deadline + 2 s; connection closed; no file deleted/recreated | T-F1-07, T-F1-16 |
| S7 | lock released after ~0.3 s | succeeds with `attempts ≥ 2` | T-F1-09 |
| S8 | corrupt file | `SqliteNotADatabase`, `attempts == 1`, `< 2 s`, message contains "database"; AU-35 still green | T-F1-10 |
| S9 | any | directory delta ⊆ {`-wal`, `-shm`} | T-F1-11 |
| S10 | forced `journal_mode` reply `"delete"` | `SqliteInitializationError` — a PRAGMA that did not take effect is a failure | T-F1-12 |
| S11 | ledger with `journal=J` | helper **not** called; journal's mode untouched | T-F1-13 |
| S12 | each adapter | `check_same_thread` behaviour identical to today | T-F1-15 |

### 3.8 Requirements trace for §3

| Requirement | Design item | Where |
| --- | --- | --- |
| 1 shared initializer + reason | F-1-D1 | §3.3 |
| 2 BUSY/LOCKED incl. **extended codes** | F-1-D1 §3.4.1 | §3.4.1 |
| 3 bounded backoff + **deadline as true cap** | §3.4.2, §3.4.4 | §3.4 |
| 4 no unbounded retry | §3.4.2 | §3.4 |
| 5 fail closed + clear error | §3.5 | §3.5 |
| 6 close on constructor failure | §3.3 step 10 | §3.3, T-F1-08 |
| 7 WAL actually verified | §3.3 step 7 | §3.3, T-F1-01/T-F1-12 |
| 8 new/existing/WAL/concurrent | §3.7 | §3.7 |
| 9 no parallel DB / deletion / path change | §3.3 "Never" | §3.3, T-F1-11 |
| 10 one policy for all three | §3.3 + §3.6.1 | §3.3, §3.6 |

---

## 4. F-2 Deterministic Concurrency Test Design

### 4.1 What "deterministic" must mean here

A test is **deterministic** if, for a given source tree, it yields the same verdict on every run on every
supported platform, with no dependence on scheduling. A test is a **valid detector** of a mutation if it fails,
**every time**, on the mutated tree.

**Fact shaping the design:** the current A7 harness achieves 7.5 % sensitivity against a faithful mutant and
0 % against removal of `_in_immediate`. Running the same recipe more often is not a design.

### 4.2 The three mechanisms, each with an explicit claim

| Mechanism | What it actually does | What it **may** claim | What it **may not** claim |
| --- | --- | --- | --- |
| **M-A. Same-transaction interleaving simulation** | Inside the adapter's own transaction, a hook executes one "advance" statement on the **same connection**, then the adapter's own write proceeds | "The adapter's write must be **column-targeted** / **guarded**; a whole-row write from a stale in-memory snapshot reverts the advance" | **It is not** a competing committed write. It does **not** prove isolation between transactions, and it must **never** be cited as multi-process evidence. |
| **M-B. Lock-level witness** | While the adapter is paused **inside** its transaction, a **second independent connection** with `busy_timeout=0` attempts `BEGIN IMMEDIATE` | "The adapter holds a **RESERVED** lock at that point" — i.e. the serialization primitive is actually in force | It does not prove that a real second process would be *blocked out*; it proves the lock is **held**. |
| **M-C. Real multi-process race** | Two OS processes contend end-to-end, as today | Nothing on its own — reported as an observed rate | Not a witness; never quoted as proof (class **P**). |

**Why M-B is the right instrument for `BEGIN IMMEDIATE` removal (C-4):**
if the mutation removes `_in_immediate`, the adapter's `SELECT` runs in **autocommit**, so no RESERVED lock is
held; the second connection's `BEGIN IMMEDIATE` **succeeds**. With the fix in place, it **fails with SQLITE_BUSY**
(probe §2.7 confirms `busy_timeout=0` yields code 5 immediately). The outcome is therefore a **deterministic
function of the presence of the primitive**, not of timing — the pause is controlled by an in-process `Event`.

### 4.3 Test catalogue

Every entry names its mechanism, actors, exact order, sync points, expectation, mutation and the reason the
mutation is **certainly** detected.

---

#### T-F2-01 — stale duplicate bump cannot revert an advanced row *(mechanism M-A, class D)*

* **Goal:** the duplicate counter bump must not write back any column other than `duplicate_requests`.
* **Initial state:** fresh file; one ENTRY op `APPROVED`, `claim_attempts=0`, `duplicate_requests=0`.
* **Actors:** ledger `L1` (one connection); the hook installed on `L1._conn.execute`; no second connection.
* **Exact order:**
  1. `record()` the operation.
  2. Install `PausingExecute` on `L1._conn`; it intercepts the statement matching
     `UPDATE operation_ledger SET … duplicate_requests`.
  3. A worker thread calls `L1.record(<same intent>)` → enters `_bump_duplicate` → `BEGIN IMMEDIATE` →
     targeted-parameter read (`_load`) → reaches the hook → **signals `reached`, blocks on `go`**.
  4. Main, **on the same connection, inside the same open transaction**, executes
     `UPDATE operation_ledger SET state='SUBMITTING', claim_attempts=1 WHERE op_id=?`.
  5. Main signals `go`; the worker's UPDATE executes; worker commits; `join(timeout=10)`.
* **Synchronization:** two `threading.Event`s (`reached`, `go`). No sleep anywhere.
* **Expected (fixed):** final `state == 'SUBMITTING'`, `claim_attempts == 1`, and `claim_for_send()` returns
  `False`. `duplicate_requests` is reported, **not asserted** (§4.5).
* **Mutation M-1:** `_bump_duplicate` → `self._update(self._apply_bump(existing))` (whole-row write).
* **Why certainly detected:** under M-1 the write is a full-row `UPDATE` built from the `existing` snapshot that
  was loaded **before** step 4, so it writes `state='APPROVED'` unconditionally. The advance in step 4 is
  guaranteed to have happened, therefore the revert is guaranteed. Probability 1.
* **Honest label in the test docstring (mandatory):** *"same-transaction interleaving simulation — verifies
  column-targeted writes; it is NOT a multi-process isolation proof."*

---

#### T-F2-02 — guarded digest normalisation rejects a changed hash *(mechanism M-A, class D)*

* **Goal:** the `WHERE op_id=? AND payload_hash=?` guard must reject a normalisation whose precondition no longer
  holds.
* **Initial state:** one op stored with a v1 hash, `digest_scheme='v1'`.
* **Exact order:** hook on the normalisation `UPDATE`; worker enters `record()` with legacy hashes; on the hook,
  main executes `UPDATE operation_ledger SET payload_hash='<other>' WHERE op_id=?` **on the same connection**;
  release.
* **Expected (fixed):** `record()` raises `LedgerUnavailable` (the guard found 0 rows, `:551`); the stored hash
  is the one main wrote; `digest_scheme` unchanged.
* **Mutation M-3:** drop `AND payload_hash=?`.
* **Why certainly detected:** with M-3 `rowcount == 1`, the other hash is overwritten, the assertion on the stored
  hash fails. Probability 1.

---

#### T-F2-03 — the serialization primitive is actually held *(mechanism M-B, class D)* — **the M-2 witness**

* **Goal:** deterministically detect removal of `BEGIN IMMEDIATE` from `transition()`.
* **Initial state:** one op in `APPROVED`.
* **Actors:** `L1` (the adapter's connection, paused), and `L2` = a **second independent `sqlite3` connection** to
  the same file, created with `timeout=0.0` and `PRAGMA busy_timeout=0`.
* **Exact order:**
  1. Install a pause hook on `L1` that fires **after** the in-transaction `_load` and **before** the `UPDATE`.
  2. Worker thread calls `L1.transition(op, EXPIRED)` → `BEGIN IMMEDIATE` → `_load` → **hook fires**:
     signals `reached`, blocks on `go`.
  3. Main opens `L2` and attempts `L2.execute("BEGIN IMMEDIATE")`.
  4. Main records the outcome, then signals `go`.
* **Synchronization:** the two `Event`s; the entire measurement happens while the worker is provably stopped.
* **Expected (fixed):** step 3 raises `sqlite3.OperationalError` with `primary_code == 5`. Then, after `go`, the
  operation reaches `EXPIRED` and `claim_for_send()` is `False`.
* **Expected (mutant M-2, `_in_immediate` removed):** step 3 **succeeds** — no exception — because no RESERVED
  lock is held.
* **Mutation M-2:** `transition()` calls `run()` instead of `self._in_immediate(run)`.
* **Why certainly detected:** the discriminator is "does `BEGIN IMMEDIATE` from a second connection raise BUSY at
  a moment when the first connection is provably inside its transaction". That is a logical consequence of lock
  state, not a race. The pause removes timing from the question entirely.
* **Complement:** a `Connection.set_trace_callback` assertion (stdlib) additionally records that `BEGIN IMMEDIATE`
  appears between the `SELECT` and the `UPDATE`. It is a **secondary, structural** witness and is **not** the
  acceptance instrument.

---

#### T-F2-04 — real two-process race *(mechanism M-C, class P — complementary only)*

* **Goal:** keep a genuine cross-process net and confirm the deterministic model is not fantasy.
* **Actors:** 2 `spawn`ed processes; construction **after** the barrier (F-2-D6, §5.7).
* **Expected:** exactly one `OperationClaimed`; `claim_attempts == 1`; exactly one submission line; final state
  `SUBMITTING`; `integrity_check == 'ok'`.
* **Reporting rule:** the result is quoted as **"N runs, M failures"**, never as proof.
* **Explicit:** this test is **not** the F-2 acceptance witness.

---

#### T-F2-05 — the safety consequence, asserted directly *(class D)*

* **Goal:** after any interleaving, no path may license a resend.
* **Expected:** `claim_for_send()` is `False`; `recover_incomplete()` leaves `UNKNOWN` untouched;
  `RESOLVED_ABSENT` unreachable without a complete query (already covered by the FIX-03 tests).

#### T-F2-06 — mutation harness *(class M, meta)*

* **Goal:** prove mechanically that the tests fail on their mutants.
* **Design:** copy the source tree to `tmp_path` (asserting the copy root is **not** under the repository),
  apply a scripted textual mutation, run the designated test in a subprocess against that copy, assert a non-zero
  exit **and** the expected failing test ID.
* **Required-catch list (minimum):** M-1 → T-F2-01 · M-2 → **T-F2-03** · M-3 → T-F2-02.
* **Expected:** every listed mutant caught, **5 of 5** harness runs.

#### T-F2-07 — M-2 must be in the required-catch list *(structural guard)*

* **Goal:** prevent a future edit from quietly dropping the hardest case.
* **Expected:** the harness's declared list contains `M-2` mapping to `T-F2-03`; the test fails if it is removed.

### 4.4 Claim boundaries (C-4) — binding on every report written under this spec

| Statement | Allowed? |
| --- | --- |
| "T-F2-01/02 prove the write is column-targeted / guarded" | **Yes** |
| "T-F2-03 proves the serialization primitive is held during the RMW" | **Yes** |
| "T-F2-01..03 prove two processes are isolated" | **No** |
| "The deterministic tests prove the absence of a Lost Update **across processes**" | **No** |
| "T-F2-04, over N runs, produced M failures" | **Yes** |
| "A7 is now deterministic" | **No** — A7 remains class P |

**Inference to be stated plainly in any future report:** same-process, same-transaction interleaving and
lock-level witnessing are *stronger than* the current probabilistic test for the properties they capture, but
they are *not* a substitute for cross-process evidence. Cross-process safety remains evidenced only by (a) the
lock being held — which is cross-process by nature, since SQLite locks are file-level — and (b) the class-P test.

### 4.5 What the deterministic tests must not do

| Anti-pattern | Why refused |
| --- | --- |
| `sleep()` to widen a window | Non-deterministic by construction. |
| Repeat a race N times and require "most pass" | Cannot distinguish 1 % from 0 %; E-2 measured exactly this. |
| Assert an exact `duplicate_requests` | The contract permits either +1 or a detected change; pinning one would over-specify and couple the test to the implementation. **The invariant is asserted; the counter is reported.** |
| Use an in-process `threading.Lock` as the "other actor" | Proves nothing about the database; M-A uses the real connection, M-B uses a real second connection. |
| Edit or delete A7 | Forbidden. A7 stays; its only permitted change is the disclosed strengthening (§5.7). |

### 4.6 F-2-D7 — document the atomicity classes (comment-only)

Add a docstring block to `sqlite_operation_ledger.py` tabulating §2.4: which methods are `_in_immediate`
read-modify-write and which are single-statement atomic, and why the second class is safe **only** while it
remains one statement. Rationale: the audit brief itself mis-stated this, which proves it is not discoverable
from the code. Comment-only = still Gate-2.

---

## 5. F-3 Windows Portability Design

### 5.1 Files and lines in scope

| File | Line(s) | Change |
| --- | --- | --- |
| `tests/test_phase_a_audit_multiprocess.py` | 66, 112 | `mp.get_context("fork")` → `get_mp_context()` (spawn) |
| `tests/test_fix05_counter_semantics.py` | 181 | same |
| `tests/test_phase_a_audit_multiprocess.py` | 36–37, 220–221 (construction), 39, 257 (barrier) | move construction **after** the barrier — **strengthening, separate gate** |
| new `tests/mp_support.py` | — | context factory, bounded teardown, diagnostics |

### 5.2 F-3-D1 — `spawn` on every platform

**Proposal:** one start method (`spawn`) everywhere. **Justification:** a test running a different mechanism on
the developer machine than on the target is not evidence about the target; `fork` inherits the parent's SQLite
connections (undefined for SQLite) and can produce false passes; `spawn` matches production. **Cost:** ~0.3–0.8 s
per worker; accounted for in §7.2.

### 5.3 F-3-D2 — worker contract

1. **Module-level function only** — never a closure/lambda/bound method. *(Fact: all three workers already
   comply.)*
2. **Picklable, minimal arguments** — `str`, `int`, Manager proxies. *(Fact: compliant.)*
3. **No inherited mutable state.** *(Fact: the modules use immutable constants only — `SCOPE`, `INTENT`,
   `DIGEST`.)*
4. **Mandatory module-level import guard** in every spawning test module:

   ```python
   import sys
   from pathlib import Path
   _SRC = Path(__file__).resolve().parents[1] / "src"
   if str(_SRC) not in sys.path:
       sys.path.insert(0, str(_SRC))
   ```

   It runs in the parent **and** in every spawned child, making the `goldbot` import independent of
   `conftest.py` and of `pytest.ini`'s `pythonpath`. **This is the concrete mitigation for A-3.**
5. **`if __name__ == "__main__":`** — **correction of a common claim:** under pytest this is **not** required,
   because the module is imported by the pytest entry point, not executed as `__main__`. Add the guard only in
   standalone reproduction scripts under `tools/`, and document *why* it is unnecessary under pytest so nobody
   "fixes" a non-problem.

### 5.4 F-3-D3 — primitives that survive `spawn`

| Primitive | Under `spawn` | Verdict |
| --- | --- | --- |
| `multiprocessing.Barrier` (plain) | not shareable | **Forbidden** cross-process |
| `Manager().Barrier(n)` | proxy — picklable | **Required** *(already used: `:68`, `:114`)* |
| `Manager().Queue()` | proxy | **Required** *(already used)* |
| plain `multiprocessing.Event()` | not shareable cross-process | **Forbidden** cross-process; fine for **threads** (T-F2-01..03) |
| `Manager().Event()` | proxy | allowed where genuinely needed |

**Inference:** because the in-scope tests already use Manager proxies, switching to `spawn` is essentially
one line per site plus the import guard. **F-3 is cheap; this must be stated so it is not scoped as a rewrite.**

### 5.5 F-3-D4 — module purity

A spawned child re-imports the test module; therefore in-scope modules must have **no module-level side effects**
(no I/O, no DB opening, no clock reads). *(Fact: currently compliant.)* Guarded by T-F3-06.

### 5.6 F-3-D5 — bounded teardown, no new dependency

**Facts:** no `pytest-timeout`; no `psutil` in `requirements.txt`; a dead worker currently manifests as a stall
and, per E-2, a **60.10 s** red run whose message was `BrokenBarrierError` — the harness consumed the signal
instead of reporting the cause.

**Proposal — `tests/mp_support.py`:**

| Function | Behaviour |
| --- | --- |
| `get_mp_context()` | `mp.get_context("spawn")` — single source of truth |
| `run_workers(ctx, target, args_list, *, join_timeout, result_timeout)` | start all; join each; on timeout `terminate()` → `join(5)` → `kill()`; return `[(exitcode, payload)]`; raise a diagnostic naming the dead worker and its exit code |
| `finish(manager)` | `manager.shutdown()` in `finally` — today no manager is explicitly shut down *(Fact)* |
| `assert_clean_exit(results)` | fail with `exitcode` + captured `repr` when a worker reported `error` |

**Thresholds (Proposal):** `join_timeout = 120 s` and `result_timeout = 60 s` — unchanged from today *(Fact:
`proc.join(timeout=120)`, `results.get(timeout=60)`)*. **Ordering constraint:** the F-1 deadline (30 s) must stay
**below** `result_timeout` (60 s) so a lock timeout surfaces as `SqliteBusyTimeout`, not a queue timeout.

**Stdlib only.** `terminate`/`join`/`kill`/`exitcode` suffice; `requirements.txt` stays one line.

### 5.7 F-2/F-3 overlap — one disclosed strengthening of A7

**Proposal F-2-D6:** move `SqliteJournal(...)`/`SqliteOperationLedger(...)` **after** `go.wait()` in AU-22
(`:36–37`) and AU-23 (`:220–221`), keeping every existing assertion. This strictly **adds** coverage
(initialization becomes part of the race), so it does not weaken anything.

**Hard ordering constraint:** this must land **after** F-1. Implementing it first would make A7 fail at roughly
the F-1 rate (4.4 %–46.7 %), which would look exactly like a regression introduced by a test change.

### 5.8 Windows risks this spec does **not** resolve

| Risk | Status |
| --- | --- |
| The real F-1 rate on Windows file locking | **Unknown — NOT RUN** |
| `msvcrt` vs `fcntl` in `LiveInstanceLock` (`live_instance_lock.py:44–62`) | **Out of scope** (OOS-6) |
| Antivirus/indexer holding the DB open on a VPS | **Unknown**; mitigated only by the explicit deadline failure |
| `spawn` cost on a 2-vCPU Windows VPS | **Unknown**; affects wall-clock, not verdict |
| Path length / unicode path handling | **Unknown**, unchanged here |

---

## 6. Source File Impact Matrix

| File | Change | Lines (current) | Requirement | Risk | Gate |
| --- | --- | --- | --- | --- | --- |
| `src/goldbot/adapters/sqlite_init.py` | **NEW** (~140 lines) | — | F1-R1…R18 | Low — additive | 1 |
| `src/goldbot/adapters/sqlite_store.py` | modify | 153–169 | F1-R1, R5, R6, R13 | **Medium** — 58 call sites; hash `a74e06db…` will change | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | modify | 147–160, 191–213 | F1-R1, R13, R16, R17 | Medium — order path | 1 |
| `src/goldbot/adapters/sqlite_state_store.py` | modify | 27–37 | F1-R1, R2, R13 | Low | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | comment-only | near `:440` | F-2-D7 | None | 2 |
| `tests/test_f1_wal_initialization.py` | **NEW** | — | F-1 acceptance (fast lane) | Low | 1 |
| `tests/test_f1_wal_initialization_slow.py` | **NEW** (env-gated `GOLDBOT_WAL_ACCEPTANCE=1`) | — | F-1 acceptance (300-lane) | Low | 1 |
| `tests/test_f2_deterministic_lost_update.py` | **NEW** | — | F-2 | Low | 2 |
| `tests/test_f2_mutation_harness.py` | **NEW** | — | F-2 (T-F2-06/07) | Medium — copies+muts source in tmp | 2 |
| `tests/mp_support.py` | **NEW** | — | F-3 | Low | 3 |
| `tests/test_phase_a_audit_multiprocess.py` | modify (strengthen) | 36–37, 66, 112, 220–221 | F3-R1, F2-D6 | **Medium** — audit test | 3 (after 1) |
| `tests/test_fix05_counter_semantics.py` | modify | 181 (+ guard) | F3-R2 | Low | 3 |
| `docs/adr/` | **NEW ADR** | — | records §3.3/§3.4/§3.5 | None | 1 |
| `SOURCE-MANIFEST.txt` | regenerate | — | project rule | — | 4 |

**Explicitly untouched:** `domain/fsm/order.py`, `ports/ledger.py` contracts, `app/order_coordinator.py`,
`adapters/mt5_broker.py`, `runtime/*`, `web/*`, any DDL/schema, any magic/risk/strategy file, `requirements.txt`.

**Carry-forward for the implementation report:** `sqlite_store.py`'s SHA-256 will no longer be `a74e06db…`;
both the old and new hashes must be recorded so the "pre-existing" claim stays auditable.

---

## 7. Test Matrix and Acceptance Criteria

### 7.1 Test classes

| Class | Meaning | Acceptance witness? |
| --- | --- | --- |
| **D** Deterministic | forced interleaving / logical lock-state witness; same verdict every run | **Yes** |
| **P** Probabilistic | scheduling-dependent; reported as a rate | **No** |
| **S** Sampling/stress | N repetitions bounding a rate | Yes, for F-1 with §7.2's power |
| **M** Meta/mutation | proves another test detects a mutation | **Yes**, for the mutation claim |
| **X** Not runnable | Windows / PowerShell / MT5 | **No** — carries `NOT RUN` |

### 7.2 Sample size and lanes

With residual per-iteration failure probability *p*, *n* iterations miss it with probability (1−p)ⁿ. To claim
"residual ≤ 1 % at 95 % confidence": (1−0.01)ⁿ ≤ 0.05 ⇒ **n ≥ 298 ⇒ n = 300**, acceptance = **0 failures in 300**.

| n | Smallest rate detectable at 95 % power | Verdict |
| --- | --- | --- |
| 10 | ~26 % | useless |
| 40 | **~7.2 %** | **insufficient** — E-2 measured 4.4 %; a 4.4 % regression would pass |
| 100 | ~3.0 % | marginal |
| **300** | **~1.0 %** | **acceptance lane** |

| Lane | Trigger | n | Purpose |
| --- | --- | --- | --- |
| `fast` | default pytest | 40/adapter, 1 scenario | smoke |
| `acceptance` | `GOLDBOT_WAL_ACCEPTANCE=1` | **300**/adapter (S4) + 100 (S5) | the F-1 witness |
| `deadline` | default | 3 | deterministic fail-closed proof |

**Cost estimate:** 300 iterations × 3 adapters at ~0.5–0.8 s ≈ **8–12 min** — hence opt-in. **The 40-iteration
fast lane may never be cited as proof** (§7.2 table).

### 7.3 F-1 test matrix

| ID | Class | Scenario | Key assertions | Acceptance |
| --- | --- | --- | --- | --- |
| T-F1-01 | D | S1, ×3 adapters | no exception; `PRAGMA journal_mode` == `wal` | 3/3 |
| T-F1-02 | D | S2 rollback DB, 100 rows | mode becomes wal; row values byte-identical | pass |
| T-F1-03 | D | S3 existing WAL | no error; no data change | pass |
| T-F1-04 | S | S4 two processes | all constructors return; one file set | **0/300** |
| T-F1-05 | S | S4 **pre-fix tree** (negative control) | — | **≥1 failure in 300** — proves the lane is sensitive |
| T-F1-06 | S | S5 four processes | as T-F1-04 | **0/100** |
| T-F1-07 | D | S6 lock beyond deadline | `SqliteBusyTimeout`; `elapsed ≤ deadline + 2 s`; message has path/attempts/deadline | pass |
| T-F1-08 | D | connection leak | patched `sqlite3.connect` recorder; every recorded conn raises "closed" | **0 leaks / 100** |
| T-F1-09 | D | S7 short lock (~0.3 s) | succeeds; `attempts ≥ 2` | pass |
| T-F1-10 | D | S8 corrupt file | `SqliteNotADatabase`; `attempts == 1`; `elapsed < 2 s`; message contains "database"; **AU-35 unmodified still green** | pass |
| T-F1-11 | D | S9 no stray files | `set(os.listdir)` delta ⊆ {`-wal`, `-shm`}; nothing deleted | pass |
| T-F1-12 | D | S10 forced `"delete"` reply | `SqliteInitializationError` | pass |
| T-F1-13 | D | S11 shared-journal branch | helper not called; journal mode untouched; `migrate()` still serialized | pass |
| **T-F1-15** | D | **S12 `check_same_thread` regression** | journal: usable from a worker thread (no `ProgrammingError`); ledger & state: using from a worker thread raises `ProgrammingError` — for **all three adapters**, on first use, not just at construction | **3/3** |
| **T-F1-16** | D | **deadline is a hard cap** | `deadline_s=1.0`, foreign `BEGIN EXCLUSIVE` held 10 s → `SqliteBusyTimeout` with `elapsed ≤ 3.0 s` | pass |
| **T-F1-17** | D | **per-attempt invariant** | from `InitReport`: for every attempt, `busy_timeout_ms_used ≤ remaining_ms_before`; and `sum(delays) ≤ budget` | pass |
| **T-F1-18** | D | **migrate() must not swallow BUSY** | hold `BEGIN EXCLUSIVE` while a v1→v3 migration runs → raises (`LedgerUnavailable`/`SqliteBusyTimeout`), never silently `pass`; `integrity_check` still `ok` afterwards | pass |
| **T-F1-19** | D | **extended-code classification** | synthetic errors carrying codes `5, 261, 517, 773, 6, 262` → retryable; `26, 14, 8, 3, 11` → non-retryable, `attempts == 1`; **code absent → message fallback used** | pass |
| T-F1-14 | X | Windows | — | **NOT RUN** |

### 7.4 F-2 test matrix

| ID | Class | Mechanism | Property | Mutation caught | Sensitivity |
| --- | --- | --- | --- | --- | --- |
| T-F2-01 | D | M-A | column-targeted bump cannot revert a claim | M-1 | **100 %** |
| T-F2-02 | D | M-A | guarded normalisation rejects changed hash | M-3 | **100 %** |
| **T-F2-03** | **D** | **M-B** | **serialization primitive is held during the RMW** | **M-2** | **100 %** |
| T-F2-04 | P | M-C | real two-process claim | — | reported only |
| T-F2-05 | D | — | no resend ever licensed | — | 100 % |
| T-F2-06 | M | — | harness catches M-1/M-2/M-3 | itself | **5/5** |
| T-F2-07 | M | — | M-2 stays in the required-catch list | edit of the list | 100 % |

**Acceptance:** T-F2-01/02/03/05 pass on the unmutated tree; T-F2-06 fails on every listed mutant in **5/5**
runs — on Linux **and** on Windows once Gate-5 exists. **T-F2-04 is never quoted as proof** (§4.4).

### 7.5 F-3 test matrix

| ID | Class | Assertion | Linux | Windows |
| --- | --- | --- | --- | --- |
| T-F3-01 | D | `get_mp_context().get_start_method() == "spawn"` | to be run | **NOT RUN** |
| T-F3-02 | D | every worker target `pickle.dumps`-able | to be run | **NOT RUN** |
| T-F3-03 | D | a raising worker yields a diagnosable failure, not a hang | to be run | **NOT RUN** |
| T-F3-04 | D | a hung worker is terminated within `2+5+ε` s | to be run | **NOT RUN** |
| T-F3-05 | D | no plain `Barrier`/`Event` cross-process | to be run | **NOT RUN** |
| T-F3-06 | D | module import performs no I/O | to be run | **NOT RUN** |
| T-F3-07 | D | `mp.active_children()` empty after each test | to be run | **NOT RUN** |
| T-F3-08 | X | AU-22/AU-23/crash test under `spawn` | 20/20 on Linux | **NOT RUN** |

**Reporting rule (C-6):** every table in every future report must carry **two** result columns — `Linux` and
`Windows`. Windows cells are `NOT RUN` until Gate-5 produces real logs from a Windows host.

---

## 8. Failure Handling and Observability

| Condition | Detected by | Behaviour | Operator sees | Test |
| --- | --- | --- | --- | --- |
| Transient BUSY (< deadline) | retry loop | retries with jittered backoff | nothing (INFO line) | T-F1-09 |
| Persistent BUSY (> deadline) | deadline | **fail closed** | `SqliteBusyTimeout`: path, attempts, elapsed, deadline, `last_code`, name, chained cause | T-F1-07/16 |
| WAL not applied | post-transition verify | fail closed | `SqliteInitializationError`, reason `journal_mode is 'delete', not 'wal'` | T-F1-12 |
| Corrupt / non-DB | immediate classify | fail fast (< 2 s), `attempts == 1` | `SqliteNotADatabase`; message contains "database" | T-F1-10 |
| Other non-retryable | immediate classify | fail fast | `SqliteUnretryableError` with code and name | T-F1-19 |
| Migrate `BEGIN IMMEDIATE` busy | F1-R17 | raise, **never** silent `pass` | `LedgerUnavailable` with cause | T-F1-18 |
| Worker dies | `run_workers` | terminate→join(5)→kill | failure naming worker + exit code | T-F3-03 |
| Worker hangs | `join_timeout` | same, bounded | ditto | T-F3-04 |
| Hook never fires | test-side `Event.wait(timeout)` | test fails "hook never fired" | distinguishes "path changed" from "assertion failed" | T-F2-01/03 |
| Mutant escapes | harness | harness fails | names the escaping mutant | T-F2-06 |

**Proposal — one structured INFO line per initialization**, using the project's existing convention
(`log.warning(..., extra={"fields": {...}})`, `runtime/live.py:361`):
`path`, `attempts`, `elapsed_ms`, `journal_mode_after`, `created`, `last_code`.
Rationale: F-1 is a *rate*; without this line an operator cannot learn that a deployment is contending.

**Not proposed:** overloading `migration_report()` with connection state — it is a schema artefact and coupling
two concerns would be a defect of its own (Q-7).

---

## 9. Risks and Non-Goals

### 9.1 F-1 risks

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| New exception type escapes a caller that catches `sqlite3.Error` | **Unknown — A-4** | The Gate-1 review mandated in §3.5.4 lists every reachable `except` clause; cause chains preserved |
| 30 s too short on a loaded Windows VPS | Medium | Parameterised; T-F1-07/16 pin the *contract* not the value; Q-3 |
| Retry masks a permanent lock and turns a fast failure into a 30 s wait | Medium | Only codes 5/6 retry; everything else fails in `attempts == 1` (T-F1-19) |
| `migrate()` re-clamping changes migration timing | Low | F1-R16 clamps **downward only**, never above the configured 10 s |
| `sqlite_store.py` hash changes | Low | Old+new hashes recorded (§6) |

### 9.2 F-2 risks

| Risk | Mitigation |
| --- | --- |
| M-A is judged "not a real race" | It is **declared** as a same-transaction simulation and claims nothing beyond column-targeting; M-B supplies the lock-state evidence; M-C is kept separately |
| M-B proves "lock held" but not "process blocked" | Stated in §4.4; the property SQLite guarantees is file-level locking, so held-reserved is the correct witness |
| The mutation harness mutates the wrong tree | It must copy to `tmp_path` and **assert** the copy root is outside the repository (§4.3 T-F2-06) |
| Pausing inside a transaction holds a lock for the pause duration | Bounded by in-process Events (milliseconds); the test asserts the pause fired; no sleeps |

### 9.3 F-3 risks

| Risk | Mitigation |
| --- | --- |
| `spawn` child cannot import the module or `goldbot` | Mandatory module-level `sys.path` guard (§5.3 rule 4) |
| Wall-clock grows 3–5× | Fast lane stays small; acceptance lane opt-in |
| A7 becomes flaky if changed before F-1 | §10 enforces the order; this is the single most important sequencing rule |

### 9.4 Non-goals (no implementation may expand into these)

Order Coordinator architecture · trading/strategy/indicator logic · broker/MT5 behaviour · schema or migration
DDL changes · database path or name changes · deletion/recreation of any real database · MT5/broker connection ·
demo or real orders · VPS configuration · deployment to a Windows VPS · operational soak · PHASE B · PHASE C ·
production-readiness claims · `requirements.txt` changes · `LiveInstanceLock` semantics.

**This spec proposes no DDL change at all.** If implementation finds one is required, it must stop and request a
separate gate.

---

## 10. Implementation Sequence

| Step | Content | Gate | Blocking constraint |
| --- | --- | --- | --- |
| 0 | Owner reads v1.1 and answers Q-1…Q-9 | **Gate-0** | nothing proceeds |
| 1 | `sqlite_init.py` + its unit tests (T-F1-01/02/03/07/08/10/11/12/15/16/17/19) | 1 | — |
| 2 | Wire the three adapters (§3.6.1) + F1-R16/R17 in `migrate()` | 1 | **No test file edited in this step** |
| 3 | `test_f1_wal_initialization.py` (T-F1-04/05/06/09/13/18) + slow lane | 1 | T-F1-05 must fail on the pre-fix tree |
| 4 | Gate-1 evidence: §3.5.4 `except`-clause review + acceptance lane (300×3) + fast lane | 1→4 | Linux only; Windows `NOT RUN` |
| 5 | F-3 `mp_support.py`, `spawn`, import guard, teardown | 3 | **Step 2 merged first** |
| 6 | F-2-D6: construction after the barrier in AU-22/AU-23 | 3 | **Only after step 2** |
| 7 | F-2 tests T-F2-01…03, 05…07 | 2 | new files only |
| 8 | F-2-D7 docstring; regenerate `SOURCE-MANIFEST.txt`; ADR | 2 | — |
| 9 | Independent verification of a new ZIP (hash, manifest, suite, mutation harness) | 4 | Linux PASS / Windows NOT RUN, separately |
| 10 | Windows execution: T-F1-14, T-F3-08 | **5** | requires a real Windows host; `NOT RUN` otherwise |

**Hard rules**
1. **F-1 before the A7 edit.** Reversing it makes an audit test intermittent.
2. **No test file edited in steps 1–3.** The "before" measurement must survive.
3. **One ZIP per gate**, so each gate is independently verifiable.

---

## 11. Independent Verification Plan

| # | Check | Method | Expected | Platform |
| --- | --- | --- | --- | --- |
| V-1 | Delivery integrity | `sha256sum <zip>`; `sha256sum -c SHA256SUMS.txt` | recorded hash; 100 % OK | Linux |
| V-2 | Manifest | recompute `SOURCE-MANIFEST.txt` | `files: <n> \| problems: 0` | Linux |
| V-3 | No weakened tests | diff the test tree against the previous ZIP; assert **no removed assertion** in pre-existing files | only disclosed strengthenings | Linux |
| V-4 | Suite | `GOLDBOT_SOAK=0 pytest -q` | `1224 + <new> passed, 6 skipped` | Linux |
| V-5 | F-1 acceptance | acceptance lane, n=300, 3 adapters | 0 failures; raw log attached | **Linux** |
| V-6 | F-1 **negative control** | same lane on the pre-fix tree | **>0 failures** | **Linux** |
| V-7 | F-1 deadline | T-F1-16 | error at `≤ deadline + 2 s`; connection closed | Linux |
| V-8 | F-2 determinism | T-F2-01/02/03 × 20 consecutive | **20/20 identical verdict** | Linux |
| V-9 | F-2 mutation | T-F2-06 | every mutant caught, 5/5 | Linux |
| V-10 | F-3 start method | T-F3-01 + `grep '"fork"'` = 0 | pass | Linux |
| V-11 | F-3 hang safety | T-F3-04 | bounded termination | Linux |
| V-12 | Windows items | T-F1-14, T-F3-08 | **NOT RUN** unless a Windows host is provided | **Windows** |

**Rule:** V-6 is mandatory. An acceptance lane not shown to be *sensitive* is not evidence of absence — E-2's
0/40 on A7 against a tree with a reproducible 4.4 % startup failure is the proof of that risk.

---

## 12. Explicit Authorization Gates (C-6 — corrected boundaries)

Each gate reports **Linux** and **Windows** separately. **A gate may never be marked `CLOSED` while any of its
Windows items is `NOT RUN`;** such a gate is `PARTIALLY CLOSED — WINDOWS OUTSTANDING`.

| Gate | Authorizes | Does **not** authorize | Closes on | Linux | Windows |
| --- | --- | --- | --- | --- | --- |
| **Gate-0** | Reading and commenting on this spec | **any edit** | owner's answers to §14 | — | — |
| **Gate-1** | `sqlite_init.py`; the three adapter wirings; F1-R16/R17; F-1 tests; ADR | F-2, F-3, **any edit to an existing test**, any DDL | §7.3 (T-F1-01…13, 15…19) + V-5 + **V-6** + §3.5.4 review | `CLOSED` allowed if all green | **NOT RUN** → gate is `PARTIALLY CLOSED` |
| **Gate-2** | F-2 test files, mutation harness, comment-only docstring | any behavioural source change | §7.4 + T-F2-06 5/5 | `CLOSED` allowed | **NOT RUN** → `PARTIALLY CLOSED` |
| **Gate-3** | `mp_support.py`, `spawn`, import guards, A7 strengthening | **any assertion removal** | §7.5 T-F3-01…07 on Linux | `CLOSED` allowed | **NOT RUN** → `PARTIALLY CLOSED` |
| **Gate-4** | Independent verification of the resulting ZIP | any further change | V-1…V-11 | `CLOSED` allowed | **NOT RUN**; the report must state it |
| **Gate-5** | Windows execution and any Windows-specific adjustment | anything else | V-12 with real Windows-host logs | — | **the only gate that can close Windows items** |

**Standing rules**
* The delivery ZIP is never edited in place: extract → hash → verify → independent copy.
* One new ZIP per gate; each gate's ZIP hash is recorded in its report.
* **No gate may contain the phrase "Windows tested"** without logs from a Windows host in that gate's evidence
  folder. Absent that, the only permitted wording is `Windows: NOT RUN`.

---

## 13. Traceability Matrix

Columns: **ID · Finding · Source Location · Required Change · Test ID · Acceptance · Evidence**

### 13.1 F-1

| ID | Finding | Source Location | Required Change | Test ID | Acceptance | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F1-R1** | F-1 | `sqlite_store.py:160` | WAL transition via `open_sqlite_connection()` | T-F1-01, 04 | 0/300 pairs; 3/3 single | E-1 56/120; E-2 11/250 |
| | | `sqlite_operation_ledger.py:153` | same | T-F1-01, 04, 13 | as above; shared branch untouched | E-1 23/40 |
| | | `sqlite_state_store.py:33` | same **+ add missing `busy_timeout`** | T-F1-01, 04 | as above | E-1 22/40; Fact D-3 |
| **F1-R2** | F-1 | `sqlite_store.py:165`, `sqlite_operation_ledger.py:155`, `sqlite_state_store.py` (absent) | uniform `busy_timeout` inside the helper | T-F1-13 | all three report the same value | §2.2 |
| **F1-R3** | F-1 | three sites | bounded backoff + deadline as a **true cap** | T-F1-07, 09, **16**, **17** | S7 ≥2 attempts; S6 fails ≤ deadline+2 s; per-attempt invariant holds | §3.4.4 |
| **F1-R4** | F-1 | three sites | no unbounded retry | T-F1-16 | elapsed ≤ deadline + 2 s | §3.4.3 |
| **F1-R5** | F-1 | three constructors | fail closed with a typed, diagnosable error | T-F1-07, 10, 19 | taxonomy correct; fields present | §3.5 |
| **F1-R6** | F-1 | three constructors | close the connection on every failure path | T-F1-08 | 0 leaks / 100 | D-5 |
| **F1-R7** | F-1 | three constructors | verify `journal_mode == 'wal'` | T-F1-01, 12 | 100 %; forced fallback raises | no existing test |
| **F1-R8** | F-1 | three constructors | defined behaviour S1…S12 | T-F1-01…06, 09, 13, 15 | §3.7 covered | §3.7 |
| **F1-R9** | F-1 | three constructors | no parallel DB / deletion / path change | T-F1-11 | delta ⊆ {`-wal`,`-shm`} | requirement 9 |
| **F1-R10** | F-1 | all three | one policy, one helper | T-F1-13 + review | `PRAGMA journal_mode=WAL` appears **once** in `src/` | §2.2; D-3/D-7 |
| **F1-R11** | F-1 | §2.3 | document that the Live lock is not the remedy | — (doc) | §2.3 present | E-2 |
| **F1-R12** | F-1 | `sqlite_operation_ledger.py:141–146` | shared-journal branch must **not** call the helper | T-F1-13 | mode untouched | Proposal |
| **F1-R13** | F-1 (**new v1.1**) | `sqlite_store.py:158` (`False`), `sqlite_operation_ledger.py:151` / `sqlite_state_store.py:31` (default `True`) | `check_same_thread` is an explicit helper parameter; **per-adapter value frozen** | **T-F1-15** | journal usable cross-thread; ledger/state raise `ProgrammingError` | Fact (only the journal sets it) |
| **F1-R14** | F-1 (**new v1.1**) | three constructors | deadline caps the **whole** init; remaining recomputed per attempt; `connect(timeout=)` and `busy_timeout` clamped | **T-F1-16, T-F1-17** | `elapsed ≤ deadline + 2 s`; `busy_timeout_used ≤ remaining` | §3.4.4 |
| **F1-R15** | F-1 (**new v1.1**) | helper classification | primary code = `code & 0xFF`; retry `{5,6}`; text only if code absent; others fail fast | **T-F1-19** | 6 retryable / 5 non-retryable / fallback case | probe §2.7 |
| **F1-R16** | F-1 (**new v1.1, Finding N-1**) | `sqlite_operation_ledger.py:192` | `clamp_busy_timeout(...)` instead of literal 10 000 | T-F1-17 (ledger variant) | clamped value ≤ remaining | §2.7 |
| **F1-R17** | F-1 (**new v1.1, Finding N-2**) | `:208–213` | swallow **only** "transaction within transaction"; re-raise BUSY/LOCKED | **T-F1-18** | busy during migrate raises; no silent `pass` | §2.7 |
| **F1-R18** | F-1 (**new v1.1**) | helper + adapters | 4-way taxonomy, cause chains, message token "database", AU-35 untouched | T-F1-07, 10, 19 + AU-35 | all four classes reachable; AU-35 green | §3.5; probe |

### 13.2 F-2

| ID | Finding | Source Location | Required Change | Test ID | Acceptance | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F2-R1** | F-2 | `test_phase_a_audit_multiprocess.py:36–37, 220–221` | deterministic tests added in a **new** file | T-F2-01…03 | 20/20 identical | E-2 3/40 |
| **F2-R2** | F-2 | `sqlite_operation_ledger.py:503–523` | prove column-targeted bump | T-F2-01, 05 | M-1 caught 5/5 | §2.4 |
| **F2-R3** | F-2 | `:446–501` | prove the serialization primitive is held | **T-F2-03**, 05 | M-2 caught 5/5 | E-2 0/60 |
| **F2-R4** | F-2 | `:525–560` | prove the hash guard rejects | T-F2-02 | M-3 caught 5/5 | §2.4 |
| **F2-R5** | F-2 | `:562–592`, `:594–614` | document the single-statement atomic class | T-F2-07 + docstring | docstring present; no behaviour change | Fact correction §2.4 |
| **F2-R6** | F-2 | new harness | mechanical mutation detection | T-F2-06 | 5/5 catches | §4.3 |
| **F2-R7** | F-2 | A7 file | only the disclosed strengthening | T-F2-04 (P) | no assertion removed | §5.7 |
| **F2-R8** | F-2 (**new v1.1**) | new tests | simulation labelled **same-transaction interleaving**; no cross-process claim | all F-2 tests' docstrings | §4.4 wording honoured | C-4 |
| **F2-R9** | F-2 (**new v1.1**) | new test | **lock-level witness** catches `BEGIN IMMEDIATE` removal | **T-F2-03** | M-2 → 100 % detection | probe §2.7 (code 5 at `busy_timeout=0`) |
| **F2-R10** | F-2 (**new v1.1**) | A7 file | real multi-process test kept, labelled **P**, never a witness | T-F2-04 | reported as rate only | §4.4 |

### 13.3 F-3

| ID | Finding | Source Location | Required Change | Test ID | Acceptance (Linux) | Windows |
| --- | --- | --- | --- | --- | --- | --- |
| **F3-R1** | F-3 | `test_phase_a_audit_multiprocess.py:66, 112` | `spawn` via factory | T-F3-01 | zero `"fork"` | NOT RUN |
| **F3-R2** | F-3 | `test_fix05_counter_semantics.py:181` | same | T-F3-01 | zero `"fork"` | NOT RUN |
| **F3-R3** | F-3 | three workers | module-level, picklable, no parent state | T-F3-02 | pickle OK | NOT RUN |
| **F3-R4** | F-3 | in-scope modules | module-level `sys.path` guard | T-F3-06 | child imports OK | NOT RUN |
| **F3-R5** | F-3 | `mp_support.py` | bounded teardown + `Manager.shutdown()` | T-F3-04, 07 | ≤7 s; no children | NOT RUN |
| **F3-R6** | F-3 | in-scope tests | diagnosable worker death | T-F3-03 | names worker + code | NOT RUN |
| **F3-R7** | F-3 | in-scope tests | Manager proxies only | T-F3-05 | pass | NOT RUN |
| **F3-R8** | F-3 | — | Windows execution | T-F3-08, T-F1-14 | — | **NOT RUN (Gate-5 only)** |
| **F3-R9** | F-3 (**new v1.1**) | every report | Linux and Windows reported in **separate** columns | — | enforced | — |

### 13.4 Out-of-Scope Findings (recorded, not addressed)

| ID | Finding | Location | Why out of scope | Owner decision |
| --- | --- | --- | --- | --- |
| **OOS-1** | State store has no `busy_timeout` | `sqlite_state_store.py:31–36` | In scope **only** via F1-R2 | — |
| **OOS-2** | Atomicity class of `claim_for_send`/`mark_submit_attempt` undocumented and mis-stated in the brief | `:562`, `:594` | Doc-only (F-2-D7) | Gate-2 |
| **OOS-3** | 58 `SqliteJournal(` sites, no factory | `src/` 12, `tests/` 46 | Architectural | PHASE C |
| **OOS-4** | `check_same_thread=False` on the journal connection | `sqlite_store.py:158` | Thread-safety not assessed; **value frozen by C-1** | PHASE C |
| **OOS-5** | CLI/web open the same DB as a running Live process with no lock | §2.3 | Ownership model | PHASE C |
| **OOS-6** | `LiveInstanceLock` `msvcrt`/`fcntl`, no Windows test | `live_instance_lock.py:44–62` | Not F-1/F-2/F-3 | Gate-5 |
| **OOS-7** | No test asserts `journal_mode == 'wal'` | whole suite | **Inside** F-1 (T-F1-01/12) | Gate-1 |
| **OOS-8** | `migrate()` backup contention (~20 s stall, per handover §5) | `:185–220` | Recorded as already mitigated; not re-verified | Gate-1 review |
| **OOS-9** | `_backup()` opens two un-timed connections outside any deadline | `:307`, `:313` | **New in v1.1 (N-3)** — needs its own decision | Q-9 |

---

## 14. Open Questions / Assumptions

### 14.1 Assumptions

| ID | Assumption | Impact if false | How to check |
| --- | --- | --- | --- |
| **A-1** | The Linux-measured F-1 rate characterises the **mechanism**, not the platform's rate | The 300-sample may be mis-powered on Windows | Gate-5 run; adjust n only with a stated power rationale |
| **A-2** | `sqlite_errorcode` is populated and carries the extended code in CPython 3.13 | The `& 0xFF` extraction would be inert; text fallback would carry the load | **Probe §2.7 already confirms it** — reduces this to a runtime-version caveat |
| **A-3** | A `spawn` child under pytest can import the test module | F-3 needs the mandatory `sys.path` guard (already specified) | T-F3-06 |
| **A-4** | No caller relies on catching `sqlite3.Error` around construction | A start could crash uncaught where it previously printed a message | **§3.5.4 review is the Gate-1 condition** |
| **A-5** | The same-transaction simulation is an acceptable stand-in **for the column-targeting property only** | F-2's claim would be weaker than stated | Enforced by §4.4 wording; M-B supplies the lock evidence |
| **A-6** | 30 s is acceptable for CLI paths | A slow report becomes a 30 s wait instead of an instant error | Q-3 |
| **A-7** | A hard wall-clock cap over DDL is not required | The deadline does not bound schema/migration execution | Q-8 |

### 14.2 Open questions (each blocks a gate)

| ID | Question | Default if unanswered | Blocks |
| --- | --- | --- | --- |
| **Q-1** | Is Gate-1 authorized as specified (helper + three adapters + F1-R16/R17)? | no work begins | Gate-0 |
| **Q-2** | Acceptance lane at **300** (8–12 min), or a smaller n with an explicitly weaker claim? | 300; no weaker claim permitted | Gate-1 close |
| **Q-3** | One 30 s deadline, or 5 s for CLI and 30 s for Live? | single 30 s | Gate-1 |
| **Q-4** | May the journal path change exception type (needs the A-4 review), or must it keep `sqlite3.OperationalError` and only enrich the message? | new type, subject to the review | Gate-1 |
| **Q-5** | Mutation harness inside `tests/`, or under `tools/` and acceptance-only? | `tools/`, acceptance-only | Gate-2 |
| **Q-6** | Which logger name/level for the initialization INFO line? | `goldbot.adapters.sqlite_init`, INFO | Gate-1 |
| **Q-7** | Expose connection state via `migration_report()`? | no (§8) | Gate-2 |
| **Q-8** (**new**) | Is the deadline allowed to stop at the PRAGMA/verification boundary (current design), or must DDL also be hard-capped (requires a thread + cancellation protocol)? | boundary design; DDL only inherits the clamped `busy_timeout` | Gate-1 |
| **Q-9** (**new**) | `_backup()`'s two un-timed connections (N-3 / OOS-9) — include in Gate-1, or defer? | defer with a recorded finding | Gate-1 |

### 14.3 Explicitly NOT established by this document

* Whether the **current** implementation contains a Lost Update — **no evidence was found that it does**.
* Whether F-1 occurs on **Windows**, or at what rate.
* Whether 30 s is adequate in production.
* Whether any test listed here **passes** — **no project test was executed for v1 or v1.1.** The only executed
  code was the stdlib interpreter probe disclosed in §2.7.
* Any statement about MT5, the broker, demo execution, soak behaviour, Cross-VPS fencing or production readiness.

---

## 15. Confirmation of No Change

| Check | Result |
| --- | --- |
| Source files changed for v1.1 | **None** |
| Test files changed for v1.1 | **None** |
| Schema / migration / config changed | **None** |
| Project tests executed for v1.1 | **None** |
| Delivery ZIP re-generated | **None** |
| Delivery integrity at time of writing | `sha256sum -c SHA256SUMS.txt` → **316/316 OK**; ZIP `9375170378…52d1758d` unchanged |
| **Content hash of every delivered file** | `SOURCE-MANIFEST.txt` recomputed: **299/299 entries identical**, of which **205/205 `.py` files identical (0 mismatches)** |
| Code executed for v1.1 | One **stdlib-only** interpreter probe (§2.7), no project import, no project DB, no project test |
| Gates opened by this document | **None** — Gate-0 is owner review only |

**Disclosure — mtime artefact (so the "no change" claim is auditable, not merely asserted).**
Every file in the extracted delivery carries `mtime = ctime = 2026-11-26 02:39:44 UTC`, i.e. the timestamp of the
**workspace snapshot restore for this round**, not of any edit. A naive `find -newermt` therefore flags all
205 `.py` files as "recently modified" even though none was touched.

The decisive check is therefore **content**, not timestamps:

```
manifest entries checked: 299
CONTENT MISMATCHES: 0
.py entries: 205 | content mismatches: 0
```

**Inference:** combined with the 316/316 `SHA256SUMS.txt` result, the delivered source and tests are
**byte-identical** to the v1 baseline. The only new artefact of this round is this specification file.

**End of specification v1.1.**

```
STATUS: SPECIFICATION ONLY — NO IMPLEMENTATION, NO PROJECT TEST EXECUTION, NO DELIVERY CHANGE
AWAITING OWNER REVIEW (GATE-0)
```
