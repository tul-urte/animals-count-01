# Session Queue — the front door (read first; the record lives in notes/)

The pair to [session_notes.md](./session_notes.md) (the record, S113 2026-11-18).
This file is the PLAN: what is open, in what order, what depends on what.
It is **not a state document** — a row here is a claim from a session's
close, verified only when the work actually runs (the handoff-drift law).

## The laws

1. **SESSION START**: read this file; the top of NOW is the suggested
   agenda — the user steers.
2. **SESSION CLOSE** (three writes, the session is not finished without
   all three): (a) `notes/YYYY-MM-DD S###.md` + ledger row in
   session_notes.md; (b) this file — new carries IN, retired items OUT
   (one-line note under DONE/RETIRED), `last-confirmed` stamps refreshed;
   (c) the next session's suggested follow-ups land at the TOP of NOW.
3. **Only the user's verbatim command moves a gate** (LIVE deploys above
   all; the 2026-11-18 ruling).
4. Big designs get their own `plan-*.md`; the queue links to them rather
   than restating them.

## Entry shape

`Q# — name`: one line of what; `origin:` S### (date); `priority` +
`depends-on`/`blocks`; `last-confirmed:` the last session that touched or
re-verified the work; `finish-line:` the evidence that closes it (a probe,
a suite, a rendered page) — never "it ran once".

## NOW (the natural carries; top = the agenda for the next session)

**G1 — the LIVE deploy of the design era: WAITING ON THE USER'S CALL.**
The qa-bank gate is GREEN (S112: batteries 21+22+18; consistency 33+9;
validate_pages 127/0/0; the UAT walk 37/37 — all on the WSL qa estate
after --fresh). The ladder's dev leg rode green earlier (S110h).
- *origin*: S110h (2026-11-18) close; UNBLOCKED by S112 (2026-11-18).
- *the command shape*: `python 06-deployment/imagebank/pi_deploy.py
  --live` is the USER'S verbatim command (the assistant NEVER launches
  it; `--check` is the assistant's tool); the restart afterwards is the
  by-PID idiom; schema rides pi_migrate.py after the deploy.
- *finish-line*: the live routes answer; the product's OWN probes (the
  sitemap's 8 cards, the dialog's chain, the stamped nav) — never
  "server restarted + login 200".

**Q3 — the popup's gesture face: `$op`-through-the-frame as the true test cockpit.**
- *origin*: S110f (2026-11-18) close; *priority 3*.
- *last-confirmed*: S112 (2026-11-18) — carried untouched.

**Q4 — the first userRole consumer: a pages.js-driven renderer that
hides/shows on `$userRole`.**
- *origin*: S110h (2026-11-18) close; *priority 4*; "the natural first
  consumer when a suite wants it" — not before it's needed.
- *last-confirmed*: S112 (2026-11-18) — carried untouched.

**Q5 — the remaining uat_* suites' honest re-cut onto the verbs**
(uat_fammap.js, uat_retrain_lane.js still ride the compat faces).
- *origin*: S110f close (the Q2 umbrella's remainder); *priority 5*;
  "when a suite is next touched" (the S112 precedent: the touched
  suites ride the verbs fully now).

**Q6 — the design model's META-SPEC** (the keys + the prose vocabulary
+ the db outside the code dir; the plan:
[plan-design-meta-spec.md](./plan-design-meta-spec.md)).
- *origin*: S113 (2026-11-18, the review thread); the user's rulings:
  battery-enforcement only; d:anatomy yes; d:rules prose-only leg 3;
  d:algorithm held; the db's move is the user's note (Part D).
- *last-confirmed*: S113 (2026-11-18) — **leg 1 DONE + COMMITTED**
  (a912e92: d:anatomy built; the master's 3 splits; the sitemap+dialog
  render it; BOTH estates' full sweeps green; the master 18362a4a).
  Two live-proven defects rode it: the DEAD prose TEXT-ONLY refusal
  (fixed: the composite marks the open lease's kids; A5b pins it);
  the --skip-estate db wipe (the stagekeep guard interim; leg 4 is
  the real fix). The S112 'no such table: users' was the same
  defect's first catch — the misdiagnosis is corrected in the plan.
- *legs*: 2. the key law's battery (Part A's checks; the renames
  case-by-case); 3. d:rules + the derived-gates battery (+ the
  $state consumer); 4. **the db outside the code dir** (Part D; the
  live migration rides the LIVE gate).


## OPEN, OLDER (verify before use; last-confirmed = origin session)

**O1 — RETIRED by Q1's run (S112)**: the consistency suites' live
rehearsal rode the qa-bank sweep (33+9 green on the fresh estate).

**O2 — S093 BRAIN-BANK items (2026-11-15)**: the user's two ideas for a
next session, recorded in [2026-11-15 S093 BRAIN-BANK.md](../notes/2026-11-15%20S093%20BRAIN-BANK.md).
- *re-read when idle; the user decides if either is still wanted.*

**O3 — picods label-thresholds.json on picods101 is kit-default** after
the re-flash; retune at the panel if needed.
- *origin*: S073 (2026-11-12) close; *node-side, low*.

**O4 — test_housekeep_why2.py's tail prints an ssh probe's noise** (a
manual diagnostic riding run_all's tally; cosmetic, run_all still
counts it "ok").
- *origin*: S112 (2026-11-18) close; *low; cleanup when the tests' dir
  is next touched.*

## DONE / RETIRED (the short tail log)

- **2026-11-18 S113**: Q6's leg 1 (d:anatomy built + committed a912e92;
  the meta-spec plan landed; the dead TEXT-ONLY refusal fixed; the
  --skip-estate db wipe caught+guarded; Part D the db-out-of-code
  note at the user's naming).
- **2026-11-18 S112**: Q1 (the qa-bank rehearsal, gates green: 21+22+18
  + 33 + 9 + 127/0/0 + the UAT walk 37/37) + Q2's touched-suite legs
  (consistency_menus/states/uat_glitch_match ride the compound + the
  verbs). Commit 619f36b + the master's tag-honesty fixes (the
  h:h3 whitelist; validate_pages 97 misses -> 0).
- **2026-11-18 S111**: the notes re-org (one file per session + the index
  + this queue file). Done (ee0b8df + a805068).
