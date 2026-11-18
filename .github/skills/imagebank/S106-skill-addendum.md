# S106 addendum — the QA-bank lives in the browser (the UAT law) +
# the slice-2 qa estate + the make-qa-bank exhaustion/fire/bank-guard
# laws (2026-11-16, 01:00-02:10)

## THE BRIEF the next session inherits (read this FIRST)

The last session's story END-TO-END, all through the LIVE BROWSER (the
user watched every stage; screenshots are the record):
    fresh slice-2 build (exhaust laws) -> login -> the assessed world
    (546 robin rows, CAM-02 2022) -> 9 reference rows ticked -> g-claim
    (confirm '9 selected') -> toast '9 of 9 done' (gclaim 0->9) -> the
    gclaim review: allocate -> 'no verses yet' -> analyse (the read
    clusters 5+2+2, the read never writes) -> seed 'g-2022-10' (5 of 5
    allocated, seed window opens) -> band tick (5) -> g-publish (5 of
    5, glitched 5) -> train (n 5, fit 0.6486, coh 0.9547, window
    auto-closed) -> the assessed queue's FAMILIES toggle -> the CAM-02
    chip (fam=root:CAM-02:) -> the de-glitch lozenge -> WHOLE scope
    (537) -> the confirm names the walk -> 10 moved (assessed 527,
    gjunk 10) -> pass 2 converged (0 more).

## THE USER'S LAWS (each earned in this session; the skill carries them)

1. QA SUITES ARE USE-CASES — compound stories in sequence; NOT idempotent
   unit tests; the estate rebuilds only when necessary.
2. QA IS USER ACCEPTANCE TESTING: the browser session IS the test
   subject. Every state change rides the product's own UI gestures in
   the SHARED VISIBLE tab; every stage shown AS IT HAPPENS (the tab
   moves through the surfaces). Route POSTs, DB writes, headless shots
   cannot perform a UAT use-case. The playwright suite is the
   REPLAYABLE ENCODING of the human session (headed by default).
3. ALWAYS OPEN A BROWSER WHEN POSSIBLE — by default, not on request;
   the tab opens at the build's fire step (before ingest), not after
   the completion notification. The user had to ask three times; make
   it impossible to miss: FIRST TOOL CALL of any QA action = open/
   screenshot the tab.
4. Scripts over ~1 minute LOG PROGRESS per pass/bite (flush prints).
5. NO MOJIBAKE — EVER (skills carry the law; byte-exact matches on
   pre-existing mojibake; parse-check any written file).
6. A fresh qa build must NOT delete the bank (the frozen corpus):
   wipe the db + reception only; re-unzip only on a manifest
   fingerprint change (pairs|days|md5(selection)).
7. Ingest + assess run to EXHAUSTION on qa (the consistent initial
   state: new=0); progress lines prove the drain.

## THE MAKE-QA-BANK LAWS (all live-verified this session)

- THE FIRE RACE: Popen('wsl.exe ... nohup ... &') is a RACE (build 3937
  lost it — the server never came up; a manual fire saved the build).
  THE FIX: setsid-detach inside ONE blocking wsl call that backgrounds,
  disowns, and WAITS FOR ITS OWN PORT before exiting
  (`(setsid env ... &) && for i in 1..15; ss -tln | grep :PORT && break`).
  The server SURVIVES the spawner's exit (verified: --smoke-only + the
  later ps/ss probe). The smoke step VERIFIES and LEAVES RUNNING.
- THE RECEPTION RE-SLICE: zip members are named 'reception/...' (the
  prefix at the string START). '/reception/' in n matched NOTHING and
  the build ingested 0 (caught by its own output: bites 1, moved 0).
  Match by STARTSWITH and extract to WSL_BANK.
- THE BANK GUARD: fingerprint = pairs|days|md5(json.dumps(selection,
  sort_keys)) from the manifest's slice2 block (probed: 'pairs=2727|
  days=90|sel=2f710c327f' both sides). On a match: PRESERVED (only the
  db + reception reset). On a mismatch: full re-unzip.
- THE EXHAUST LAWS: the ingest range(200)/break moved==0; the assess
  range(200)/break new==0 (build 3958 first proved it: new 0 at the
  end; the last build stopped at its 40-pass cap and left 727 stuck).

## THE UAT JOURNEY'S SELECTOR MAP (for the suite encoding)

The surfaces + gestures the next session's suite plays (the real DOM
handles, not guesses):
- login: input[name=username] / input[name=password], form button.
- the filter panel: #filters <details>; the day fields are 'day from'/
  'day to' (#f-day_after/#f-day_before do NOT exist on the fam face;
  the snapshot's refs are the truth); apply = the panel's submit.
- the ticks: #tbody tr's input[data-sel]; the selection counter:
  #sel-count ('N selected'); the bar: #selbar. Dialogs: page.waitForEvent
  ('dialog') — the confirm TEXT names the scope + count ('Apply
  g-claim to the 9 selected image(s)?').
- the gclaim review: the band row carries '⌗ analyse' + the read line
  ('the read never writes; seeding is your call'); '⌗ seed this
  family' opens the name modal (input + create/cancel); the band's
  '✓ tick band' tickes ONLY the band (the re-render replaces refs —
  RE-READ after every action, never reuse stale refs across renders).
- train: the '⚗ train' bar button opens the family select modal
  ('train a family' + the option '2022-10 (0 landed)'); the verdict
  line: 'trained <verse>: n N · fit F · coh C (the seed window
  auto-closed)'.
- the walk: the assessed queue's filter panel carries THE FAMILIES
  TOGGLE (#fam-toggle): families mode replaces the free fields with
  the fam preset chips (input[name=fampick]; the change event applies
  IMMEDIATELY via PUT - fam=root:CAM-02: rides the URL); the
  '⌂ de-glitch ("<root>")' lozenge = #deglitch-bar (visible ONLY while
  a fam filter is APPLIED + QID==assessed). The S103f scope law: select
  scope with no ticks = 'tick rows first (the de-glitch scope is
  empty)' (an honest refusal — the suite asserts it once); the WHOLE
  radio (input[name=selmode][value=whole]) arms the root-walk; the
  confirm text: "Run EVERY family of the '<root>' root over the WHOLE
  root-filtered queue (N rows)? ... any FAMILY band walks to gjunk".
  The run modal's verdict line reads 'done: N of M re-fitted (T
  tagged) · K → g-junk' when it survives; the DOORWAY COUNTS are the
  authoritative read (assessed/gjunk chips).
- THE CONVERGENCE ASSERT: a second walk moves 0 (the envelope's
  verdict is stable) — the suite's end-state law.

## THE ESTATE'S SHAPE (what a fresh slice-2 build gives the suite)

new 0 / assessed 1,206 / junk 1,521; CAM-02 1,787 + CAM-05 940; the
server left RUNNING on :8451 (the WSL IP prints in the build's fire
banner). The suite's login: qa/qa. The walk's world: 537 (CAM-02 2022);
the pre-walk fam slice: the 9 reference rows sit at 2022-10-24..26;
the verse lives in g_universes ('g-2022-10', seed_mode auto-closed
after the first train; a REBUILD re-opens nothing — the suite seeds
its own verse name if it wants the seed flow).
