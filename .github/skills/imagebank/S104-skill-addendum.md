# S104 addendum — THE FILTER ROOTS (the multi-family de-glitch) — draft for the next skill refresh

*2026-11-15. Extends the S096 addendum (the glitch mini-flow + the universe
law) and the S081 band law. The user's rulings ride each section.*

## THE ROOT LAW (the user's model — the filter unit is NOT the verse)

- **A root = (camera/model, class_label)** — the verses sharing the pair sit
  in one root; the day blocks are the verses' OWN mutable state ("advance/
  recede as work gets done"), never root identity. Root id shape
  `root:<cam>:<class>` (the class leg may be NULL → a camera-wide root).
- **One root applied at a time** (the user's ruling; no multi-root select).
- **The derivation (from gclaimed):** `gfam.root_members(con, in_play_only)`
  keeps roots with a TRAINED verse or a gclaim allocation; the archive face
  reads in_play_only=False (a root with landed mass shows even untrained).
- **The fam= expansion** (api.apply_fam + routes._fam_expand, mirrored): a
  ROOT fam= expands camera + class ONLY (no day legs); a verse fam= keeps
  the S103f shape. THE NULL-LEG BIND: SQL `IS ?` matches NULL but never ''
  — the empty class leg needs `(class_label IS ? OR (?='' AND class_label=''))`.
- **The archive bind** (api.filter_fragments): a fam on a view carrying
  g_version scopes `g_version IN (the root's verses)` (a verse fam= rides
  as itself — back-compat).

## THE COMPOSITE PLUGIN (gfamroot — the multi-family walk)

- `gfamroot:<cam>:<class>` (assess_plugins.plugin_of): one pass, EVERY
  trained verse of the root fits the row; the evaluator writes each verse's
  OWN `gfam:<verse>` tag (banded, actor 'gfamroot' — the bands-ride
  precedent) and returns the MAX fit (the any-verse rule). **NO composite
  tag**: a `gfamroot:*` tag would ride the meters as raw TEXT (the user's
  catch) and adds nothing the per-verse tags lack.
- ladder.edges_of carries the `gfamroot:` prefix arm (the gfam ladder);
  routes' walk_target + the ladder gate + the fam-kind split grew the arm.
- THE FAMILY-SCOPE GUARD (assess_plugins.write_tags): a `gfam:<verse>`
  tag rides only rows INSIDE the verse's own camera (the hard wall; the
  camera is the verse's enforceable scope) — a write outside is DROPPED.
  A run on one root can never tag another root's rows. (Retro-wash: the
  live-caught contamination — 696 cross-root tags — was deleted + the
  pairs cache rebuilt; the NEXT REBUILD must not re-create them: the user's
  NB. The composite evaluator also skips foreign-camera verses BEFORE fit.)

## THE FACES (queue.js — all browser-verified)

- **The families toggle** (both ruled): FIRST control of the filter form
  (order:-1); the exclusive faces (famMode = the APPLIED fam or the
  toggle; FAM_UI_MODE module var carries the user's last ask, reset on a
  queue hop). Families face = the root chips ONLY; free face = the
  .freefield members (camera/classes/light/focus/day/apply/reset + band
  widgets). THE APPLIED TAG: #fam-drops.dataset.applied carries the fam=
  truth (the free-face apply preserves it via famSelection()).
- **The root chips**: one radio per root — the short name + the ROOT'S
  families in line + the count word (holotypes on assessed, LANDED on the
  archive); the ✕ clear; applyFam(verse) posts the stored row + hops.
- **The de-glitch lozenge**: toolbar ONLY (the user's ruling), the applied
  fam state drives it, ASSESSED only (the walk is assess-scoped; the
  archive's chips are reads). **THE SCOPE LAW (the S104d catch): the
  lozenge honours the bar's scope radio — selection walks the ticked rows
  (ids posted, NO fam re-slice), page this page's rows, whole the whole
  root-filtered queue (the fam= expansion rides whole-mode only). The
  toolbar verb whose context is a scope must read the scope.**
- **The archive's instruments**: the fam chips + ⌗ maps + ✕ close (no
  writes — glitched keeps no out-edge by the user's standing ruling).
- **THE FAMILY MAPS** (⌗ maps, glitched only): /api/queue/glitched/family-map
  (PURE READ — per root, the landed rows as points + every trained verse's
  envelope 2-sd box + res/fit/cohesion); one scatter per root (score x
  area), one hue per verse, dot click = fam=<verse>, wheel zoom. A MAP IS
  A READ: no derived truth (never invent a stored per-point fit column).
- **THE TRAIN GESTURE** (S104d): the gclaim bar's ⚗ train (the in-modal
  verse select; the enable rule's isTrain arm — a train is NOT a
  rows-gesture) + POST /action/verse-train (gfam.train's live seam; the
  reviewer's act: editor/publisher/admin; the honest refusal). The
  per-CAMERA heads page stays the OTHER trainers (it refuses honestly on
  a negatives-empty bank — never conflate the two).
- **The analyse read respects the holds**: api.analyse_unassigned_json
  drops rows WITH an allocation from the cluster sweep (never re-propose
  a held row — the duplicate-seed law).

## THE LIVE PROOFS (the dev bank, all browser + DB verified)

- Seed→publish→train: 3 verse seeds (13/12/7 allocated), publishes 13/12/7,
  the landed corpus {robin 3, starling 4, 1005b 13, pigeon-0246 12,
  pigeon-1319 7} = glitched 39, gclaim 2; g-export REFERENCE rows only
  (39, ZERO file writes — the internal-subflow law); each verse its
  g_batch (open).
- Trained (through the LIVE face): 0246 n12 fit .665 coh .956 · 1319 n7
  fit .666 coh .932 · 1005b n13 fit .644 coh .895; the seed windows
  auto-closed.
- The multi-verse walks (the user's order CAM-02 first): CAM-02 root 523
  rows, ONE pass, 24 → gjunk; CAM-05 root 636 rows, 31 → gjunk. THE
  MAX-FIT RULE in the moved rows' tags: every moved row wears ALL the
  root's verse tags (one family's band held it). Estate after: assessed
  499, gjunk 61, glitched 39 (the walk NEVER publishes), gclaim 2.
- The scope-aware lozenge: 3 ticked rows → confirm "over the 3 ticked
  row(s)" → "done: 3 of 3".

## THE NEXT SESSION'S PLAN (the user's own words)

1. **Iterate to exhaustion**: reduce the assessed head root by root —
   analyse gclaim → seed → train → walk — until the families run out; the
   user g-claims new items as they arise; problems solved en route.
2. **The capture**: at the exhaustion point, record everything the replay
   needs (the notes carry the reference numbers so far: S104/S104b/S104c/
   S104d).
3. **The junk-compound action**: the gclaim-from-assessed becomes a
   compound via junk ("the glitches never leave junk — sucked into an
   internal dimensional space entirely inside junk, invisible from the
   outside"); build before the battery. **DONE (S105, REAL walk — see the
   S105 addendum): the direct edge retired, COMPOUNDS walks the legs.**
4. **The full rebuild + the QA replay**: rebuild-devbank from the live
   slice, then run the QA battery (the chain battery AFTER the
   junk-compound shape lands so it pins the FINAL flow).

## THE WORKING RULES RE-EARNED (S104 era)

- The degenerate case is not coverage: a multi-family feature verified
  with one-verse roots proves the pipe, not the composition.
- The unbound button is the silent dead click (the lozenge's render but
  no wire froze the modal at "starting…" — probe the click, not the
  render).
- The NULL-leg bind: `IS ?` never matches '' — a root id with an empty
  leg needs both arms.
- A read must respect the holds (the analyse read's re-proposal fix).
- A map is a read (no invented columns to look richer).
- The scope law above (the lozenge catch).
- By-PID restart discipline RE-proven live: the composite refused 400
  "unknown plugin" for exactly the pre-fix-module window (file mtime vs
  the process START TIME names it).
# S104 addendum - the multi-family root, the qa-bank, the restructure (2026-11-15, S104d-g)

## THE RESTRUCTURE (S104g, the four siblings - the user's layout law)

`06-deployment/` carries FOUR siblings; the repo-side names NEVER mirror
into the runtime hosts:
- `imagebank/` - THE DEPLOYABLE ARTIFACT: exactly what `pi_deploy.FILES`
  stages (the app code, web/, sql/, model/, the .sh's, run-v2-server.sh,
  the SHIPPED pi_* tools, pi_deploy.py itself; the machine-local
  `properties` override - git-ignored, per machine). The Pi still runs
  /home/pi/imagebank-dev (deployed by pi_deploy).
- `dev-bank/` - ALL DEVELOPMENT ARTEFACTS: the estate in `bank/` (the
  tree the windows dev server serves), the dev db + its baks, logs/,
  __shots*, the dev-only pi_* tools in `pi-devtools/`, the runbooks.
- `dev-slice/` - the slice zips + the slicers.
- `qa-bank/` - THE QA SPEC: `make-qa-bank.py` (build/destroy the WSL
  qa-bank), `rebuild-devbank.py` (reads the sibling layout), `README.md`,
  `tests/` (the batteries + run_all.py + the browser faces).

THE LIVING DOCS: `imagebank/README.md` (the layout), `qa-bank/README.md`
(the qa-bank spec + its laws), the SKILL's "Where the code lives" section.
The historical session_notes/plan-* refs to imagebank-dev are HISTORY
(append, never rewrite).

## THE QA-BANK (the WSL stand-up; the release rehearsal)

- SHAPE (the user's catch): the REAL imagebank's names on the qa box -
  user qa; `/home/qa/imagebank/` = flat code + `harvest-v2.db` INSIDE it
  (the Pi idiom); the estate = `/opt/cameras` (unzipped DIRECTLY:
  cameras/ + reception/ + assessment/ + manifest.json; bank_root pinned
  by the staged properties override); the zip at `/home/qa/zip/`; the
  venv = /home/imx/imx500-env (py 3.11 aarch64 = the Pi 5's own
  instruction set; numpy+pillow exact). NO REPO-SIBLING NAMES in WSL.
- LIFECYCLE (the user's laws): build RE-USES a healthy estate by default
  (integrity ok + images>0 + cameras/ present, probed INSIDE WSL - a
  \\wsl$ read races the WAL); `--fresh` forces; `--destroy` keeps the
  zip by default (`--no-keep-slices` takes it). The smoke: fire :8451
  PLAIN HTTP (no --tls-dir), curl /login, pkill.
- THE FOUR BUILD LAWS (each caught by evidence):
  1. THE BINARY LAW: .onnx = RAW copy + RAW md5 (LF-normalising corrupts
     the flatbuffer while a stripped-vs-stripped md5 PASSES).
  2. THE CLEAN-DB LAW: wipe the db + WAL before db.bootstrap (it
     schema-creates only when ABSENT; a leftover empty db skips the
     schema: 'no such table: cameras').
  3. THE CROSS-TREE STAMP LAW: a pair's legs live in SIBLING trees
     (reception/<CAM>/<day>/ jpg + reception/<CAM>-CLASSIFIED/<day>/<graph>/
     xml); walk the JPG side, stamp BOTH legs; a per-dir walk pairs
     nothing (the ingest stalls at the window cap).
  4. THE SUITE RIDES THE DEPLOYED SHAPE: tests/ stages into the qa tree's
     debug/ (beside the code, the manifest's own layout); the suite runs
     against the staged code exactly as it will on the Pi.
- THE GREEN: 50 batteries failed:0 in WSL (glitch_slice2 168, byexample
  83, multirole 55, gjunk 64, ladder 50, publish_gt 60, autoreject 47...);
  the first run's catches: the MANIFEST GAP (gfam.py + autorepeat.py
  imported by deployed files but absent from pi_deploy.FILES - FIXED;
  the next live deploy would have 500'd at first /q), a Windows-only
  path assert in test_adopt_publish (made OS-neutral), node in WSL.

## THE GLITCH-FAMILY LATE JOINERS (S104e)

- A ready verse has NO re-train lane (the bar's train button hides when
  no un-ready verse exists): a 2-row top-up never re-sharpens the
  envelope through the UI; gfam.train re-run module-side reads the full
  corpus. An in-face 're-sharpen' gesture on the ready verses is the
  open ruling.
- The frame-mapping decision: HARVEST-ERA ONLY for now (the
  normaliser/cross-frame wiring deferred until significant new-picods
  traffic); the verified frame-attr facts live in S104e's notes.

## THE MAP TOOLTIPS (S104e)

- The family maps carry a DOM tooltip overlay (#fammap-tip): dot face =
  verse-coloured header + score/area/aspect/density + the click hint;
  envelope face = corpus n + mean fit + cohesion + centre + the 2-sd
  spread + the 2-sd-box hint. Native svg titles RETIRED (the datalist
  law: OS-drawn, invisible to QA). The tip dies on mouseleave, panel
  close, re-render.
