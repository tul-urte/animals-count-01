# How we work together

This is the cross-cutting working agreement for this repo. It applies
everywhere; the domain skills (`.github/skills/picods`, `.../imagebank`) and
the path-scoped instruction (`.github/instructions/`) hold the technical
doctrine. Keep this file short — it is loaded into context on every turn.

## How you like to work

- **Step by step, smallest-verifiable-first.** Break work into the smallest
  step that can be checked on its own, do that step, confirm it, then move on.
  Do not batch several steps and surface them together "at the end".
- **Check every step of the way.** A step is done when its evidence is in
  front of us (a test that passed, a command's real output, a live page), not
  when I *describe* having done it.
- **Understanding outranks the built result.** Prefer an explanation that
  makes the *why* clear over a change that merely works. When I hand you
  something, assume you'll want the mechanism, not just the outcome.
- **Make a plan, then execute it — and keep you talking through it.** Surface
  the plan before acting on non-trivial work, and report back at each step
  rather than going quiet and dumping a finished pile at the end.
- **Unit tests + UAT are the finish line, not an afterthought.** Code changes
  get a test that pins the behaviour; user-facing changes get a UAT step we
  walk through together before calling them done.
- **You like being asked.** When a step has a real fork (not a cosmetic one),
  prefer a short, specific question over a guess-and-hope.
- **Work smart, collaborate.** Don't grind — when stuck, the break usually
  comes from outside the loop. Ping me with a lateral seed or let me ask
  you; that's the cheaper path than brute-forcing harder in the same
  direction.

## Your known friction (work around these)

- **Forgetfulness across sessions.** Do not assume I carry state in memory.
  Current truth lives in the docs (the session index
  `documents/session_notes.md` + one file per session in `notes/`, node
  records, kit README), not in a previous conversation. Re-read before acting
  on anything state-sensitive, and say when I'm relying on something a doc
  should actually be asserting.
- **Session start and close are file rituals, not habits.** Start: read
  `documents/session_queue.md` (the plan; the NOW top is the suggested
  agenda — the user steers). Close: the notes file + ledger row **and**
  the queue update (carries in, retired out, stamps refreshed). A close
  that skips the queue write leaves "un-requited follow-ups" that nobody
  will ever see again.
- **Handoff claims that drift from disk.** A previous session may *say* "X is
  staged" when X is not on disk. Treat such claims as leads, then verify.
  (Example: the kit.zip handoff — I could not find `kit.zip` anywhere under
  `C:\projects` when I checked.)

## How I (the assistant) work, and my known flaws

- **I can sound done before I am.** My default failure mode is narrating a fix
  as complete before I've run it. Countermeasure I owe you: no step is "done"
  until I've shown you the evidence, and I label anything not-yet-verified
  as such explicitly.
- **I drift toward batching when I get confident** — which is exactly against
  your step-by-step preference. If I start chaining steps quietly, call it out;
  I'll split.
- **Rut-prone: I iterate inside apparent context.** Searching again is the
  default reflex, and it's the right default — but past a point it becomes a
  rut. The escape usually isn't "search harder"; it's a lateral move (a
  different command, a different file, a different assumption) that's hard to
  see from inside. That's what *you* are for: an exogenous seed. So: when I
  start re-searching, that's the signal to stop and ask you, or to try one
  visibly different angle — not the same angle once more.
- **I understate uncertainty.** I can state a guess with the confidence of a
  fact. When I'm inferring rather than reading it from source, I should say
  so — hold me to it.
- **I confuse "the doc says" with "I verified".** Distinguish provenance:
  what a previous session asserted vs. what I checked this session.

## Observations log

Append, don't rewrite. Each entry: date, what we observed, and the working
rule that follows from it. Both of us may add to this.

- 2026-10-31 — The user values step granularity and per-step evidence over
  end-state delivery; understanding > built result. → Default to small
  verifiable steps, report at each, and ask at real forks.
- 2026-10-31 — Handoff summaries can assert things not present on disk
  (e.g. "kit.zip staged"). → Verify handoff claims against the filesystem
  before acting on them.
- 2026-10-31 — Both of us are rut-prone and the escape is usually exogenous:
  a lateral seed from the other side, not more effort in the same direction.
  → Iterating is the right default; but the *signal* to stop and collaborate
  (ask the user, or try one visibly different angle) is repeated re-search /
  re-verify without a new idea. "Work smart, collaborate" — don't grind.
- 2026-11-13 — A "restart" that cannot kill (pi vs a root-owned server:
  `Operation not permitted`) verifies the OLD process and produces phantom
  findings. → Check the process START TIME against the deploy time before
  any post-restart verification; escalate to sudo when the owner differs.
- 2026-11-13 — Native `<datalist>` is OS-drawn: never reliably opens on a
  pre-filled field, invisible to scripted verification. → Build own
  suggestion lists in plain DOM when behaviour must be guaranteed.
- 2026-11-13 — Programmatic `.value =` writes fire no events; event-driven
  dirty trackers miss them. → Any component that edits rows programmatically
  must call the dirty hook itself (as setRow does for geometry).
- 2026-11-14 — A temp-file+`mv` deploy does NOT carry the target's
  ownership/permissions: the shipped file takes the staging dir's
  (pi:pi 644), and `callhome.sh`'s exec bit died with the move — the
  every-minute `./callhome.sh ping` died with "Permission denied",
  swallowed by `|| true`, and pings silently stopped for 15 minutes
  (manual `bash callhome.sh` masked it during verification).
  → The deploy procedure's final step is `chmod --reference` (or an
  explicit 755/644) from the `.pre-` backup on EVERY pushed file, plus
  a behaviour check that exercises the exact invocation the runtime
  uses (exec `./callhome.sh`, not `bash callhome.sh`).
- 2026-11-14 — Manual fires of a scheduled job "prove" the code path but
  NOT the scheduling path: a cron-driven runner may exec the file by name
  (`./x`), and only an exec-by-name test catches a missing exec bit.
  → Verify scheduled files by running them the way cron does (by name,
  through the same shell line), never by explicit interpreter.
- 2026-11-15 — The browser killed a whole wrong design in one look (the
  tags_rank view column ordered one row's string by mixed-unit ranks);
  the computed style caught a CSS selector a code re-read had missed
  (`td .tagscell` vs `td.tagscell`). → A rendered page + one computed
  probe outranks re-reading the code you just wrote; build the page
  check into the loop, not after it.
- 2026-11-15 — A PowerShell bulk string-replace emptied BOTH schema SQL
  files (the replace target had already been reverted; the replace built
  on a phantom and wrote empty). The next run caught it because the dev
  DB's view re-creation failed. → A blind file rewrite is not a verified
  step: check file sizes + loadability (read the file through the code
  that parses it) before anything builds on it.
- 2026-11-15 — The S080 alias trap fired a THIRD live time: the band
  sort's ORDER BY subquery bound the bare view name inside a wrapped
  subquery ("no such column: assessed_pairs.camera"). → Any new
  ORDER BY/subquery over a queue view binds the WRAPPER's alias, never
  the view name; the law is now in the skill addendum too.
- 2026-11-15 � A test battery that silently SKIPS its checks on the state
  a fresh setup actually produces (qa-devbank's D refusal battery found 0
  'new' rows on the assess-era rebuild and said "consumed them") is a
  false-green, not coverage. ? Derive reference rows from what the estate
  has (fallback new->assessed) and PRINT which path ran; an expected check
  that never fires is a failure to surface, not an optimization.
- 2026-11-15 � UI faces are STATE-DEPENDENT far beyond visibility: a
  0-row filtered queue never hydrates the toolbar (scopes/bar/filter form
  all absent), and the selection column's FIRST checkbox is the header's
  peek switch (the S091a law, live again � a driver ticking "the first
  checkbox" flips the preview and acts on the wrong row). ? Browser
  batteries bind probes to a specific row's own <tr>, assert the scope's
  own face (whole-scope counter for the filtered total), and fall through
  to a populated queue for face tests that need hydration.
- 2026-11-15 � Bulk QA probes must be REVERSIBLE and never touch policy
  leaves: the browser driver's dialog test initially fired reject on a
  whole filtered CAM-0102 slice (a policy leave). ? One-row
  junk->rejected->junk with the triple captured BEFORE the action; the
  recorded policy lives in the battery, the driver only exercises faces.
- 2026-11-15 - A whole-queue by-example run on live (13,574 rows, ~12 min)
  launched from a 3-seed QA create without narrowing filters first; the
  user corrected mid-run. -> Date (and camera) filters go on BEFORE any
  whole-slice action on the live estate; a QA probe never needs the full
  estate to prove its face.
- 2026-11-15 - A live "restart" is not verified by its HTTP 200: the
  nine-commit deploy rode a deploy-manifest gap (ladder.py/examples.py
  never in pi_deploy's FILES), a 23-for-24 carry INSERT, regex view
  scrapes that truncate at the tags `'; '` literal, and a band sort whose
  COALESCE sat inside a no-row scalar subquery - every one caught before
  or during the QA, none by the restart's green light. -> The deploy's
  finish line is the feature's OWN probe on the live estate (api fetch /
  page face / DB row), never "server restarted + login 200".
- 2026-11-15 - A demo walkthrough walked a REAL COCO detection (stock COCO
  on CAM-0101: 'potted plant' 0.56, a correct detection) through the new
  g-stage/g-publish, landing anti-truth on it; the user caught the class
  of error in one line, then SIMPLIFIED the remedy in the next round: no
  model-kind gates at all - one head per CAMERA, tied to its configured
  universe, trained ONLY on that universe's verified rows (both classes:
  the anti-truth verdicts + the published keeps), because the
  position-density feature is scene-local and the score/area statistics
  model-local - cross-universe rows teach the head another scene as if
  it were this one's. -> Anti-truth is a human act (the review flow);
  a trainer is camera-scoped and class-balanced; a walkthrough probe
  never publishes truth (or anti-truth).
- 2026-11-16 - A page-bounded read that TITLED its slice with the root's
  own count word ("50 landed" under a chip promising 78) read as the
  truth and lied - the payload's WORDS deserve the honesty the route
  refusals get. -> An inventory read (family-map) is bounded by its own
  identity (the root's verses), never by the table's paging; the battery
  lands >-one-page rows so the slice bug cannot hide.
- 2026-11-16 - "The batteries ran green" meant WSL-shaped paths for 40+
  files (repo runs were impossible until the dual-path head landed, and
  the first repo sweep found 14 latent path assumptions + pre-S105
  assertions). -> A suite that runs in one home only verifies one
  layout; the path head resolves BOTH (db.py probe + ART), and the
  re-pointed assertions are re-cut HONESTLY (the compound asserts named
  their new truths, never silenced).
- 2026-11-16 - A compound whose legs are REAL transitions needs no new
  machinery, only a retirement: the S105 g-claim rode /action/selected
  as-built the moment the direct edge died - the ONE seam that needed
  surgery was the glitch-walk's gate (it must accept the compound legs).
  -> Retiring an edge means re-pointing EVERY reader of it (routes,
  gates, batteries, the client's SEL_WALKS); the events' [route ...]
  stamps are the proof, the flash is not.
- 2026-11-16 - A build's wipe raced a RUNNING (root-owned) server: the
  server re-opened the db per request, recreated an EMPTY harvest-v2.db,
  and the bootstrap read fresh=False (schema skipped; 'no such table:
  cameras'). A plain kill failed 'Operation not permitted' and
  check=False ate it - two phantom-looking failures before the cause.
  -> The stop-first law: build_db() stops the server AND VERIFIES
  ('LEFT=0') before any wipe, sudo-escalating; if it cannot, the build
  REFUSES. And: bash -c strings carrying $( ) ride the wsl.exe -- bash
  -c argv passthrough EMPTY - stage such blocks as files and execute.
- 2026-11-16 - The page model became the UAT instrument: a hierarchical
  JSON ($type-declared nodes, $anchor hops riding the product's own
  data-test-id stamps, $root escape, $maxWaitSeconds wait knowledge)
  whose hierarchy mirrors the DOM's; a suite speaks loc() chains, never
  re-typed selectors. -> The product carries its QA contract (the
  stamps); the model is the design document the suites execute; a
  selector that stops resolving is the product's regression, loudly.
- 2026-11-16 - THE PAGE MODEL IS XML (the user's ruling; pages.xml IS
  the master, page-xml.js parses it, the parity gate held both trees
  deep-equal, the json retired). Every element names its accessor
  (key=/title= camelised, else the parse refuses; tags are grouping
  words); version + maxwaitseconds are the ONLY numeric conversions;
  $params/$traps ride the <param>/<trap> slots (no attribute form);
  from="a.xml,b.xml" includes build a DUMMY from me, first-in-list
  wins per KEY, then MY attrs+children overlay (provenance traces).
  -> A script element's comment carrying a literal <script silently
  breaks the ELEMENT (the HTML parser's escaped state eats the real
  </script>): comments spell such tags BROKEN; the rendered page is
  the only probe that caught it. The XML tree carries $type on EVERY
  node — the reference page's walk became pure node-typing (iterate
  plain keys, detect the type, render ONCE; the json-era double-head
  render died with its cause).
- 2026-11-16 - A walk's verdict line says 'done: 537 of 537 re-fitted
  (537 tagged) - 10 -> g-junk': re-fitted = JUDGED (all rows), the
  g-junk leg = MOVED (few). Asserting assessed == judged-count fell -10
  rows was my mis-read twice. -> Parse the VERBS: 're-fitted' is the
  envelope's read; the MOVE leg is the only row-count; convergence =
  pass 2 carrying NO move leg.
- 2026-11-17 - The assistant launched `pi_deploy.py` (a PRODUCTION
  push) because "deploy" rode its step plan and the user's "1a/2b"
  approved the approach; the user intercepted mid-run ("deploy to the
  live pi?... to dev-bank only I hope"). The push had reached scp
  staging (no backups yet, no mv — recovery proved live untouched by
  mtime + content + the backup-stamp inventory). -> The law: a plan
  step NAMING production is not authorization to touch it; the
  push is the USER'S verbatim command, re-confirmed per run; a
  blocking failure pauses for a ruling — never "fix + retry in the
  same breath". Enforcement is structural: pi_deploy.py is GATED
  (a bare run refuses; the push runs ONLY on --live).
- 2026-11-17 - The same incident's residue law: a stopped deploy can
  only leave a STAGING dir (mkdir+scp precede the backup+mv), so the
  recovery inventory is the backups dir (a stamp for THIS run = mvs
  began) + the staging dir, and the verdict rides mtime + content
  (stamp counts) + the process start time, not the --check DIFF (it
  compares the WORKING TREE — every uncommitted session edit reads
  DIFF against an untouched live). -> Probe read-only, prove the
  boundary, then restore from the backup dir by the backup's own md5 -
  never assume the mixed state.

- 2026-11-18 - A page's EMBEDDED SCRIPT rebuilt through a PowerShell
  here-string lost the mid-dot and the check (the chars died in the
  here-string's own encoding; the em-dash survived) - the S110f
  sitemap rebuild shipped 'h?' / '?click' and the browser showed it.
  -> The mojibake law's newest vector: scripted FILE CONTENT via
  here-strings mangles non-ASCII exactly like re-typed files do; a
  rebuild's write is verified by the RENDERED PAGE (or a codepoint
  audit), never by "compiles". The repair rides codepoints
  (chr(0x2713)/chr(0x00B7)) with context anchors, and the ?-residue
  audit distinguishes LEGITIMATE ternaries from eaten chars.
- 2026-11-18 - The notes re-org's follow-up: the plan and the record are
  different files with different rots (session_queue.md vs
  session_notes.md). A queue row is a CLAIM from a session close,
  verified only when the work runs - handoff-drift applies to it
  doubly. -> The queue closes only by being touched every session
  (carry in / retire out / refresh stamps), every row carries origin +
  last-confirmed + a finish-line of evidence, and "the queue says X"
  is never read as "X is true on disk".

- 2026-11-18 - I answered "role predicates on nav elements" with an
  invented framework (a roles() fetch + satisfies() gate + a SKIP
  face) when the user's design was ONE DUMB ATTRIBUTE
  (userRole="admin") read by the consuming caller; "what else is
  involved?" caught it twice.
  -> The $userRole law: an attribute rides verbatim; NO model-side
  machinery; a consumer reads it and acts as it wishes. When the user
  names a declarative shape, build exactly that shape - consumers are
  the callers' business.

- 2026-11-18 - S110h's two symptoms (the sitemap's 4 cards, "MISS
  dashboard.nav.site (an empty chain)") both pointed at the MODEL
  master, not the product: the product's faces were right; the master
  mirrored an incomplete estate + a prose-only nav type (whose d:descs
  were ALSO dying silently on a dead prose namespace).
  -> When a design-perspective view disagrees with the served product,
  the product is authoritative: fix the master, never "fix" the
  product to match the model.

- 2026-11-18 - Clicking through the design dialog cost three rounds:
  a group row click TOGGLES its kids, so a re-click collapses them
  again - page.click then times out on rows that DID resolve
  ("element is not visible").
  -> Drivers of collapse/expand UIs track the toggle state they
  created; a resolved-but-invisible row means collapsed, not absent.

- 2026-11-18 - Splitting session_notes.md by STR char-offsets silently
  mangled 70 sections: the file's multi-byte chars (mojibake arrows,
  math glyphs) make str-index ≠ byte-offset, and the mixed
  CRLF/LF line-ending eras made every "rebuild the slice" formula
  wrong somewhere. Each fix passed its local check and the NEXT
  section failed. -> For byte-faithful document surgery: read the
  file RAW BYTES and slice by BYTE offsets (re.finditer on the
  bytes), verify by REASSEMBLY (concat of slices + head + tail ==
  source), and treat any "the check itself says 146/148" as a
  finding, never as noise. The final run (bytes-only) went green
  first try after the wipe-and-restart.

- 2026-11-18 (S112) - The UAT suite's stage-2b filter assert read ""
  and the selMode radios never rendered; three probe rounds deep
  before the real cause surfaced (the estate carried a PRIOR run's
  g-claim residue; the premise was never fresh). -> A suite's
  premise gate rides AT THE TOP (stage 1.5), and a walked estate
  refuses FAST with the remedy named (exit + remedy line) — never
  downstream-assert-cascades. "The failed assert" is not the same
  question as "is the premise true"; ask the premise question
  FIRST, even when the failing assert seems unrelated.

- 2026-11-18 (S112) - The S107-era suite's stage-3 carried a `whole`
  re-check BEFORE the g-claim ticks: whole mode HIDES the checkbox
  column (the S090 scope law), so the ticks resolved 0 — but the
  run stayed green for months because a rerun-honest wrapper's TRUE
  branch masked the leg. -> A wrapper's skip-branch HIDES regressions
  under it: when retiring a skip (the wrapper), run the un-masked
  legs against a FRESH estate before trusting them; a "green" leg
  that never executed is not green, it is unverified.

- 2026-11-18 (S113) - The prose TEXT-ONLY refusal read frame.kids —
  which nothing ever set; a child element inside any d: face was
  silently accepted for the whole design era (probe-proven on
  d:desc, the ORIGINAL face). The composite marks hasChildren on
  OBJECT frames only, and the lease frame carries no object. ->
  A refusal in a specialist is not a law until a battery check
  FIRES it: every refusal shape gets its own negative fixture
  (the A5b pattern), and "the message exists" is not "the refusal
  fires". The fix rides the REAL signal (the composite marks the
  open lease's kids), not the never-set field.

- 2026-11-18 (S113) - make-qa-bank's --skip-estate stage wiped the
  estate's db: the db lives INSIDE the code dir that stage_code
  rm -rf's, and the running server re-opened an EMPTY
  harvest-v2.db (the S107 phantom shape — the smoke's GET /login
  200 rides a db with 2 tables). The S112 'no such table: users'
  was the same defect, misdiagnosed as a stop-first race. ->
  When the same symptom recurs under a DIFFERENT operation, hunt
  the shared path, not the operation: both wipes shared the
  "db inside the code dir" layout. The real fix (Part D/leg 4):
  the db moves OUT of the code dir — a deployment must never
  touch data; the interim stagekeep guard is the guard, not the fix.
