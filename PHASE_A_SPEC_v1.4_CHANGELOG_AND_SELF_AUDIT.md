# PHASE A — SPEC v1.4 CHANGELOG AND SELF-AUDIT

**Document reviewed:** `PHASE_A_WAL_INITIALIZATION_AND_CONCURRENCY_REMEDIATION_SPEC_v1.4.md`
**Supersedes:** v1.3 (`0fb6487b4261dd242b53090b9a323a2e8513c08aec1108d088c620211807c1f5`)
**v1.4 SHA-256:** `2abfd63cedb28d2d5dd5b10367021c6f632b63f485cdb4e27b17393bd5a85098`
**Size:** 1 904 lines · 160 519 bytes
**Date:** 2026-11-26
**Nature:** Specification revision only.

---

## 1. Disposition of each assigned correction

| Case | Subject | Status | What was actually done |
| --- | --- | --- | --- |
| **C-11** | Backup deadline in the migration path | **FIXED** (design) / **OPEN — OWNER DECISION REQUIRED** (value) | See §2 below. |
| **C-12** | Exact WAL initialization deadline algorithm | **FIXED** | §3.4.6 — 15 numbered steps P-1…P-15, literal Python, one connection re-armed per attempt, per-attempt `busy_timeout` derived from remaining, close on every failure path, late success treated as failure. |
| **C-13** | BUSY vs LOCKED separation in migration | **FIXED** | §3.5.1 split, §3.5.2 dual chain, §3.5.2.1 12-row mapping table, A14 corrected, T-F1-24 added, F1-R27. |
| **C-14** | Probe count | **FIXED** | §2.9 heading corrected `three` → **four**; whole-document scan returns zero live "three/two probes" claims. No probe invented to fix a count. |
| **C-15** | T-F1-16 `budget_s` wording | **FIXED** | Precondition 5 now reads `monkeypatch.setattr(sqlite_init, "WAL_INIT_BUDGET_S", 1.0)`; explicit statement that no `budget_s` argument is passed. |
| **C-16** | Runtime evidence for BUSY / LOCKED | **FIXED** | Both families **observed**, not assumed. §2.9.1 records Python 3.13.18 / SQLite 3.46.1 and the exact code/name/text. T-F1-25 keeps the assertion in the suite. |
| **C-17** | Timeout units | **FIXED** | §3.4.7 normative table + `_ms`/`_s` naming rule + PR-20 hazard demonstration + review item V-14. |
| **Q-12** | `mkdir` decision | **OPEN — OWNER DECISION REQUIRED** (unchanged) | Still **Proposal P-1**; still blocks Gate-1; still no default. **Not touched by this revision except to keep it open.** |
| Other | New contradictions discovered | see §5 | 5 new questions raised (Q-14…Q-16), 1 new unknown (U-7), 2 v1.3 claims withdrawn. |

---

## 2. C-11 in detail — the most consequential change

### 2.1 The defect in v1.3

`_backup(version)` is invoked at `sqlite_operation_ledger.py:207`, **before** `BEGIN IMMEDIATE` at `:209`.
Backup is therefore **inside** the migration path, not adjacent to it. v1.3 recorded `RQ-BACKUP-DEADLINE` as
*"deferred, unapproved, not designed"* — leaving an operation inside migration that OD-4 requires to be bounded.

Worse, v1.3 believed this was a theoretical gap. **It is not.** PR-16 ran `backup()` under a held `BEGIN EXCLUSIVE`
against a DELETE-mode file and killed it at an **external 25-second wall clock while it was still running**. The
call does not merely retry slowly; it **does not return**.

### 2.2 Why `sleep=` and `connect(timeout=)` cannot be the deadline — demonstrated

| Claim | Why it is true | Probe |
| --- | --- | --- |
| `sleep=` sets the **gap between attempts**, nothing else | 0.6 s window: `sleep=0.0` → 8 252 callbacks; `sleep=0.25` → **4**. Same non-termination, different rate. | PR-22 |
| `connect(timeout=)` bounds **one** busy-handler wait | `sleep=0.05` survived an 8 s wall (PR-9b) and a 25 s wall (PR-16) | PR-9b, PR-16 |
| `pages=-1` — **the current call at `:309`** — gives **no** intervention point | 2 M rows copied in 0.016 s with **1** callback | PR-17 |
| `progress=` alone is unreachable while blocked | with the inherited 10 s handler the first callback arrived at **5.01 s** | PR-10b |

**Conclusion: no single stock parameter is a deadline.** v1.3 was right to forbid the claim and wrong to stop there.

### 2.3 What works — and was measured

Independent source connection with `busy_timeout=0`, `pages=1`, `progress=` raising our own exception, deadline
**2.00 s** → **aborted at 2.01 s (+0.01 s overshoot)**, 41 callbacks, destination 0 B. (**PR-11**)

This is **cooperative**, not preemptive: the check happens between page steps, so up to one page step plus scheduler
latency can overshoot. §3.9.2 applies unchanged. It is strictly better than `sleep=` (which has no bound at all) and
it is the best the standard library offers **without** out-of-band termination (RQ-HARD-CAP — still unapproved).

**Status: DESIGNED. Not implemented. Not tested against project code.**

### 2.4 Partial file on timeout

| Observation | Consequence |
| --- | --- |
| PR-12 / PR-18 / PR-21 → **0 bytes, no tables** | deletion is simple |
| PR-19 → abort landed **after** the copy finished; file was a **complete** backup under a timeout exception | **deletion is mandatory, not cosmetic** |
| **No** run produced a partially-populated, openable, `integrity_check`-clean but silently-wrong backup | stated as **evidence, not proof** — which is exactly why §3.10.4 mandates `_unlink_partial()` |

### 2.5 What the owner must still decide

| ID | Question | Why this document cannot answer it |
| --- | --- | --- |
| **OD-6 / Q-15** | the **numeric value** of `BACKUP_DEADLINE_S` | depends on **U-7** — measured backup duration on production-sized data, which was **not** performed here. Any number in this document is the **symbol**, never a decision. |
| **Q-14** | reuse the ledger's borrowed connection (**A**, written up as Proposal) or open an independent source (**B**) | **A tensions C-01** (it makes the ledger a temporary second policy owner). The document gives the trade-off; the owner must choose. |
| **Q-16** | should migration BUSY retry at all, given each statement already waits 10 000 ms? | a policy choice that multiplies worst-case migration time |

---

## 3. Owner Decisions — verification that nothing changed

| ID | Status in v1.4 | Changed by this revision? |
| --- | --- | --- |
| OD-1 — 30 s WAL budget, all adapters, no per-surface override | CONFIRMED — BINDING | **No** |
| OD-2 — five-operation scope | CONFIRMED — BINDING | **No** |
| OD-3 — `mkdir` | **WITHDRAWN → Proposal P-1** | **No** — remains withdrawn, remains Q-12 |
| OD-4 — schema/migration outside the 30 s; lock waits bounded & detectable | CONFIRMED — BINDING | **No** |
| OD-5 — documentation only | CONFIRMED — BINDING | **No** |
| OD-6 — backup deadline value | **OPEN — OWNER DECISION REQUIRED** | **Added as OPEN, not as a decision** |

**The 30-second figure was not altered, reinterpreted, extended or shortened anywhere.** The backup deadline is
**structurally independent** of it — separate token, separate start, sequential (never nested) execution. Verified by
the scope table in §3.10.6.

**Nothing was silently promoted.** Three things were deliberately left unpromoted: the `mkdir` boundary (P-1), the
backup value (OD-6), and the backup source-connection choice (Q-14).

---

## 4. C-13 in detail — a v1.3 claim that had to be withdrawn

v1.3 justified `SQLITE_LOCKED → non-retryable` with *"P-2 proves shared cache is never enabled, so cross-process
contention cannot produce 6"*, and then implied 6 was unreachable. **Two separate findings:**

1. **The rule is right; the reasoning was too strong.** PR-5a produced a **real** code 6 — `OperationalError`,
   `sqlite_errorcode = 6`, `sqlite_errorname = 'SQLITE_LOCKED'`, message **`database table is locked`** — with **no
   shared cache**, merely an unexhausted read cursor followed by `DROP TABLE`. Reachable in principle.
2. **Not reachable in this codebase today.** PR-23 checked every cursor touching a migration: `_column_names()`
   (`:272–273`), `_has_ledger_table()` (`:164`), `sqlite_store.py:194` — all **fully materialised** set
   comprehensions before any DDL. A materialised cursor + `ALTER TABLE` produced no error.

**Net effect on the design: none.** 6 stays fail-fast — but now for the *right* reason: it signals a programming
error, and fail-fast is exactly right for a programming error. What changed is the **justification** and the
**honesty of the document**.

**And the contradiction is gone.** v1.3's A14 routed "BUSY/LOCKED" into a single `SqliteMigrationBusyTimeout`,
contradicting its own taxonomy. v1.4 raises `SqliteMigrationLockError` for 6/262, **never** enters the BUSY loop,
`attempts == 1`, and the message must contain `locked` and must **not** contain `timeout`/`exhausted`.

**Runtime identity, both families, observed:**

| Family | Exception | code | name | message |
| --- | --- | --- | --- | --- |
| BUSY | `sqlite3.OperationalError` | **5** | `SQLITE_BUSY` | `database is locked` |
| LOCKED | `sqlite3.OperationalError` | **6** | `SQLITE_LOCKED` | `database table is locked` |

Both float on the **same** exception class — which is precisely why classification must be by **code first** and by
**text only when the code is absent**, and why the text patterns must be ordered. Confirmed by PR-6 and PR-5d.

---

## 5. New contradictions / issues discovered during this revision

| # | Issue | Handling |
| --- | --- | --- |
| **N-1 (v1.4)** | v1.3's `RQ-BACKUP-DEADLINE` said the deadline would need *"out-of-band termination"* — **wrong**. PR-11 shows cooperative `progress=`/`pages=` suffices for the **contention** case. | §3.9.4 rewritten; deferral narrowed to RQ-HARD-CAP only. |
| **N-2 (v1.4)** | v1.3 implied `SQLITE_LOCKED` was unreachable here. | Withdrawn; PR-5a/PR-23 recorded. |
| **N-3 (v1.4)** | The proposed backup design **contradicts C-01** by re-arming a borrowed connection. v1.3's C-01 rule is binding and this is a genuine architecture question. | Raised as **Q-14**, Proposal A vs B, **no default**. |
| **N-4 (v1.4)** | `pages=1` on a production ledger may be slow; nobody has measured it, so OD-6 cannot be sized responsibly. | **U-7** recorded; **Q-15** blocks. |
| **N-5 (v1.4)** | Whether migration should retry BUSY at all — each statement already waits 10 000 ms, so retrying multiplies worst-case migration time. | **Q-16** raised. |
| **N-6 (v1.4)** | A WAL-mode source **does not block** `backup()` at all (PR-15/PR-24/PR-25). Since A2 makes the ledger WAL **before** A7 runs, the hang requires the DELETE-mode window — which is exactly the brand-new-file window F-1 is about. | Recorded in §3.10.1 and **T-F1-23 precondition**: the test **must** use `journal_mode=DELETE`. |
| **N-7 (v1.4)** | A timed-out backup can leave a **complete** file (PR-19), so an operator could mistake a failure for a success. | §3.10.4 mandates deletion, does not rely on emptiness. |

---

## 6. C-14 — the counting error, and how it is prevented from recurring

v1.3's Change-Log row said **four**, but §2.9's **heading** still said **three**. The row was corrected; the section
was not. That is the same class of error C-10 was meant to eliminate.

**Fixed and verified:** heading = **four**; disclosure line = **four**; a whole-document scan for
`three probes` / `3 probes` / `two probes` returns **zero live claims** (three historical mentions remain inside the
Change Log, where they describe what was wrong).

**Discipline adopted:** the v1.4 runtime probes are a **separate table** (§2.9.1, PR-5…PR-26) with their own count
(twenty-two), so the authoring count can never be disturbed again by adding evidence. **No probe was created to
repair a number** — every probe added answers a question that C-11/C-13/C-16/C-17 raised.

---

## 7. C-15 / C-17 — the two silent-failure corrections

**C-15.** T-F1-16 precondition 5 now reads the literal call and states explicitly that **no `budget_s` argument is
passed**. Audit of every surviving `budget_s`: (i) the internal `InitDeadline` field — a dataclass field, not a
parameter; (ii) historical Change-Log text. Both labelled. **The helper signature contains zero `budget_s`.**

**C-17.** The units table is §3.4.7. The important part is not the table but the **hazard**:

```
sqlite3.connect(path, timeout=5000)   # intended 5 000 ms
  → PRAGMA busy_timeout = 5 000 000 ms = 5 000 SECONDS   (PR-20)
  → accepted silently; there is no exception to catch
```

**This is why the fix is a naming convention, not a comment:** every field and local carries `_ms` or `_s`, so a unit
error becomes a visible typo. Backed by review item **V-14**.

---

## 8. Explicit declarations

| Question | Answer |
| --- | --- |
| Was **only** the specification changed? | **Yes.** Two files were created: the v1.4 spec and this report. |
| Are the **source files** untouched? | **Yes.** `SHA256SUMS` → **316/316 OK**; `SOURCE-MANIFEST.txt` → **205/205 `.py` byte-identical**, 0 mismatches. |
| Are the **test files** untouched? | **Yes.** Not one test file was opened for writing; AU-35 in particular is unmodified. |
| Was **any project test** executed? | **No.** No `pytest`, no test module imported. |
| Was **any Migration or Database operation** performed? | **No.** No project database was opened, created, migrated or deleted. All 26 probes used throw-away files in `tempfile.mkdtemp()`. |
| Was **any implementation ZIP** created? | **No.** The delivery ZIP is unchanged: `9375170378…52d1758d`. |
| Is **Gate-1 still closed**? | **Yes.** Also Gate-2 and Gate-3. **No Gate-0 approval is claimed.** |
| Which Owner Decisions still need the owner? | **OD-6** (backup deadline value) · **Q-12** (`mkdir` boundary) · **Q-13** (migration chain) · **Q-14** (backup source connection) · **Q-15** (= OD-6) · **Q-16** (migration retry) · plus **Q-1, Q-9, Q-11** carried from v1.3 |

### Statements NOT made by this revision

* No claim of **Gate-0 approval**.
* No claim of production readiness, Windows PASS, or Demo/Live correctness.
* No claim that any test passes — **nothing was run against project code**.
* No claim that `SQLITE_LOCKED` is impossible in this project — only that it is **not reachable in the present code**.
* No claim that the cooperative backup deadline is a **preemptive** guarantee.

---

## 9. Files

| File | SHA-256 | Bytes |
| --- | --- | --- |
| `PHASE_A_WAL_INITIALIZATION_AND_CONCURRENCY_REMEDIATION_SPEC_v1.4.md` | `2abfd63cedb28d2d5dd5b10367021c6f632b63f485cdb4e27b17393bd5a85098` | 160 519 |
| `PHASE_A_SPEC_v1.4_CHANGELOG_AND_SELF_AUDIT.md` | *(this file)* | — |
| `…SPEC_v1.3.md` (superseded, retained) | `0fb6487b4261dd242b53090b9a323a2e8513c08aec1108d088c620211807c1f5` | 107 266 |
| `…SPEC_v1.2.md` (superseded, retained) | `6435482075b9793b479ad37f4eb803690d50ff0149866505e86977f97bdc0cc6` | 92 851 |
| `…SPEC_v1.1.md` (superseded, retained) | `d0d34f3fb6c59479b616348a79df8279217b7aa0171a69abb9d4037f296e3249` | — |

---

```
SPECIFICATION REVISION ONLY
NO SOURCE FILES MODIFIED
NO PROJECT TESTS EXECUTED
NO IMPLEMENTATION ZIP CREATED
NO GATE-1 AUTHORIZATION
AWAITING OWNER REVIEW OF v1.4
```
