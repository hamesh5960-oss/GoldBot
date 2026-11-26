# PHASE_A_WAL_INITIALIZATION_AND_CONCURRENCY_REMEDIATION_SPEC_v1.4

**Project:** GoldBot v8.2.5 · PHASE A
**Document type:** Technical Remediation Specification (**document only — no implementation authorization**)
**Version:** **v1.4** (supersedes v1.3; preserves every valid section, evidence item and confirmed Owner Decision)
**Date:** 2026-11-26
**Baseline:** `GoldBot_v8.2.5_PHASE_A_TARGETED_REMEDIATION_2026-11-25.zip`
SHA-256 `937517037894c99916e6b30d4b5ff3a63530e9bf9931f3b10616384f52d1758d`
**Source tree referenced:** `GoldBot_Source_v8.2.5` (299-entry `SOURCE-MANIFEST.txt`, problems: 0)
**Authority:** project owner — the only party that may authorize implementation.

> **Status vocabulary.**
> **Fact** = read from the delivered source, or established by an isolated stdlib probe (§2.9, §2.9.1).
> **Inference** = conclusion drawn from Facts, stated as such.
> **Proposal** = recommended design; **not implemented, not approved**.
> **Owner Decision** = binding instruction from the project owner (**§2.1 only**).
> **Assumption** = unverified premise the design relies on.
> **Deferred Requirement** = recorded need, explicitly **not** approved or designed here.
> **Unknown** = not established by any evidence available.
> **NOT RUN / NOT VERIFIED** = explicitly not executed — never "passed".

---

## 0. Change Log

### 0.1 v1.3 → v1.4 (this revision)

Every change traces to a numbered correction from the owner's v1.4 revision order (C-11 … C-17). **No Owner
Decision was altered.** No open decision was closed. Nothing was implemented, nothing was tested.

| # | Correction | v1.3 state | v1.4 state | Where |
| --- | --- | --- | --- | --- |
| **C-11** | Backup inside the migration path | `_backup()` (`:207`) runs **before** `BEGIN IMMEDIATE` (`:209`), so Backup is *inside* migration. v1.3 recorded this as `RQ-BACKUP-DEADLINE` — "deferred, unapproved, not designed" — and therefore left a **potentially unbounded operation inside migration**, which OD-4 does not license. | Backup **stays** in migration but receives its **own separate deadline**, structurally independent of the 30 s WAL budget. The numeric value is **OWNER DECISION REQUIRED** — no number is chosen. A **feasible, API-real** mechanism is designed and **runtime-proven** here: `pages=1` stepping + a `progress=` callback that raises our own exception at the deadline (PR-11…PR-21). Explicit statement of *why* `sleep=` and `connect(timeout=)` alone cannot bound it (they cannot: `sleep` sets the gap *between* attempts, and the loop has **no exit condition**). Deterministic failure behaviour, partial-artefact handling, and the independent-vs-shared-source-connection trade-off are all specified. | §3.6.4, §3.10 (new), §5 (new class), §7.3 (T-F1-23), §13.1 (F1-R23…R26), §14 |
| **C-12** | WAL retry algorithm precision | The signature, `InitDeadline`, `Attempt`, `InitReport` and rules R-a…R-f existed, but there was **no executable algorithm** — no statement of whether the connection is rebuilt or re-armed per attempt, no per-attempt `busy_timeout` derivation, no close-on-every-failure-path enumeration. | A complete, **literal-to-Python** pseudocode algorithm (§3.4.6, P-1…P-13) with: one connection **re-armed** per attempt (not rebuilt — justified by PR-7), per-attempt `busy_timeout` **derived from remaining budget and proven re-armable at runtime** (PR-7: 30 000 → 50 ms, measured 51 ms; PR-8: 800 ms, measured 802 ms), upward **and** downward re-arm, explicit `try/except/finally` close paths, final `busy_timeout=10000` after read-back verification, and a handover that must precede expiry. Units are stated at every API boundary (C-17). | §3.4.6, §3.3, §3.4.5 |
| **C-13** | BUSY vs LOCKED in the migration path | §3.5 said `SQLITE_LOCKED` (6) is **non-retryable**, while Sequence A step A14 routed "a BUSY/LOCKED on any statement in A8–A13" into a **single** `SqliteMigrationBusyTimeout`. That conflates two conditions the document itself separates. | Split: **BUSY** → retried inside the migration deadline, then `SqliteMigrationBusyTimeout`. **LOCKED** → **fail-fast**, never enters the BUSY loop, raises its **own** type `SqliteMigrationLockError`. Both wrap to `LedgerUnavailable` at the public boundary with the cause chain preserved. A timeout message is **forbidden** for LOCKED. Full exception-mapping table added. **New runtime fact (PR-5a/PR-23) that materially qualifies the design:** `SQLITE_LOCKED` (6) is **reachable without shared cache** — an unexhausted read cursor on the *same* connection followed by DDL raises it — but this codebase materialises every cursor before any DDL, so the branch is **defensive**. Recorded honestly rather than asserting unreachability. | §3.5.1, §3.5.2, §3.5.6, §3.6.3 (A14), §7.3 (T-F1-24), §13.1 (F1-R27) |
| **C-14** | Probe count | The **§2.9 heading still read "three probes"** while its own table listed PR-1…PR-4. v1.3 corrected only the Change-Log row, not the section. | Heading, disclosure line and every reference now say **four**. A whole-document scan for `three probes`, `3 probes`, `two probes` returns **zero live claims**. **No probe was invented to fix a counting error** — PR-1…PR-4 are unchanged, and every probe added in v1.4 (PR-5…PR-26) is a *new, separately motivated runtime measurement* for C-11/C-13/C-16/C-17, listed in a **separate table**. | §2.9, §2.9.1, §15 |
| **C-15** | T-F1-16 budget wording | T-F1-16 precondition 5 still read `budget_s = 1.0 via the module-constant patch`, re-introducing the name that C-06 deleted from the public API. | Replaced with the literal call `monkeypatch.setattr(sqlite_init, "WAL_INIT_BUDGET_S", 1.0)` and an explicit sentence: **no `budget_s` argument is passed to the helper.** Every remaining `budget_s` occurrence was audited: the only survivors are (i) the **field** of `InitDeadline` (internal, not an API parameter) and (ii) historical Change-Log text describing what C-06 removed. Both are labelled. | §7.3 T-F1-16, §3.3, §13.1 |
| **C-16** | Runtime evidence for BUSY / LOCKED text | Classification rested on **PR-1** (BUSY) only. The `SQLITE_LOCKED` text path was **assumed**, never observed; tests mock the classifier and therefore cannot prove the runtime produces the expected code/name/message pair. | **Eight new runtime probes executed** (PR-5a, PR-6, PR-15, PR-16, PR-23, plus re-confirmation of PR-1) capturing Python version, SQLite runtime version, exception type, `sqlite_errorcode`, `sqlite_errorname` and message **verbatim** for both families. Result: **BUSY → `OperationalError`, code 5, `SQLITE_BUSY`, `database is locked`**; **LOCKED → `OperationalError`, code 6, `SQLITE_LOCKED`, `database table is locked`**. Both texts are **observed**, not inferred. A design-level probe (T-F1-25) is specified to keep asserting them at run time; the suite may not rely on mocks alone. | §2.9.1, §2.9.2, §7.3 (T-F1-25), §13.1 (F1-R28) |
| **C-17** | Timeout units | Units were implied, never tabulated; `connect(timeout=)` (seconds) and `PRAGMA busy_timeout` (ms) sat in the same sentences, and PR-20 shows `connect(timeout=5000)` is **silently accepted** as 5 000 **seconds**. | A single normative units table (§3.4.7) with a **milliseconds suffix rule** for every internal variable and field, a **runtime-demonstrated** hazard (PR-20: `connect(timeout=5000)` installs `busy_timeout = 5 000 000`), and a review checklist. Every code sample, pseudocode line and table cell in the document was swept. | §3.4.7, §3.3, §3.4.6, §3.6.4 |

**Cross-cutting consequence of C-13 + C-11.** v1.3's Sentence *"an unbounded operation remains inside migration"* is
now **false**, and v1.3's *"`SQLITE_LOCKED` cannot arise here"* is now **qualified**. Both changes are corrections to
**this document's** claims, not to Owner Decisions. OD-1…OD-5 are untouched.

**No Finding from v1.3 was removed.** N-1…N-7, OOS-1…OOS-11, F1-R1…F1-R22, A-1…A-11, U-1…U-6, Q-1…Q-13 all survive.

### 0.2 v1.2 → v1.3 (previous revision — retained for audit trail)

Every change traces to a numbered correction from the owner's v1.3 revision order. Nothing else was altered.

| # | Correction | v1.2 state | v1.3 state | Where |
| --- | --- | --- | --- | --- |
| **C-01** | §2 C-01 Migration `busy_timeout` ownership | §3.6.3 contained **contradictory** instructions (re-arm after the helper / delete `:192` / keep `:192` as owner / avoid double arming) and left "either A or B" alternatives | **One unambiguous sequence** for the Ledger path constructor and for the Journal/State constructors; **one owner per connection: the connection's creator**; `migrate():192` is **deleted** (it currently mutates a borrowed connection); explicit position of `busy_timeout=10000` relative to Backup, `BEGIN IMMEDIATE`, DDL and commit; shared branch confirmed to neither call the helper nor alter policy. | §3.6.3 |
| **C-02** | §3 Text fallback retries `SQLITE_LOCKED` | fallback accepted **both** `database is locked` **and** `database table is locked`, contradicting the "6 ⇒ non-retryable" decision | Classifier corrected: `database is locked` → BUSY → retry **only when the code is absent**; **`database table is locked` → LOCKED → fail fast**; **any ambiguous text without a reliable code → fail closed**. LOCKED pattern matched **before** BUSY. New tests T-F1-22a/T-F1-22b; T-F1-20 and **F1-R15** updated. | §3.4.2, §7.3, §13.1 |
| **C-03** | §4 Backup API claims | F1-R19 / T-F1-21 **implied** that an explicit `connect(timeout=…)` yields a bounded total backup | Claim **removed**. `Connection.backup()` has its **own** BUSY/LOCKED retry loop with a `sleep` parameter (**Fact**: `sleep=0.25` default on this interpreter; `_backup()` passes no arguments at `:309`). Phase A scope limited to: explicit documented `timeout=` on both independent connections + correct cleanup; **no claim of bounded total backup duration**; total-backup deadline recorded as **RQ-BACKUP-DEADLINE (deferred, unapproved)**. | §3.6.4, §7.3, §13.1 |
| **C-04** | §5 Migration exception taxonomy | §3.5.1 said migration failure ⇒ `LedgerUnavailable`; §3.6.3 said exhausted migration wait **may** map to `SqliteBusyTimeout` — not interchangeable | **One chain defined:** `SqliteMigrationBusyTimeout` → wrapped as `LedgerUnavailable` at the Ledger public boundary, original `sqlite3` error preserved in `__cause__`. WAL's 30 s class and migration's 10 s class are **distinct types** so they can never be misrepresented. Catch sites inventoried; tests assert **both** levels of the chain. | §3.5, §3.6.3, T-F1-18 |
| **C-05** | §6 `mkdir` boundary | recorded as **OD-3**, a binding Owner Decision | **Withdrawn from the Owner Decision table** and reclassified as **Proposal P-1 (pending owner confirmation)**. Boundary kept visible. **Q-12 stays OPEN and blocking Gate-1**; no approval inferred from any default. | §2.1, §3.4.1, §14 |
| **C-06** | §7 Constant vs test override | helper signature exposed `budget_s` publicly while the text called 30 s non-configurable | **`budget_s` removed from the public signature entirely.** One module constant `WAL_INIT_BUDGET_S = 30.0`. Production callers have **no parameter** to pass. Tests shorten the budget by patching the constant (stdlib mechanism), never through the API. | §3.3, §3.4.3, T-F1-16 |
| **C-07** | §8 T-F1-17 vs OS scheduling | required `Σ delay_ms_actual ≤ budget`, contradicting §3.9.2's admission that the scheduler may oversleep | Requirement made consistent: **planned** backoff must not exceed the remaining budget; **actual** backoff is measured and **reported**; remaining is recomputed **after every sleep**; **no retry may start after expiry**; **no claim** that actual sleep is bounded by the requested duration. `budget + 2 s` remains confined to T-F1-16. | §3.3, §3.4.5, §3.9, §7.3, F1-R3/R4 |
| **C-08** | §9 T-F1-16 preconditions | did not establish that the DB was in rollback mode, nor that the blocker actually held the lock | Seven preconditions specified explicitly, including **verification that the file is not already WAL**, **verification that the blocker holds `BEGIN EXCLUSIVE`** (independent probe), and `finally` cleanup. The test must exercise the **actual transition**. | T-F1-16 |
| **C-09** | §10 T-F1-11 directory delta | did not say whether the main DB file exists before the snapshot | Three-way classification: **expected main DB** / **expected `-wal` / `-shm` sidecars** / **anything else = failure**. | T-F1-11 |
| **C-10** | §11 probe count and evidence consistency | §2.8 said "two probes" but listed PR-1, PR-2, PR-3; §15 said three | Count corrected to **four** (`PR-1`…`PR-4`) — a **fourth** probe (PR-4, `Connection.backup()` signature) was added in this revision and the earlier "two/three" wording is superseded. A **three-source provenance table** now separates the handover's reported figures, the shipped artefacts' figures and the independent audit's figures — with the **disclosed discrepancy** that the handover reports the extracted run as `1224 passed, 6 skipped — 18.41 s` while the shipped log `FINAL_extracted_copy.log` records `1224 passed, 6 skipped, 3 warnings in 18.31 s`. Neither number is silently preferred. | §2.9, §2.10, §15 |

**No Finding from v1.2 was removed.** N-1…N-7 and OOS-1…OOS-10 all survive; N-3 is now expressed through
F1-R19 with the C-03 correction applied.

---

## 1. Executive Summary

### 1.1 Status vocabulary — strictly separated (v1.4 requirement 9)

Every capability in this document carries **exactly one** status. The ladder is one-way and a later rung is never
implied by an earlier one.

| Status | Means | Evidence required to claim it | What it does **not** mean |
| --- | --- | --- | --- |
| **NEEDS** | A requirement exists and is not yet designed | the requirement ID | nothing is known |
| **DESIGNED** | A design exists on paper, traceable to requirements, with acceptance criteria | this document | nothing runs |
| **IMPLEMENTED** | Source code exists | an implementation with a diff, reviewed | nothing is tested |
| **TESTED** | Project tests executed and green | **project** test logs with dates and counts | not verified independently |
| **VERIFIED** | Independent verification reproduced the result | a separate verifier's report | not production-ready |

**Status of every capability in this document: `DESIGNED` at most.** No capability is `IMPLEMENTED`, `TESTED` or
`VERIFIED`. The runtime probes in §2.9/§2.9.1 are **library behaviour measurements on a throw-away file** — they
support design decisions; they are **not** project tests and confer **no** rung above `DESIGNED` on anything.

**Gate-0 is not claimed, and Gate-1/2/3 remain closed.**

This document specifies — and only specifies — the remediation design for three findings from the independent
audit of 2026-11-26, under the owner's confirmed decisions.

| ID | Class | Statement | Blast radius |
| --- | --- | --- | --- |
| **F-1** | Availability / Startup defect (**pre-existing**) | `PRAGMA journal_mode=WAL` can fail with `database is locked` when two processes initialize a **brand-new** SQLite file simultaneously. Three adapters perform this initialization independently. | CLI, report, web, Live startup; **no corruption observed** |
| **F-2** | Verification gap | The A7 multi-process test does not deterministically prove absence of a Lost Update, and cannot reliably detect re-introduction of the pre-fix whole-row write. | Test credibility only; **no evidence the implementation is defective** |
| **F-3** | Test portability gap | Three test sites hard-code `multiprocessing.get_context("fork")`; Windows — the target platform — does not support it. | Test executability on the production platform; no source defect |

**Headline Facts (re-verified against the reference tree for v1.4)**

* `PRAGMA journal_mode=WAL` appears in exactly **three** locations: `sqlite_store.py:160`,
  `sqlite_operation_ledger.py:153`, `sqlite_state_store.py:33`. No fourth exists.
* `busy_timeout` appears in exactly **three** places — `sqlite_store.py:165`, `sqlite_operation_ledger.py:155`,
  `sqlite_operation_ledger.py:192` — all literal `10000`. `SqliteRiskStateStore` declares none but inherits
  **10 s** from `sqlite3.connect(timeout=10.0)`.
* **No test anywhere** asserts `journal_mode == 'wal'`.
* `claim_for_send` (`:562`) and `mark_submit_attempt` (`:594`) do **not** use `_in_immediate`; `transition`
  (`:457`, call `:497`), `_bump_duplicate` (`:503`, call `:516`), `_normalise_digest` (`:522`, call `:555`) do.
* A faithful re-introduction of the pre-fix whole-row write reproduced failures in **3 of 40** A7 runs (7.5 %).
* `get_context("fork")` at `test_phase_a_audit_multiprocess.py:66`, `:112`, `test_fix05_counter_semantics.py:181`.
* `Connection.backup()` exposes `sleep=0.25` and retries BUSY/LOCKED itself; `_backup()` calls it with **no
  arguments** (`:309`), so **no part of this design bounds its total duration**.

**Headline Proposals**

* One shared initializer with a **30 s monotonic budget** (OD-1) covering exactly five operations (OD-2).
* **The connection's creator owns its lock-wait policy**; `migrate()` re-arms nothing and mutates nothing.
* Distinct exception types for the WAL budget (30 s) and the migration lock wait (10 s), chained explicitly.
* Deterministic tests under **strict thread ownership**; `BEGIN IMMEDIATE` removal caught by a lock-level witness.
* `spawn` everywhere with bounded teardown and no new dependency.
* **Nothing in this document is implemented. No project test has been executed for v1.0–v1.4.**

---

## 2. Owner Decisions, Evidence and Scope

### 2.1 Owner Decisions (binding) — and one withdrawn

| ID | Decision | Status |
| --- | --- | --- |
| **OD-1** | **One 30-second SQLite WAL initialization budget for all three adapters** (`SqliteJournal`, `SqliteRiskStateStore`, `SqliteOperationLedger` path-based branch), with **identical policy for Live, CLI, Web, Report and administrative callers**. | **CONFIRMED — BINDING** |
| **OD-2** | The budget covers exactly: **connection opening · WAL transition · recoverable retries/backoff · WAL read-back verification · required helper PRAGMAs and handover of a ready connection**. | **CONFIRMED — BINDING** |
| **OD-3** | *(former text)* "`path.parent.mkdir` is outside the 30 s budget." | **WITHDRAWN FROM OWNER DECISIONS.** The owner approved the WAL budget scope and the Schema/Migration exclusion but **did not separately confirm the `mkdir` boundary**. Reclassified as **Proposal P-1** (§3.4.1) pending explicit confirmation. **Q-12 remains OPEN and blocks Gate-1.** No approval may be inferred from a default answer. |
| **OD-4** | **Schema creation and Migration are outside the 30-second wall-clock budget**, but **migration lock waits must nevertheless be bounded and detectable**. | **CONFIRMED — BINDING** |
| **OD-5** | This revision authorizes **documentation only**: no source change, test change, project test execution, implementation ZIP, or gate opening. | **CONFIRMED — BINDING** |
| **OD-6** | **The numeric value of the Backup deadline (C-11) is not yet determined by the owner.** The design in §3.10 requires *a* value; **this document deliberately chooses none** and refers to it only as the symbolic `BACKUP_DEADLINE_S`. Any number appearing anywhere in this document is illustrative of *form*, never of *decision*. | **OPEN — OWNER DECISION REQUIRED.** Not binding, not defaulted. **Blocks Gate-1** for that one value; the rest of §3.10 is designable without it. |

**What is deliberately absent from this table.** There is **no** OD for `mkdir` (withdrawn, now P-1/Q-12), **no** OD
for the backup value (OD-6 open), and **no** OD for the backup source-connection choice (Q-14 open). A reviewer who
finds a number in §3.10 has found the **symbol**, not a decision.

**Rules applied to this table**
1. No Proposal or Assumption may be promoted into this table without explicit owner approval.
2. `OD-3` keeps its identifier so earlier documents remain traceable; its status line is the authoritative record.
3. Every consequence elsewhere in this document names the OD it rests on.

### 2.2 Evidence sources

| # | Source | Nature | Used for |
| --- | --- | --- | --- |
| **E-1** | Owner audit, 2026-11-26 (F-1/F-2/F-3; 120/40/40 process pairs) | Owner-supplied **Fact** | Failure rates per adapter |
| **E-2** | Independent audit, 2026-11-26 (`INDEPENDENT_VERIFICATION_2026-11-26.md`, `/home/user/evidence/`) | Independent measurement | Root cause; fresh-vs-existing split; mutation sensitivity |
| **E-3** | Delivered source tree, read directly (`grep`/`sed`/`awk`) | **Fact** | Every line number here |
| **E-4** | `SOURCE_DIFF_FULL.diff` | **Fact** | Proves `sqlite_store.py` untouched by the Targeted Remediation |
| **E-5** | Isolated stdlib interpreter probes (§2.9) | **Fact** | N-4, N-5, N-8; extended-code and backup mechanics |
| **E-6** | Shipped artefacts: `REPORTS/TEST_LOGS/*.log`, `REPORTS/REPRODUCTION.md`, `GOLDBOT_HANDOVER_2026-11-25.md` | **Fact with provenance** | §2.10 evidence reconciliation |

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
*smallest* rate we care about.
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

### 2.5 Fact inventory (established by direct read for v1.3; unchanged in v1.4)

| # | Fact | Basis |
| --- | --- | --- |
| **P-1** | `busy_timeout` appears exactly at `sqlite_store.py:165`, `sqlite_operation_ledger.py:155`, `sqlite_operation_ledger.py:192` — all `10000`. None in `sqlite_state_store.py`. | `grep -rn busy_timeout src/` |
| **P-2** | **Shared cache is never enabled**: no `cache=shared`, `shared_cache`, `cache_shared`, `PRAGMA cache` in `src/` or `tests/`. All connections are file-based. | `grep -rn "cache=shared\|shared_cache\|PRAGMA cache" src/ tests/` → none |
| **P-3** | `connection.in_transaction` is used **nowhere** today. | `grep -rn in_transaction src/ tests/` → none |
| **P-4** | `sqlite3.connect` sites: `sqlite_state_store.py:31` `timeout=10.0`; `sqlite_store.py:158` `timeout=10.0, check_same_thread=False`; `sqlite_operation_ledger.py:151` `timeout=10.0`; **`:307` and `:313` — no timeout argument (stdlib default 5.0)**. | `grep -rn "sqlite3.connect(" src/` |
| **P-5** | `check_same_thread` set **only** by the journal (`:158`, `False`); ledger path branch (`:151`) and state store (`:31`) use the default `True`. | `grep -rn check_same_thread src/` |
| **P-6** | `migrate()` exact sequence: `busy_timeout` PRAGMA `:192` (wrapped in `except sqlite3.Error: pass`, `:193–194`) → `_version()` `:196` → early return `:197–199` → `_backup()` `:207` → `BEGIN IMMEDIATE` `:209` inside `except sqlite3.OperationalError: pass` `:210–213` → re-check `:215–220` → `_upgrade_columns()` `:224` → DDL loop `:225–227` → version upsert `:228–232` → `commit()` `:233` → `except Exception: rollback(); raise` `:234–236`. | `awk` `:181–240` |
| **P-7** | `_backup()` opens `destination = sqlite3.connect(target)` at `:307` and `check = sqlite3.connect(target)` at `:313`, **both without a timeout argument**, and calls `self._conn.backup(destination)` at `:309` **with no `pages`/`progress`/`sleep` arguments**. | `awk` `:293–325` |
| **P-8** | The shared-journal branch is `:141–146`: `self._conn = journal.connection`, `self._owns_connection = False`. `migrate()` is then called on that borrowed connection at `:160`. | direct read |

### 2.6 F-2 — the correction to the audit brief, carried forward

| Method | Definition | `_in_immediate` call | Targeted SQL | Safety argument |
| --- | --- | --- | --- | --- |
| `transition` | `:457` | **`:497`** | full-row `UPDATE` from `_apply_transition` | RMW serialized |
| `_bump_duplicate` | `:503` | **`:516`** | `… duplicate_requests = duplicate_requests + 1 …` | RMW serialized |
| `_normalise_digest` | `:522` | **`:555`** | conditional `UPDATE … WHERE op_id=? AND payload_hash=?` (guard `:537` `rowcount != 1`) | guarded write + re-read |
| `claim_for_send` | `:562` | **none** | `… SET state=?, claim_attempts = claim_attempts+1 WHERE op_id=? AND state IN (?,?)` (`:572`) | one conditional statement is atomic; `rowcount` is the CAS witness |
| `mark_submit_attempt` | `:594` | **none** | `… submit_call_markers = submit_call_markers + 1 WHERE op_id=?` (`:603`) | one incrementing statement is atomic |

**Inference:** no defect is implied, but the distinction must be documented (F-2-D7): a future refactor adding a
second statement to either single-statement method would silently lose serialization.

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

### 2.8 Integrity evidence

| Check | Result | Status |
| --- | --- | --- |
| Delivery ZIP SHA-256 | `9375170378…52d1758d` | **Fact** |
| `SHA256SUMS.txt` | 316/316 OK (re-verified at end of this round) | **Fact** |
| `SOURCE-MANIFEST.txt` | files: 299 \| problems: 0 | **Fact** |
| `sqlite_store.py` SHA-256 | `a74e06db297c9f9ee8d6b6d8ef6942daa6528a70cbacad44db5a0f1fd29cfcba` | **Fact** |
| `sqlite_store.py` in `SOURCE_DIFF_FULL.diff` | 0 occurrences | **Fact** — F-1 is pre-existing |
| `domain/fsm/order.py` SHA-256 | `bf4154c6…f1774d0f` | **Fact** — untouched |

### 2.9 Environment probes and full disclosure — **four** authoring probes (C-10, C-14)

> **Disclosure.** **Four** stdlib-only probes were executed while authoring v1.1–v1.3 (retained unchanged), each in a temporary
> directory with a throw-away file. They imported **no project code**, opened **no project database**, ran **no
> project test**, and produced no artefact inside the delivery.
> A **further twenty-two** stdlib-only probes (PR-5 … PR-26) were executed while authoring **v1.4** for C-11, C-13,
> C-16 and C-17; they are listed separately in §2.9.1 and are subject to the **same** disclosure terms.
> **No project test was executed for v1.4.**

| Probe | Question | Result (**Fact**) |
| --- | --- | --- |
| **PR-1** | What does a contended write raise, and does the exception carry extended codes? | `sqlite3.OperationalError`; `sqlite_errorcode = 5`; `sqlite_errorname = 'SQLITE_BUSY'`; `5 & 0xFF = 5` |
| **PR-2** | What does a corrupt file raise on `PRAGMA journal_mode=WAL`? | **`sqlite3.DatabaseError`**, `sqlite_errorcode = 26`, message `file is not a database` |
| **PR-3** | Does `sqlite3.connect(timeout=10.0)` alone install a 10 s busy handler? | **Yes** — a contended write waited **10.02 s**, then raised `database is locked` |
| **PR-4** | What is `Connection.backup()`'s signature on this interpreter? | `(self, /, target, *, pages=-1, progress=None, name='main', **sleep=0.25**)` — a `sleep` parameter exists, confirming an internal retry loop between attempts |

**Inferences:** (a) `& 0xFF` primary extraction is sound on Python 3.13; (b) the corrupt-file path is
`DatabaseError`, not `OperationalError`; (c) the corrupt message contains "database" — the accidental reason AU-35
passes today, which §3.5.3 makes deliberate; (d) the state store already waits up to 10 s, so F1-R2 concerns
**declared, owned policy**, not repair; (e) **PR-4 establishes that a `connect()` timeout cannot bound a backup** —
C-03.

**Finding from these four probes, restated for v1.4:** PR-4 is the *only* reason C-03 exists. It also means the
`progress=`/`pages=` mechanism in §3.10 is the **only** lever in the standard library that can interrupt a backup —
a fact v1.3 could not state because it had not tested it.

### 2.9.1 Runtime probes executed for v1.4 (C-11, C-13, C-16, C-17)

> **Execution record.** All probes below were **actually executed** on the authoring host during v1.4 preparation.
> Environment, captured at the top of each run: **Python 3.13.18**, **SQLite runtime 3.46.1**, `sqlite3` module
> `2.6.0`. Each used a throw-away file inside a fresh `tempfile.mkdtemp()` directory. **No project code was
> imported, no project database was opened, no project test was run.**
> Nothing here is a prediction. Where a probe produced **no** answer, that is stated as such.

**C-16 — the runtime identity of the two families (this is the evidence v1.3 lacked).**

| Probe | Question | **Observed** result |
| --- | --- | --- |
| **PR-5a** | Can `SQLITE_LOCKED` (6) be produced **without shared cache**? | **Yes.** An **unexhausted** read cursor on the *same* connection, then `DROP TABLE`: `sqlite3.OperationalError`, `sqlite_errorcode = 6`, `sqlite_errorname = 'SQLITE_LOCKED'`, message **`database table is locked`**. |
| **PR-6** | What does cross-connection contention look like? | `sqlite3.OperationalError`, `sqlite_errorcode = 5`, `sqlite_errorname = 'SQLITE_BUSY'`, message **`database is locked`**. |
| **PR-23** | Is the LOCKED branch reachable in **this** codebase's migration path? | **No.** Every cursor in `_column_names()` (`:272–273`), `_has_ledger_table()` (`:164`) and `sqlite_store.py:194` is **fully materialised** (set comprehension) before any DDL. A materialised cursor + `ALTER TABLE` produced **no error**. The LOCKED branch is therefore **defensive**, not reachable-by-design — but PR-5a proves it is **not impossible in principle**, so v1.3's flat claim is withdrawn. |
| **PR-5d** | Can a *read* or a *DDL* under `BEGIN EXCLUSIVE` be confused with LOCKED? | **No.** Both raised code **5** / `SQLITE_BUSY` / `database is locked`. The two texts are **disjoint in practice**, not merely by substring reasoning. |

**Consequence for the classifier (supersedes v1.3's reasoning, keeps v1.3's rule).** v1.3 justified
`6 ⇒ non-retryable` with *"P-2 proves shared cache is never enabled, so cross-process contention cannot produce
6"*. That inference is **correct about cross-process contention** and PR-6/PR-5d confirm it. But v1.3 then implied
6 could not happen here at all; PR-5a/PR-23 show it can in principle (same-connection, unfinished cursor) and
cannot in this codebase's present code. **The rule does not change** — 6 stays fail-fast — only the
**justification** is corrected from "impossible" to "reachable only by a programming error, and fail-fast is
exactly the right response to a programming error."

**C-12 — can a live connection be re-armed per attempt? (this decides "rebuild" vs "re-arm")**

| Probe | Question | **Observed** result |
| --- | --- | --- |
| **PR-7** | Does `PRAGMA busy_timeout=N` re-arm an **existing** connection downward? | **Yes.** `connect(timeout=30.0)` installed `30000` ms; `PRAGMA busy_timeout=50` changed it to `50` ms and the next contended write waited **51 ms**, not 30 s. **The 30 s handler was genuinely displaced.** |
| **PR-8** | And upward? | **Yes.** `PRAGMA busy_timeout=800` under a held lock waited **802 ms**. |
| **PR-20** | What does a unit error cost? | `connect(timeout=5000)` — intended as 5 000 ms — installs `busy_timeout = 5 000 000` ms = **5 000 seconds**, silently. `connect(timeout=5.0)` and `PRAGMA busy_timeout=5000` both give 5 000 ms. **This is why C-17 exists.** |

**Consequence:** a single connection can be **re-armed** between attempts, so the algorithm in §3.4.6 does **not**
rebuild the connection. PR-7 is the proof that the re-arm is real rather than cosmetic — without it the
per-attempt `busy_timeout` derivation would be decorative.

**C-11 — bounding the backup (this is the substantive new evidence)**

| Probe | Question | **Observed** result |
| --- | --- | --- |
| **PR-9b** | Does `backup()` return under a held `BEGIN EXCLUSIVE`? | **No.** Killed by an external 8 s wall clock while still inside `backup()` with `sleep=0.05`. |
| **PR-16** | Is that a slow loop or genuinely unbounded? | **Unbounded.** Killed by an external **25 s** wall clock. `backup()` has **no internal cap** — v1.3's phrasing "retry loop" understated it. |
| **PR-22** | What does `sleep=` control? | The **gap between attempts only**. In 0.6 s: `sleep=0.0` → 8 252 progress callbacks; `sleep=0.25` → **4**. It changes *rate*, never *termination*. **This is the proof that `sleep=` cannot be a deadline.** |
| **PR-17** | Does `pages=` change interruptibility? | **Decisively.** `pages=-1` (the **current** call at `:309`) gave **1** callback for 2 M rows in 0.016 s — **no opportunity to intervene**. `pages=1` gave **1 941** callbacks. A deadline is only possible with page-stepping. |
| **PR-10b** | Is the `progress` callback reachable **while the source is locked**? | **Yes, but only after the source's own busy handler gives up.** With the source at its inherited 10 s handler, the first callback arrived at **5.01 s**. With the source re-armed to `busy_timeout=0` (PR-11), callbacks arrive every sleep cycle. |
| **PR-11** | Does the composite design actually hold a deadline? | **Yes.** Independent source connection + `busy_timeout=0` + `pages=1` + `progress` deadline of **2.00 s** → aborted at **2.01 s** (**+0.01 s overshoot**), 41 callbacks, destination 0 B. |
| **PR-21** | What is left on disk after an early abort? | **0 bytes; `integrity_check` = `ok`; zero tables.** The leftover is an **empty shell**, not a silently-truncated database. |
| **PR-12 / PR-18 / PR-19** | And after a later abort? | Same class of result: the destination held **no table** (`PR-12`, `PR-18`) or, in one run that had already completed the copy, a **complete** copy (`PR-19`). **No run produced a partially-populated, openable, `integrity_check`-clean but silently-wrong backup.** That is *evidence*, not proof — §3.10.4 therefore still mandates deletion. |
| **PR-15 / PR-24 / PR-25** | Does the WAL-mode source (which is what `:309` actually sees, since A2 precedes A7) block at all? | **No.** A WAL reader uses a snapshot: `backup()` completed in ~0.00 s even under a concurrent `BEGIN EXCLUSIVE`/`BEGIN IMMEDIATE`. |
| **PR-26** | Then when does it hang? | **When the source is in `journal_mode=DELETE`.** Killed by a 6 s wall clock. This matters because a **brand-new** ledger file is in DELETE mode until `:153` runs — so the hang is reachable in exactly the window F-1 is about. |

**Two conclusions that change the document, stated plainly:**

1. **`sleep=` and `connect(timeout=)` cannot bound a backup — now measured, not argued** (PR-9b, PR-16, PR-22).
   v1.3 asserted this from a signature alone; v1.4 demonstrates it.
2. **A deadline *is* achievable with the standard API** — `pages=1` + `progress=` + our own exception (PR-11). No
   watchdog, no thread, no process, no new dependency, no non-existent API. This is a **design**, not an
   implementation: **status = DESIGNED** on the ladder of §1.1. It is **not** implemented and **not** tested
   against project code.

### 2.10 Evidence reconciliation with provenance (C-10)

Three independent sources report the same suite at slightly different values. **None is silently preferred.**

| Source | Passed | Skipped | Warnings | Duration | Provenance class |
| --- | --- | --- | --- | --- | --- |
| `GOLDBOT_HANDOVER_2026-11-25.md` §2 row 13 and §7.2 ("from the extracted ZIP") | 1224 | 6 | **not stated** | **18.41 s** | **Reported by the author of that round** |
| `REPORTS/TEST_LOGS/FINAL_extracted_copy.log` (primary artefact, same run category) | 1224 | 6 | **3** | **18.31 s** | **Shipped raw log** |
| `REPORTS/TEST_LOGS/FINAL_full_suite.log` | 1224 | 6 | 3 | 16.55 s | Shipped raw log |
| `REPORTS/TEST_LOGS/BASELINE_full_suite.log` | 1142 | 6 | 3 | 18.62 s | Shipped raw log |
| `REPORTS/REPRODUCTION.md` (line 45) | 1224 | 6 | 3 | ≈17 s | Shipped expectation |
| Independent audit, 2026-11-26 (10 consecutive runs, E-2) | 1224 | 6 | 3 | 19.31–20.62 s | **Independent measurement** |

**Disclosed discrepancies (stated, not resolved by preference):**

| # | Discrepancy | Assessment |
| --- | --- | --- |
| **D-1** | Handover reports **18.41 s** for the extracted run; the shipped `FINAL_extracted_copy.log` records **18.31 s**. | **Unknown** which is authoritative; the log is the closer-to-primary artefact, but the 0.10 s delta is immaterial to any claim in this document. Both are recorded. |
| **D-2** | Handover states the extracted run as `1224 passed, 6 skipped` with **no warning count**; the shipped log records **3 warnings**; `REPRODUCTION.md` expects **3**. | The two shipped artefacts agree. The handover is **incomplete**, not contradictory. Treat **3 warnings** as the evidenced value for that run. |
| **D-3** | Handover §7.1 records `pydantic 2.14.3`; the independent audit measured `pydantic 2.14.4`. | Environment drift between rounds; neither run is invalidated. Recorded as **Unknown** for exact reproducibility. |
| **D-4** | Durations differ across every source (16.55 s … 20.62 s). | Expected: different hosts and loads. **No duration claim is made in this document.** |

**Consequence for this specification:** no acceptance criterion may rest on a duration or a warning count.
Acceptance rests on **pass/fail counts** and on the **re-executed** verification plan (§11).

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

### 3.2 Defect inventory

| ID | Defect | Location | Consequence |
| --- | --- | --- | --- |
| **D-1** | WAL transition attempted exactly once, no retry | A:160, B:153, C:33 | Any momentary lock → constructor failure |
| **D-2** | `busy_timeout` set **after** the transition | A:165, B:155 | Cannot help the statement that needs it (reordering does not help — §2.3) |
| **D-3′** | **Declared** lock-wait policy missing in the state store; the effective value is an **invisible inherited** 10 s from `connect(timeout=10.0)` | C (absent) | Policy is unowned and undiscoverable at the call site. *(Corrected from v1.1's "no busy_timeout at all" — N-5.)* |
| **D-4** | Resulting journal mode never read back | A, B, C | A silent fallback to `delete` goes unnoticed |
| **D-5** | No `try/except`; connection not closed on failure | A:158–169, B:147–160, C:31–37 | Leaked handle per failed construction; on Windows a leaked handle can block deletion |
| **D-6** | Operator sees a bare `sqlite3.OperationalError`/`DatabaseError` | A, B, C | No path, attempt count, elapsed time or guidance |
| **D-7** | Three independent copies of one policy | A, B, C | Already diverged |
| **D-8** | Ledger's shared-connection branch inherits the journal's policy **and `migrate()` currently re-sets `busy_timeout` on it** | B:141–146, `:160` → `:192` | The ledger **mutates a connection it does not own**. Corrected by C-01: the ledger stops touching the policy. |
| **D-9** | `migrate()` re-sets `busy_timeout=10000` unconditionally | B:192 | Duplicate ownership; the second owner is the defect, not the value |
| **D-10** | `migrate()` swallows **any** `OperationalError` on `BEGIN IMMEDIATE` | `:210–213` | A genuine BUSY is misread as "already inside a transaction"; DDL then proceeds unlocked (**N-2**) |
| **D-11** | No deadline anywhere in initialization | A, B, C | An operator cannot distinguish "slow" from "stuck" |
| **D-12** | Migration-path connections have no explicit timeout | `:307`, `:313` | They silently inherit the 5 s stdlib default — an unowned policy in the path that can stall startup (**N-3**) |
| **D-13** | `_backup()`'s call at `:309` passes no `sleep`/`pages`, so `Connection.backup()` runs **its own** BUSY retry loop at its default `sleep=0.25` with **no total bound from this codebase** | `:309`, P-7, PR-4 | Total backup duration is unbounded by design and by accident. **Recorded, not fixed** (C-03, RQ-BACKUP-DEADLINE). |

### 3.3 The initializer, the report structure, and the budget constant (C-06)

```python
WAL_INIT_BUDGET_S: Final[float] = 30.0      # module constant — the ONLY place 30 exists

@dataclass(frozen=True)
class InitDeadline:                         # monotonic, created exactly once per initialization
    started: float                          # time.monotonic()
    budget_s: float                         # copied from WAL_INIT_BUDGET_S (or a patched test value)
    def remaining(self) -> float: ...
    def remaining_ms(self, *, floor_ms: int = 1) -> int: ...
    def expired(self) -> bool: ...

@dataclass(frozen=True)
class Attempt:
    index: int
    remaining_ms_before: int
    busy_timeout_ms_used: int
    delay_ms_planned: int
    delay_ms_actual: int
    elapsed_ms: int
    error_code: int | None
    error_name: str | None

@dataclass(frozen=True)
class InitReport:
    path: str
    attempts: tuple[Attempt, ...]
    elapsed_s: float
    journal_mode_after: str
    created: bool

def open_sqlite_connection(
    path: str | Path,
    *,
    row_factory: bool = True,
    check_same_thread: bool = True,
) -> tuple[sqlite3.Connection, InitReport]:
    ...
```

**Public API carries no budget parameter (C-06).** There is therefore **no production call site that can pass a
different value**, and the API cannot even express "this adapter uses 10 s". The 30 s value lives in exactly one
place, `WAL_INIT_BUDGET_S`.

**Test override mechanism (C-06):** tests shorten the budget by patching the **constant**, not by calling a
parameter:

```python
monkeypatch.setattr(sqlite_init, "WAL_INIT_BUDGET_S", 1.0)   # test-only, stdlib, no API surface
```

`InitDeadline` snapshots the constant at entry, so a patch applied before the constructor call takes effect and one
applied later cannot retroactively change a running deadline. **`delay_ms_planned` and `delay_ms_actual` are
separate fields** so a backoff that is clamped or skipped is visible rather than inferred (C-07).

### 3.4 Deadline semantics — the WAL budget (OD-1, OD-2, P-1)

#### 3.4.1 Exact scope of the 30 s budget

| Inside the budget (OD-1, OD-2) | Outside the budget |
| --- | --- |
| 1. Opening the connection | Schema creation (`executescript(SCHEMA)`, `:166` / `:36`) — **OD-4** |
| 2. `PRAGMA journal_mode=WAL` | Ledger migration: DDL, version row, commit/rollback (`:181–236`) — **OD-4** |
| 3. Retry and backoff for recoverable errors | `_backup()` and its two connections (`:298–325`) — **OD-4** |
| 4. Re-read and confirmation that `journal_mode == 'wal'` | Runtime order-path behaviour |
| 5. Required helper PRAGMAs and handover of a ready connection | **`path.parent.mkdir` — P-1 below** |

**Proposal P-1 (NOT an Owner Decision — C-05):** `path.parent.mkdir(parents=True, exist_ok=True)` executes
**before** the deadline token is created and is therefore **not** bounded by OD-1.
**Why this is a Proposal and not a Decision:** the owner confirmed the WAL budget scope (OD-2) and the
Schema/Migration exclusion (OD-4), but **did not separately confirm** where `mkdir` sits.
**Consequence:** an NFS-stalled `mkdir` would not be bounded by the 30 s budget. This is the **visible boundary**
the owner must accept, reject or re-scope. **Q-12 is OPEN and blocks Gate-1**; no approval is inferred.

#### 3.4.2 Retry classification (C-02 correction)

```
primary(code) = code & 0xFF                 # 517 → 5, 261 → 5, 773 → 5, 262 → 6
RETRYABLE_PRIMARY = {5}                     # SQLITE_BUSY family ONLY
```

| Raw | Name | Primary | Decision | Why |
| --- | --- | --- | --- | --- |
| 5 | `SQLITE_BUSY` | 5 | **Retry** | Cross-process file contention is exactly F-1; waiting resolves it |
| 261 | `SQLITE_BUSY_RECOVERY` | 5 | **Retry** | Restore/recovery in progress; transient by definition |
| 517 | `SQLITE_BUSY_SNAPSHOT` | 5 | **Retry**, with re-open note | Deferred-snapshot upgrade collided with a WAL change. This codebase holds no deferred read transaction across a write; if that ever changes, re-examine. |
| 773 | `SQLITE_BUSY_TIMEOUT` | 5 | **Retry** | Busy handler expired; same handling as 5. The **budget** bounds the loop. |
| **6** | **`SQLITE_LOCKED`** | 6 | **NOT retryable → fail fast** | SQLite raises it for a lock held **inside the same connection/process**, principally under shared cache. **P-2 proves shared cache is never enabled**, so cross-process contention cannot produce 6. Retrying would burn the full 30 s on a condition that cannot clear. |
| **262** | **`SQLITE_LOCKED_SHAREDCACHE`** | 6 | **NOT retryable → fail fast** | Requires shared cache, which is not used (P-2) |
| 26 | `SQLITE_NOTADB` | 26 | **NOT retryable** | Corrupt/foreign file (PR-2) |
| 11, 14, 8, 3 | `CORRUPT`, `CANTOPEN`, `READONLY`, `PERM` | — | **NOT retryable** | Waiting cannot change them |

**Text fallback — corrected (C-02).** Used **only** when `sqlite_errorcode` is absent. The patterns are matched
**in this order**, and the order is load-bearing:

| # | Message pattern (case-insensitive) | Classification | Rationale |
| --- | --- | --- | --- |
| 0 | `database table is locked` | **LOCKED → fail fast** (`SqliteUnretryableError`) | Matched **before** the BUSY pattern so a LOCKED message can never be read as generic BUSY. |
| 1 | `database is locked` | **BUSY → retry** | The only text that may enter the retry loop without a code. |
| 2 | anything else, or no message | **Fail closed** (`SqliteUnretryableError`) | An unrecognised condition must not consume a 30 s budget on speculation. |

**Fact supporting the ordering:** the literal `database is locked` is **not** a substring of
`database table is locked` (the word `table` intervenes), so a naive `in` test would not have conflated them —
but the explicit ordering makes the intent unambiguous and survives future message rewording.

**Re-open condition:** if any path ever enables shared cache (URI `cache=shared`, `PRAGMA cache_shared` or
equivalent), the `6 → NOT_RETRYABLE` decision **must be revisited** and this table re-issued (**Q-10**).

**Fast-fail requirement:** every non-retryable code surfaces with `attempts == 1` and `elapsed < 2 s`.

#### 3.4.3 The budget is a constant, not a knob (C-06 / OD-1)

OD-1 requires one value for Live, CLI, Web, Report and administrative callers. This is enforced **structurally**:
`open_sqlite_connection()` has no budget parameter, and `WAL_INIT_BUDGET_S` exists once. A reviewer can verify OD-1
compliance with a single `grep` for `WAL_INIT_BUDGET_S` and for any literal `30` in the adapters (expected: one
definition, zero literal duplications).

#### 3.4.4 Backoff schedule

```
delay(n) = min(250 ms, 10 ms · 2^(n−1)) · uniform(0.8, 1.2)      n = 1, 2, 3, …
```

* The contended party holds the lock only for its own WAL transition — sub-100 ms in every successful contention
  case observed by E-2. A 10 ms first step means the uncontended path never sleeps.
* The 250 ms cap bounds wake-ups: at the cap a full 30 s budget costs ≤ ~120 wake-ups.
* ±20 % jitter prevents lock-step retries — the thundering-herd pattern that matters precisely because both
  processes start microseconds apart (E-1's harness).

#### 3.4.5 Budget accounting that survives OS scheduling (C-07)

The requirement was restated because v1.2 demanded `Σ delay_ms_actual ≤ budget`, which §3.9.2 itself contradicts.

| Rule | Statement |
| --- | --- |
| **R-a** | Before sleeping, compute `remaining`. **If `planned_delay > remaining`, do not sleep** — fail closed with `SqliteBusyTimeout`. |
| **R-b** | After **every** sleep, **recompute** the monotonic remaining budget. |
| **R-c** | **No new retry may begin after the deadline has expired** (`remaining <= 0` ⇒ fail closed before the attempt). |
| **R-d** | `delay_ms_actual` is **measured** (`time.monotonic()` deltas) and **reported** in `InitReport`. |
| **R-e** | **No claim is made** that `delay_ms_actual ≤ delay_ms_planned`. The OS may deschedule the process arbitrarily; measuring is how we *observe* it, not how we bound it. |
| **R-f** | The `budget + 2 s` acceptance form is confined to **T-F1-16** under its documented test conditions (§3.9.3). No other test or report may use it. |

**Inference:** R-a…R-c bound the *decision* to retry; R-d makes any scheduler overshoot *visible* in the report;
R-e prevents the document from claiming a bound it cannot honour. This is the only self-consistent set.

#### 3.4.6 C-12 — the executable WAL initialization algorithm

v1.3 specified the data structures and the rules but not the **procedure**. Below is the procedure, written to be
transcribed into Python without further design decisions. Every numbered line **P-1 … P-13** is referenced by the
traceability matrix.

**Design decisions embedded here, each with its reason:**

| Decision | Choice | Reason |
| --- | --- | --- |
| Rebuild or re-arm the connection per attempt? | **Re-arm the one connection** | PR-7 proves `PRAGMA busy_timeout=N` displaces an already-installed handler (30 000 → 50 ms, measured 51 ms). Rebuilding would discard the file handle, re-pay the open cost against the same budget, and multiply the close paths that can leak (T-F1-08). One connection = one close path. |
| How is per-attempt `busy_timeout` chosen? | `floor(remaining_ms)` capped at 10 000 | The attempt must never outlive the deadline (§3.4.5 R-a) and must never idle longer than the steady-state policy. |
| What if `remaining_ms` is below a usable value? | fail closed **before** opening the attempt | R-c: no retry starts after expiry, and a sub-millisecond wait cannot succeed. |
| Where does the deadline token start? | **before** the connection is opened, **after** `mkdir` | OD-2 covers connection opening; `mkdir` is **P-1/Q-12 — still open**. A-10 records that if the owner brings `mkdir` inside, P-1 moves and the token is created earlier. |
| Final `busy_timeout` | **10 000 ms, set after read-back verification** | Same value every creator-owned connection carries (§3.6.3); setting it after verification avoids the *appearance* that the loop's last attempt value is the policy. |

```python
# ---- P-1: ONE deadline token, created before anything is opened -------------
# WAL_INIT_BUDGET_S is the module constant; there is NO budget parameter (C-06).
deadline = InitDeadline(started=time.monotonic(), budget_s=WAL_INIT_BUDGET_S)
attempts: list[Attempt] = []
conn: sqlite3.Connection | None = None

try:
    # ---- P-2: fail closed before opening if the budget is already gone ------
    if deadline.expired():
        raise SqliteBusyTimeout(path, attempts, deadline)      # no connection to close

    while True:
        remaining_ms = deadline.remaining_ms()                 # P-3: recomputed EVERY pass
        if remaining_ms <= 0:                                  # P-4: no retry after expiry (R-c)
            raise SqliteBusyTimeout(path, attempts, deadline) from last_exc

        # ---- P-5: per-attempt wait, in MILLISECONDS, never exceeding remaining
        attempt_timeout_ms = min(remaining_ms, MAX_BUSY_TIMEOUT_MS)   # MAX = 10_000

        if conn is None:
            # ---- P-6: connect() takes SECONDS (C-17). Ratio keeps the two consistent.
            conn = sqlite3.connect(
                str(path),
                timeout=attempt_timeout_ms / 1000.0,           # SECONDS, e.g. 10_000 ms -> 10.0
                check_same_thread=check_same_thread,
            )
            if row_factory:
                conn.row_factory = sqlite3.Row
        # ---- P-7: else KEEP the connection and RE-ARM it (PR-7 proves this is real) ----
        conn.execute(f"PRAGMA busy_timeout={attempt_timeout_ms}")   # MILLISECONDS

        t_attempt = time.monotonic()
        try:
            # ---- P-8: the WAL transition itself ----------------------------
            conn.execute("PRAGMA journal_mode=WAL")

            # ---- P-9: read-back verification, INSIDE the budget (OD-2) -----
            mode = conn.execute("PRAGMA journal_mode").fetchone()[0]
            if str(mode).lower() != "wal":
                raise SqliteInitializationError(
                    f"SQLite initialization failed for database {path}: "
                    f"journal_mode is {mode!r}, expected 'wal'"
                )          # NOT retryable: a accepted-but-not-applied pragma will not change

            # ---- P-10: required helper PRAGMAs -----------------------------
            conn.execute("PRAGMA synchronous=FULL")

            # ---- P-11: final lock-wait policy, set only after verification --
            conn.execute("PRAGMA busy_timeout=10000")          # MILLISECONDS, steady state

            # ---- P-12: OPTIONAL auto-checkpoint policy — see note (a) ------
            # conn.execute("PRAGMA wal_autocheckpoint=...")  # NOT SET HERE; see §3.4.6 note (a)

            break                                               # success -> leave the loop
        except sqlite3.Error as exc:
            last_exc = exc
            verdict = classify(exc)          # §3.4.2 — codes FIRST, text ONLY if no code

            if verdict is not Classify.RETRY:                   # ---- non-retryable (includes LOCKED) --
                raise _typed_for(verdict, exc, path, attempts, deadline) from exc

            # ---- P-13: FAIL-CLOSED PATH — one close, one re-raise ----------
            delay_ms = backoff_plan(len(attempts))              # §3.4.4
            remaining_ms = deadline.remaining_ms()              # R-b: recompute AFTER the failure
            if delay_ms >= remaining_ms:                        # R-a: never sleep past the deadline
                raise SqliteBusyTimeout(path, attempts, deadline) from exc
            t0 = time.monotonic()
            time.sleep(delay_ms / 1000.0)                       # SECONDS
            actual_ms = int((time.monotonic() - t0) * 1000)     # R-d: MEASURED, not assumed
            attempts.append(Attempt(...))                       # records planned AND actual
            # loop continues; P-3 recomputes remaining at the top

    # ---- P-14: handover must be INSIDE the budget --------------------------
    if deadline.expired():
        # We got a verified WAL connection, but too late: a late success is a failure to the
        # caller's contract, and silently returning it would hide a budget breach.
        raise SqliteBusyTimeout(path, attempts, deadline)
    return conn, InitReport(...)

except BaseException:
    # ---- P-15: EVERY failure path closes the connection it opened ---------
    if conn is not None:
        try:
            conn.close()
        except sqlite3.Error:
            pass                     # never mask the original failure with a close error
    raise
```

**Notes that are part of the design, not commentary:**

**(a) No `wal_autocheckpoint` statement is introduced.** An `Attempt`-produced checkpoint can block on a reader
and would be an unbounded wait *inside* the 30 s budget. The default (1 000 pages) is left untouched. This is a
**deliberate non-change**, recorded so a reviewer does not read its absence as an oversight.

**(b) P-14 is a real requirement, not decoration.** v1.3 said "handover before expiry" but did not say what
happens when verification succeeds *after* expiry. Returning the connection would be the tempting, wrong choice:
the caller asked for a connection within 30 s. **A late success is a failure.**

**(c) `last_exc` is always bound when used.** `SqliteBusyTimeout(...) from last_exc` appears only after at least
one `except sqlite3.Error` has bound it, or after the loop exhausted a retry — never on the first pass.

**(d) Timeout units are stated at the call site, in the code, as comments.** `connect(timeout=)` is **seconds**;
`PRAGMA busy_timeout` is **milliseconds**; `time.sleep()` is **seconds**; the deadline is **monotonic seconds**
internally and **integer milliseconds** at every report boundary. PR-20 shows what a mixed unit costs.

**(e) What this algorithm guarantees — and what it does not.** It is **algorithmically hard-bounded**: no attempt
is scheduled with a wait longer than the remaining budget, no retry begins after expiry, and the loop terminates.
It is **not** a wall-clock guarantee: `time.sleep()` may oversleep, the OS may deschedule the process, and a
filesystem stall can block inside any `execute()` without ever reaching the busy handler (§3.9.2). **The deadline
is hard-bounded in the algorithm and best-effort in wall-clock time.** No other claim is made.

#### 3.4.7 C-17 — timeout units, normative

Units at every boundary in this document. **The rule is a suffix, not a comment:** every internal variable, field
and message carries `_ms` or `_s` in its name, so a unit error is a **typo**, not a silent factor of 1 000.

| API / surface | Unit | Correct | Incorrect |
| --- | --- | --- | --- |
| `sqlite3.connect(timeout=...)` | **seconds** (float) | `connect(path, timeout=10.0)`, `timeout=attempt_ms / 1000.0` | `timeout=10000` — **silently 10 000 s** (PR-20) |
| `PRAGMA busy_timeout=N` | **milliseconds** (integer) | `PRAGMA busy_timeout=10000` | `PRAGMA busy_timeout=10.0` |
| `Connection.backup(sleep=...)` | **seconds** (float) | `backup(dst, sleep=0.25)` | `sleep=250` — **250 s between attempts** (PR-20) |
| `Connection.backup(pages=...)` | **pages** (integer), `-1` = all in one step | `backup(dst, pages=1, ...)` — required for interruptibility (PR-17) | `pages=-1` when a deadline is needed |
| `InitDeadline.budget_s` / `.remaining()` | **seconds** (monotonic) | `remaining()` → `float` seconds | passing seconds where ms is expected |
| `InitDeadline.remaining_ms()` | **milliseconds** (int) | `remaining_ms()` → `10000` | feeding it to `connect(timeout=)` |
| `Attempt.remaining_ms_before`, `busy_timeout_ms_used`, `delay_ms_planned`, `delay_ms_actual`, `elapsed_ms` | **milliseconds** | field names carry `_ms` | a field named `delay` with no unit |
| `InitReport.elapsed_s` | **seconds** | `elapsed_s` | `elapsed` |
| `time.sleep(x)` | **seconds** | `time.sleep(delay_ms / 1000.0)` | `time.sleep(delay_ms)` |
| `time.monotonic()` | **seconds**, opaque origin | differences only | comparing to `time.time()` |
| Messages (§3.5.3) | mixed, **labelled in the text** | `(budget 30.0 s)`, `(lock wait 10000 ms per statement)` | a bare number |

**Runtime evidence for this table:** PR-20 measured `connect(timeout=5000)` installing
`busy_timeout = 5 000 000` ms and stated the failure mode plainly — **the wrong unit is accepted without
complaint**. There is no exception to catch, so only naming discipline prevents it.

**Review checklist (V-item V-13):** no call site passes a literal to `connect(timeout=)`; no `PRAGMA busy_timeout`
receives a non-integer; every `_ms`/`_s` field is used only with a matching API; the divisor `1000.0` appears
wherever the two families meet.


### 3.5 Exception contract (C-04)

#### 3.5.1 Taxonomy — with the WAL and migration classes kept distinct

```
GoldbotError (domain/errors.py:22)
├── SqliteInitializationError                 # WAL-side base
│   ├── SqliteBusyTimeout                     # BUSY persisted past the 30 s WAL budget (OD-1)
│   ├── SqliteNotADatabase                    # NOTADB (26)
│   └── SqliteUnretryableError                # other non-retryable, incl. 6 / 262
├── SqliteMigrationBusyTimeout                # migration BUSY wait exhausted (retryable class)
├── SqliteMigrationLockError                  # migration LOCKED (6 / 262) — fail-fast, NEVER a timeout
└── SqliteBackupTimeout                       # backup deadline expired (§3.10) — value set by OD-6

LedgerUnavailable                             # ports/ledger.py:77 — unchanged public contract
```

**Why three separate migration/backup classes (C-04 + C-13 + C-11):** a 30 s WAL failure, a 10 s migration
lock-wait failure, a `LOCKED` fail-fast and a backup-deadline expiry are **four different operational events** with
different causes and different remediation. Giving any two of them one type makes a wrong-subsystem report not
merely possible but likely. **They are never interchangeable**, and no code path may raise one class for another
class's condition.

**C-13 explicitly — the v1.3 contradiction and its correction.** v1.3 §3.5 declared `SQLITE_LOCKED` **non-retryable**
and then, in Sequence A step A14, routed *"a BUSY/LOCKED on any statement in A8–A13"* into the single
`SqliteMigrationBusyTimeout`. That is a contradiction: a fail-fast condition cannot also be a timeout. v1.4 splits
them:

| Condition | Retried? | Type | Message must say |
| --- | --- | --- | --- |
| **`SQLITE_BUSY`** (5, 261, 517, 773) during migration | **Yes**, inside the migration deadline, with backoff | `SqliteMigrationBusyTimeout` on exhaustion | **"lock wait exhausted … after N attempts"** |
| **`SQLITE_LOCKED`** (6, 262) during migration | **No** — never enters the retry loop | `SqliteMigrationLockError` | **"locked"** — must **NOT** contain "timeout" or "exhausted" |

**Why LOCKED must not be a timeout (the design reason, stated once):** a timeout says *"we waited as long as we
agreed to."* A LOCKED condition will **not** clear no matter how long we wait — PR-5a shows it arises from an
unfinished cursor on the same connection. Reporting it as a timeout sends the operator to look for a competing
process that does not exist. The message must say `locked`, and the type must say `LockError`.

**A fact v1.3 got wrong, corrected here.** v1.3 justified `6 ⇒ non-retryable` with *"P-2 proves shared cache is
never enabled, so cross-process contention cannot produce 6"*, and implied 6 was unreachable. **PR-5a proves
`SQLITE_LOCKED` is reachable without shared cache** (unexhausted read cursor → `DROP TABLE`, code 6, message
`database table is locked`), and **PR-23 proves this codebase cannot currently trigger it** (every cursor is
materialised before DDL). Net: **the rule is unchanged, the justification is corrected** from "impossible" to
"reachable only by a programming error — and fail-fast is exactly right for a programming error."

#### 3.5.2 The one exception chain for migration (C-04, C-13)

```
LedgerUnavailable                     ← what the caller of the Ledger sees (public contract unchanged)
  └── __cause__: SqliteMigrationBusyTimeout   OR   SqliteMigrationLockError
        └── __cause__: sqlite3.OperationalError / sqlite3.DatabaseError   (the original)
```

Rules:
1. `migrate()` raises `SqliteMigrationBusyTimeout(...) from <original>` when the BUSY retry loop is exhausted.
2. `migrate()` raises `SqliteMigrationLockError(...) from <original>` **immediately** on LOCKED — with **no**
   backoff sleep and **no** retry attempt.
3. The **Ledger public boundary** converts either: `raise LedgerUnavailable(...) from exc`.
4. **Both** levels of `__cause__` must be present and asserted by T-F1-18 / T-F1-24.
5. `SqliteBusyTimeout` (the WAL class) **may never** be raised for a migration failure, and the ledger's
   `LedgerUnavailable` for a migration failure **may never** wrap the WAL class.
6. **Neither** migration class may wrap the other.

#### 3.5.2.1 Complete exception mapping table (C-13 requirement)

| # | Condition | Origin | Inner type | Public surface (Ledger) | Public surface (Journal / State) | Retried? | `attempts` | Message token |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| M-1 | BUSY family persists past the migration deadline | `migrate()` | `SqliteMigrationBusyTimeout` | `LedgerUnavailable` | n/a | yes | ≥ 2 | "lock wait exhausted", "database" |
| M-2 | **LOCKED (6/262) on any migration statement** | `migrate()` | **`SqliteMigrationLockError`** | `LedgerUnavailable` | n/a | **no** | **1** | **"locked"**, "database" — **never "timeout"** |
| M-3 | Ambiguous sqlite error, no reliable code, during migration | `migrate()` | `SqliteMigrationLockError` (fail closed) | `LedgerUnavailable` | n/a | **no** | 1 | "locked" |
| M-4 | NOTADB (26) during migration | `migrate()` | `SqliteNotADatabase` | `LedgerUnavailable` | n/a | no | 1 | "database" |
| M-5 | Any other `sqlite3.Error` (permission, I/O, corrupt) | `migrate()` | original, re-raised | `LedgerUnavailable` (unchanged `:234–236`) | n/a | no | 1 | original text |
| W-1 | BUSY past the 30 s WAL budget | helper | `SqliteBusyTimeout` | `LedgerUnavailable` | as-is | yes | ≥ 1 | "initialization", "budget", "database" |
| W-2 | LOCKED / ambiguous during WAL init | helper | `SqliteUnretryableError` | `LedgerUnavailable` | as-is | no | 1 | "database" |
| W-3 | `journal_mode` read back ≠ `wal` | helper | `SqliteInitializationError` | `LedgerUnavailable` | as-is | no | 1 | "database" |
| W-4 | NOTADB during WAL init | helper | `SqliteNotADatabase` | `LedgerUnavailable` | as-is | no | 1 | "database" |
| B-1 | **Backup deadline expired (§3.10)** | `_backup()` | **`SqliteBackupTimeout`** | `LedgerUnavailable` | n/a | **no** (deadline already spent) | — | **"backup"**, "deadline", "database" |

**Enforcement rule (gate condition):** the mapping is a **table in code** — one `dict` from verdict → exception
type — not a chain of `if isinstance`. A reviewer diffs the table against this one.

#### 3.5.3 Messages


```
SQLite initialization failed for database {path}: {reason} after {attempts} attempt(s)
in {elapsed:.2f}s (budget {budget:.1f}s)                       # WAL side

SQLite migration lock wait exhausted for database {path}: {reason}
after {elapsed:.2f}s (lock wait {busy_timeout_ms} ms per statement)   # migration side — BUSY only

SQLite migration failed for database {path}: database is locked by this connection
({sqlite_errorname}) — no retry attempted                        # migration side — LOCKED (C-13)

SQLite backup did not finish for database {path} within {budget_used:.2f}s
(backup deadline {backup_budget_s:.2f}s, aborted after {pages_copied} page step(s))   # backup (§3.10)
```

The literal token **"database"** is mandatory in every message (AU-35 protection). The LOCKED message must contain
**"locked"** and must **not** contain "timeout" or "exhausted" (M-2). The backup message must contain **"backup"**
so an operator does not confuse it with a WAL-initialization failure (B-1).

#### 3.5.4 AU-35 is preserved by construction

`tests/test_phase_a_audit_fsm_and_live.py:251`:

```python
assert "database" in repr(excinfo.value).lower() or "ledger" in repr(excinfo.value).lower()
```

Today it passes **accidentally** (the corrupt message is `file is not a database`, PR-2). The templates above make
the token deliberate. **AU-35 must not be edited, weakened or skipped**; T-F1-10 re-runs it unmodified.

#### 3.5.5 Compatibility inventory of catch sites (gate condition)

| File:line | Handler | Effect of the new taxonomy |
| --- | --- | --- |
| `sqlite_operation_ledger.py:193` | `except sqlite3.Error: pass` around a `busy_timeout` PRAGMA | **Removed with `:192`** (C-01) |
| `:210` | `except sqlite3.OperationalError: pass` around `BEGIN IMMEDIATE` | **Replaced** by the `in_transaction` pre-check (F1-R17) |
| `:341, :389, :399, :416, :434, :500, :519, :558, :590, :615, :628` | `except sqlite3.Error` around writes | Unaffected — they wrap `self._conn.execute`, not a constructor |
| `order_coordinator.py:244` | `except LedgerUnavailable` | **Preserved** — the ledger still raises `LedgerUnavailable` |
| `web/app.py:203, 211, 516, …` | `except Exception` | **Preserved** — every new type derives from `Exception` |

**Gate-1 condition:** a one-page inventory of **every** `except` reachable from a journal/ledger/state constructor,
stating for each whether the new type is caught and what the user sees. **A-4** is discharged only by that review.

| Failure | Raised by | Adapter surface | `attempts` | `elapsed` | Cause chain |
| --- | --- | --- | --- | --- | --- |
| BUSY family past the WAL budget | `SqliteBusyTimeout` | journal/state as-is; ledger → `LedgerUnavailable` | ≥1 | ≈ budget | `from last_exc` |
| Corrupt file | `SqliteNotADatabase` | as above | **1** | **< 2 s** | `from DatabaseError` |
| `LOCKED` (6 / 262), incl. via text fallback | `SqliteUnretryableError` | as above | **1** | **< 2 s** | `from exc` |
| Ambiguous text, no code | `SqliteUnretryableError` | as above | **1** | **< 2 s** | `from exc` |
| WAL not applied | `SqliteInitializationError` | as above | 1 | small | cause of the PRAGMA, or `from None` |
| **Migration lock wait exhausted** | **`SqliteMigrationBusyTimeout`** | **`LedgerUnavailable`** | — | per-statement wait | **two-level chain, §3.5.2** |

### 3.6 Per-adapter wiring

#### 3.6.1 Frozen values

| Adapter | Change | **Frozen — must not change** |
| --- | --- | --- |
| `SqliteJournal` (`sqlite_store.py:153–169`) | replace 158–165 with `open_sqlite_connection(self.path, row_factory=True, check_same_thread=False)`; keep schema + migrations + commit | `SCHEMA`; migration semantics; **`check_same_thread=False`** |
| `SqliteOperationLedger` path branch (`:147–160`) | replace 151–155 with the helper (**`check_same_thread` omitted → `True`**); keep `_owns_connection=True`; keep `migrate()` | shared-journal branch; `migrate()`'s `BEGIN IMMEDIATE` + version re-check; **`True`** |
| `SqliteRiskStateStore` (`:27–37`) | replace 31–36 with the helper (**`True`**) — this **declares** what today is merely inherited (D-3′) | `SCHEMA`; `key` semantics; **`True`** |

**Why `check_same_thread` matters:** `False` (journal) allows cross-thread use; `True` (ledger, state) raises
`sqlite3.ProgrammingError` on use from another thread. Both frozen; T-F1-15 asserts the **observable** behaviour —
which is also why §4.3 forbids moving a connection across threads in tests.

#### 3.6.2 Shared-journal branch (C-01 item 5)

`SqliteOperationLedger.__init__` with `journal is not None` (`:141–146`):
* **does not call the helper** (it does not own the connection);
* **does not set, change or reset the borrowed connection's `busy_timeout`** — including inside `migrate()`
  (this removes today's mutation at `:192`, D-8);
* runs `migrate()` using whatever policy the journal already installed (`sqlite_store.py:165` = 10 000 ms).

#### 3.6.3 C-01 — Migration lock-wait ownership: one unambiguous sequence

**Withdrawn from v1.2:** §3.6.3 previously offered "re-arm after the helper **or** delete `:192` **or** keep `:192`
as the owner". That is replaced by the single rule below.

> **THE RULE — one owner per connection.**
> **The party that creates a connection owns its lock-wait policy. No other party may set, re-arm or reset it.**
> The helper is therefore the owner for every connection it returns; `SqliteJournal.__init__` is the owner for the
> connection it borrows out; and **`migrate()` owns nothing** — it neither arms nor re-arms.

**Consequences, stated once:**

| Connection | Creator = owner | Value | Where set |
| --- | --- | --- | --- |
| Ledger path-branch connection | `open_sqlite_connection()` | **10 000 ms** | helper step 9 |
| Journal connection | `open_sqlite_connection()` | **10 000 ms** | helper step 9 |
| State-store connection | `open_sqlite_connection()` | **10 000 ms** | helper step 9 |
| Ledger **borrowed** (shared-journal) connection | `SqliteJournal.__init__` | **10 000 ms** | `sqlite_store.py:165` (unchanged) |

**`migrate():192` and its `try/except sqlite3.Error: pass` (`:193–194`) are DELETED.** They were the second owner.
Deleting them changes **no effective value** for any path — every connection already carries 10 000 ms when
`migrate()` is reached — but it removes the ability of the ledger to mutate a connection it does not own (D-8), and
removes a silent `except` that could hide a real error.

---

**Sequence A — `SqliteOperationLedger.__init__`, path branch** (`path` given, `journal is None`)

| Step | Action | lock-wait in force | Inside the 30 s budget? |
| --- | --- | --- | --- |
| A1 | `self.path.parent.mkdir(parents=True, exist_ok=True)` | n/a | **No — P-1 (pending)** |
| A2 | `open_sqlite_connection(self.path)` → **opens connection**, WAL retry loop, **verify `wal`**, `synchronous=FULL`, **`busy_timeout=10000`** | busy 10 000 after A2 | **Yes — OD-2** |
| A3 | `self._owns_connection = True` | — | — |
| A4 | `self.migrate()` begins — **no policy statement** (the v1.1/v1.2 `:192` is gone) | 10 000 (from A2) | **No — OD-4** |
| A5 | `self._version()` — table-existence read | 10 000 | No |
| A6 | **steady state:** `version >= LEDGER_SCHEMA_VERSION` and table present → `_report_for(...)`, `return`. **Backup is not taken, `BEGIN IMMEDIATE` is not issued, no DDL runs.** | 10 000 | No |
| A7 | otherwise, if an `operation_ledger` table already exists (`_has_ledger_table()`) → `self._backup(version)`; `_backup` opens `destination` (`:307`) and `check` (`:313`), **each with an explicit `timeout=`** (F1-R19); `Connection.backup()` runs its **own** retry loop (`sleep=0.25`) — **not bounded by us** (C-03) | 10 000 on the ledger conn; explicit timeout on the two backup conns | No |
| A8 | `if not self._conn.in_transaction:` → `self._conn.execute("BEGIN IMMEDIATE")`; **if the connection was already in a transaction, do nothing and issue no statement** | 10 000 on the `BEGIN IMMEDIATE` | No |
| A9 | **re-check** `_version()` **inside** the transaction (`:215–220`); if another process migrated, `commit()` and return with `changed=False` | 10 000 | No |
| A10 | if `existing_table`: `self._upgrade_columns()` (`ALTER TABLE`, `:275`) — **DDL** | 10 000 | No |
| A11 | `for statement in LEDGER_SCHEMA.split(";"): self._conn.execute(statement)` — **DDL** (`:225–227`) | 10 000 | No |
| A12 | version row upsert (`:228–232`) | 10 000 | No |
| A13 | `self._conn.commit()` (`:233`) | — | No |
| A14 | on any exception: `self._conn.rollback()`; if the failure is a **BUSY** family error on **any** statement in A8–A13 → `SqliteMigrationBusyTimeout` → wrapped `LedgerUnavailable` (§3.5.2); if the failure is **LOCKED (6/262)** or ambiguous → **`SqliteMigrationLockError`, no retry**, wrapped `LedgerUnavailable` (C-13, M-2/M-3); otherwise re-raise with cause (`:234–236` behaviour preserved) | — | No |

**Answering C-01 item 4 explicitly — where `busy_timeout=10000` sits:**

| Statement | Position relative to `busy_timeout=10000` |
| --- | --- |
| **Backup** (A7) | **After** it is in force (set at A2). The two backup connections have **their own** explicit timeouts. |
| **`BEGIN IMMEDIATE`** (A8) | **After** it is in force — this is the whole point: the lock acquisition is covered. |
| **DDL** (A10, A11) | **After** it is in force; every `ALTER TABLE`/`CREATE`/index statement waits up to 10 000 ms. |
| **Commit** (A13) | After; commit itself does not wait for other writers beyond what the transaction already holds. |

---

**Sequence B — `SqliteJournal.__init__`** (`sqlite_store.py`)

| Step | Action | lock-wait | Budget? |
| --- | --- | --- | --- |
| B1 | `path.parent.mkdir(...)` | n/a | **No — P-1** |
| B2 | `open_sqlite_connection(path, row_factory=True, check_same_thread=False)` → open, WAL retry, **verify**, `synchronous=FULL`, **`busy_timeout=10000`** | 10 000 | **Yes — OD-2** |
| B3 | `executescript(SCHEMA)` — note it issues an implicit `COMMIT` first | 10 000 | No — OD-4 |
| B4 | `_migrate_trades()` — `PRAGMA table_info` then `ALTER TABLE` per missing column | 10 000 | No |
| B5 | `_migrate_signals()` — `executescript(SIGNAL_SCHEMA)` | 10 000 | No |
| B6 | `commit()` | — | No |

---

**Sequence C — `SqliteRiskStateStore.__init__`** (`sqlite_state_store.py`)

| Step | Action | lock-wait | Budget? |
| --- | --- | --- | --- |
| C1 | `path.parent.mkdir(...)` | n/a | **No — P-1** |
| C2 | `open_sqlite_connection(path)` (**`check_same_thread=True`**) → open, WAL retry, **verify**, `synchronous=FULL`, **`busy_timeout=10000`** — *this converts today's invisible inherited 10 s into a declared policy, D-3′* | 10 000 | **Yes — OD-2** |
| C3 | `executescript(SCHEMA)` | 10 000 | No — OD-4 |
| C4 | `commit()` | — | No |

---

**Confirmation — C-01 item 5.** The shared-connection Ledger branch **does not invoke the helper** and **does not
change the borrowed connection's policy**. Before this revision, `migrate():192` *did* change it; after this
revision there is no statement anywhere in the ledger that touches `busy_timeout` on any connection. A reviewer can
verify this with `grep -n busy_timeout src/goldbot/adapters/sqlite_operation_ledger.py` → **zero matches expected**
after implementation.

#### 3.6.4 C-03 — Backup: scope and honest limitations

**Facts**

| # | Fact | Basis |
| --- | --- | --- |
| B-1 | `_backup()` opens `destination` (`:307`) and `check` (`:313`) with **no timeout argument**, inheriting the **stdlib default 5.0 s** | P-4, P-7 |
| B-2 | `_backup()` calls `self._conn.backup(destination)` at `:309` with **no `pages`, `progress` or `sleep` argument** | P-7 |
| B-3 | `Connection.backup()` accepts `sleep=0.25` by default and **retries BUSY/LOCKED internally between attempts** | PR-4 |
| B-4 | The **destination** is a brand-new file, so contention there is minimal; the contention that matters is on the **source**, handled by `self._conn`'s own policy plus `backup()`'s internal loop | P-7, PR-4 |

**Inference (why v1.2 was wrong):** an explicit `timeout=` on `destination`/`check` bounds *those connections'*
lock waits. It does **not** bound `Connection.backup()`'s own retry loop, and therefore **cannot** be described as
a total backup deadline. v1.2's F1-R19/T-F1-21 implied otherwise; the implication is removed.

**Phase A scope — adopted (limited approach)**

| # | Requirement |
| --- | --- |
| **F1-R19 (revised)** | Both independent backup connections receive **explicit, documented `timeout=` values** (proposed 5 000 ms — the value they inherit today, made visible). Resource cleanup verified. |
| **F1-R19-b** | **No claim** is made that `timeout=` bounds total backup duration. No report may state or imply a bounded backup. |
| **F1-R19-c** | No watchdog, supervisor or out-of-band terminator is added. |
| **RQ-BACKUP-DEADLINE** | *(Deferred Requirement — recorded, NOT approved, NOT designed.)* A true total-backup deadline would require bounding `Connection.backup()`'s internal retry loop — e.g. via `progress`/`pages` stepping with our own deadline check — or out-of-band termination. **Out of Phase A scope.** Requires separate owner approval. |

**T-F1-21 is correspondingly narrowed:** it verifies **configuration** (explicit `timeout=` on both connections,
value documented) and **cleanup** (both closed on success and on failure; no leaked handle). It **does not** and
**must not** assert a backup duration.

---

### 3.7 Behaviour matrix

| Scenario | Initial state | Expected after remediation | Test |
| --- | --- | --- | --- |
| S1 | absent | WAL verified, schema created, no error | T-F1-01 |
| S2 | exists, rollback mode, with data | transitions to WAL; data unchanged | T-F1-02 |
| S3 | exists, already WAL | no-op; data unchanged | T-F1-03 |
| S4 | absent, 2 processes | both succeed; 0 transient errors over N | T-F1-04, T-F1-05 |
| S5 | absent, 4 processes | all succeed; one file set | T-F1-06 |
| S6 | lock held past budget | `SqliteBusyTimeout` at budget + ≤2 s under §3.9.3 conditions; connection closed | T-F1-07, T-F1-16 |
| S7 | lock released after ~0.3 s | succeeds with `attempts ≥ 2` | T-F1-09 |
| S8 | corrupt file | `SqliteNotADatabase`, `attempts == 1`, `< 2 s`, message contains "database"; AU-35 green | T-F1-10 |
| S9 | any | directory delta classified three ways (C-09) | T-F1-11 |
| S10 | forced `journal_mode` reply `"delete"` | `SqliteInitializationError` | T-F1-12 |
| S11 | ledger with `journal=J` | helper **not** called; borrowed policy **unchanged**; zero `busy_timeout` statements in the ledger | T-F1-13 |
| S12 | each adapter | `check_same_thread` behaviour identical to today | T-F1-15 |
| S13 | migration under a held lock | bounded 10 s wait, then `LedgerUnavailable` → `SqliteMigrationBusyTimeout` → original; **never** the WAL class; **never** silent | T-F1-18 |
| S14 | backup connections | explicit timeout present and documented; cleanup verified; **no duration claim** | T-F1-21 |
| S15 | each retryable / non-retryable code | classification per §3.4.2 | T-F1-19, T-F1-20 |
| S16 | LOCKED text without a code | fail fast, `attempts == 1` | T-F1-22a |
| S17 | ambiguous text without a code | fail closed, `attempts == 1` | T-F1-22b |
| **S18** | **backup blocked, deadline expired (C-11)** | `SqliteBackupTimeout` → `LedgerUnavailable`; source `busy_timeout` restored; partial target deleted | **T-F1-23** |
| **S19** | **migration hits LOCKED (6/262) (C-13)** | `SqliteMigrationLockError`; `attempts == 1`; no timeout wording; wrapped `LedgerUnavailable` | **T-F1-24** |
| **S20** | **runtime text identity (C-16)** | BUSY → code 5 / `SQLITE_BUSY` / `database is locked`; LOCKED → code 6 / `SQLITE_LOCKED` / `database table is locked` | **T-F1-25** |

### 3.8 Requirements trace for §3

| Requirement | Design item | Where |
| --- | --- | --- |
| one shared initializer | §3.3 | §3.3 |
| BUSY vs LOCKED classification, extended codes, **corrected text fallback** | §3.4.2 | §3.4.2 |
| bounded backoff; budget as a true cap over OD-2's five operations | §3.4.4, §3.4.5 | §3.4 |
| budget is one constant; no public override | §3.4.3 (C-06) | §3.4.3 |
| fail closed, typed errors, cause chains, **migration chain** | §3.5 | §3.5 |
| close on constructor failure | §3.3 | T-F1-08 |
| verify WAL | §3.3 | T-F1-01/T-F1-12 |
| **one lock-wait owner per connection** | §3.6.3 (C-01) | §3.6.3 |
| schema/migration independent of the WAL budget | §3.6.3 (OD-4) | §3.6.3 |
| **backup claims limited; no total-duration promise** | §3.6.4 (C-03) | §3.6.4 |
| independent connections have explicit timeouts | F1-R19 | T-F1-21 |
| `mkdir` boundary | **P-1, pending** | §3.4.1 |

### 3.9 What `sqlite3` timeouts do — and do not — guarantee

#### 3.9.1 Controlled

`sqlite3.connect(timeout=n)` and `PRAGMA busy_timeout=n` bound the connection's **busy-handler** wait for a
conflicting SQLite lock (PR-3 measured 10.02 s for `timeout=10.0`). Our retry loop bounds how many times the **WAL
transition statement** is re-attempted, and the monotonic budget bounds that loop.

#### 3.9.2 **Not** controlled

* **File-system stalls** (network share, failing disk, antivirus filter) — execution can block inside the OS
  without ever reaching the busy handler.
* **Scheduler / OS suspension** — a process can be descheduled arbitrarily. This is exactly why C-07 forbids the
  claim `delay_ms_actual ≤ delay_ms_planned`.
* **`mkdir`** — outside the budget under **P-1 (still pending)**.
* **Schema / migration** — outside the budget by **OD-4**; bounded only per statement by `busy_timeout`.
* **`Connection.backup()`'s internal retry loop** — bounded by nothing in this codebase (**C-03**).
* Anything before the deadline token is created, and anything after the helper returns.

#### 3.9.3 Consequences for acceptance

* The 30 s figure is a **Monotonic Initialization Budget** — the cooperative engineering bound over OD-2's five
  operations — **not** a mathematical guarantee about wall-clock time.
* `elapsed ≤ budget + 2 s` is permitted **only** in **T-F1-16**, under its documented conditions (§7.3): local
  filesystem, a cooperating blocker connection, a machine not otherwise loaded, and a budget deliberately
  shortened to **1 s** so the 2 s tolerance is a large fraction of it and overshoot is visible.
* **No document, report or commit message may present `deadline + 2 s` as protection against OS or disk stalls.**

#### 3.9.4 Deferred requirements

> **RQ-HARD-CAP** — an absolute wall-clock bound over **all** initialization work, including `mkdir`, DDL and
> migration, would require out-of-band termination (a supervisor or watchdog), because cooperative timeouts cannot
> bound code that never runs. **Not proposed here; not approved.**
>
> **RQ-BACKUP-DEADLINE** — *status changed in v1.4.* v1.3 recorded this as deferred and undesigned. **C-11 now
> designs it** (§3.10): page-stepping with `pages=1` under a `progress=` callback that raises, with **no** out-of-band
> termination required. It is **DESIGNED, not implemented, not tested against project code**, and its **numeric value
> is unset (OD-6)**. The residual that remains deferred is narrower: whether a *preemptive* wall-clock cap (one that
> survives a filesystem stall) is required — that is still **RQ-HARD-CAP**, and still unapproved.

---

### 3.10 C-11 — Backup inside migration: a separate, independent deadline

**The problem v1.3 left open.** `_backup(version)` is called at `:207`, **before** `BEGIN IMMEDIATE` at `:209` —
so Backup is not adjacent to migration, it is **inside** it. OD-4 removes *Schema creation and Migration* from the
30 s WAL budget and simultaneously requires that **no unbounded wait remains in the migration path** (owner item 6).
v1.3 resolved this by recording `RQ-BACKUP-DEADLINE` as *"deferred, unapproved, not designed"* — which left an
operation that PR-16 shows to be **genuinely unbounded** (25 s wall, still running) sitting inside a path OD-4
says must be bounded. **That is the defect C-11 corrects.**

**Owner decisions restated — nothing here overrides them.**

| # | Owner decision (unchanged) |
| --- | --- |
| 1 | The 30 s WAL budget (OD-1) is **not** extended, shortened or reinterpreted by this section. |
| 2 | The backup deadline is **independent** of the 30 s WAL budget — separate token, separate start, separate scope. |
| 3 | The **numeric value of the backup deadline is `OWNER DECISION REQUIRED` (OD-6, §2.1).** **No number is chosen in this document.** Every reference is the symbolic `BACKUP_DEADLINE_S`, deliberately undefined. |
| 4 | Backup **stays** in migration. It is not moved, not made optional, not made asynchronous. |

**3.10.1 Why `sleep=` and `connect(timeout=)` cannot be the deadline — measured, not asserted**

| Mechanism | What it actually does | Why it cannot bound total duration | Evidence |
| --- | --- | --- | --- |
| `Connection.backup(sleep=S)` | sets the **gap between attempt cycles** | it changes *rate*, not *termination*. The retry loop has **no exit condition** and no total cap. With `sleep=0.25`, 0.6 s produced **4** callbacks; with `sleep=0.0`, **8 252** — both still running. | **PR-22**, **PR-9b**, **PR-16** |
| `connect(timeout=)` / `PRAGMA busy_timeout` on the source | bounds **one** busy-handler wait | `backup()` re-enters the loop indefinitely; each individual wait may be short while the **sum** is unbounded. `sleep=0.05` under a held `BEGIN EXCLUSIVE` survived an **8 s** wall and a **25 s** wall. | **PR-9b**, **PR-16** |
| `pages=-1` (the current call at `:309`) | copies all remaining pages in **one** step | the `progress` callback fires **once** — `pages=-1` copied 2 M rows in 0.016 s with **1** callback. There is **no point at which to intervene**. | **PR-17** |
| `progress=` alone | called between page steps | **unreachable while blocked** unless the source's busy handler is shortened: with the inherited 10 s handler, the first callback arrived at **5.01 s**. | **PR-10b** |

**Therefore: no single stock parameter is a deadline.** Any design that claims otherwise is wrong, and v1.3 was
right to forbid the claim.

**3.10.2 What does work — the composite, standard-API mechanism**

The only lever in the standard library that can interrupt a backup is a **cooperative callback that raises**,
combined with **page-stepping** so the callback is reached often enough, combined with a **short busy handler on the
source** so the callback is reached *at all* while the source is locked.

```
                        ┌─────────────────────────────────────────────┐
                        │  BACKUP DEADLINE  (OD-6 — value NOT set)    │
                        │  BACKUP_DEADLINE_S : OWNER DECISION REQ.    │
                        └──────────────────┬──────────────────────────┘
                                           │ monotonic token, created at :207 entry
                                           ▼
  source = the ledger's own conn  ──►  RE-ARM to busy_timeout=0   (PR-7 mechanism)
  destination = new file           ──►  connect(timeout=0.0), busy_timeout=0
                                           │
                                           ▼
             conn.backup(destination, pages=1, progress=_progress, sleep=0.05)
                                           │
              _progress(status, remaining, total):
                  if monotonic() - start >= BACKUP_DEADLINE_S:
                      raise SqliteBackupTimeout(path, pages_copied, budget_used)   ← from nowhere
                  pages_copied = total - remaining
```

**Why each element is required — remove any one and the deadline fails:**

| Element | Without it | Evidence |
| --- | --- | --- |
| `pages=1` | callback fires once; an 8 MB copy completes in 0.016 s with no chance to intervene | **PR-17** |
| `progress=` raising our own exception | nothing else can interrupt `backup()` | **PR-11** |
| source `busy_timeout=0` | the first callback arrives only after the source's own handler expires (5.01 s with a 10 s handler) — the deadline can be blown before we are ever consulted | **PR-10b**, **PR-11** |
| destination `busy_timeout=0`, `connect(timeout=0.0)` | the destination is a fresh file, but a fast-fail policy keeps it from adding its own 5 s default | PR-11 |
| `sleep=0.05` | with `sleep=0.0` the loop spins 8 252 times in 0.6 s — correct but wasteful; 0.05 keeps the overshoot small without burning CPU | **PR-22**, **PR-11** |

**What this buys, honestly stated.** PR-11 measured **2.01 s against a 2.00 s deadline (+0.01 s)**. That is a
**cooperative** bound: it is checked between page steps, so the overshoot is at most one page step plus scheduler
latency. It is **not** a preemptive guarantee — a single page step on a stalled filesystem can exceed it, and
§3.9.2 applies unchanged. **This is strictly better than `sleep=`, which has no bound at all, and it is the best
the standard library offers without out-of-band termination (RQ-HARD-CAP, still unapproved).**

**3.10.3 The algorithm**

```python
def _backup(self, version: int) -> tuple[Path, str]:
    deadline = InitDeadline(started=time.monotonic(), budget_s=BACKUP_DEADLINE_S)  # OD-6 value
    target = self._backup_dir / f"{Path(self.path).name}.backup-v{version}-{stamp}"
    destination = sqlite3.connect(str(target), timeout=0.0)      # SECONDS — explicit (F1-R19)
    destination.execute("PRAGMA busy_timeout=0")                 # MILLISECONDS
    original_bt = self._conn.execute("PRAGMA busy_timeout").fetchone()[0]
    pages_done = 0
    try:
        self._conn.execute("PRAGMA busy_timeout=0")              # SHORTEN so progress is reachable

        def _progress(status, remaining, total):
            nonlocal pages_done
            pages_done = total - remaining
            if deadline.expired():                               # monotonic check, inside callback
                raise SqliteBackupTimeout(...)                   # our own exception, not sqlite3

        self._conn.backup(destination, pages=1, progress=_progress, sleep=0.05)
        destination.commit()
    except SqliteBackupTimeout:
        destination.close()
        _unlink_partial(target)          # §3.10.4 — delete, do not leave a half file
        raise                             # propagates as B-1 (§3.5.2.1)
    finally:
        self._conn.execute(f"PRAGMA busy_timeout={original_bt}")   # RESTORE the owner's policy
        destination.close()               # idempotent; safe if already closed
    # ---- verification unchanged: the existing integrity + row-count checks at :313–326 ----
```

**3.10.4 Partial file on timeout — exact handling (owner item 8)**

| Question | Answer |
| --- | --- |
| What is on disk after an abort? | PR-21/PR-12/PR-18 observed **0 bytes and no tables** on early aborts; PR-19 observed a **complete** copy when the abort landed after the copy had finished. **The state is not uniform.** |
| Therefore, what must the code do? | **Delete the file.** `_unlink_partial()` removes `target` and any `-wal`/`-shm` sidecar it created. Do **not** rely on it being empty. |
| Why not leave it for diagnosis? | Because PR-19 shows a timed-out run can leave a **complete-looking** file under a timeout exception. An operator who finds it later cannot tell a timed-out-but-complete backup from a good one. **A file that may or may not be valid must not survive a failure.** |
| Is a partially written file ever treated as a valid backup? | **Never.** The verification block at `:313–326` runs only on the success path, and it is unchanged. |
| Is deletion logged? | Yes — at `WARNING`, with the target path and the bytes removed. |

**3.10.5 Should Backup use its own source connection? (owner item 9)**

| | **A — reuse the ledger's connection** (chosen) | **B — open an independent read-only source** |
| --- | --- | --- |
| Deadline reachability | **works**, but requires re-arming the owner's connection to `busy_timeout=0` and restoring it in `finally` | works without touching the owner's policy |
| Conflicts with **C-01** ("creator owns the policy") | **Yes — a real tension.** The ledger borrowed `self._conn` from the helper; re-arming it here makes the ledger a *second* policy owner for the duration of the call. | **No conflict** — a new connection has a new owner |
| Consistency of the copy | `backup()` on the live connection gives a **consistent snapshot** at its own read transaction | also consistent — `backup()` takes its own snapshot |
| Extra handles / close paths | none added | one more handle; one more close path in every failure branch |
| Risk if the restore in `finally` is missed | the borrowed connection keeps `busy_timeout=0` for the **life of the process**, silently disabling lock waits on the ledger's own writes | n/a |
| Risk if the source is locked | must shorten the handler to make progress reachable | same |

**Chosen: A, with a mandatory `try/finally` restore — and this is flagged as a decision the owner may reverse.**
The reason for A is that B opens a *second* connection to a file the ledger is simultaneously migrating, adding a
handle and a failure path for no consistency benefit. The reason this is **not** settled is that A contradicts
C-01's ownership rule for the duration of the call. **Q-14 records the choice and asks the owner to confirm it or
mandate B.** Until answered, **A is a Proposal, not a Decision.**

**3.10.6 Migration, WAL and backup — the three scopes, side by side (owner item 7 requirement)**

| | **WAL initialization** | **Backup** | **Migration** |
| --- | --- | --- | --- |
| Deadline | `WAL_INIT_BUDGET_S` = **30.0 s** (OD-1) — **CONFIRMED** | `BACKUP_DEADLINE_S` = **OWNER DECISION REQUIRED** (OD-6) | per-statement `busy_timeout=10000 ms`; **no total cap** (OD-4) |
| Starts | before connection open (P-1) | at `:207` entry | at `migrate()` entry, after backup |
| Ends | at handover (P-14) | at return or `SqliteBackupTimeout` | at commit/rollback |
| Bounded by | monotonic algorithm (§3.4.6) | cooperative callback (§3.10.3) | per-statement handler only |
| Failure type | `SqliteBusyTimeout` / `SqliteNotADatabase` / `SqliteUnretryableError` | `SqliteBackupTimeout` | `SqliteMigrationBusyTimeout` / `SqliteMigrationLockError` |
| Public surface | `LedgerUnavailable` at the ledger boundary | `LedgerUnavailable` | `LedgerUnavailable` |
| Additive to the others? | **No** — they are sequential, not nested | **No** | **No** |
| Total worst case | 30 s | + `BACKUP_DEADLINE_S` | + 10 s **per statement** × N statements |

**Order of execution (owner item 7):** `mkdir (P-1)` → **WAL init (30 s)** → `_version()` → **[steady state: return]**
→ **backup (OD-6)** → `BEGIN IMMEDIATE` → **migration (per-statement 10 s)** → commit. The three deadlines are
**sequential and non-overlapping**; none is nested inside another, so no budget is double-counted.

**3.10.7 Status and what is still needed**

| Item | Status |
| --- | --- |
| Backup deadline mechanism | **DESIGNED** (§3.10.3; runtime-proven at library level by PR-11) |
| Numeric value | **NEEDS — OWNER DECISION (OD-6)** |
| Source connection choice | **DESIGNED as Proposal**, pending **Q-14** |
| Interplay with C-01 ownership | **OPEN** — Q-14 |
| Whether `pages=1` is acceptable performance-wise on a large ledger | **NEEDS MEASUREMENT** — U-7 |

---

## 4. F-2 Deterministic Concurrency Test Design

### 4.1 What "deterministic" must mean here

A test is **deterministic** if, for a given tree, it yields the same verdict on every run on every supported
platform with no scheduling dependence. A test is a **valid detector** of a mutation if it fails **every time** on
the mutated tree. **Fact:** A7 achieves 7.5 % sensitivity against a faithful mutant and 0 % against removal of
`_in_immediate`. Repetition is not a design.

### 4.2 The three mechanisms and their exact claims

| Mechanism | What it does | May claim | May **not** claim |
| --- | --- | --- | --- |
| **M-A. Same-transaction interleaving simulation** | inside the adapter's own transaction, a hook performs one "advance" statement **on the same connection**, then the adapter's write proceeds | "the write must be **column-targeted** / **guarded**" | not a competing committed write; not isolation evidence; **never** multi-process evidence |
| **M-B. Lock-level witness** | while the adapter is paused inside its transaction, an **independent** connection with `busy_timeout=0` attempts `BEGIN IMMEDIATE` | "the adapter holds a **RESERVED** lock at that point" | does not prove a real process was *blocked out*; proves the lock is **held** |
| **M-C. Real multi-process race** | two OS processes contend end-to-end | nothing on its own — an observed rate | never proof |

### 4.3 Thread-ownership invariant (**F2-R11 — unchanged, binding**)

> **A SQLite connection is created, used and closed in one and the same thread.** No test may rely on
> `check_same_thread=False`, and `check_same_thread` is **never** changed to make a test pass.

**Why:** `SqliteOperationLedger` builds its connection with the stdlib default **`True`** (P-5). A hook touching
`L1` from the main thread while the worker owned it would raise `sqlite3.ProgrammingError` — a test that fails for
the wrong reason, or passes because the interleaving never happened.

#### **T-F2-01 — column-targeted bump cannot revert an advanced row** *(M-A · class D)*

| Aspect | Design |
| --- | --- |
| Thread ownership | `L1` created, used and closed **only in the worker thread**; the main thread never touches a connection |
| Initial state | fresh file; one ENTRY op `APPROVED`, `claim_attempts=0`, `duplicate_requests=0` |
| Order | worker creates `L1` → records the op → installs the pause hook on its own connection → calls `record()` again → `_bump_duplicate` (`:503`) → `_in_immediate` (`:516`) → `BEGIN IMMEDIATE` → `_load` → **hook**: `reached.set(); go.wait(timeout=10)` → main sets `go` (touching no connection) → **in the hook, same thread, same connection, same transaction**: `UPDATE operation_ledger SET state='SUBMITTING', claim_attempts=1 WHERE op_id=?` → hook returns → adapter's UPDATE runs → commit → worker closes `L1` |
| Expected (fixed) | final `state='SUBMITTING'`, `claim_attempts=1`, `claim_for_send()` → `False`; `duplicate_requests` **reported, not asserted** |
| Mutation | **M-1**: `_bump_duplicate` → `self._update(self._apply_bump(existing))` |
| Why certainly detected | under M-1 the write is a full-row `UPDATE` from the snapshot read **before** the advance → writes `state='APPROVED'` unconditionally. Probability 1. |
| Docstring (mandatory) | *"same-transaction interleaving simulation — verifies column-targeted writes; NOT a multi-process isolation proof."* |

#### **T-F2-02 — guarded normalisation rejects a changed hash** *(M-A · class D)*

Thread ownership as above. Hook fires on the normalisation `UPDATE` before it executes; inside the hook, same
thread/connection/transaction, `UPDATE … SET payload_hash='<other>' WHERE op_id=?`. Expected: `record()` raises
`LedgerUnavailable`, `rowcount == 0`, the other hash survives, `digest_scheme` unchanged. Mutation **M-3** (drop
`AND payload_hash=?`) ⇒ `rowcount == 1` ⇒ regression detected with probability 1. The test asserts **both** the
exception **and** the guard outcome, so a mutant that merely rewrites the message cannot pass.

#### **T-F2-03 — the serialization primitive is actually held** *(M-B · class D — the M-2 witness)*

| Aspect | Design |
| --- | --- |
| Thread ownership | **`L1` is constructed inside the worker thread**; begin, read, update, commit and **close** all happen there. **`L2` is constructed and used only in the main thread.** The main thread performs **zero** operations on `L1`. |
| Order | worker creates `L1` → records the op → installs a hook that fires after the in-transaction `_load` and before the `UPDATE` → calls `transition(op, EXPIRED)` → `BEGIN IMMEDIATE` → `_load` → **hook**: `reached.set(); go.wait(timeout=10)`; **main** creates `L2 = sqlite3.connect(path, timeout=0.0)` + `PRAGMA busy_timeout=0`, attempts `BEGIN IMMEDIATE`, records the outcome; main sets `go`; worker completes, commits, **closes `L1` in its own thread** |
| Expected (fixed) | step 3 raises `sqlite3.OperationalError` with `primary_code == 5`; after `go` the op is `EXPIRED`; `claim_for_send()` → `False` |
| Expected (M-2) | step 3 **succeeds** — no RESERVED lock is held |
| Mutation | **M-2**: `transition()` calls `run()` instead of `self._in_immediate(run)` |
| Why certainly detected | a **lock-state fact** observed while the worker is provably stopped — a logical consequence of the primitive, not a race |
| Secondary witness | `set_trace_callback` records `BEGIN IMMEDIATE` between the `SELECT` and the `UPDATE`; structural only, **not** the acceptance instrument |
| Cleanup | `try/finally` in both threads guarantees closure; joins have timeouts |

#### **T-F2-04 — real two-process race** *(M-C · class P — complementary only)*

Two `spawn`ed processes; construction **after** the barrier (F-2-D6, §5.7). Expected: exactly one
`OperationClaimed`, `claim_attempts == 1`, one submission line, final `SUBMITTING`, `integrity_check == 'ok'`.
**Reported as "N runs, M failures"; never proof; not the F-2 witness.**

#### **T-F2-05 — the safety consequence** *(class D)*

`claim_for_send()` → `False`; `recover_incomplete()` leaves `UNKNOWN` untouched; `RESOLVED_ABSENT` unreachable
without a complete query.

#### **T-F2-06 — mutation harness** *(class M)*

Copy the tree to `tmp_path` (asserting the copy root is **not** under the repository), apply a scripted textual
mutation, run the designated test in a subprocess, assert non-zero exit and the expected failing test ID.
**Required-catch list:** M-1 → T-F2-01 · **M-2 → T-F2-03** · M-3 → T-F2-02. **5 of 5** harness runs must catch all.

#### **T-F2-07 — the list cannot be quietly shrunk** *(class M)*

Asserts `M-2 → T-F2-03` is present in the declared list; fails if removed.

### 4.4 Claim boundaries (binding on every report)

| Statement | Allowed? |
| --- | --- |
| "T-F2-01/02 prove the write is column-targeted / guarded" | **Yes** |
| "T-F2-03 proves the serialization primitive is held during the RMW" | **Yes** |
| "T-F2-01..03 prove two processes are isolated" | **No** |
| "The deterministic tests prove absence of a Lost Update **across processes**" | **No** |
| "T-F2-04 over N runs produced M failures" | **Yes** |
| "A7 is now deterministic" | **No** — A7 remains class P |

### 4.5 Forbidden in the deterministic tests

`sleep()` to widen a window · repeat-and-hope · asserting an exact `duplicate_requests` · an in-process
`threading.Lock` as the "other actor" · **moving a connection between threads** (F2-R11) · editing or deleting A7
beyond the disclosed strengthening.

### 4.6 F-2-D7 — document the atomicity classes (comment-only)

A docstring block tabulating §2.6, including why the single-statement class is safe **only** while it remains one
statement. Comment-only = still Gate-2.

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

5. **`if __name__ == "__main__":`** — **not required under pytest** (the module is imported by the pytest entry
   point, not executed as `__main__`). Required only in standalone reproduction scripts under `tools/`, with the
   reason documented.

### 5.4 F-3-D3 — primitives that survive `spawn`

| Primitive | Under `spawn` | Verdict |
| --- | --- | --- |
| plain `Barrier` | not shareable | **Forbidden** cross-process |
| `Manager().Barrier(n)` | proxy | **Required** *(already used: `:68`, `:114`)* |
| `Manager().Queue()` | proxy | **Required** *(already used)* |
| plain `Event()` | not shareable cross-process | **Forbidden** cross-process; fine for **threads** |
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
| `run_workers(ctx, target, args_list, *, join_timeout, result_timeout)` | start all; join each; on timeout `terminate()` → `join(5)` → `kill()`; raise a diagnostic naming the dead worker and exit code |
| `finish(manager)` | `manager.shutdown()` in `finally` *(today no manager is explicitly shut down)* |
| `assert_clean_exit(results)` | fail with `exitcode` + captured `repr` |

**Thresholds:** `join_timeout = 120 s`, `result_timeout = 60 s` — unchanged. **Constraint:** the WAL budget (30 s)
must stay **below** `result_timeout` (60 s) so a lock timeout surfaces as `SqliteBusyTimeout`, not a queue timeout.
**Stdlib only** — `requirements.txt` stays one line.

### 5.7 F-2/F-3 overlap — one disclosed strengthening of A7

**Proposal F-2-D6:** move construction **after** `go.wait()` in AU-22 (`:36–37`) and AU-23 (`:220–221`), keeping
every assertion — strictly **adds** coverage. **Hard ordering:** lands **after** F-1; otherwise A7 fails at roughly
the F-1 rate (4.4 %–46.7 %) and looks like a regression introduced by a test change.

### 5.8 Windows risks this spec does not resolve

The real F-1 rate on Windows file locking · `msvcrt`/`fcntl` in `LiveInstanceLock` (`live_instance_lock.py:44–62`,
OOS-6) · antivirus/indexer holding the DB · `spawn` cost on a 2-vCPU VPS · path length / unicode paths.
**All Unknown; all NOT RUN.**

---

## 6. Source File Impact Matrix

| File | Change | Lines (current) | Requirement | Risk | Gate |
| --- | --- | --- | --- | --- | --- |
| `src/goldbot/adapters/sqlite_init.py` | **NEW** | — | F1-R1…**R22, R29, R30** | Low — additive | 1 |
| `src/goldbot/adapters/sqlite_store.py` | modify | 153–169 | F1-R1, R5, R6, R13 | **Medium** — 58 call sites; hash `a74e06db…` changes | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | modify | 147–160, 181–236 (delete `:191–194`), **298–326 (`_backup` deadline + partial-file deletion)** | F1-R1, R13, R16, R17, R19, **R23…R28** | **Medium–High** — order path **and** the backup now raises; every `except` around `_backup()` must be re-checked against §3.5.5 | 1 |
| `src/goldbot/adapters/sqlite_state_store.py` | modify | 27–37 | F1-R1, R2, R13 | Low | 1 |
| `src/goldbot/adapters/sqlite_operation_ledger.py` | comment-only | near `:439` | F-2-D7 | None | 2 |
| `tests/test_f1_wal_initialization.py` | **NEW** (fast lane) | — | F-1 acceptance | Low | 1 |
| `tests/test_f1_wal_initialization_slow.py` | **NEW** (env-gated) | — | F-1 acceptance (300-lane) | Low | 1 |
| `tests/test_f2_deterministic_lost_update.py` | **NEW** | — | F-2 | Low | 2 |
| `tests/test_f2_mutation_harness.py` | **NEW** | — | F-2 (T-F2-06/07) | Medium — copies+muts source in tmp | 2 |
| `tests/mp_support.py` | **NEW** | — | F-3 | Low | 3 |
| `tests/test_phase_a_audit_multiprocess.py` | modify (strengthen) | 36–37, 66, 112, 220–221 | F3-R1, F2-D6 | **Medium** — audit test | 3 (after 1) |
| `tests/test_fix05_counter_semantics.py` | modify | 181 (+ guard) | F3-R2 | Low | 3 |
| `docs/adr/` | **NEW ADR** | — | records OD-1/OD-2/OD-4, **OD-6 (open)**, §3.4–§3.10 | None | 1 |
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

### 7.2 Statistical basis — per adapter, and its limits

**The 95 % claim is per adapter and independent.** Each lane is powered for a 1 % residual rate:

$$\Pr(\text{miss}) = (1-p)^n \le 0.05 \;\text{with}\; p = 0.01 \;\Rightarrow\; n \ge \frac{\ln 0.05}{\ln 0.99} \approx 298 \;\Rightarrow\; n = 300$$

| n | Smallest rate detectable at 95 % power | Verdict |
| --- | --- | --- |
| 10 | ~26 % | useless |
| 40 | **~7.2 %** | **insufficient** — E-2 measured 4.4 %; a 4.4 % regression would pass |
| 100 | ~3.0 % | marginal |
| **300** | **~1.0 %** | **acceptance lane** |

**If a joint three-adapter claim is wanted, the arithmetic changes and must be stated.** Three independent lanes at
α = 0.05 give a family-wise false-pass probability `1 − 0.95³ ≈ 0.143` (**≈ 14.3 %**). A joint 95 % claim would
require Bonferroni: α′ = 0.05/3 ≈ 0.0167 ⇒ `n ≥ ln(0.0167)/ln(0.99) ≈ 408` per adapter.
**Decision for v1.3 (unchanged):** keep **n = 300 per adapter**, report **each adapter separately**, and **prohibit
any joint "95 % for all three" sentence** (Q-11).

**Explicit non-claim:** *"0 failures in 300"* bounds the residual rate at ≈1 % with 95 % confidence **under
independence and stationarity**. It is **not** proof of absence.

**Cost:** 300 × 3 adapters at ~0.5–0.8 s/iteration ≈ **8–12 min** — hence opt-in.

| Lane | Trigger | n | Purpose |
| --- | --- | --- | --- |
| `fast` | default pytest | 40/adapter, 1 scenario | smoke |
| `acceptance` | `GOLDBOT_WAL_ACCEPTANCE=1` | **300**/adapter (S4) + 100 (S5) | the F-1 witness |
| `deadline` | default | 3 | deterministic fail-closed proof |

**The 40-iteration fast lane may never be cited as proof.**

### 7.3 F-1 test matrix

| ID | Class | Scenario | Key assertions | Acceptance |
| --- | --- | --- | --- | --- |
| T-F1-01 | D | S1 ×3 adapters | no exception; `journal_mode` == `wal` | 3/3 |
| T-F1-02 | D | S2 rollback DB, 100 rows | mode becomes wal; row values identical | pass |
| T-F1-03 | D | S3 existing WAL | no error; data unchanged | pass |
| T-F1-04 | S | S4 two processes | all constructors return; one file set | **0/300 per adapter, reported per adapter** |
| T-F1-05 | S | S4 **pre-fix tree** (negative control) | — | **≥1 failure in 300** — proves sensitivity |
| T-F1-06 | S | S5 four processes | as T-F1-04 | **0/100** |
| T-F1-07 | D | S6 lock past budget | `SqliteBusyTimeout`; message has path/attempts/budget | pass |
| T-F1-08 | D | connection leak | patched `connect` recorder; every recorded conn raises "closed" | **0 leaks / 100** |
| T-F1-09 | D | S7 short lock (~0.3 s) | succeeds; `attempts ≥ 2` | pass |
| T-F1-10 | D | S8 corrupt file | `SqliteNotADatabase`; `attempts == 1`; `< 2 s`; message has "database"; **AU-35 unmodified green** | pass |
| **T-F1-11** | D | **S9 directory delta (C-09)** | snapshot `set(os.listdir(dir))` **before** the constructor runs, with the file **explicitly absent**; after: classify **three ways** — (i) expected main DB `<name>` **(allowed, and must exist)**; (ii) expected sidecars `<name>-wal`, `<name>-shm` **(allowed, optional)**; (iii) **anything else = failure**, in particular any `<other>.db`, `.backup-*` or unrelated artefact; nothing present before may disappear | pass |
| T-F1-12 | D | S10 forced `"delete"` | `SqliteInitializationError` | pass |
| **T-F1-13** | D | **S11 shared-journal branch (C-01)** | helper **not** called; borrowed connection's `busy_timeout` **identical before and after** `SqliteOperationLedger(journal=J)`; **`grep busy_timeout` in the ledger module returns zero matches**; `migrate()` still serialized | pass |
| T-F1-15 | D | S12 `check_same_thread` regression | journal usable cross-thread; ledger & state raise `ProgrammingError` on first use in another thread | 3/3 |
| **T-F1-16** | D | **budget is a hard cap — controlled conditions only (C-08)** | see the seven preconditions below | pass |
| **T-F1-17** | D | **budget accounting (C-07)** | see the invariants below | pass |
| **T-F1-18** | D | **S13 migration chain (C-04)** | hold `BEGIN EXCLUSIVE` during a v1→v3 migration → public exception is `LedgerUnavailable`; `__cause__` is `SqliteMigrationBusyTimeout`; `__cause__.__cause__` is the original `sqlite3` error; **`SqliteBusyTimeout` never appears**; no silent `pass`; `in_transaction` was `False` before the begin; after release, a retry succeeds; `integrity_check == 'ok'` | pass |
| T-F1-19 | D | non-retryable fail-fast | codes `26, 14, 8, 3, 11` → `attempts == 1`, `< 2 s` | pass |
| **T-F1-20** | D | **per-code classification (extended codes)** | BUSY(5) retried; BUSY_RECOVERY(261) retried; BUSY_SNAPSHOT(517) retried; BUSY_TIMEOUT(773) retried; **LOCKED(6) → `SqliteUnretryableError`, `attempts == 1`**; **LOCKED_SHAREDCACHE(262) → same** | 6/6 |
| **T-F1-22a** | D | **text fallback, BUSY message, no code (C-02)** | error with `sqlite_errorcode` removed/absent and message `database is locked` → **retried** (≥2 attempts), subject to the budget | pass |
| **T-F1-22b** | D | **text fallback, LOCKED message, no code (C-02)** | error with `sqlite_errorcode` absent and message **`database table is locked`** → **`SqliteUnretryableError`, `attempts == 1`, NOT retried** | pass |
| **T-F1-22c** | D | **ambiguous text, no code (C-02)** | message that matches neither pattern (e.g. `database disk image is malformed`) → **fail closed**, `attempts == 1` | pass |
| T-F1-21 | D | S14 backup configuration and cleanup (C-03) | both `:307` and `:313` connections pass an **explicit `timeout=`**; the value is documented; both closed on success and on failure; **no duration assertion and no bounded-backup claim** | pass |
| **T-F1-23** | D | **S18 backup deadline (C-11)** | source in **`journal_mode=DELETE`** (else `backup()` does not block at all — PR-15/PR-25); blocker holds `BEGIN EXCLUSIVE`; `BACKUP_DEADLINE_S` patched to **1.0 s**; assert: `SqliteBackupTimeout` raised; `elapsed ≤ 1.0 + 2.0 s`; source connection's `busy_timeout` **restored to 10 000** in `finally`; destination file **deleted**; `LedgerUnavailable` at the public boundary with `SqliteBackupTimeout` as `__cause__`; **no** `integrity_check` is run on a deleted file | pass |
| **T-F1-24** | D | **S19 migration LOCKED ≠ timeout (C-13)** | force code **6** into a migration statement; assert `SqliteMigrationLockError`; **`attempts == 1`**; **no backoff sleep**; message contains `locked` and **does not contain** `timeout`/`exhausted`; `LedgerUnavailable.__cause__` is `SqliteMigrationLockError`; **`SqliteMigrationBusyTimeout` never appears**; contrast case M-1 in the same test file raises the *other* type | pass |
| **T-F1-25** | D | **S20 runtime text identity (C-16)** | with **real** SQLite, not a mock: (i) cross-connection contention → assert `sqlite_errorcode == 5`, `sqlite_errorname == 'SQLITE_BUSY'`, message `database is locked`; (ii) unexhausted cursor + `DROP TABLE` → assert `sqlite_errorcode == 6`, `sqlite_errorname == 'SQLITE_LOCKED'`, message `database table is locked`. **If the runtime ever reports different text, the test fails** — the classifier's text fallback is thereby anchored to an observation, not to a guess | 2/2 |
| T-F1-14 | X | Windows | — | **NOT RUN** |

**T-F1-16 — the seven preconditions required by C-08** (the test must exercise the **actual** transition):

| # | Precondition | How it is established |
| --- | --- | --- |
| 1 | Create a throw-away database in **`journal_mode=DELETE`** | `PRAGMA journal_mode=DELETE` on a fresh file; commit |
| 2 | **Verify it is not already WAL** | `PRAGMA journal_mode` must return `delete`; the test **fails** if it returns `wal` |
| 3 | A competing connection acquires **`BEGIN EXCLUSIVE`** | separate `sqlite3.connect(path, timeout=0)` + `busy_timeout=0`; `BEGIN EXCLUSIVE` |
| 4 | **Verify the holder actually holds the lock** | a third probe connection's write attempt must raise BUSY, proving the lock is real before the helper is invoked |
| 5 | Invoke the helper with a **shortened test-only budget** — **no `budget_s` argument is passed** (C-15) | `monkeypatch.setattr(sqlite_init, "WAL_INIT_BUDGET_S", 1.0)` |
| 6 | Verify BUSY classification, retry termination, typed exception, cleanup | `SqliteBusyTimeout`; `attempts ≥ 1`; connections recorded by the leak recorder are closed; `elapsed ≤ 3.0 s` **only here** |
| 7 | Release and close **all** test connections in `finally` | blocker `rollback()`+`close()`; probe closed; temp dir removed |

**T-F1-17 — invariants required by C-07** (aligned with §3.4.5):

| # | Invariant | Basis |
| --- | --- | --- |
| I-1 | for every attempt: `busy_timeout_ms_used ≤ remaining_ms_before` | R-a/R-b |
| I-2 | for every attempt: `delay_ms_planned ≤ remaining_ms_before` **or** the attempt is the terminal one (fail closed before sleeping) | R-a |
| I-3 | `remaining_ms_before` is strictly **non-increasing** across attempts | proves recomputation |
| I-4 | every attempt has `remaining_ms_before > 0` — **no retry starts after expiry** | R-c |
| I-5 | `delay_ms_actual` is present and **≥ 0** for every attempt that slept | R-d |
| I-6 | `Σ delay_ms_actual` is **REPORTED**, and **is NOT asserted to be ≤ budget** | **R-e — this is the C-07 correction** |
| I-7 | `Σ delay_ms_planned ≤ budget` | R-a |

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

**Additional acceptance (F2-R11):** no F-2 test may reference a connection created in a different thread — enforced
by review plus the fact that the stdlib raises `ProgrammingError`, which the tests must not suppress.

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

**Reporting rule:** every table carries **two** result columns; Windows cells are `NOT RUN` until Gate-5 produces
real logs from a Windows host.

---

## 8. Failure Handling and Observability

| Condition | Detected by | Behaviour | Operator sees | Test |
| --- | --- | --- | --- | --- |
| Transient BUSY (< budget) | retry loop | retries with jittered backoff | nothing (INFO line) | T-F1-09 |
| BUSY past budget | budget | **fail closed** | `SqliteBusyTimeout`: path, attempts, elapsed, budget, `last_code`, name, cause | T-F1-07/16 |
| `LOCKED` / `LOCKED_SHAREDCACHE` (by code or text) | classifier | **fail fast**, no retry | `SqliteUnretryableError`, `attempts == 1` | T-F1-20, T-F1-22b |
| Ambiguous text, no code | classifier | **fail closed** | `SqliteUnretryableError`, `attempts == 1` | T-F1-22c |
| WAL not applied | post-transition verify | fail closed | reason `journal_mode is 'delete', not 'wal'` | T-F1-12 |
| Corrupt / non-DB | classifier | fail fast | `SqliteNotADatabase`, message has "database" | T-F1-10 |
| **Migration lock wait exhausted** | F1-R17 + migration policy | **typed inner cause, wrapped at the boundary** | `LedgerUnavailable` → `SqliteMigrationBusyTimeout` → original | T-F1-18 |
| Backup lock-wait | F1-R19 | controlled error **per connection**; **no total-duration promise** | message names target + timeout | T-F1-21 |
| Worker dies | `run_workers` | terminate→join(5)→kill | failure naming worker + exit code | T-F3-03 |
| Worker hangs | `join_timeout` | bounded | ditto | T-F3-04 |
| Hook never fires | test-side `Event.wait(timeout)` | fails "hook never fired" | distinguishes "path changed" from "assertion failed" | T-F2-01/03 |
| Mutant escapes | harness | harness fails | names the escaping mutant | T-F2-06 |

**Proposal — one structured INFO line per initialization**, using the project's existing convention
(`log.warning(..., extra={"fields": {...}})`, `runtime/live.py:361`): `path`, `attempts`, `elapsed_ms`,
`journal_mode_after`, `created`, `last_code`. Rationale: F-1 is a *rate*; without this an operator cannot learn that
a deployment is contending. **Not proposed:** overloading `migration_report()` with connection state (Q-7).

---

## 9. Risks and Non-Goals

### 9.1 F-1 risks

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| New exception types escape a caller catching `sqlite3.Error` | **Unknown — A-4** | §3.5.5 inventory is the Gate-1 condition |
| Migration lock wait (10 s × statements) makes a stuck migration slow to fail | Medium | Bounded **per statement**, explicit typed error, two-level cause; **no** claim about total migration time (OD-4) |
| A migration failure is misreported as a WAL timeout (or vice versa) | Low | **Distinct classes** (C-04) + T-F1-18 asserts the WAL class never appears |
| `LOCKED` classified non-retryable and a real case appears later | Low | Evidence-based (P-2) + **re-open condition Q-10** |
| `mkdir` stall is unbounded | **Proposal P-1, unconfirmed** | Kept visible; **Q-12 blocks Gate-1**; RQ-HARD-CAP records the gap |
| Backup appears bounded but is not | Medium | **C-03**: claim removed; T-F1-21 asserts configuration only; RQ-BACKUP-DEADLINE recorded |
| `sqlite_store.py` hash changes | Low | Old+new hashes recorded |

### 9.2 F-2 risks

| Risk | Mitigation |
| --- | --- |
| M-A judged "not a real race" | Declared as same-transaction simulation; M-B supplies lock-state evidence; M-C kept separately |
| M-B proves "lock held" not "process blocked" | Stated in §4.4; SQLite locks are file-level, so held-reserved is the correct witness |
| Thread-ownership mistakes re-introduce `ProgrammingError` confusion | F2-R11 invariant + §7.4 check |
| Mutation harness mutates the wrong tree | Copies to `tmp_path` and asserts the copy root is outside the repository |

### 9.3 F-3 risks

| Risk | Mitigation |
| --- | --- |
| `spawn` child cannot import module or `goldbot` | Mandatory module-level `sys.path` guard |
| Wall-clock grows 3–5× | Fast lane stays small; acceptance lane opt-in |
| A7 flaky if changed before F-1 | §10 enforces the order |

### 9.4 Non-goals (no implementation may expand into these)

Order Coordinator architecture · trading/strategy/indicator logic · broker/MT5 behaviour · schema or migration
**DDL** changes · database path or name changes · deletion/recreation of any real database · MT5/broker connection ·
demo or real orders · VPS configuration · deployment to a Windows VPS · operational soak · PHASE B · PHASE C ·
production-readiness claims · `requirements.txt` changes · `LiveInstanceLock` semantics · **any process watchdog or
supervisor** (covers RQ-HARD-CAP and RQ-BACKUP-DEADLINE) · bounding `Connection.backup()` internals.

---

## 10. Implementation Sequence

| Step | Content | Gate | Blocking constraint |
| --- | --- | --- | --- |
| 0 | Owner reviews v1.3 and answers Q-1, Q-2, Q-4…Q-7, Q-9…Q-13 | **Gate-0** | nothing proceeds; **Q-12 must be answered or Gate-1 cannot open** |
| 1 | `sqlite_init.py` + unit tests (T-F1-01/02/03/07/08/10/11/12/15/16/17/19/20/22a-c) | 1 | — |
| 2 | Wire the three adapters (§3.6.1) + **delete `:191–194`** + F1-R17 + F1-R19 | 1 | **No test file edited in this step** |
| 3 | `test_f1_wal_initialization.py` (T-F1-04/05/06/09/13/18/21) + slow lane | 1 | T-F1-05 must fail on the pre-fix tree |
| 4 | Gate-1 evidence: §3.5.5 inventory + acceptance lane (300 per adapter, reported separately) + fast lane | 1→4 | Linux only; Windows `NOT RUN` |
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
| V-4 | Suite | `GOLDBOT_SOAK=0 pytest -q` | `1224 + <new> passed, 6 skipped` — **counts only; no duration or warning claim** | Linux |
| V-5 | F-1 acceptance | 300 per adapter; **reported per adapter** | 0 failures each; raw logs attached | **Linux** |
| V-6 | F-1 **negative control** | same lane on the pre-fix tree | **>0 failures** | **Linux** |
| V-7 | F-1 budget | T-F1-16 with the seven preconditions | error at `≤ budget + 2 s` under §3.9.3 conditions | Linux |
| V-8 | F-2 determinism | T-F2-01/02/03 × 20 consecutive | **20/20 identical verdict** | Linux |
| V-9 | F-2 mutation | T-F2-06 | every mutant caught, 5/5 | Linux |
| V-10 | F-3 start method | T-F3-01 + `grep '"fork"'` = 0 | pass | Linux |
| V-11 | F-3 hang safety | T-F3-04 | bounded termination | Linux |
| V-12 | **Policy ownership** | `grep -n busy_timeout src/goldbot/adapters/sqlite_operation_ledger.py` | **zero matches** (C-01) | Linux |
| V-13 | Windows items | T-F1-14, T-F3-08 | **NOT RUN** unless a Windows host is provided | **Windows** |
| **V-14** | **Timeout units (C-17)** | static review against §3.4.7: no literal passed to `connect(timeout=)`; every `PRAGMA busy_timeout` argument an integer; every `_ms`/`_s` field matched to its API; the `1000.0` divisor present wherever seconds and milliseconds meet | **zero mismatches** | Linux |
| **V-15** | **No chosen backup value (C-11/OD-6)** | `grep -nE 'BACKUP_DEADLINE_S\s*='` in the implementation | **zero assignment**; the symbol alone appears, so OD-6 must be answered before code is written | Linux |
| **V-16** | **BUSY/LOCKED separation actually implemented (C-13)** | review of the verdict→exception table against §3.5.2.1; `grep` for `SqliteMigrationLockError` and `SqliteMigrationBusyTimeout` at **distinct** raise sites | two distinct types at two distinct sites; no site raises both | Linux |

**Rules:** V-6 is mandatory (an acceptance lane not shown to be *sensitive* is not evidence of absence — E-2's
0/40 on A7 against a tree with a reproducible 4.4 % startup failure proves the risk). **V-12 is mandatory** — it is
the mechanical proof that C-01 was implemented rather than described. **V-14 and V-15 are mandatory** for the same
reason: C-17 and OD-6 are the two corrections whose violation is *silent* (PR-20 shows a wrong unit is accepted
without complaint; a hard-coded backup value would look like a decision the owner never made).

---

## 12. Explicit Authorization Gates

Each gate reports **Linux** and **Windows** separately. **No gate may be recorded as `CLOSED` while any of its
Windows items is `NOT RUN`;** such a gate is `PARTIALLY CLOSED — WINDOWS OUTSTANDING`.

| Gate | Authorizes | Does **not** authorize | Closes on | Linux | Windows | Overall status |
| --- | --- | --- | --- | --- | --- | --- |
| **Gate-0** | Reading and commenting on this spec | **any edit** | owner answers to §14 | — | — | open |
| **Gate-1** | `sqlite_init.py`; three adapter wirings; **deletion of `:191–194`**; F1-R17; F1-R19; **F1-R23…R29** (backup deadline, C-13 split, C-12 algorithm, C-17 units); F-1 tests; ADR | F-2, F-3, **any edit to an existing test**, any DDL | §7.3 (T-F1-01…13, 15…25) + V-5 + **V-6** + **V-12** + §3.5.5 inventory + **V-14, V-15, V-16** + **Q-12, Q-13, Q-14, Q-15 answered** | `PASS` required | `NOT RUN` (T-F1-14) | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`** |
| **Gate-2** | F-2 test files, mutation harness, comment-only docstring | any behavioural source change | §7.4 + T-F2-06 5/5 | `PASS` required | `NOT RUN` | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`** |
| **Gate-3** | `mp_support.py`, `spawn`, import guards, A7 strengthening | **any assertion removal** | §7.5 T-F3-01…07 on Linux | `PASS` required | `NOT RUN` (T-F3-08) | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`** |
| **Gate-4** | **Independent verification of the Linux half only** | any further change; **and it may not close Windows** | V-1…V-12 | `PASS` or `FAIL` | **remains `NOT RUN`** | **`PARTIALLY CLOSED — WINDOWS OUTSTANDING`**; never described as "Linux + Windows verified" |
| **Gate-5** | Windows execution and Windows-specific adjustment | anything else | V-13 with real Windows-host logs | — | `PASS`/`FAIL` | the **only** gate that can close Windows items |

**Standing rules**
1. The delivery ZIP is never edited in place: extract → hash → verify → independent copy.
2. One new ZIP per gate; each gate's ZIP hash is recorded in its report.
3. **No gate may contain the phrase "Windows tested"** without logs from a Windows host in that gate's evidence
   folder. Absent that, the only permitted wording is `Windows: NOT RUN`.
4. The overall project status for Gates 1–4 is **exactly** `PARTIALLY CLOSED — WINDOWS OUTSTANDING`; the bare word
   `CLOSED` may not appear as a gate's overall status before Gate-5.

---

## 13. Traceability Matrix

Columns: **ID · Finding · Source Location · Required Change · Test ID · Acceptance · Evidence**

### 13.1 F-1

| ID | Finding | Source Location | Required Change | Test ID | Acceptance | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| **F1-R1** | F-1 | `sqlite_store.py:160`; `sqlite_operation_ledger.py:153`; `sqlite_state_store.py:33` | WAL transition via `open_sqlite_connection()` — all three | T-F1-01, 04 | 3/3 single; 0/300 per adapter | E-1 56/120, 23/40, 22/40; E-2 11/250 |
| **F1-R2** | F-1 (**N-5**) | `sqlite_store.py:165`; `sqlite_operation_ledger.py:155`; `sqlite_state_store.py:31` (declares none, inherits 10 s) | **declare and own** the lock-wait policy in one place | T-F1-13 | effective `busy_timeout` explicit and identical (10 000 ms) | P-1, P-4, PR-3 |
| **F1-R3** | F-1 (**C-07**) | helper loop | planned backoff must not exceed remaining; **actual measured and reported, not bounded**; remaining recomputed after each sleep; no retry after expiry | T-F1-07, 09, 16, **17** | I-1…I-7 hold | §3.4.5 |
| **F1-R4** | F-1 (**C-07**) | helper loop | no unbounded retry; `budget + 2 s` **only** in T-F1-16 | T-F1-16, 17 | I-4 holds; no other test uses `budget + 2 s` | §3.9.3 |
| **F1-R5** | F-1 | three constructors | fail closed with a typed, diagnosable error | T-F1-07, 10, 19, 20, 22a-c | taxonomy correct; fields present | §3.5 |
| **F1-R6** | F-1 | three constructors | close the connection on every failure path | T-F1-08 | 0 leaks / 100 | D-5 |
| **F1-R7** | F-1 | three constructors | verify `journal_mode == 'wal'` | T-F1-01, 12 | 100 %; forced fallback raises | no existing test |
| **F1-R8** | F-1 | three constructors | defined behaviour S1…S17 | T-F1-01…06, 09, 13, 15, 18, 21, 22a-c | §3.7 covered | §3.7 |
| **F1-R9** | F-1 | three constructors | no parallel DB / deletion / path change | **T-F1-11** | three-way classification (C-09) | OD-2 |
| **F1-R10** | F-1 | all three | one policy, one helper | T-F1-13 + review | `PRAGMA journal_mode=WAL` appears **once** in `src/` | P-1 |
| **F1-R11** | F-1 | §2.4 | the Live lock is not the remedy (documented) | — (doc) | §2.4 present | E-2 |
| **F1-R12** | F-1 | `sqlite_operation_ledger.py:141–146` | shared-journal branch must **not** call the helper **and must not alter the borrowed policy** | **T-F1-13**, **V-12** | borrowed `busy_timeout` identical before/after; zero `busy_timeout` statements in the ledger | P-8, D-8, C-01 |
| **F1-R13** | F-1 | `:158` (`False`); `:151`, `sqlite_state_store.py:31` (default `True`) | `check_same_thread` explicit; **per-adapter value frozen** | T-F1-15 | journal usable cross-thread; ledger/state raise `ProgrammingError` | P-5 |
| **F1-R14** | F-1 (**C-06**) | helper API | **OD-1 30 s budget as a module constant with NO public parameter**; tests patch the constant only | T-F1-16, 17 | no call site can pass a budget; `WAL_INIT_BUDGET_S` defined once | OD-1, §3.4.3 |
| **F1-R15** | F-1 (**C-02 — UPDATED**) | helper classifier | `primary = code & 0xFF`; retry **only {5 family}**; `6`/`262` fail fast; **text fallback: `database table is locked` ⇒ LOCKED (matched first), `database is locked` ⇒ BUSY (retry, code-absent only), anything else ⇒ fail closed** | **T-F1-19, T-F1-20, T-F1-22a, T-F1-22b, T-F1-22c** | 6/6 codes + 3/3 text cases | P-2, PR-1, §3.4.2 |
| **F1-R16** | F-1 (**C-01 — REWRITTEN**) | `sqlite_operation_ledger.py:191–194`, `:209–213` | **one lock-wait owner per connection = its creator**; `migrate()` owns nothing; `:192`–`:194` **deleted**; migration bounded per statement at 10 000 ms; no total-migration-time claim | **T-F1-13, T-F1-18, V-12** | zero `busy_timeout` statements in the ledger; bounded per-statement wait; controlled error | OD-4, P-1, P-6, P-8 |
| **F1-R17** | F-1 (**C-01/C-04**) | `:208–213` | **`conn.in_transaction` checked before `BEGIN IMMEDIATE`**; begin only when needed; BUSY/LOCKED when not in a transaction propagates | **T-F1-18** | no silent `pass`; cause present | N-2, P-3, A-8 |
| **F1-R18** | F-1 (**C-04**) | helper + adapters | **4-way taxonomy + `SqliteMigrationBusyTimeout`**; two-level chain for migration; message token "database" in all; **AU-35 untouched** | T-F1-07, 10, 18, 19, 20 + AU-35 | every class reachable; chain asserted both levels; AU-35 green | §3.5, PR-2 |
| **F1-R19** | F-1 (**C-03 — REVISED**) | `:307`, `:313` | both independent backup connections get **explicit documented `timeout=`**; cleanup verified; **no total-backup-duration claim**; no watchdog | **T-F1-21** | configuration + cleanup only | P-4, P-7, PR-4 |
| **F1-R20** | F-1 (**C-05**) | helper entry | `mkdir` boundary documented as **Proposal P-1, pending**; **Q-12 open** | — (doc) | §3.4.1 present; OD table shows WITHDRAWN | C-05 |
| **F1-R21** | F-1 | §3.9 | document what sqlite3 timeouts do **not** guarantee; `budget + 2 s` restricted; deferred requirements recorded | — (doc) | §3.9 present; no contrary claim elsewhere | §3.9 |
| **F1-R22** | F-1 (**new — C-03**) | `:309` | record that `Connection.backup()` runs its own retry loop (`sleep=0.25`) and is **not** bounded here | T-F1-21 (documentation assertion) | D-13 recorded; RQ-BACKUP-DEADLINE raised | P-7, PR-4 |
| **F1-R23** | F-1 (**new — C-11**) | `:207`, `:298–326` | Backup receives its **own deadline**, independent of the 30 s WAL budget, implemented with `pages=1` + a `progress=` callback that raises `SqliteBackupTimeout`; source re-armed to `busy_timeout=0` and **restored in `finally`** | **T-F1-23** | deadline holds; handler restored; file deleted | **PR-9b, PR-11, PR-16, PR-17, PR-22** |
| **F1-R24** | F-1 (**new — C-11**) | `:306` | a timed-out backup **deletes** its partial target and any sidecar; never leaves a file whose validity is unknown | T-F1-23 | `target` absent after failure | PR-12, PR-18, PR-19, PR-21 |
| **F1-R25** | F-1 (**new — C-11**) | §3.10 | the backup deadline **value** is `OWNER DECISION REQUIRED`; the symbol `BACKUP_DEADLINE_S` is used everywhere; no literal appears | — (doc) + review V-14 | document contains no chosen value | OD-6, Q-9 |
| **F1-R26** | F-1 (**new — C-11**) | §3.10.5 | backup source-connection choice (reuse vs independent) is recorded as a **Proposal**, pending **Q-14**; the C-01 ownership tension is stated, not hidden | review | §3.10.5 present; Q-14 open | C-01, Q-14 |
| **F1-R27** | F-1 (**new — C-13**) | `migrate()` A8–A13 | **BUSY and LOCKED are separated**: BUSY → retried → `SqliteMigrationBusyTimeout`; LOCKED/ambiguous → **fail-fast** → `SqliteMigrationLockError`, never a timeout | **T-F1-18, T-F1-24** | two distinct types reachable; LOCKED `attempts == 1`; message tokens correct | PR-5a, PR-6, PR-5d, PR-23, Q-13 |
| **F1-R28** | F-1 (**new — C-16**) | §3.4.2 | the text fallback is anchored to **observed** runtime messages by a real-SQLite test, not to mocked exceptions | **T-F1-25** | both families asserted with code + name + text | PR-5a, PR-6 |
| **F1-R29** | F-1 (**new — C-12**) | §3.4.6 | the retry procedure is specified as an executable algorithm: one connection **re-armed** per attempt, per-attempt `busy_timeout = min(remaining_ms, 10 000)`, final `busy_timeout=10000` after verification, close on **every** failure path, **late success = failure** | **T-F1-16, T-F1-17** | every line P-1…P-15 has a test or a review item | PR-7, PR-8 |
| **F1-R30** | F-1 (**new — C-17**) | §3.4.7 | units are normative: `connect(timeout=)` **s**, `PRAGMA busy_timeout` **ms**, `backup(sleep=)` **s**, deadline **s**; every internal field carries `_ms`/`_s` | review **V-14** | no literal passed to `connect(timeout=)`; no non-integer `busy_timeout` | PR-20 |

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
| **F2-R8** | F-2 | new tests | simulation labelled **same-transaction interleaving**; no cross-process claim | all F-2 docstrings | §4.4 wording honoured | §4.4 |
| **F2-R9** | F-2 | new test | **lock-level witness** catches `BEGIN IMMEDIATE` removal | **T-F2-03** | M-2 → 100 % | PR-1 |
| **F2-R10** | F-2 | A7 file | real multi-process test kept, labelled **P**, never a witness | T-F2-04 | reported as a rate | §4.4 |
| **F2-R11** | F-2 | new + rewritten F-2 tests | **strict thread ownership**; `check_same_thread` never changed for tests | T-F2-01/02/03 + §7.4 check | no cross-thread connection use | P-5 |

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
| **F3-R9** | F-3 | every report | Linux and Windows in **separate** columns | — | enforced | enforced |

### 13.4 Out-of-Scope Findings (recorded, not addressed)

| ID | Finding | Location | Why out of scope | Owner decision |
| --- | --- | --- | --- | --- |
| **OOS-1** | State store declares no lock-wait policy (inherits 10 s) | `sqlite_state_store.py:31` | **In scope** via F1-R2 | — |
| **OOS-2** | Atomicity class of `claim_for_send`/`mark_submit_attempt` undocumented and mis-stated in the brief | `:562`, `:594` | Doc-only (F-2-D7) | Gate-2 |
| **OOS-3** | 58 `SqliteJournal(` sites, no factory | `src/` 12, `tests/` 46 | Architectural | PHASE C |
| **OOS-4** | `check_same_thread=False` on the journal connection | `sqlite_store.py:158` | Thread-safety not assessed; **value frozen** | PHASE C |
| **OOS-5** | CLI/web open the same DB as a running Live process with no lock | §2.4 | Ownership model | PHASE C |
| **OOS-6** | `LiveInstanceLock` `msvcrt`/`fcntl`; no Windows test | `live_instance_lock.py:44–62` | Not F-1/F-2/F-3 | Gate-5 |
| **OOS-7** | No test asserts `journal_mode == 'wal'` | whole suite | **Inside** F-1 (T-F1-01/12) | Gate-1 |
| **OOS-8** | `migrate()` backup contention (~20 s stall per handover §5) | `:185–220` | Recorded as mitigated by the handover; **not re-verified** (U-4) | Gate-1 review |
| **OOS-9** | **N-3 — carried forward:** `_backup()` opens two connections with no explicit timeout (`:307`, `:313`), inheriting the **5.0 s** stdlib default | `:298–325` | **Promoted to F1-R19** with the **C-03 limitation applied**: explicit timeout + documentation only; **no bounded-backup claim** | Q-9 |
| **OOS-10** | `mkdir` latency unbounded | helper entry | **P-1, pending confirmation** | **Q-12 (blocks Gate-1)** |
| **OOS-11** | `Connection.backup()`'s internal BUSY retry loop is unbounded by this codebase | `:309` | **RQ-BACKUP-DEADLINE** deferred; no watchdog approved | Q-9 |

---

## 14. Open Questions / Assumptions

### 14.1 Assumptions and Unknowns

| ID | Type | Statement | Impact if false | How to check |
| --- | --- | --- | --- | --- |
| **A-1** | Assumption | The Linux-measured F-1 rate characterises the **mechanism**, not the platform's rate | The 300-sample may be mis-powered on Windows | Gate-5 run |
| **A-2** | Assumption (narrowed) | `sqlite_errorcode` carries the extended code on the production runtime | `& 0xFF` becomes inert; the text fallback carries the load | **PR-1 confirms it on 3.13**; version caveat remains |
| **A-3** | Assumption | A `spawn` child under pytest can import the test module | The mandatory `sys.path` guard mitigates | T-F3-06 |
| **A-4** | Assumption | No caller relies on catching `sqlite3.Error` around construction | A start could crash where it previously printed a message | **§3.5.5 inventory is the Gate-1 condition** |
| **A-5** | Assumption | The same-transaction simulation is a faithful stand-in **for the column-targeting property only** | F-2's claim would be weaker | §4.4 |
| **A-6** | ~~Assumption~~ | ~~30 s acceptable for CLI~~ | — | **DELETED — replaced by OD-1** |
| **A-7** | Assumption | A hard wall-clock cap over DDL is not required for Phase A | The budget does not bound DDL | Closed by **OD-4**; residual = RQ-HARD-CAP |
| **A-8** | Assumption | `in_transaction` faithfully reports state for explicit `BEGIN IMMEDIATE` and is not confused by `executescript`'s implicit commit | F1-R17's pre-check could mis-decide | T-F1-18 asserts the pre-check's value |
| **A-9** | Assumption | The three per-adapter lanes are **independent samples** | The Bonferroni arithmetic would be wrong | distinct `tmp_path` roots by construction; asserted in T-F1-04 |
| **A-10** | Assumption (**new — C-05**) | `mkdir` is intended to sit outside the 30 s budget | If the owner wants it inside, P-1 must become a requirement and the deadline token must be created **before** `mkdir` | **Q-12 — must be answered before Gate-1** |
| **A-11** | Assumption (**new — C-03**) | Phase A does not require a bounded total backup duration | Startup can, in principle, stall inside `backup()` | **RQ-BACKUP-DEADLINE**; Q-9 |
| **U-1** | **Unknown** | The real F-1 rate on Windows file locking | Gate-5 planning only | **NOT RUN** |
| **U-2** | **Unknown** | Whether elevated `busy_timeout` on Windows changes user-visible startup latency | — | **NOT RUN** |
| **U-3** | **Unknown** | Whether any deployment runs GoldBot over a network share | §3.9.2's filesystem-stall caveat becomes material | Ask the owner |
| **U-4** | **Unknown** | Actual duration of a real v1→v3 migration on a large ledger | Migration policy adequacy unquantified | OOS-8; needs a measured run |
| **U-5** | **Unknown (C-10)** | Which of `18.41 s` (handover) and `18.31 s` (shipped log) describes the extracted run, and why the handover omitted the warning count | No acceptance criterion depends on either | §2.10 D-1, D-2 |
| **U-6** | **Unknown — PARTIALLY RESOLVED in v1.4** | *v1.3: "Whether the true total backup duration can be bounded without out-of-band termination."* **v1.4: it can, cooperatively** — `pages=1` + `progress=` + our own exception (PR-11, 2.01 s against 2.00 s). What remains unknown is whether the **cooperative** bound is tight enough on a real, large ledger, and whether a single page step can itself stall arbitrarily. | the deadline may overshoot by up to one page step; §3.9.2 still applies | **U-7**; needs a measured run on production-sized data |
| **U-7** | **Unknown (new — C-11)** | The **throughput cost** of `pages=1` on a production-sized ledger versus `pages=-1`, and therefore the realistic overshoot magnitude | The owner cannot size `BACKUP_DEADLINE_S` (OD-6) without it | needs a measured backup on production-sized data — **not** performed here |

### 14.2 Open questions (each blocks a gate)

| ID | Question | Default if unanswered | Blocks |
| --- | --- | --- | --- |
| **Q-1** | Is Gate-1 authorized as specified (helper + three adapters + deletion of `:191–194` + F1-R17/R19)? | no work begins | Gate-0 |
| **Q-2** | Acceptance lane **300 per adapter** (reported separately), as retained? | 300 per adapter, separate claims | Gate-1 close |
| **Q-3** | ~~5 s vs 30 s~~ | — | **CLOSED — OWNER DECISION (OD-1)** |
| **Q-4** | May the journal path change exception type (needs the A-4 inventory), or must it keep `sqlite3.OperationalError` and only enrich the message? | new type, subject to the inventory | Gate-1 |
| **Q-5** | Mutation harness inside `tests/`, or under `tools/` and acceptance-only? | `tools/`, acceptance-only | Gate-2 |
| **Q-6** | Which logger name/level for the initialization INFO line? | `goldbot.adapters.sqlite_init`, INFO | Gate-1 |
| **Q-7** | Expose connection state via `migration_report()`? | no (§8) | Gate-2 |
| **Q-8** | ~~Hard cap over DDL?~~ | — | **CLOSED — OWNER DECISION (OD-4)**; residual = RQ-HARD-CAP |
| **Q-9** | Backup: is F1-R19 (explicit timeout + documentation, **no duration claim**) sufficient for Gate-1, with RQ-BACKUP-DEADLINE deferred? | yes, limited scope; defer | Gate-1 |
| **Q-10** | **Re-open condition:** if shared cache is ever introduced, must the `SQLITE_LOCKED → non-retryable` decision be revisited? | yes, mandatory | future |
| **Q-11** | Three **separate** 95 % claims at n = 300 each, or one **joint** claim requiring n ≈ 408 per adapter? | three separate claims at 300 | Gate-1 close |
| **Q-12** | **`mkdir` boundary (C-05): is `path.parent.mkdir` outside the 30 s budget — confirmed, or must it be brought inside?** | **NO DEFAULT. Unanswered ⇒ Gate-1 cannot open.** | **Gate-1** |
| **Q-13** | Approve the **`SqliteMigrationBusyTimeout` / `SqliteMigrationLockError` → `LedgerUnavailable`** two-level chain (C-04, C-13), or choose a different design with the three safeguards in §3.5.2? | prefer the stated chain | Gate-1 |
| **Q-14** | **Backup source connection (C-11):** reuse the ledger's borrowed connection (**Proposal A**, requires temporarily re-arming it to `busy_timeout=0` and restoring it in `finally`, which makes the ledger a second policy owner and therefore **tensions C-01**), or open an **independent source connection (Proposal B)**, which respects C-01 but adds a handle and a close path? | **NO DEFAULT** — A is written up as a Proposal, not a Decision | **Gate-1** |
| **Q-15** | **Backup deadline value (OD-6):** what wall-clock value should `BACKUP_DEADLINE_S` take? No value can be recommended until **U-7** measures real backup duration. | **NO DEFAULT** | **Gate-1** |
| **Q-16** | **Migration retry (C-13):** should the BUSY branch inside migration retry at all, or fail on the **first** BUSY (the per-statement 10 000 ms handler already waits once)? Retrying multiplies the worst-case migration time by the attempt count. | retry, bounded by the per-statement handler | Gate-1 |

### 14.3 Explicitly NOT established by this document

* Whether the **current** implementation contains a Lost Update — **no evidence was found that it does**.
* Whether F-1 occurs on **Windows**, or at what rate.
* Whether 30 s is adequate in production — it is an **Owner Decision**, not a measured optimum.
* Whether any test listed here **passes** — **no project test was executed for v1.0–v1.4.** Only the stdlib probes
  in §2.9 (four, authoring) and §2.9.1 (twenty-two, v1.4 runtime evidence) were run, and they exercise **library
  behaviour on throw-away files**, not this project.
* The real wall-clock duration of any migration **or of any backup**, and therefore the value OD-6 needs (**U-7**).
* Whether `SQLITE_LOCKED` can be raised by **this project's** code — PR-23 says no *in the present code*, but that is
  evidence about today's cursor discipline, **not** a guarantee about future code.
* Whether the cooperative backup deadline (§3.10) is tight enough on a production-sized ledger.
* Which of the two reported durations for the extracted run is authoritative (§2.10).
* Any statement about MT5, the broker, demo execution, soak behaviour, Cross-VPS fencing or production readiness.

---

## 15. Confirmation of No Change

| Check | Result |
| --- | --- |
| Source files modified | **None** |
| Test files modified | **None** |
| Schema / DDL / migration executed or changed | **None** |
| Project tests executed | **None** |
| Implementation ZIP created | **None** |
| Source committed or pushed | **None** |
| MT5 / broker / VPS / Live settings changed | **None** |
| Demo or Live orders executed | **None** |
| Windows PASS claimed | **None** — `NOT RUN` everywhere |
| Production readiness claimed | **None** |
| Gate-1 / Gate-2 / Gate-3 opened | **None** |
| Delivery integrity at time of writing | `sha256sum -c SHA256SUMS.txt` → **316/316 OK**; ZIP `9375170378…52d1758d` unchanged |
| Content hash of every delivered file | `SOURCE-MANIFEST.txt` recomputed: **299/299 identical**, including **205/205 `.py` files** |
| Code executed for v1.4 | **No project code.** 22 additional **stdlib-only** probes (§2.9.1, PR-5…PR-26) were run on throw-away files in `tempfile.mkdtemp()` for C-11/C-13/C-16/C-17. **No project module was imported, no project database opened, no project test run.** |
| Probe count (C-10, C-14) | **Four** authoring probes (`PR-1`…`PR-4`) in §2.9, **plus** twenty-two runtime probes (`PR-5`…`PR-26`) in §2.9.1. The §2.9 heading, which still read "three" in v1.3, now reads **four**; a whole-document scan for `three probes` / `3 probes` / `two probes` returns **zero live claims**. |
| Status of every capability | **`DESIGNED` at most** — §1.1. Nothing is IMPLEMENTED, TESTED or VERIFIED. |

**Disclosure — `mtime` artefact.** Every file in the extracted delivery carries `mtime = ctime = the workspace
snapshot-restore timestamp`, so a naive `find -newermt` flags all 205 `.py` files as "recently modified" even
though none was touched. The decisive check is **content**, not timestamps: 299/299 manifest entries and 205/205
`.py` files are byte-identical, and `SHA256SUMS.txt` is 316/316. The only new artefact of this round is this
specification file.

---

**End of specification v1.4.**

```
SPECIFICATION REVISION ONLY
NO SOURCE FILES MODIFIED
NO PROJECT TESTS EXECUTED
NO IMPLEMENTATION ZIP CREATED
NO GATE-1 AUTHORIZATION
AWAITING OWNER REVIEW OF v1.4
```
