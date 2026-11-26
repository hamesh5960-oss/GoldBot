# PHASE_A_WAL_INITIALIZATION_AND_CONCURRENCY_REMEDIATION_SPEC_v1.2

**Project:** GoldBot v8.2.5 · PHASE A
**Document type:** Technical Remediation Specification (**design only — no code**)
**Version:** **v1.2** (supersedes v1.1)
**Date:** 2026-11-26
**Baseline:** `GoldBot_v8.2.5_PHASE_A_TARGETED_REMEDIATION_2026-11-25.zip`
SHA-256 `937517037894c99916e6b30d4b5ff3a63530e9bf9931f3b10616384f52d1758d`
**Source tree referenced:** `GoldBot_Source_v8.2.5` (299-entry `SOURCE-MANIFEST.txt`, problems: 0)
**Authority:** project owner — the only party that may authorize implementation.

> **Status vocabulary.**
> **Fact** = read from the delivered source, or established by an isolated interpreter probe (§2.8).
> **Inference** = conclusion drawn from Facts, stated as such.
> **Proposal** = recommended design; **not implemented**.
> **Owner Decision** = binding instruction from the project owner (§2.1) — **not** an assumption.
> **Assumption** = unverified premise the design relies on.
> **Unknown** = not established by any evidence available.
> **NOT RUN / NOT VERIFIED** = explicitly not executed — never "passed".

---

## 0. Change Log — v1.1 → v1.2

Every change below traces to a numbered instruction from the owner's revision order.

| # | Source instruction | v1.1 state | v1.2 state | Where |
| --- | --- | --- | --- | --- |
| **R-1** | §1 Owner Decision | deadline was a parameter with a 30 s default, and Q-3/A-6 still framed it as an open choice (5 s vs 30 s) | **30 s is a binding Owner Decision for all three adapters**, identical across Live/CLI/Web/Report/admin. Q-3 **CLOSED — OWNER DECISION**; A-6 **deleted and replaced** by the decision; **every** 5-vs-30 reference removed. New **§2.1 Owner Decisions** table. | §2.1, §3.4.3, §14 |
| **R-2** | §1 / §2.2 scope boundary | `path.parent.mkdir` was inside the helper without an explicit budget statement | **`path.parent.mkdir` is explicitly OUTSIDE the 30 s budget**; the budget covers exactly five operations, enumerated. | §3.4.1 |
| **R-3** | §2.2 / §2.3 / F1-R16 | `InitDeadline` was passed into `migrate()` and migration's `busy_timeout` was clamped to the *remaining WAL budget* | **Reversed.** Schema/Migration are **independent of the 30 s WAL deadline**. After WAL succeeds, `busy_timeout` is **re-armed to the migration policy (10 000 ms)**; migration gets its own bounded lock-wait. `InitDeadline` is **not** used as a migration time cap. **F1-R16 rewritten.** | §3.6.3, §13.1 |
| **R-4** | §2.4 / F1-R17 | handler discriminated by catching `OperationalError` and matching text | **`connection.in_transaction` is checked BEFORE `BEGIN IMMEDIATE`**; `BEGIN IMMEDIATE` is issued only when needed; a BUSY/LOCKED that occurs when the connection was genuinely not in a transaction is **never swallowed**. **F1-R17 rewritten.** | §3.6.3, §13.1 |
| **R-5** | §3 | `deadline + 2 s` was presented as a design property | A dedicated section states precisely **what sqlite3 timeouts do and do not guarantee**, reframes the 30 s as a **Monotonic Initialization Budget**, restricts `deadline + 2 s` to **defined controlled test conditions**, and records a hard absolute wall-clock cap as **out of scope**. | §3.9 |
| **R-6** | §4 | primary codes `{5,6}` were both retried | **`SQLITE_LOCKED` no longer retried automatically.** Retry set = `{5}` (BUSY family) **plus `6` only when `SQLITE_LOCKED_SHAREDCACHE` (262) can actually occur** — which §2.5 proves it cannot here, so `6` is **non-retryable** in this codebase, with a documented rationale and a re-open condition. Per-code classification tests added (T-F1-20). | §3.4.2, §7.3 |
| **R-7** | §5 | T-F2-01/02/03 used `threading.Event` hooks that could touch a `check_same_thread=True` connection from the main thread | All three tests rewritten with **strict thread ownership**: no connection ever crosses a thread boundary; T-F2-03 builds `L1` **inside the worker thread**. `check_same_thread` is **not** changed to help tests. New invariant **F2-R11**. | §4.3, §13.2 |
| **R-8** | §6 | `InitReport` had 6 fields | Expanded to 9 fields including **`delay_ms_planned` vs `delay_ms_actual`** and per-attempt `elapsed_ms`; T-F1-17 consumes them, with criteria aligned to the real deadline scope (§3.4.1). | §3.3, §7.3 |
| **R-9** | §7 | Gate-4 said "closes on V-1…V-11" while also containing a Windows row — an internal contradiction | Gate-4 **closes Linux only**; Windows remains `NOT RUN` until Gate-5; overall status must read **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`**. No gate row may pair `CLOSED` with an unexplained `NOT RUN`. | §12 |
| **R-10** | §8 | "n = 300 ⇒ 95 % confidence" was stated once, without saying *per adapter* | **The 95 % claim is per-adapter and independent.** If a joint three-adapter claim is desired, the multiple-comparison consequence is analysed (**0.95³ ≈ 0.857 ⇒ ≈ 14.3 % family-wise false-pass risk**) and a Bonferroni-corrected per-adapter α is offered. **n = 300 per adapter is retained** (the owner requires documented mathematical counter-evidence to change it; §7.2 supplies the opposite). "0 failures in 300" is explicitly **not** proof of absence. | §7.2 |
| **R-11** | §9 | — | All intact v1.1 content retained verbatim in substance; **N-3 (`_backup()`) is carried forward, not hidden** — and is now **promoted**: §2.2.3 requires every independent migration-path connection to have an explicit timeout, so N-3 becomes **F1-R19** with its stdlib default documented. | §13.1, OOS-9 |

**Findings that changed status in v1.2** (each disproves or refines something previously written):

| ID | Finding | Evidence | Effect on v1.1 text |
| --- | --- | --- | --- |
| **N-5** | **D-3 in v1.1 was wrong.** `SqliteRiskStateStore` is *not* missing a lock wait. `sqlite3.connect(path, timeout=10.0)` **installs a 10 s busy handler on the connection**, as the stdlib documents. Probe: a contended write blocked **10.02 s** then raised `database is locked`. | probe §2.8 | **Correction**: D-3 is **not** "no busy_timeout". It is "**no explicit, declared** policy — it inherits a value from `connect()` that is invisible at the call site and unowned by the module". F1-R2 is restated accordingly; the substantive change is documentation + an owned policy, not a behavioural repair. |
| **N-6** | **`SQLITE_LOCKED` (6) cannot arise from cross-process contention in this codebase.** It is raised when a table is locked *within the same process*, principally under shared cache. `grep` finds **no** `cache=shared` / `shared_cache` / shared-cache URI anywhere in `src/` or `tests/`; all connections are file-based. | `grep` §2.5 | Makes R-6 decision **evidenced**, not arbitrary: `6` is **non-retryable** here; if shared cache is ever introduced, the decision must be revisited (§3.4.2 re-open condition). |
| **N-7** | `connection.in_transaction` is **not used anywhere** in `src/` or `tests/` today. | `grep` §2.5 | F1-R17 introduces a stdlib facility with **zero existing precedent** in this codebase → carries its own test (T-F1-18) and an assumption (A-8). |

---

## 1. Executive Summary

This document specifies — and only specifies — the remediation design for three findings from the independent
audit of 2026-11-26, with the owner's binding decisions of the same date applied.

| ID | Class | Statement | Blast radius |
| --- | --- | --- | --- |
| **F-1** | Availability / Startup defect (**pre-existing**) | `PRAGMA journal_mode=WAL` can fail with `database is locked` when two processes initialize a **brand-new** SQLite file simultaneously. Three adapters perform this initialization independently. | CLI, report, web, Live startup; **no corruption observed** |
| **F-2** | Verification gap | The A7 multi-process test does not deterministically prove absence of a Lost Update, and cannot reliably detect re-introduction of the pre-fix whole-row write. | Test credibility only; **no evidence the implementation is defective** |
| **F-3** | Test portability gap | Three test sites hard-code `multiprocessing.get_context("fork")`; Windows — the target platform — does not support it. | Test executability on the production platform; no source defect |

**Headline Facts (all re-verified against the reference tree for v1.2)**

* `PRAGMA journal_mode=WAL` appears in exactly **three** locations: `sqlite_store.py:160`,
  `sqlite_operation_ledger.py:153`, `sqlite_state_store.py:33`. No fourth exists.
* **`busy_timeout` appears in exactly three places** (`sqlite_store.py:165`, `sqlite_operation_ledger.py:155`
  and `:192`), all literal `10000`. `SqliteRiskStateStore` declares **none** — but inherits **10 s** from
  `sqlite3.connect(timeout=10.0)` (N-5).
* **No test anywhere** asserts `journal_mode == 'wal'`.
* `claim_for_send` (`:562`) and `mark_submit_attempt` (`:594`) do **not** use `_in_immediate`; `transition`
  (`:457`, call `:497`), `_bump_duplicate` (`:503`, call `:516`), `_normalise_digest` (`:522`, call `:555`) do.
* A faithful re-introduction of the pre-fix whole-row write reproduced failures in **3 of 40** A7 runs (7.5 %).
* `get_context("fork")` at `test_phase_a_audit_multiprocess.py:66`, `:112`, `test_fix05_counter_semantics.py:181`.

**Headline Proposals**

* One shared initializer with a **30 s monotonic budget (Owner Decision)** covering exactly five operations.
* Schema/Migration **decoupled** from that budget, with their own bounded lock waits.
* Four-way exception taxonomy, cause chains preserved, AU-35 never weakened.
* Deterministic tests under **strict thread ownership**; `BEGIN IMMEDIATE` removal caught by a lock-level witness.
* `spawn` everywhere with bounded teardown and no new dependency.
* **Nothing in this document is implemented. No project test has been executed for this or any prior version.**

---

## 2. Owner Decisions, Evidence and Scope

### 2.1 Owner Decisions (binding — not assumptions)

| ID | Decision | Scope of the decision | Status |
| --- | --- | --- | --- |
| **OD-1** | **SQLite Initialization Deadline = 30 seconds, identical for all three adapters** (`SqliteJournal`, `SqliteRiskStateStore`, `SqliteOperationLedger` path-based branch). | A single shared, centralised policy. **No** separate value for Live, CLI, Web, Report or administrative tools. | **BINDING** |
| **OD-2** | The 30 s budget covers exactly: open connection · `PRAGMA journal_mode=WAL` · retry/backoff on recoverable errors · re-read and confirmation that `journal_mode == 'wal'` · completion of required helper PRAGMAs and handing over a ready connection. | — | **BINDING** |
| **OD-3** | **`path.parent.mkdir` is outside the budget.** | Documented explicitly so nobody later argues the boundary either way. | **BINDING** |
| **OD-4** | **Schema creation and migration are outside the 30 s hard wall-clock deadline.** Their lock-wait expectation must be bounded and detectable instead. | — | **BINDING** |
| **OD-5** | The owner sanctioned this specification revision **only**. No source, test, schema, DDL, VPS, MT5 or live setting may change; no gate is opened. | — | **BINDING** |

**Consequences applied in v1.2 (each is a Fact about this document):**
`Q-3` is **CLOSED — OWNER DECISION**; `A-6` is **deleted** and replaced by **OD-1**; **every** sentence that
offered a 5 s CLI / 30 s Live split has been removed. `Q-8` (hard cap over DDL) is also closed by **OD-4**: DDL
is out of the hard cap, and the residual need is recorded as a separate, unapproved requirement (§3.9.4).

### 2.2 Evidence sources

| # | Source | Nature | Used for |
| --- | --- | --- | --- |
| **E-1** | Owner audit, 2026-11-26 (F-1/F-2/F-3; 120/40/40 process pairs) | Owner-supplied **Fact** | Failure rates per adapter |
| **E-2** | Independent audit, 2026-11-26 (`INDEPENDENT_VERIFICATION_2026-11-26.md`, `/home/user/evidence/`) | Independent measurement | Root cause; fresh-vs-existing split; mutation sensitivity |
| **E-3** | Delivered source tree, read directly (`grep`/`sed`) | **Fact** | Every line number here |
| **E-4** | `SOURCE_DIFF_FULL.diff` | **Fact** | Proves `sqlite_store.py` untouched by the Targeted Remediation |
| **E-5** | Isolated stdlib interpreter probes (§2.8) | **Fact** | N-5, extended-code mechanics, `NOTADB` behaviour |

### 2.3 F-1 — measured failure rates

| Adapter | Pairs | Pairs with ≥1 failure | Rate | Source |
| --- | --- | --- | --- | --- |
| `SqliteJournal` | 120 | 56 | **46.7 %** | E-1 |
| `SqliteRiskStateStore` | 40 | 22 | **55.0 %** | E-1 |
| `SqliteOperationLedger` (path) | 40 | 23 | **57.5 %** | E-1 |
| `SqliteJournal` — independent, **fresh** DB | 250 | 11 | **4.4 %** | E-2 |
| `SqliteJournal` — independent, DB **already WAL** | 250 | **0** | **0.0 %** | E-2 |
| Bare-SQL stepwise probe | 150 | 13 | **8.7 %, all 13 at `PRAGMA journal_mode=WAL`** | E-2 |

**Inference:** the rates differ by an order of magnitude because the rate is a function of **how tightly the two
constructors are aligned in time**, not a constant of the product. §7.2 powers the acceptance lane for the
*smallest* rate we care about, never the largest observed.

**Fact (root cause):** the failing statement is the **journal-mode transition**, not the schema script.
**Fact:** installing `busy_timeout` *before* the transition does not fix it (8/200 vs 5/200).

### 2.4 Exposure map

| Entry point | `LiveInstanceLock`? | Files opened | Concurrent exposure |
| --- | --- | --- | --- |
| `run_live` (`runtime/live.py:198`; decorator `:175`; journal built in `build_components()` at `:341`) | **Yes** | journal + `state_db_path(db_path)` (`:228`) | Protected vs a second **Live** process; **not** vs CLI/web |
| `cmd_ack/status/override/health/level/approve/onboarding` (`cli.py:389,454,489,572,616,736,744`) | **No** | risk state (7 sites), some journal | **Yes** |
| `cmd_report` (`:167`), `cmd_signals` (`:794`), `_run` (`:176`), `cmd_serve` (`:275`) | **No** | journal, risk state | **Yes** |
| `web/app.py:210,249,503,526` (journal), `:200` (state) | **No** | journal, risk state | **Yes** |
| `build_components` (`bootstrap.py:117`, `:247`) | Inherited | journal + ledger + state | **Yes** for CLI/web callers |

**Fact:** `SqliteJournal(` is constructed at **58 sites** (12 in `src/`: 1 bootstrap, 7 CLI, 4 web; 46 in `tests/`).
**Inference:** the Live lock does **not** solve F-1; the CLI/report path is the operator's tool during exactly the
incident when a second process is most likely.

### 2.5 Fact inventory established for v1.2 (all by direct read)

| # | Fact | Command basis |
| --- | --- | --- |
| **P-1** | `busy_timeout` appears exactly at `sqlite_store.py:165`, `sqlite_operation_ledger.py:155`, `sqlite_operation_ledger.py:192` — all `10000`. **No other site, and none in `sqlite_state_store.py`.** | `grep -rn busy_timeout src/` |
| **P-2** | **Shared cache is never enabled**: no `cache=shared`, `shared_cache`, `cache_shared` or `PRAGMA cache` in `src/` or `tests/`. All connections are file-based. | `grep -rn "cache=shared\|shared_cache\|PRAGMA cache" src/ tests/` → none |
| **P-3** | `connection.in_transaction` is used **nowhere** today. | `grep -rn in_transaction src/ tests/` → none |
| **P-4** | `sqlite3.connect` sites and timeouts: `sqlite_state_store.py:31` `timeout=10.0`; `sqlite_store.py:158` `timeout=10.0, check_same_thread=False`; `sqlite_operation_ledger.py:151` `timeout=10.0`; **`sqlite_operation_ledger.py:307` and `:313` — no timeout argument (stdlib default 5.0)**. | `grep -rn "sqlite3.connect(" src/` |
| **P-5** | `check_same_thread` is set **only** by the journal (`:158`, `False`); the ledger path branch (`:151`) and the state store (`:31`) use the stdlib default `True`. | `grep -rn check_same_thread src/` |
| **P-6** | Migration executes DDL as separate statements in a loop (`:225` `LEDGER_SCHEMA.split(";")`), commits at `:233`, rolls back at `:235`; `_upgrade_columns` at `:275`; backup at `:207` → `_backup()` `:295+`. | direct read |

### 2.6 F-2 — the correction to the audit brief, carried forward

| Method | Definition | `_in_immediate` call | Targeted SQL | Safety argument |
| --- | --- | --- | --- | --- |
| `transition` | `:457` | **`:497`** | full-row `UPDATE` from `_apply_transition` | RMW serialized |
| `_bump_duplicate` | `:503` | **`:516`** | `… duplicate_requests = duplicate_requests + 1 …` | RMW serialized |
| `_normalise_digest` | `:522` | **`:555`** | conditional `UPDATE … WHERE op_id=? AND payload_hash=?` (guard at `:537` `rowcount != 1`) | guarded write + re-read |
| `claim_for_send` | `:562` | **none** | `… SET state=?, claim_attempts = claim_attempts+1 WHERE op_id=? AND state IN (?,?)` (`:572`) | one conditional statement is atomic; `rowcount` is the CAS witness |
| `mark_submit_attempt` | `:594` | **none** | `… submit_call_markers = submit_call_markers + 1 WHERE op_id=?` (`:603`) | one incrementing statement is atomic |

**Inference:** no defect is implied, but the distinction must be documented (F-2-D7), because a future refactor
adding a second statement to either single-statement method would silently lose serialization.

### 2.7 F-3 — exact sites and constraints

| File | Line | Call |
| --- | --- | --- |
| `tests/test_phase_a_audit_multiprocess.py` | **66** | `mp.get_context("fork")` (AU-22) |
| `tests/test_phase_a_audit_multiprocess.py` | **112** | `mp.get_context("fork")` (AU-23) |
| `tests/test_fix05_counter_semantics.py` | **181** | `mp.get_context("fork")` (crash-evidence) |

**Facts:** 86 test modules, **no `tests/__init__.py`**; `conftest.py` injects `src` into `sys.path` in the parent
only; `psutil` present in the audit workspace but **absent from `requirements.txt`** (one line: `pydantic`);
`pytest-timeout` and `pytest-xdist` **not installed**.
**Unknown:** whether a `spawn`ed child under pytest can import the test module by name — §5.3 makes the design
independent of the answer.

### 2.8 Environment probes and full disclosure

> **Disclosure.** Two stdlib-only probes were executed while authoring v1.1/v1.2, each in a temporary directory
> with a throw-away file. They imported **no project code**, opened **no project database**, ran **no project
> test**, and produced no artefact inside the delivery. Statement: **no project test was executed for v1.2.**

| Probe | Question | Result (**Fact**) |
| --- | --- | --- |
| **PR-1** | What does a contended write raise, and does the exception carry extended codes? | `sqlite3.OperationalError`; `sqlite_errorcode = 5`; `sqlite_errorname = 'SQLITE_BUSY'`; `5 & 0xFF = 5` |
| **PR-2** | What does a corrupt file raise on `PRAGMA journal_mode=WAL`? | **`sqlite3.DatabaseError`**, `sqlite_errorcode = 26`, message `file is not a database` |
| **PR-3** | Does `sqlite3.connect(timeout=10.0)` alone install a 10 s busy handler? | **Yes** — the contended write waited **10.02 s** and then raised `database is locked`. Confirms the stdlib contract and **disproves v1.1's D-3 wording** (N-5). |

**Inferences:** (a) `& 0xFF` primary extraction is sound on this interpreter; (b) the corrupt-file path is
`DatabaseError`, not `OperationalError`, so classification must cover `DatabaseError` (N-4); (c) the corrupt
message contains the word "database" — the accidental reason AU-35 passes today, which §3.5.3 makes deliberate;
(d) the state store already waits up to 10 s, so F1-R2 is about **declared, owned policy**, not about repair.

### 2.9 Integrity evidence carried forward

| Check | Result | Status |
| --- | --- | --- |
| Delivery ZIP SHA-256 | `9375170378…52d1758d` | **Fact** |
| `SHA256SUMS.txt` | 316/316 OK (re-verified at end of this round) | **Fact** |
| `SOURCE-MANIFEST.txt` | files: 299 \| problems: 0 | **Fact** |
| `sqlite_store.py` SHA-256 | `a74e06db297c9f9ee8d6b6d8ef6942daa6528a70cbacad44db5a0f1fd29cfcba` | **Fact** |
| `sqlite_store.py` in `SOURCE_DIFF_FULL.diff` | 0 occurrences | **Fact** — F-1 is pre-existing |
| `domain/fsm/order.py` SHA-256 | `bf4154c6…f1774d0f` | **Fact** — untouched |
| Full suite on the delivered tree | `1224 passed, 6 skipped, 3 warnings` | **Fact — from the prior audit round; not re-run for v1.2** |

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
151:  self._conn = sqlite3.connect(self.path, timeout=10.0)   # check_same_thread defaults True
153:  self._conn.execute("PRAGMA journal_mode=WAL")
154:  self._conn.execute("PRAGMA synchronous=FULL")
155:  self._conn.execute("PRAGMA busy_timeout=10000")
156:  self._owns_connection = True
160:  self.migrate()                                           # :192 → busy_timeout=10000 again
```

**C. `src/goldbot/adapters/sqlite_state_store.py:30–37`**

```python
31:  self._conn = sqlite3.connect(self.path, timeout=10.0)   # check_same_thread defaults True
33:  self._conn.execute("PRAGMA journal_mode=WAL")
35:  self._conn.execute("PRAGMA synchronous=FULL")
36:  self._conn.executescript(SCHEMA)
37:  self._conn.commit()                                     # no explicit busy_timeout (inherits 10 s)
```

### 3.2 Defect inventory (v1.2 wording)

| ID | Defect | Location | Consequence |
| --- | --- | --- | --- |
| **D-1** | WAL transition attempted exactly once, no retry | A:160, B:153, C:33 | Any momentary lock → constructor failure |
| **D-2** | `busy_timeout` set **after** the transition | A:165, B:155 | Cannot help the statement that needs it (and reordering does not help — §2.3) |
| **D-3′** | **Declared** lock-wait policy missing in the state store; the effective value is an **invisible inherited** 10 s from `connect(timeout=10.0)` | C (absent) | Policy is unowned and undiscoverable at the call site; a future edit to `connect()` silently changes durability behaviour. *Corrected from v1.1's "no busy_timeout at all" — see N-5.* |
| **D-4** | Resulting journal mode never read back | A, B, C | A silent fallback to `delete` goes unnoticed |
| **D-5** | No `try/except`; connection not closed on failure | A:158–169, B:147–160, C:31–37 | Leaked handle per failed construction; on Windows a leaked handle can also block deletion |
| **D-6** | Operator sees a bare `sqlite3.OperationalError`/`DatabaseError` | A, B, C | No path, attempt count, elapsed time or guidance |
| **D-7** | Three independent copies of one policy | A, B, C | Already diverged (only the journal/ledger declare a timeout) |
| **D-8** | Ledger's shared-connection branch (`:141–146`) inherits the journal's policy and runs `migrate()` on it | B:141–160 | Correct by design; the helper must be **forbidden** there |
| **D-9** | `migrate()` re-sets `busy_timeout=10000` unconditionally | B:192 | Belongs to the migration policy, must **not** be tied to the WAL budget (OD-4) |
| **D-10** | `migrate()` swallows **any** `OperationalError` on `BEGIN IMMEDIATE` | `:208–213` | A genuine BUSY is misread as "already inside a transaction"; DDL proceeds unlocked (**N-2**) |
| **D-11** | No deadline anywhere in initialization | A, B, C | An operator cannot distinguish "slow" from "stuck"; harnesses time out before the code speaks |
| **D-12** | Migration-path connections have no explicit timeout | `:307`, `:313` | They silently inherit the 5 s stdlib default — an unowned policy in the exact path that can stall startup (**N-3**) |

### 3.3 The initializer and the report structure

```python
@dataclass(frozen=True)
class InitDeadline:                 # monotonic, created exactly once per initialization
    started: float                  # time.monotonic()
    budget_s: float = 30.0          # OD-1 — a constant, not a knob
    def remaining(self) -> float: ...
    def remaining_ms(self, *, floor_ms: int = 1) -> int: ...
    def expired(self) -> bool: ...

@dataclass(frozen=True)
class Attempt:                      # one row per WAL attempt (R-8)
    index: int
    remaining_ms_before: int        # budget left when the attempt started
    busy_timeout_ms_used: int       # value actually applied for this attempt
    delay_ms_planned: int           # backoff chosen before sleeping
    delay_ms_actual: int            # measured sleep (time.monotonic() deltas)
    elapsed_ms: int                 # duration of this attempt
    error_code: int | None          # raw sqlite_errorcode
    error_name: str | None          # e.g. "SQLITE_BUSY", "SQLITE_BUSY_SNAPSHOT"

@dataclass(frozen=True)
class InitReport:
    path: str
    attempts: tuple[Attempt, ...]
    elapsed_s: float
    journal_mode_after: str
    created: bool

def open_sqlite_connection(
    path, *, budget_s: float = 30.0, busy_timeout_ms: int = 10_000,
    row_factory: bool = True, check_same_thread: bool = True,
) -> tuple[sqlite3.Connection, InitReport]
```

**Why `delay_ms_planned` *and* `delay_ms_actual` (R-8):** a backoff implementation that silently rounds, clamps or
is skipped can look correct in planned values alone. Measuring the actual sleep is the only way T-F1-17 can prove
the invariant `Σ planned ≤ budget` **and** `Σ actual ≤ budget`, and detect a scheduler that overslept. The test
asserts both and reports the delta.

### 3.4 Deadline semantics — the WAL budget (OD-1, OD-2, OD-3)

#### 3.4.1 Exact scope of the 30 s budget

| Inside the budget | Outside the budget |
| --- | --- |
| 1. Opening the connection | **`path.parent.mkdir(parents=True, exist_ok=True)`** (OD-3) |
| 2. `PRAGMA journal_mode=WAL` | Schema creation (`executescript(SCHEMA)`, `:166` / `:36`) |
| 3. Retry and backoff for recoverable errors | Ledger migration, DDL, version row, commit/rollback (`:181–235`) |
| 4. Re-read and confirmation that `journal_mode == 'wal'` | `_backup()` and its two connections (`:295+`, `:307`, `:313`) |
| 5. Required helper PRAGMAs and handing over a ready connection | All runtime order-path behaviour |

**Inference recorded deliberately:** `mkdir` precedes the deadline token, so an NFS-stalled directory creation is
**not** bounded by OD-1. This is the owner's stated boundary and is documented so the limitation is visible rather
than discovered later.

#### 3.4.2 Retry classification (R-6 / N-6)

```
primary(code) = code & 0xFF                      # 517 → 5, 261 → 5, 773 → 5, 262 → 6

RETRYABLE   = {5}                                # SQLITE_BUSY family only
LOCKED_POLICY = {6: NOT_RETRYABLE}               # see below — evidence-based
```

| Raw code | Name | Primary | Decision | Why |
| --- | --- | --- | --- | --- |
| 5 | `SQLITE_BUSY` | 5 | **Retry** | Cross-process contention on the file is exactly F-1; waiting resolves it. |
| 261 | `SQLITE_BUSY_RECOVERY` | 5 | **Retry** | Restore/recovery in progress; transient by definition. |
| 517 | `SQLITE_BUSY_SNAPSHOT` | 5 | **Retry**, but see note | Raised when a deferred snapshot is upgraded and the WAL moved on. In this codebase every write is either a single statement or inside `BEGIN IMMEDIATE`, so **no deferred read transaction is held across a write**. If new code introduces deferred transactions, 517 must be re-examined rather than assumed harmless. |
| 773 | `SQLITE_BUSY_TIMEOUT` | 5 | **Retry** | The busy handler itself expired; identical handling to 5. The **deadline** — not this code — is what bounds the loop. |
| 6 | `SQLITE_LOCKED` | 6 | **NOT retryable** | SQLite defines this as a lock held **inside the same connection/process**, typically under shared cache. **P-2 proves shared cache is never enabled here**, so a genuine cross-process lock cannot produce 6. Retrying would burn the full 30 s on a condition that cannot clear by waiting. |
| 262 | `SQLITE_LOCKED_SHAREDCACHE` | 6 | **NOT retryable** | Requires shared cache, which is not used (P-2). |
| 26 | `SQLITE_NOTADB` | 26 | **NOT retryable** | Corrupt/foreign file (PR-2). |
| 11, 14, 8, 3 | `CORRUPT`, `CANTOPEN`, `READONLY`, `PERM` | — | **NOT retryable** | Waiting cannot change them. |

**Re-open condition (binding on future work):** if any code path ever enables shared cache (URI `cache=shared`,
`PRAGMA cache_shared`, or an equivalent), the `{6 → NOT_RETRYABLE}` decision **must be revisited** and the
classification table above re-issued. Recorded as **Q-10**.

**Text fallback (R-6.6):** permitted **only** when `sqlite_errorcode` is absent (older runtimes) **and** the message
matches `database is locked` / `database table is locked` case-insensitively. Never the primary discriminator.

**Fast-fail requirement:** a non-retryable code must surface with `attempts == 1` and `elapsed < 2 s`; a retry
storm on `NOTADB` would convert an instant, honest failure into a 30 s wait (T-F1-19/T-F1-20).

#### 3.4.3 The budget is a constant, not a knob (OD-1)

`budget_s: float = 30.0` is a **module constant**, not a per-caller argument threaded through Live, CLI, Web or
report. A test may override it to shorten a test (e.g. `budget_s=1.0` in T-F1-16); **production call sites never
pass it**. This is how OD-1's "no different deadline anywhere" is enforced *structurally* rather than by
convention: there is only one place the number exists.

#### 3.4.4 What the budget can and cannot guarantee — see §3.9

The `deadline + 2 s` figure is **not** introduced here as a universal promise; §3.9 states its exact, restricted
meaning and the conditions under which the acceptance test (T-F1-16) may use it.

### 3.5 Exception contract

#### 3.5.1 Taxonomy

```
GoldbotError (domain/errors.py:22)
└── SqliteInitializationError
    ├── SqliteBusyTimeout          # BUSY family persisted past the budget
    ├── SqliteNotADatabase         # NOTADB (26)
    └── SqliteUnretryableError     # any other non-retryable code (incl. 6 / 262 — §3.4.2)
LedgerUnavailable                  # ports/ledger.py:77 — unchanged public contract
```

| Failure | Helper raises | Adapter surface | `attempts` | `elapsed` | Cause chain |
| --- | --- | --- | --- | --- | --- |
| BUSY family past budget | `SqliteBusyTimeout` | journal/state as-is; ledger → `LedgerUnavailable` | ≥1 (logged) | ≈ budget | `from last_exc` |
| Corrupt file | `SqliteNotADatabase` | as above | **1** | **< 2 s** | `from DatabaseError` |
| `LOCKED` (6 / 262) | `SqliteUnretryableError` | as above | **1** | **< 2 s** | `from exc` |
| Other non-retryable | `SqliteUnretryableError` | as above | **1** | **< 2 s** | `from exc` |
| WAL not applied after transition | `SqliteInitializationError` | as above | 1 | small | cause of the PRAGMA, or `from None` |
| Ledger migration failure | — | `LedgerUnavailable` (existing, `:341`) | — | — | unchanged |

**Non-negotiable:** every raise uses `from exc` (or `from None` only when no cause exists), preserving
`__cause__` end-to-end — modelled on the existing `raise LedgerUnavailable(...) from exc` at `:500/:519/:558`.

#### 3.5.2 Message template

```
SQLite initialization failed for database {path}: {reason} after {attempts} attempt(s)
in {elapsed:.2f}s (budget {budget:.1f}s)
```

The literal token **"database"** is mandatory in every subclass.

#### 3.5.3 AU-35 is preserved by construction

`tests/test_phase_a_audit_fsm_and_live.py:251`:

```python
assert "database" in repr(excinfo.value).lower() or "ledger" in repr(excinfo.value).lower()
```

Today this passes **accidentally** (the corrupt-file message is `file is not a database`, PR-2). The template above
makes the token deliberate. **AU-35 must not be edited, weakened or skipped**; T-F1-10 re-runs it unmodified.

#### 3.5.4 Compatibility review of existing catch sites (Gate-1 condition)

| File:line | Handler | Relationship to the new type |
| --- | --- | --- |
| `sqlite_operation_ledger.py:193` | `except sqlite3.Error: pass` around a `busy_timeout` PRAGMA | unaffected — direct connection call |
| `:210` | `except sqlite3.OperationalError: pass` around `BEGIN IMMEDIATE` | **must be replaced** — F1-R17 |
| `:341, :389, :399, :416, :434, :500, :519, :558, :590, :615, :628` | `except sqlite3.Error` around writes | unaffected — they wrap `self._conn.execute`, not the constructor |
| `order_coordinator.py:244` | `except LedgerUnavailable` | **preserved**, because the ledger maps to `LedgerUnavailable` |
| `web/app.py:203, 211, 516, …` | `except Exception` | **preserved** — the new type derives from `Exception` |

**Gate-1 condition:** a one-page inventory of **every** `except` reachable from a journal/ledger/state constructor,
stating for each whether the new type is caught and what the user sees. **A-4** is discharged only by that review.

### 3.6 Per-adapter wiring

#### 3.6.1 Frozen values (unchanged from v1.1 — R-9/§9 of the order)

| Adapter | Change | **Frozen — must not change** |
| --- | --- | --- |
| `SqliteJournal` (`sqlite_store.py:153–169`) | replace 158–165 with `open_sqlite_connection(self.path, row_factory=True, check_same_thread=False)`; keep `executescript(SCHEMA)`, `_migrate_trades()`, `_migrate_signals()`, `commit()` | `SCHEMA`; migration semantics; **`check_same_thread=False`** |
| `SqliteOperationLedger` path branch (`:147–160`) | replace 151–155 with the helper (**`check_same_thread` omitted → `True`**); keep `_owns_connection=True`; keep `migrate()` | shared-journal branch (`:141–146`); `migrate()`'s `BEGIN IMMEDIATE` + version re-check; **`True`** |
| `SqliteRiskStateStore` (`:27–37`) | replace 31–36 with the helper (**`True`**) — this **declares** the policy instead of inheriting it (D-3′) | `SCHEMA`; `key` semantics; **`True`** |

**Why `check_same_thread` is behaviourally significant:**

| Value | Observable behaviour |
| --- | --- |
| `False` (journal) | connection usable from a worker thread without `ProgrammingError` |
| `True` (ledger, state) | use from another thread raises `sqlite3.ProgrammingError` |

Both are frozen; T-F1-15 asserts the **observable** behaviour of each, so a future "cleanup" that unifies the
keyword fails loudly. **This is also why §4.3 forbids moving a connection across threads in tests.**

#### 3.6.2 Shared-journal branch

`SqliteOperationLedger.__init__` with `journal is not None` (`:141–146`) **must not** call the helper: it does not
own the connection, and a second WAL transition on a borrowed connection is exactly the side effect OD-2's scope
forbids. It inherits the journal's verified mode.

#### 3.6.3 Schema and Migration are a separate budget (OD-4; R-3, R-4) — **F1-R16 / F1-R17 rewritten**

**The v1.1 design is withdrawn.** v1.1 passed the WAL `InitDeadline` into `migrate()` and clamped migration's
`busy_timeout` to the *remaining WAL budget*. That contradicts OD-4 and is replaced by the following.

**F1-R16 (rewritten) — migration has its own bounded lock wait:**

| Requirement | Detail |
| --- | --- |
| Independence | The WAL `InitDeadline` **must not** be used as a time cap for schema/migration. |
| Re-arm | Immediately after the helper returns and **before** `executescript(SCHEMA)` / `migrate()`, the connection's lock-wait policy is **re-armed to the migration policy**: `PRAGMA busy_timeout=10000` (the current literal, preserved — P-1). |
| Single ownership | The re-arm happens **once**, in the adapter, not in the helper and not in both. `migrate()`'s own `:192` statement is **removed** so the policy has exactly one owner; if the owner is kept at `:192` instead, the constructor must not duplicate it — **one of the two, never both**. Decision: **keep `:192` as the owner** and have the journal/state paths rely on the helper's `busy_timeout_ms=10_000` default; the ledger must not re-arm twice. |
| Bounded | The wait is bounded by `busy_timeout` (10 s) **per statement**, not by a global migration wall clock. |
| Detectable | Exhausting it raises a controlled error (§3.5.1, `LedgerUnavailable` with cause) — never silently continues. |
| No global claim | **No statement in any report may claim the whole migration finishes within 30 s.** |
| Honest metric | The acceptance metric is **per-operation lock wait** bound, not total wall-clock. |

**F1-R17 (rewritten) — `in_transaction` check instead of a catch-all:**

```
if not conn.in_transaction:                 # P-3: this facility is new to the codebase
    conn.execute("BEGIN IMMEDIATE")         # BUSY/LOCKED here must propagate
else:
    pass                                    # genuinely already inside a transaction
```

| Requirement | Detail |
| --- | --- |
| Pre-check | `connection.in_transaction` is evaluated **before** issuing `BEGIN IMMEDIATE`. |
| Conditional begin | `BEGIN IMMEDIATE` is issued **only** when the connection is not already in a transaction. |
| No swallowing | If the connection was **not** in a transaction and `BEGIN IMMEDIATE` fails with BUSY/LOCKED, the error **must propagate** (mapped to `SqliteBusyTimeout` if the migration lock wait is exhausted, else `LedgerUnavailable`). |
| No misreading | A real BUSY may **never** be interpreted as "already inside a transaction" — the exact defect of `:208–213` (N-2, D-10). |
| Testable cause | The raised exception's `__cause__` is the original `sqlite3` error; T-F1-18 asserts the chain. |

**Inference (why this is better than v1.1's text matching):** `in_transaction` is a *state query*; text matching is a
*guess about a message*. The state query cannot be fooled by locale, version or message rewording.

**Assumption A-8:** `sqlite3.Connection.in_transaction` faithfully reports the state for connections whose
transactions are started by explicit SQL (`BEGIN IMMEDIATE`) — which is what this codebase does (P-6) — and is not
confused by `executescript`'s implicit commit. Verified by T-F1-18.

### 3.7 Behaviour matrix

| Scenario | Initial state | Expected after remediation | Test |
| --- | --- | --- | --- |
| S1 | absent | WAL verified, schema created, no error | T-F1-01 |
| S2 | exists, rollback mode, with data | transitions to WAL; data unchanged | T-F1-02 |
| S3 | exists, already WAL | no-op; data unchanged | T-F1-03 |
| S4 | absent, 2 processes | both succeed; 0 transient errors over N | T-F1-04, T-F1-05 |
| S5 | absent, 4 processes | all succeed; one file set | T-F1-06 |
| S6 | lock held past budget | `SqliteBusyTimeout` at budget + ≤2 s under §3.9 conditions; connection closed | T-F1-07, T-F1-16 |
| S7 | lock released after ~0.3 s | succeeds with `attempts ≥ 2` | T-F1-09 |
| S8 | corrupt file | `SqliteNotADatabase`, `attempts == 1`, `< 2 s`, message contains "database"; AU-35 green | T-F1-10 |
| S9 | any | directory delta ⊆ {`-wal`, `-shm`} | T-F1-11 |
| S10 | forced `journal_mode` reply `"delete"` | `SqliteInitializationError` | T-F1-12 |
| S11 | ledger with `journal=J` | helper not called; mode untouched | T-F1-13 |
| S12 | each adapter | `check_same_thread` behaviour identical to today | T-F1-15 |
| S13 | migration under a held lock | bounded wait, then a controlled error with cause; **not** tied to the WAL budget | T-F1-18 |
| S14 | backup path connections | explicit timeout present and documented (5 s stdlib default today) | T-F1-21 |
| S15 | each retryable/non-retryable code | classification per §3.4.2 | T-F1-19, T-F1-20 |

### 3.8 Requirements trace for §3

| Requirement | Design item | Where |
| --- | --- | --- |
| one shared initializer | F-1-D1 | §3.3 |
| BUSY/LOCKED classification incl. extended codes | §3.4.2 | §3.4.2 |
| bounded backoff; budget as true cap | §3.4.1, §3.9 | §3.4 |
| deadline is **OD-1 constant** | §3.4.3 | §3.4.3 |
| fail closed + typed error + cause chain | §3.5 | §3.5 |
| close on constructor failure | §3.3 step 10 | §3.3, T-F1-08 |
| verify WAL | §3.3 step 7 | T-F1-01/T-F1-12 |
| new/existing/WAL/concurrent | §3.7 | §3.7 |
| no parallel DB / deletion / path change | §3.3 "Never" | T-F1-11 |
| identical policy, all three adapters | §3.3 + §3.6.1 | §3.3, §3.6 |
| schema/migration **independent** of the 30 s | §3.6.3 | §3.6.3 |
| migration BUSY never swallowed | §3.6.3 F1-R17 | T-F1-18 |
| independent migration connections have explicit timeouts | §3.6.3 F1-R19 | T-F1-21 |

### 3.9 What `sqlite3` timeouts do — and do not — guarantee (R-5)

This subsection is the technical boundary the owner required. It contains no new promise.

#### 3.9.1 What is actually controlled

| Mechanism | What it bounds |
| --- | --- |
| `sqlite3.connect(timeout=n)` | how long the connection's **busy handler** waits for a conflicting SQLite lock before raising "database is locked". PR-3 measured this at **10.02 s** for `timeout=10.0`. |
| `PRAGMA busy_timeout=n` | the same wait, per connection, settable at runtime; replaces the value derived from `connect()`. |
| Our retry loop | how many times the **WAL transition statement** is re-attempted, and with what backoff, until the monotonic budget expires. |

#### 3.9.2 What is **not** controlled

* **File-system stalls** (network share, failing disk, antivirus filter) — execution can block inside the OS
  without ever reaching SQLite's busy handler.
* **Scheduler and OS-level suspension** — a process can be descheduled arbitrarily; `time.monotonic()` keeps
  advancing, but the code is not running to observe it.
* **`mkdir`** — outside the budget by OD-3.
* **Schema/migration** — outside the budget by OD-4; bounded only per-statement by `busy_timeout`.
* **Anything before the deadline token is created** and anything after the helper returns.

#### 3.9.3 Consequences for the acceptance criterion

* The 30 s figure is therefore defined as a **Monotonic Initialization Budget** — the *cooperative* engineering
  bound on the helper's five operations (§3.4.1), **not** a mathematical guarantee about wall-clock time.
* The acceptance form `elapsed ≤ budget + 2 s` is permitted **only** under the test conditions in T-F1-16:
  a local filesystem, `BEGIN EXCLUSIVE` held by a cooperating second connection, a machine not otherwise loaded,
  and the budget deliberately shortened to 1 s so that the 2 s tolerance is a large fraction of it and any
  overshoot is visible rather than hidden.
* **No document, report or commit message under this specification may present `deadline + 2 s` as protection
  against OS or disk stalls.**

#### 3.9.4 If an absolute hard cap is genuinely required

Recorded **as a separate requirement outside this scope**, with **no implementation proposal here**:

> **RQ-HARD-CAP (not approved, not designed):** an absolute wall-clock bound on *all* initialization work,
> including `mkdir`, DDL and migration, would require out-of-band termination (a supervisor or watchdog process),
> because cooperative timeouts cannot bound code that never runs. This specification does **not** propose such a
> watchdog, and nothing in it may be read as delivering one.

---

## 4. F-2 Deterministic Concurrency Test Design

### 4.1 What "deterministic" must mean here

A test is **deterministic** if, for a given tree, it yields the same verdict on every run on every supported
platform with no scheduling dependence. A test is a **valid detector** of a mutation if it fails **every time** on
the mutated tree.

**Fact shaping the design:** A7 achieves 7.5 % sensitivity against a faithful mutant and 0 % against removal of
`_in_immediate`. Repetition is not a design.

### 4.2 The three mechanisms and their exact claims

| Mechanism | What it does | May claim | May **not** claim |
| --- | --- | --- | --- |
| **M-A. Same-transaction interleaving simulation** | inside the adapter's own transaction, a hook performs one "advance" statement **on the same connection**, then the adapter's write proceeds | "the write must be **column-targeted** / **guarded**; a whole-row write from a stale snapshot reverts the advance" | not a competing committed write; not isolation evidence; **never** multi-process evidence |
| **M-B. Lock-level witness** | while the adapter is paused inside its transaction, an **independent** connection with `busy_timeout=0` attempts `BEGIN IMMEDIATE` | "the adapter holds a **RESERVED** lock at that point" | does not prove a real process was *blocked out*; proves the lock is **held** |
| **M-C. Real multi-process race** | two OS processes contend end-to-end | nothing on its own — an observed rate | never proof |

### 4.3 Thread-ownership rewrite of T-F2-01/02/03 (R-7) — **F2-R11**

**Binding invariant for every F-2 test:**

> **A SQLite connection is created, used and closed in one and the same thread.** No test may rely on
> `check_same_thread=False`, and `check_same_thread` is **never** changed to make a test pass (§3.6.1 freezes it).

**Why this matters here:** `SqliteOperationLedger` builds its connection with the stdlib default **`True`**
(P-5). Any hook that touched `L1` from the main thread while the worker owned it would raise
`sqlite3.ProgrammingError` — a test that "fails for the wrong reason", or worse, passes because the interleaving
never happened.

---

#### **T-F2-01 — column-targeted bump cannot revert an advanced row** *(M-A · class D)*

| Aspect | Design |
| --- | --- |
| **Thread ownership** | `L1` is created **in the worker thread**, used **only** there, and closed **only** there. The main thread never touches `L1`. |
| Goal | the duplicate bump must not write any column other than `duplicate_requests` |
| Initial state | fresh file; one ENTRY op `APPROVED`, `claim_attempts=0`, `duplicate_requests=0` |
| Actors | worker thread (owns `L1`); `threading.Event` pair owned by the main thread |
| Exact order | 1) worker creates `L1` on the file; 2) worker records the operation; 3) worker installs the pause hook **on its own connection**; 4) worker calls `record()` again → `_bump_duplicate` (`:503`) → `_in_immediate` (`:516`) → `BEGIN IMMEDIATE` → `_load` → **hook fires**: `reached.set()`, `go.wait(timeout=10)`; 5) main sets `go` **without touching any connection**; 6) the hook, **still in the worker thread and on the same connection and inside the same transaction**, executes `UPDATE operation_ledger SET state='SUBMITTING', claim_attempts=1 WHERE op_id=?`; 7) hook returns; the adapter's own UPDATE runs; commit; 8) worker closes `L1` and exits |
| Sync points | `reached` (worker → main), `go` (main → worker). No sleeps. |
| Expected (fixed) | final `state='SUBMITTING'`, `claim_attempts=1`, `claim_for_send()` → `False`; `duplicate_requests` **reported, not asserted** |
| Mutation | **M-1**: `_bump_duplicate` → `self._update(self._apply_bump(existing))` (whole row) |
| Why certainly detected | under M-1 the write is a full-row `UPDATE` built from the snapshot loaded **before** step 6, so it writes `state='APPROVED'` unconditionally. The advance is guaranteed, therefore the revert is guaranteed: probability 1. |
| Docstring (mandatory) | *"same-transaction interleaving simulation — verifies column-targeted writes; NOT a multi-process isolation proof."* |

---

#### **T-F2-02 — guarded normalisation rejects a changed hash** *(M-A · class D)*

| Aspect | Design |
| --- | --- |
| **Thread ownership** | identical to T-F2-01: `L1` lives only in the worker thread |
| Goal | the `WHERE op_id=? AND payload_hash=?` guard (`:522`, guard check `:537`) must reject a stale precondition |
| Initial state | one op with a v1 hash, `digest_scheme='v1'` |
| Exact order | worker owns `L1`; hook fires on the normalisation `UPDATE` **before** it executes; still in the worker, still in the transaction, the hook runs `UPDATE operation_ledger SET payload_hash='<other>' WHERE op_id=?`; hook returns; the adapter's guarded UPDATE runs |
| Expected (fixed) | `record()` raises `LedgerUnavailable`; `rowcount == 0`; stored hash is the hook's value; `digest_scheme` unchanged |
| Mutation | **M-3**: drop `AND payload_hash=?` |
| Why certainly detected | with M-3 `rowcount == 1`, the hook's hash is overwritten, the stored-hash assertion fails. Probability 1. |
| Guard/rowcount proof | the test asserts **both** the observable exception **and** that `rowcount` was 0 in the fixed case, so a mutant that merely changes the error message cannot pass. |

---

#### **T-F2-03 — the serialization primitive is actually held** *(M-B · class D — the M-2 witness)*

| Aspect | Design |
| --- | --- |
| **Thread ownership (explicit)** | **`L1` is constructed inside the worker thread** and every operation on `L1` — begin, read, update, commit, close — happens **only in that thread**. `L2` is constructed and used **only in the main thread**. **The main thread performs zero operations on `L1`.** |
| Goal | deterministically detect removal of `BEGIN IMMEDIATE` from `transition()` (`:457`, call `:497`) |
| Initial state | one op in `APPROVED`; file already in WAL |
| Actors | worker thread (owns `L1`); main thread (owns `L2`) |
| Exact order | 1) worker creates `L1`, records the op, installs a pause hook that fires **after** the in-transaction `_load` and **before** the `UPDATE`; 2) worker calls `transition(op, EXPIRED)` → `BEGIN IMMEDIATE` → `_load` → **hook**: `reached.set()`, `go.wait(timeout=10)`; 3) **main** creates `L2 = sqlite3.connect(path, timeout=0.0)` + `PRAGMA busy_timeout=0` and attempts `BEGIN IMMEDIATE`, recording the outcome; 4) main sets `go`; 5) worker completes the transition, commits, **closes `L1` in its own thread**, exits |
| Sync points | `reached`, `go` — two `threading.Event`s |
| Expected (fixed) | step 3 raises `sqlite3.OperationalError` with `primary_code == 5`; after `go`, the op is `EXPIRED`; `claim_for_send()` → `False` |
| Expected (mutant M-2) | step 3 **succeeds** — no RESERVED lock is held |
| Mutation | **M-2**: `transition()` calls `run()` instead of `self._in_immediate(run)` |
| Why certainly detected | the discriminator is a **lock-state fact** observed while the worker is provably stopped — a logical consequence of the primitive, not a race. The pause removes timing from the question. |
| Secondary witness | `set_trace_callback` records that `BEGIN IMMEDIATE` appears between the `SELECT` and the `UPDATE`. Structural only; **not** the acceptance instrument. |
| Cleanup | `L2` closed in the main thread; `L1` closed in the worker; a `finally` in both threads guarantees closure even on assertion failure; `mp`/thread handles joined with timeouts. |

**Why T-F2-03 does not need `check_same_thread=False`:** it never shares a connection. `L1` stays in the worker;
`L2` stays in `main`. The lock being probed is a **file-level** SQLite lock, which is exactly why a second
*connection* is a faithful instrument.

---

#### **T-F2-04 — real two-process race** *(M-C · class P — complementary only)*

Two `spawn`ed processes; construction **after** the barrier (F-2-D6, §5.7). Expected: exactly one
`OperationClaimed`, `claim_attempts == 1`, one submission line, final state `SUBMITTING`, `integrity_check == 'ok'`.
**Reported as "N runs, M failures"; never proof; not the F-2 witness.**

#### **T-F2-05 — the safety consequence** *(class D)*

`claim_for_send()` → `False`; `recover_incomplete()` leaves `UNKNOWN` untouched; `RESOLVED_ABSENT` unreachable
without a complete query.

#### **T-F2-06 — mutation harness** *(class M)*

Copy the tree to `tmp_path` (asserting the copy root is **not** under the repository), apply a scripted textual
mutation, run the designated test in a subprocess, assert non-zero exit and the expected failing test ID.
**Required-catch list:** M-1 → T-F2-01 · **M-2 → T-F2-03** · M-3 → T-F2-02. **5 of 5** harness runs must catch all.

#### **T-F2-07 — the list cannot be quietly shrunk** *(class M)*

Asserts that `M-2` maps to `T-F2-03` in the declared list; fails if the mapping is removed.

### 4.4 Claim boundaries (binding on every report)

| Statement | Allowed? |
| --- | --- |
| "T-F2-01/02 prove the write is column-targeted / guarded" | **Yes** |
| "T-F2-03 proves the serialization primitive is held during the RMW" | **Yes** |
| "T-F2-01..03 prove two processes are isolated" | **No** |
| "The deterministic tests prove absence of a Lost Update **across processes**" | **No** |
| "T-F2-04 over N runs produced M failures" | **Yes** |
| "A7 is now deterministic" | **No** — A7 remains class P |

### 4.5 What the deterministic tests must not do

`sleep()` to widen a window · repeat-and-hope · assert an exact `duplicate_requests` (the contract permits either
+1 or a detected change; the **invariant** is asserted, the counter is reported) · use an in-process
`threading.Lock` as the "other actor" · move a connection between threads (R-7) · edit or delete A7 beyond the
disclosed strengthening.

### 4.6 F-2-D7 — document the atomicity classes (comment-only)

A docstring block tabulating §2.6: which methods are `_in_immediate` RMW and which are single-statement atomic,
and why the second class is safe **only** while it remains one statement. Comment-only = still Gate-2.

---

## 5. F-3 Windows Portability Design

### 5.1 Files and lines in scope

| File | Line(s) | Change |
| --- | --- | --- |
| `tests/test_phase_a_audit_multiprocess.py` | 66, 112 | `fork` → `get_mp_context()` (spawn) |
| `tests/test_fix05_counter_semantics.py` | 181 | same |
| `tests/test_phase_a_audit_multiprocess.py` | 36–37, 220–221 construction; 39, 257 barrier | move construction **after** the barrier (strengthening, separate gate) |
| new `tests/mp_support.py` | — | context factory, bounded teardown, diagnostics |

### 5.2 F-3-D1 — `spawn` on every platform

A test running a different mechanism on the developer's machine than on the target is not evidence about the
target; `fork` inherits the parent's SQLite connections (undefined for SQLite) and can produce false passes;
`spawn` matches production. **Cost:** ~0.3–0.8 s per worker, accounted for in §7.2.

### 5.3 F-3-D2 — worker contract

1. **Module-level function only.** *(Fact: all three workers comply.)*
2. **Picklable, minimal arguments.** *(Fact: compliant.)*
3. **No inherited mutable state.** *(Fact: only immutable constants — `SCOPE`, `INTENT`, `DIGEST`.)*
4. **Mandatory module-level import guard** in every spawning module:

   ```python
   import sys
   from pathlib import Path
   _SRC = Path(__file__).resolve().parents[1] / "src"
   if str(_SRC) not in sys.path:
       sys.path.insert(0, str(_SRC))
   ```

   It runs in the parent **and** every spawned child, making the `goldbot` import independent of `conftest.py`.
5. **`if __name__ == "__main__":`** — **not required under pytest** (the module is imported by the pytest entry
   point, not executed as `__main__`). Required only in standalone reproduction scripts under `tools/`, with the
   reason documented so nobody "fixes" a non-problem.

### 5.4 F-3-D3 — primitives that survive `spawn`

| Primitive | Under `spawn` | Verdict |
| --- | --- | --- |
| plain `Barrier` | not shareable | **Forbidden** cross-process |
| `Manager().Barrier(n)` | proxy | **Required** *(already used: `:68`, `:114`)* |
| `Manager().Queue()` | proxy | **Required** *(already used)* |
| plain `Event()` | not shareable cross-process | **Forbidden** cross-process; fine for **threads** (T-F2-01..03) |
| `Manager().Event()` | proxy | allowed where needed |

**Inference:** the in-scope tests already use Manager proxies, so F-3 is essentially one line per site plus the
import guard. **F-3 must not be scoped as a rewrite.**

### 5.5 F-3-D4 — module purity

A spawned child re-imports the module, so in-scope modules must have **no module-level side effects**. *(Fact:
compliant.)* Guarded by T-F3-06.

### 5.6 F-3-D5 — bounded teardown, no new dependency

**Facts:** no `pytest-timeout`; no `psutil` in `requirements.txt`; a dead worker currently surfaces as a stall and,
per E-2, a **60.10 s** red run whose message was `BrokenBarrierError` — the harness consumed the signal.

| Function (`tests/mp_support.py`) | Behaviour |
| --- | --- |
| `get_mp_context()` | `mp.get_context("spawn")` |
| `run_workers(ctx, target, args_list, *, join_timeout, result_timeout)` | start all; join each; on timeout `terminate()` → `join(5)` → `kill()`; return `[(exitcode, payload)]`; raise a diagnostic naming the dead worker and exit code |
| `finish(manager)` | `manager.shutdown()` in `finally` *(today no manager is explicitly shut down)* |
| `assert_clean_exit(results)` | fail with `exitcode` + captured `repr` |

**Thresholds:** `join_timeout = 120 s`, `result_timeout = 60 s` — unchanged from today. **Ordering constraint:**
the WAL budget (30 s) must stay **below** `result_timeout` (60 s) so a lock timeout surfaces as
`SqliteBusyTimeout`, not a queue timeout.

**Stdlib only** — `terminate`/`join`/`kill`/`exitcode` suffice; `requirements.txt` stays one line.

### 5.7 F-2/F-3 overlap — one disclosed strengthening of A7

**Proposal F-2-D6:** move `SqliteJournal(...)`/`SqliteOperationLedger(...)` **after** `go.wait()` in AU-22
(`:36–37`) and AU-23 (`:220–221`), keeping every assertion — strictly **adds** coverage.
**Hard ordering:** this lands **after** F-1. Implementing it first makes A7 fail at roughly the F-1 rate
(4.4 %–46.7 %) and look exactly like a regression introduced by a test change.

### 5.8 Windows risks this spec does not resolve

The real F-1 rate on Windows file locking · `msvcrt`/`fcntl` in `LiveInstanceLock` (`live_instance_lock.py:44–62`,
OOS-6) · antivirus/indexer holding the DB · `spawn` cost on a 2-vCPU VPS · path length / unicode paths.
**All Unknown; all NOT RUN.**

---

## 6. Source File Impact Matrix

| File | Change | Lines (current) | Requirement | Risk | Gate |
| --- | --- | --- | --- | --- | --- |
| `src/goldbot/adapters/sqlite_init.py` | **NEW** | — | F1-R1…R21 | Low — additive | 1 |
| `src/goldbot/adapters/sqlite_store.py` | modify | 153–169 | F1-R1, R5, R6, R13 | **Medium** — 58 call sites; hash `a74e06db…` changes | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | modify | 147–160, 191–235, 295–320 | F1-R1, R13, R16, R17, R19 | Medium — order path | 1 |
| `src/goldbot/adapters/sqlite_state_store.py` | modify | 27–37 | F1-R1, R2, R13 | Low | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | comment-only | near `:439` | F-2-D7 | None | 2 |
| `tests/test_f1_wal_initialization.py` | **NEW** (fast lane) | — | F-1 acceptance | Low | 1 |
| `tests/test_f1_wal_initialization_slow.py` | **NEW** (env-gated) | — | F-1 acceptance (300-lane) | Low | 1 |
| `tests/test_f2_deterministic_lost_update.py` | **NEW** | — | F-2 | Low | 2 |
| `tests/test_f2_mutation_harness.py` | **NEW** | — | F-2 (T-F2-06/07) | Medium — copies+muts source in tmp | 2 |
| `tests/mp_support.py` | **NEW** | — | F-3 | Low | 3 |
| `tests/test_phase_a_audit_multiprocess.py` | modify (strengthen) | 36–37, 66, 112, 220–221 | F3-R1, F2-D6 | **Medium** — audit test | 3 (after 1) |
| `tests/test_fix05_counter_semantics.py` | modify | 181 (+ guard) | F3-R2 | Low | 3 |
| `docs/adr/` | **NEW ADR** | — | records OD-1…OD-4, §3.4, §3.5, §3.9 | None | 1 |
| `SOURCE-MANIFEST.txt` | regenerate | — | project rule | — | 4 |

**Explicitly untouched:** `domain/fsm/order.py`, `ports/ledger.py` contracts, `app/order_coordinator.py`,
`adapters/mt5_broker.py`, `runtime/*`, `web/*`, any DDL/schema, any magic/risk/strategy file, `requirements.txt`,
VPS/Live settings.

**Carry-forward:** after Gate-1, `sqlite_store.py`'s SHA-256 will no longer be `a74e06db…`; both hashes must be
recorded so the "pre-existing" claim stays auditable.

---

## 7. Test Matrix and Acceptance Criteria

### 7.1 Test classes

| Class | Meaning | Witness? |
| --- | --- | --- |
| **D** Deterministic | forced interleaving / logical lock-state witness | **Yes** |
| **P** Probabilistic | scheduling-dependent; reported as a rate | **No** |
| **S** Sampling | N repetitions bounding a rate | Yes for F-1, under §7.2 |
| **M** Meta/mutation | proves another test detects a mutation | **Yes** for the mutation claim |
| **X** Not runnable | Windows / PowerShell / MT5 | **No** — `NOT RUN` |

### 7.2 Statistical basis (R-10) — per adapter, and its limits

**Scope of the claim.** The 95 % statement is **per adapter, independently**. Each adapter's lane is powered for a
1 % residual rate:

$$\Pr(\text{miss}) = (1-p)^n \le 0.05 \;\text{with}\; p = 0.01 \;\Rightarrow\; n \ge \frac{\ln 0.05}{\ln 0.99} \approx 298 \;\Rightarrow\; n = 300$$

| n | Smallest rate detectable at 95 % power | Verdict |
| --- | --- | --- |
| 10 | ~26 % | useless |
| 40 | **~7.2 %** | **insufficient** — E-2 measured 4.4 %; a 4.4 % regression would pass |
| 100 | ~3.0 % | marginal |
| **300** | **~1.0 %** | **acceptance lane** |

**If a joint three-adapter claim is wanted, the arithmetic changes and must be stated:**

* Three independent per-adapter lanes, each at α = 0.05, give a **family-wise** probability of at least one false
  pass of `1 − 0.95³ ≈ 0.143` — i.e. **≈ 14.3 %**. Stating "all three at 95 %" as a single joint claim **would be
  wrong**.
* To make a joint claim at 95 %, use Bonferroni: **α′ = 0.05/3 ≈ 0.0167** per adapter ⇒
  `n ≥ ln(0.0167)/ln(0.99) ≈ 408` per adapter.
* **Decision for v1.2 (no change without documented counter-evidence):** keep **n = 300 per adapter**, report
  **each adapter separately**, and prohibit any joint "95 % for all three" sentence. If the owner wants a joint
  claim, Q-11 asks for the choice between (a) 300 per adapter with three separate claims, or (b) 408 per adapter
  with one joint claim.

**Explicit non-claim:** *"0 failures in 300"* bounds the residual rate at ≈1 % with 95 % confidence **under the
assumption of independence and stationarity**. It is **not** proof that the defect is absent, and no report may
describe it as such.

### 7.3 F-1 test matrix

| ID | Class | Scenario | Key assertions | Acceptance |
| --- | --- | --- | --- | --- |
| T-F1-01 | D | S1 ×3 adapters | no exception; `journal_mode` == `wal` | 3/3 |
| T-F1-02 | D | S2 rollback DB, 100 rows | mode becomes wal; row values identical | pass |
| T-F1-03 | D | S3 existing WAL | no error; data unchanged | pass |
| T-F1-04 | S | S4 two processes | all constructors return; one file set | **0/300 per adapter, reported per adapter** |
| T-F1-05 | S | S4 **pre-fix tree** (negative control) | — | **≥1 failure in 300** — proves the lane is sensitive |
| T-F1-06 | S | S5 four processes | as T-F1-04 | **0/100** |
| T-F1-07 | D | S6 lock past budget | `SqliteBusyTimeout`; `elapsed ≤ budget + 2 s` **under §3.9.3 conditions**; message has path/attempts/budget | pass |
| T-F1-08 | D | connection leak | patched `connect` recorder; every recorded conn raises "closed" | **0 leaks / 100** |
| T-F1-09 | D | S7 short lock (~0.3 s) | succeeds; `attempts ≥ 2` | pass |
| T-F1-10 | D | S8 corrupt file | `SqliteNotADatabase`; `attempts == 1`; `< 2 s`; message has "database"; **AU-35 unmodified green** | pass |
| T-F1-11 | D | S9 no stray files | `set(os.listdir)` delta ⊆ {`-wal`,`-shm`} | pass |
| T-F1-12 | D | S10 forced `"delete"` | `SqliteInitializationError` | pass |
| T-F1-13 | D | S11 shared-journal branch | helper not called; mode untouched | pass |
| T-F1-15 | D | S12 `check_same_thread` regression | journal usable cross-thread; ledger & state raise `ProgrammingError` on **first use in another thread** | 3/3 |
| T-F1-16 | D | budget is a hard cap *(controlled conditions only)* | `budget_s=1.0`, foreign `BEGIN EXCLUSIVE` held 10 s → `SqliteBusyTimeout`, `elapsed ≤ 3.0 s` | pass |
| T-F1-17 | D | **per-attempt invariants from `InitReport`** | for every attempt: `busy_timeout_ms_used ≤ remaining_ms_before`; `Σ delay_ms_planned ≤ budget`; `Σ delay_ms_actual ≤ budget`; `delay_ms_actual ≥ 0`; and **`elapsed_ms` sum ≤ measurement of total** | pass |
| T-F1-18 | D | S13 migration not tied to the WAL budget | hold `BEGIN EXCLUSIVE` during a v1→v3 migration → controlled error (`LedgerUnavailable` with `__cause__`), **not** silent; `in_transaction` false before the begin; after the lock is released a retry succeeds; `integrity_check == 'ok'` | pass |
| T-F1-19 | D | non-retryable fail-fast | codes `26, 14, 8, 3, 11` → `attempts == 1`, `< 2 s` | pass |
| **T-F1-20** | D | **per-code classification (R-6.7)** | BUSY(5) → retried; BUSY_RECOVERY(261) → retried; BUSY_SNAPSHOT(517) → retried; BUSY_TIMEOUT(773) → retried; **LOCKED(6) → `SqliteUnretryableError`, `attempts == 1`**; **LOCKED_SHAREDCACHE(262) → `SqliteUnretryableError`, `attempts == 1`**; code absent + matching message → retried; code absent + non-matching message → not retried | 8/8 |
| **T-F1-21** | D | S14 backup connections have explicit timeouts | both `:307` and `:313` pass an explicit `timeout=`; the value is documented; a contended backup fails with a controlled error rather than hanging | pass |
| T-F1-14 | X | Windows | — | **NOT RUN** |

### 7.4 F-2 test matrix

| ID | Class | Mechanism | Property | Mutation | Sensitivity |
| --- | --- | --- | --- | --- | --- |
| T-F2-01 | D | M-A | column-targeted bump cannot revert | M-1 | **100 %** |
| T-F2-02 | D | M-A | guarded normalisation rejects | M-3 | **100 %** |
| **T-F2-03** | **D** | **M-B** | serialization primitive is held | **M-2** | **100 %** |
| T-F2-04 | P | M-C | real two-process race | — | reported only |
| T-F2-05 | D | — | no resend licensed | — | 100 % |
| T-F2-06 | M | — | harness catches M-1/M-2/M-3 | itself | **5/5** |
| T-F2-07 | M | — | M-2 stays in the required list | edit of the list | 100 % |

**Additional acceptance (R-7):** a static check asserts that **no F-2 test references a connection created in a
different thread** — implemented as a review checklist item plus a runtime guard (`ProgrammingError` would surface
immediately, and the tests must not suppress it).

### 7.5 F-3 test matrix

| ID | Class | Assertion | Linux | Windows |
| --- | --- | --- | --- | --- |
| T-F3-01 | D | `get_start_method() == "spawn"` | to be run | **NOT RUN** |
| T-F3-02 | D | every worker target picklable | to be run | **NOT RUN** |
| T-F3-03 | D | raising worker → diagnosable failure, not a hang | to be run | **NOT RUN** |
| T-F3-04 | D | hung worker terminated within `2+5+ε` s | to be run | **NOT RUN** |
| T-F3-05 | D | no plain `Barrier`/`Event` cross-process | to be run | **NOT RUN** |
| T-F3-06 | D | module import performs no I/O | to be run | **NOT RUN** |
| T-F3-07 | D | `mp.active_children()` empty after each test | to be run | **NOT RUN** |
| T-F3-08 | X | AU-22/AU-23/crash test under `spawn` | 20/20 on Linux | **NOT RUN** |

**Reporting rule:** every table carries **two** result columns. Windows cells are `NOT RUN` until Gate-5 produces
real logs from a Windows host.

---

## 8. Failure Handling and Observability

| Condition | Detected by | Behaviour | Operator sees | Test |
| --- | --- | --- | --- | --- |
| Transient BUSY (< budget) | retry loop | retries with jittered backoff | nothing (INFO line) | T-F1-09 |
| BUSY past budget | budget | **fail closed** | `SqliteBusyTimeout`: path, attempts, elapsed, budget, `last_code`, name, cause | T-F1-07/16 |
| `LOCKED` / `LOCKED_SHAREDCACHE` | classifier | **fail fast**, no retry | `SqliteUnretryableError`, `attempts == 1` | T-F1-20 |
| WAL not applied | post-transition verify | fail closed | reason `journal_mode is 'delete', not 'wal'` | T-F1-12 |
| Corrupt / non-DB | classifier | fail fast | `SqliteNotADatabase`, message has "database" | T-F1-10 |
| Other non-retryable | classifier | fail fast | code + name | T-F1-19 |
| Migration BUSY | F1-R17 | raise with cause, never silent | `LedgerUnavailable` (cause preserved) | T-F1-18 |
| Backup lock-wait | F1-R19 | controlled error | message names target + timeout | T-F1-21 |
| Worker dies | `run_workers` | terminate→join(5)→kill | failure naming worker + exit code | T-F3-03 |
| Worker hangs | `join_timeout` | bounded | ditto | T-F3-04 |
| Hook never fires | test-side `Event.wait(timeout)` | fails "hook never fired" | distinguishes "path changed" from "assertion failed" | T-F2-01/03 |
| Mutant escapes | harness | harness fails | names the escaping mutant | T-F2-06 |

**Proposal — one structured INFO line per initialization**, using the project's existing convention
(`log.warning(..., extra={"fields": {...}})`, `runtime/live.py:361`): `path`, `attempts`, `elapsed_ms`,
`journal_mode_after`, `created`, `last_code`. Rationale: F-1 is a *rate*; without this an operator cannot learn
that a deployment is contending. **Not proposed:** overloading `migration_report()` with connection state (Q-7).

---

## 9. Risks and Non-Goals

### 9.1 F-1 risks

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| New exception type escapes a caller catching `sqlite3.Error` | **Unknown — A-4** | §3.5.4 inventory is the Gate-1 condition |
| Migration lock-wait (10 s × statements) makes a stuck migration slow to fail | Medium | Bounded **per statement**, explicit error, cause preserved; **no** claim about total migration time (OD-4) |
| `LOCKED` classified non-retryable and a real case appears later | Low | Evidence-based (P-2) + **re-open condition Q-10** |
| `mkdir` stall is unbounded | **Fact by design** (OD-3) | Documented in §3.4.1; RQ-HARD-CAP records the gap without designing it |
| `sqlite_store.py` hash changes | Low | Old+new hashes recorded |

### 9.2 F-2 risks

| Risk | Mitigation |
| --- | --- |
| M-A judged "not a real race" | Declared as same-transaction simulation; M-B supplies lock-state evidence; M-C kept separately |
| M-B proves "lock held" not "process blocked" | Stated in §4.4; SQLite locks are file-level, so held-reserved is the correct witness |
| Thread-ownership mistakes re-introduce `ProgrammingError` confusion | **F2-R11** invariant + §7.4 acceptance check |
| Mutation harness mutates the wrong tree | Copies to `tmp_path` and asserts the copy root is outside the repository |

### 9.3 F-3 risks

| Risk | Mitigation |
| --- | --- |
| `spawn` child cannot import module or `goldbot` | Mandatory module-level `sys.path` guard |
| Wall-clock grows 3–5× | Fast lane stays small; acceptance lane opt-in |
| A7 flaky if changed before F-1 | §10 enforces the order |

### 9.4 Non-goals (no implementation may expand into these)

Order Coordinator architecture · trading/strategy/indicator logic · broker/MT5 behaviour · schema or migration
**DDL** changes · database path or name changes · deletion/recreation of any real database · MT5/broker
connection · demo or real orders · VPS configuration · deployment to a Windows VPS · operational soak · PHASE B ·
PHASE C · production-readiness claims · `requirements.txt` changes · `LiveInstanceLock` semantics · **any process
watchdog for an absolute hard cap (RQ-HARD-CAP)**.

---

## 10. Implementation Sequence

| Step | Content | Gate | Blocking constraint |
| --- | --- | --- | --- |
| 0 | Owner reviews v1.2 and answers Q-1, Q-2, Q-4…Q-12 | **Gate-0** | nothing proceeds |
| 1 | `sqlite_init.py` + unit tests (T-F1-01/02/03/07/08/10/11/12/15/16/17/19/20) | 1 | — |
| 2 | Wire the three adapters (§3.6.1) + F1-R16/R17/R19 in `migrate()`/`_backup()` | 1 | **No test file edited in this step** |
| 3 | `test_f1_wal_initialization.py` (T-F1-04/05/06/09/13/18/21) + slow lane | 1 | T-F1-05 must fail on the pre-fix tree |
| 4 | Gate-1 evidence: §3.5.4 inventory + acceptance lane (300 per adapter, reported separately) + fast lane | 1→4 | Linux only; Windows `NOT RUN` |
| 5 | F-3 `mp_support.py`, `spawn`, import guard, teardown | 3 | **Step 2 merged first** |
| 6 | F-2-D6: construction after the barrier in AU-22/AU-23 | 3 | **Only after step 2** |
| 7 | F-2 tests T-F2-01…03, 05…07 (thread-owned per §4.3) | 2 | new files only |
| 8 | F-2-D7 docstring; regenerate `SOURCE-MANIFEST.txt`; ADR | 2 | — |
| 9 | Independent verification of a new ZIP | 4 | Linux PASS / Windows `NOT RUN`, reported separately |
| 10 | Windows execution: T-F1-14, T-F3-08 | **5** | requires a real Windows host |

**Hard rules:** F-1 before the A7 edit · no test file edited in steps 1–3 · one ZIP per gate.

---

## 11. Independent Verification Plan

| # | Check | Method | Expected | Platform |
| --- | --- | --- | --- | --- |
| V-1 | Delivery integrity | `sha256sum`; `sha256sum -c SHA256SUMS.txt` | recorded hash; 100 % OK | Linux |
| V-2 | Manifest | recompute `SOURCE-MANIFEST.txt` | `files: <n> \| problems: 0` | Linux |
| V-3 | No weakened tests | diff the test tree against the previous ZIP; assert **no removed assertion** in pre-existing files | only disclosed strengthenings | Linux |
| V-4 | Suite | `GOLDBOT_SOAK=0 pytest -q` | `1224 + <new> passed, 6 skipped` | Linux |
| V-5 | F-1 acceptance | 300 per adapter; **reported per adapter** | 0 failures each; raw logs attached | **Linux** |
| V-6 | F-1 **negative control** | same lane on the pre-fix tree | **>0 failures** | **Linux** |
| V-7 | F-1 budget | T-F1-16 | error at `≤ budget + 2 s` under §3.9.3 conditions | Linux |
| V-8 | F-2 determinism | T-F2-01/02/03 × 20 consecutive | **20/20 identical verdict** | Linux |
| V-9 | F-2 mutation | T-F2-06 | every mutant caught, 5/5 | Linux |
| V-10 | F-3 start method | T-F3-01 + `grep '"fork"'` = 0 | pass | Linux |
| V-11 | F-3 hang safety | T-F3-04 | bounded termination | Linux |
| V-12 | Windows items | T-F1-14, T-F3-08 | **NOT RUN** unless a Windows host is provided | **Windows** |

**Rule:** V-6 is mandatory. A lane not shown to be *sensitive* is not evidence of absence — E-2's 0/40 on A7
against a tree with a reproducible 4.4 % startup failure is the proof of that risk.

---

## 12. Explicit Authorization Gates (R-9)

Each gate reports **Linux** and **Windows** in separate columns. **No gate may be recorded as `CLOSED` while any of
its Windows items is `NOT RUN`;** such a gate is `PARTIALLY CLOSED — WINDOWS OUTSTANDING`, with the two halves
written out explicitly.

| Gate | Authorizes | Does **not** authorize | Closes on | Linux result | Windows result | Overall status |
| --- | --- | --- | --- | --- | --- | --- |
| **Gate-0** | Reading and commenting on this spec | **any edit** | owner answers to §14 | — | — | open |
| **Gate-1** | `sqlite_init.py`; three adapter wirings; F1-R16/R17/R19; F-1 tests; ADR | F-2, F-3, **any edit to an existing test**, any DDL | §7.3 (T-F1-01…13, 15…21) + V-5 + **V-6** + §3.5.4 inventory | `PASS` required | `NOT RUN` (T-F1-14) | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`** |
| **Gate-2** | F-2 test files, mutation harness, comment-only docstring | any behavioural source change | §7.4 + T-F2-06 5/5 | `PASS` required | `NOT RUN` | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`** |
| **Gate-3** | `mp_support.py`, `spawn`, import guards, A7 strengthening | **any assertion removal** | §7.5 T-F3-01…07 on Linux | `PASS` required | `NOT RUN` (T-F3-08) | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`** |
| **Gate-4** | **Independent verification of the Linux half only** | any further change; **and it may not close Windows** | V-1…V-11 | `PASS` or `FAIL` | **remains `NOT RUN`** | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`**; never described as "Linux + Windows verified" |
| **Gate-5** | Windows execution and Windows-specific adjustment | anything else | V-12 with real Windows-host logs | — | `PASS`/`FAIL` | the **only** gate that can close Windows items; until then every overall status keeps the `WINDOWS OUTSTANDING` suffix |

**Standing rules**
1. The delivery ZIP is never edited in place: extract → hash → verify → independent copy.
2. One new ZIP per gate; each gate's ZIP hash is recorded in its report.
3. **No gate may contain the phrase "Windows tested"** without logs from a Windows host in that gate's evidence
   folder. Absent that, the only permitted wording is `Windows: NOT RUN`.
4. The overall project status string for Gates 1–4 is **exactly** `PARTIALLY CLOSED — WINDOWS OUTSTANDING`; the
   bare word `CLOSED` may not appear as a gate's overall status before Gate-5.

---

## 13. Traceability Matrix

Columns: **ID · Finding · Source Location · Required Change · Test ID · Acceptance · Evidence**

### 13.1 F-1

| ID | Finding | Source Location | Required Change | Test ID | Acceptance | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F1-R1** | F-1 | `sqlite_store.py:160` | WAL transition via `open_sqlite_connection()` | T-F1-01, 04 | 0/300 pairs (per adapter); 3/3 single | E-1 56/120; E-2 11/250 |
| | | `sqlite_operation_ledger.py:153` | same | T-F1-01, 04, 13 | as above; shared branch untouched | E-1 23/40 |
| | | `sqlite_state_store.py:33` | same | T-F1-01, 04 | as above | E-1 22/40 |
| **F1-R2** | F-1 (**reworded, N-5**) | `sqlite_store.py:165`, `sqlite_operation_ledger.py:155`/`:192`, `sqlite_state_store.py` (declares none, inherits 10 s from `:31`) | **declare and own** the lock-wait policy in one place instead of inheriting it from `connect()` | T-F1-13, 21 | every adapter's effective `busy_timeout` is explicit and identical (10 000 ms) | P-1, P-4, PR-3 |
| **F1-R3** | F-1 | three sites | bounded backoff; **budget is a true cap over the five OD-2 operations** | T-F1-07, 09, 16, 17 | S7 ≥2 attempts; S6 ≤ budget+2 s under §3.9.3; invariants hold | §3.4, §3.9 |
| **F1-R4** | F-1 | three sites | no unbounded retry | T-F1-16 | elapsed ≤ budget + 2 s (restricted) | §3.9.3 |
| **F1-R5** | F-1 | three constructors | fail closed with a typed, diagnosable error | T-F1-07, 10, 19, 20 | taxonomy correct; fields present | §3.5 |
| **F1-R6** | F-1 | three constructors | close the connection on every failure path | T-F1-08 | 0 leaks / 100 | D-5 |
| **F1-R7** | F-1 | three constructors | verify `journal_mode == 'wal'` | T-F1-01, 12 | 100 %; forced fallback raises | no existing test |
| **F1-R8** | F-1 | three constructors | defined behaviour S1…S15 | T-F1-01…06, 09, 13, 15, 18, 21 | §3.7 covered | §3.7 |
| **F1-R9** | F-1 | three constructors | no parallel DB / deletion / path change | T-F1-11 | delta ⊆ {`-wal`,`-shm`} | OD-2 |
| **F1-R10** | F-1 | all three | one policy, one helper | T-F1-13 + review | `PRAGMA journal_mode=WAL` appears **once** in `src/` | P-1 |
| **F1-R11** | F-1 | §2.4 | the Live lock is not the remedy (documented) | — (doc) | §2.4 present | E-2 |
| **F1-R12** | F-1 | `sqlite_operation_ledger.py:141–146` | shared-journal branch must **not** call the helper | T-F1-13 | mode untouched | Proposal |
| **F1-R13** | F-1 | `sqlite_store.py:158` (`False`); `sqlite_operation_ledger.py:151`, `sqlite_state_store.py:31` (default `True`) | `check_same_thread` explicit; **per-adapter value frozen** | T-F1-15 | journal usable cross-thread; ledger/state raise `ProgrammingError` | P-5 |
| **F1-R14** | F-1 | three constructors | **OD-1 30 s budget, monotonic, single shared constant**; remaining recomputed per attempt | T-F1-16, 17 | invariants hold; no per-caller override in production | OD-1, §3.4.3 |
| **F1-R15** | F-1 | helper classifier | `primary = code & 0xFF`; retry **only {5 family}**; `6`/`262` non-retryable; text fallback only if code absent | **T-F1-19, T-F1-20** | 8/8 classification cases | §3.4.2, P-2, PR-1 |
| **F1-R16** | F-1 (**REWRITTEN — R-3**) | `sqlite_operation_ledger.py:192`, `:225–235` | **migration has its own bounded lock wait (10 000 ms), independent of the WAL budget**; policy owned in exactly one place; BUSY exhausts into a controlled error; **no claim about total migration time** | **T-F1-18**, T-F1-17 | independent policy proven; controlled error; no "30 s migration" claim anywhere | OD-4, P-1, P-6 |
| **F1-R17** | F-1 (**REWRITTEN — R-4**) | `:208–213` | **`conn.in_transaction` checked before `BEGIN IMMEDIATE`**; begin only when needed; BUSY/LOCKED when not in a transaction must propagate with cause | **T-F1-18** | no silent `pass`; `__cause__` present; no misreading of BUSY | N-2, P-3, A-8 |
| **F1-R18** | F-1 | helper + adapters | 4-way taxonomy, cause chains, message token "database", **AU-35 untouched** | T-F1-07, 10, 19, 20 + AU-35 | all four classes reachable; AU-35 green | §3.5, PR-2 |
| **F1-R19** | F-1 (**new — N-3 promoted**) | `:307`, `:313` | every independent migration/backup connection has an **explicit** timeout; the current stdlib default (5.0 s) is documented | **T-F1-21** | both connections pass explicit `timeout=`; contended backup fails controlled | P-4 |
| **F1-R20** | F-1 (**new — R-2/OD-3**) | helper entry | `mkdir` is **outside** the budget and documented as such | — (doc + review) | §3.4.1 present; no code change measuring mkdir inside | OD-3 |
| **F1-R21** | F-1 (**new — R-5**) | §3.9 | the document must state what sqlite3 timeouts do **not** guarantee; `budget + 2 s` restricted to defined test conditions; absolute cap recorded as unapproved | — (doc) | §3.9 present; no contrary claim elsewhere | §3.9 |

### 13.2 F-2

| ID | Finding | Source Location | Required Change | Test ID | Acceptance | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F2-R1** | F-2 | `test_phase_a_audit_multiprocess.py:36–37, 220–221` | deterministic tests in a **new** file | T-F2-01…03 | 20/20 identical | E-2 3/40 |
| **F2-R2** | F-2 | `_bump_duplicate` `:503`, call `:516` | prove column-targeted bump | T-F2-01, 05 | M-1 caught 5/5 | §2.6 |
| **F2-R3** | F-2 | `transition` `:457`, call `:497` | prove the serialization primitive is held | **T-F2-03**, 05 | M-2 caught 5/5 | E-2 0/60 |
| **F2-R4** | F-2 | `_normalise_digest` `:522`, guard `:537`, call `:555` | prove the hash guard rejects | T-F2-02 | M-3 caught 5/5 | §2.6 |
| **F2-R5** | F-2 | `claim_for_send` `:562`, `mark_submit_attempt` `:594` | document the single-statement atomic class | T-F2-07 + docstring | docstring present; no behaviour change | §2.6 |
| **F2-R6** | F-2 | new harness | mechanical mutation detection | T-F2-06 | 5/5 catches | §4.3 |
| **F2-R7** | F-2 | A7 file | only the disclosed strengthening | T-F2-04 (P) | no assertion removed | §5.7 |
| **F2-R8** | F-2 | new tests | simulation labelled **same-transaction interleaving**; no cross-process claim | all F-2 docstrings | §4.4 wording honoured | R-7 |
| **F2-R9** | F-2 | new test | **lock-level witness** catches `BEGIN IMMEDIATE` removal | **T-F2-03** | M-2 → 100 % | PR-1 |
| **F2-R10** | F-2 | A7 file | real multi-process test kept, labelled **P**, never a witness | T-F2-04 | reported as a rate | §4.4 |
| **F2-R11** | F-2 (**new — R-7**) | new + rewritten F-2 tests | **strict thread ownership**: a connection is created, used and closed in one thread; `check_same_thread` never changed for tests | T-F2-01/02/03 + §7.4 check | no cross-thread connection use; no `ProgrammingError` in the fixed path | P-5 |

### 13.3 F-3

| ID | Finding | Source Location | Required Change | Test ID | Linux | Windows |
| --- | --- | --- | --- | --- | --- | --- |
| **F3-R1** | F-3 | `test_phase_a_audit_multiprocess.py:66, 112` | `spawn` via factory | T-F3-01 | to be run | **NOT RUN** |
| **F3-R2** | F-3 | `test_fix05_counter_semantics.py:181` | same | T-F3-01 | to be run | **NOT RUN** |
| **F3-R3** | F-3 | three workers | module-level, picklable, no parent state | T-F3-02 | to be run | **NOT RUN** |
| **F3-R4** | F-3 | in-scope modules | module-level `sys.path` guard | T-F3-06 | to be run | **NOT RUN** |
| **F3-R5** | F-3 | `mp_support.py` | bounded teardown + `Manager.shutdown()` | T-F3-04, 07 | to be run | **NOT RUN** |
| **F3-R6** | F-3 | in-scope tests | diagnosable worker death | T-F3-03 | to be run | **NOT RUN** |
| **F3-R7** | F-3 | in-scope tests | Manager proxies only | T-F3-05 | to be run | **NOT RUN** |
| **F3-R8** | F-3 | — | Windows execution | T-F3-08, T-F1-14 | — | **NOT RUN (Gate-5 only)** |
| **F3-R9** | F-3 | every report | Linux and Windows in **separate** columns; `PARTIALLY CLOSED — WINDOWS OUTSTANDING` until Gate-5 | — | enforced | enforced |

### 13.4 Out-of-Scope Findings (recorded, not addressed)

| ID | Finding | Location | Why out of scope | Owner decision |
| --- | --- | --- | --- | --- |
| **OOS-1** | State store declares no lock-wait policy (inherits 10 s) | `sqlite_state_store.py:31` | **In scope** via F1-R2 (reworded) | — |
| **OOS-2** | Atomicity class of `claim_for_send`/`mark_submit_attempt` undocumented and mis-stated in the brief | `:562`, `:594` | Doc-only (F-2-D7) | Gate-2 |
| **OOS-3** | 58 `SqliteJournal(` sites, no factory | `src/` 12, `tests/` 46 | Architectural | PHASE C |
| **OOS-4** | `check_same_thread=False` on the journal connection | `sqlite_store.py:158` | Thread-safety not assessed; **value frozen** | PHASE C |
| **OOS-5** | CLI/web open the same DB as a running Live process with no lock | §2.4 | Ownership model | PHASE C |
| **OOS-6** | `LiveInstanceLock` `msvcrt`/`fcntl`; no Windows test | `live_instance_lock.py:44–62` | Not F-1/F-2/F-3 | Gate-5 |
| **OOS-7** | No test asserts `journal_mode == 'wal'` | whole suite | **Inside** F-1 (T-F1-01/12) | Gate-1 |
| **OOS-8** | `migrate()` backup contention (~20 s stall per handover §5) | `:185–220` | Recorded as mitigated by the handover; **not re-verified** | Gate-1 review |
| **OOS-9** | **N-3 — carried forward, NOT hidden:** `_backup()` opens two connections with **no explicit timeout** (`:307`, `:313`), inheriting the **5.0 s** stdlib default; they sit outside any budget and before `BEGIN IMMEDIATE`. | `:295–320` | **Promoted to F1-R19** (explicit timeout + documentation); the deeper question of bounding total backup time remains out of scope | Q-9 |
| **OOS-10** | `mkdir` latency is unbounded (OD-3 boundary) | helper entry | Owner-drawn boundary; absolute cap is RQ-HARD-CAP, unapproved | recorded |

---

## 14. Open Questions / Assumptions

### 14.1 Assumptions and Unknowns

| ID | Type | Statement | Impact if false | How to check |
| --- | --- | --- | --- | --- |
| **A-1** | Assumption | The Linux-measured F-1 rate characterises the **mechanism**, not the platform's rate | The 300-sample may be mis-powered on Windows | Gate-5 run; adjust n only with a stated power rationale |
| **A-2** | Assumption (narrowed) | `sqlite_errorcode` carries the extended code on the runtime in production | `& 0xFF` becomes inert; the text fallback carries the load | **PR-1 already confirms it on Python 3.13**; remains a version caveat |
| **A-3** | Assumption | A `spawn` child under pytest can import the test module | F-3 needs the mandatory `sys.path` guard (already specified) | T-F3-06 |
| **A-4** | Assumption | No caller relies on catching `sqlite3.Error` around construction | A start could crash uncaught where it previously printed a message | **§3.5.4 inventory is the Gate-1 condition** |
| **A-5** | Assumption | The same-transaction simulation is a faithful stand-in **for the column-targeting property only** | F-2's claim would be weaker than stated | Enforced by §4.4; M-B supplies lock evidence |
| **A-6** | ~~Assumption~~ | ~~30 s is acceptable for CLI paths~~ | — | **DELETED — replaced by Owner Decision OD-1** |
| **A-7** | Assumption | A hard wall-clock cap over DDL is not required for Phase A | The budget does not bound schema/migration execution | Closed by **OD-4**; residual need recorded as **RQ-HARD-CAP** |
| **A-8** | Assumption (**new**) | `sqlite3.Connection.in_transaction` faithfully reports state for transactions started by explicit `BEGIN IMMEDIATE` and is not confused by `executescript`'s implicit commit | F1-R17's pre-check could mis-decide | T-F1-18 asserts the pre-check's value before the begin |
| **A-9** | Assumption (**new**) | The three per-adapter lanes are **independent samples** (separate temp directories, no shared file). If they were not, the Bonferroni analysis in §7.2 would change. | The multiple-comparison arithmetic in §7.2 would be wrong | The lanes use distinct `tmp_path` roots by construction; asserted in T-F1-04 |
| **U-1** | **Unknown** | The real F-1 rate on Windows file locking | Gate-5 planning only | **NOT RUN** |
| **U-2** | **Unknown** | Whether elevated `busy_timeout` on Windows (10 s) changes user-visible startup latency | — | **NOT RUN** |
| **U-3** | **Unknown** | Whether any deployment runs GoldBot over a network share | §3.9.2's filesystem-stall caveat would become material | Ask the owner |
| **U-4** | **Unknown** | The actual duration of a real v1→v3 migration on a large ledger (the 20 s figure in the handover is unreproduced here) | The migration policy's adequacy is unquantified | OOS-8; needs a measured run, not a guess |

### 14.2 Open questions (each blocks a gate)

| ID | Question | Default if unanswered | Blocks |
| --- | --- | --- | --- |
| **Q-1** | Is Gate-1 authorized as specified (helper + three adapters + F1-R16/R17/R19)? | no work begins | Gate-0 |
| **Q-2** | Acceptance lane **300 per adapter** (reported separately), as retained in §7.2? | 300 per adapter, separate claims | Gate-1 close |
| **Q-3** | ~~5 s vs 30 s for CLI vs Live~~ | — | **CLOSED — OWNER DECISION (OD-1)** |
| **Q-4** | May the journal path change exception type (needs the A-4 inventory), or must it keep `sqlite3.OperationalError` and only enrich the message? | new type, subject to the inventory | Gate-1 |
| **Q-5** | Mutation harness inside `tests/`, or under `tools/` and acceptance-only? | `tools/`, acceptance-only | Gate-2 |
| **Q-6** | Which logger name/level for the initialization INFO line? | `goldbot.adapters.sqlite_init`, INFO | Gate-1 |
| **Q-7** | Expose connection state via `migration_report()`? | no (§8) | Gate-2 |
| **Q-8** | ~~Hard cap over DDL?~~ | — | **CLOSED — OWNER DECISION (OD-4)**; residual recorded as RQ-HARD-CAP |
| **Q-9** | `_backup()` bounding: is F1-R19 (explicit timeout + documentation) sufficient for Gate-1, with total-backup bounding deferred? | F1-R19 only; rest deferred | Gate-1 |
| **Q-10** | **Re-open condition:** if shared cache is ever introduced, is the `SQLITE_LOCKED → non-retryable` decision to be revisited (it must be)? | yes, mandatory revisit | future |
| **Q-11** | For F-1: three **separate** 95 % claims at n = 300 each, or one **joint** claim requiring n ≈ 408 per adapter (Bonferroni)? | three separate claims at 300 | Gate-1 close |
| **Q-12** | Is the `boot`/`mkdir` boundary (OD-3) acceptable for Phase A, given an NFS-stalled `mkdir` would not be bounded by the 30 s budget? | accepted as owner-drawn | Gate-1 |

### 14.3 Explicitly NOT established by this document

* Whether the **current** implementation contains a Lost Update — **no evidence was found that it does**.
* Whether F-1 occurs on **Windows**, or at what rate.
* Whether 30 s is adequate in production — it is an **Owner Decision**, not a measured optimum.
* Whether any test listed here **passes** — **no project test was executed for v1.0, v1.1 or v1.2.** Only the
  three stdlib interpreter probes in §2.8 were run.
* The real wall-clock duration of any migration.
* Any statement about MT5, the broker, demo execution, soak behaviour, Cross-VPS fencing or production readiness.

---

## 15. Confirmation of No Change

| Check | Result |
| --- | --- |
| Source files changed for v1.2 | **None** |
| Test files changed for v1.2 | **None** |
| Schema / DDL / migration changed | **None** |
| Project tests executed for v1.2 | **None** |
| Implementation ZIP created | **None** |
| GitHub source files changed | **None** |
| MT5 / broker / VPS / Live settings changed | **None** |
| Demo or Live orders created | **None** |
| Windows PASS claimed | **None** — `NOT RUN` everywhere |
| Production readiness claimed | **None** |
| Delivery integrity at time of writing | `sha256sum -c SHA256SUMS.txt` → **316/316 OK**; ZIP `9375170378…52d1758d` unchanged |
| Content hash of every delivered file | `SOURCE-MANIFEST.txt` recomputed: **299/299 identical**, including **205/205 `.py` files** |
| Code executed for v1.2 | **None.** The three stdlib probes in §2.8 were executed for v1.1/v1.2 authoring, not for v1.2 verification |
| Gates opened by this document | **None** |

**Disclosure — `mtime` artefact.** Every file in the extracted delivery carries `mtime = ctime = the workspace
snapshot-restore timestamp`, so a naive `find -newermt` flags all 205 `.py` files as "recently modified" even
though none was touched. The decisive check is **content**, not timestamps: 299/299 manifest entries and 205/205
`.py` files are byte-identical, and `SHA256SUMS.txt` is 316/316. The only new artefact of this round is this
specification file.

---

**End of specification v1.2.**

```
SPECIFICATION REVISION ONLY
NO SOURCE FILES MODIFIED
NO PROJECT TESTS EXECUTED
NO IMPLEMENTATION ZIP CREATED
NO GATE-1 AUTHORIZATION
AWAITING OWNER REVIEW OF v1.2
```
