# S096 addendum — the GLITCH MINI-FLOW + the universe law (draft for the next skill refresh)

*Not yet merged into SKILL.md — the next skill review decides the wording;
kept alongside the skill while it's fresh (2026-11-15). It supersedes the
S095-era COCO special-case notes and extends the S081 addendum's band law.*

## THE GLITCH MINI-FLOW (S095 build-2, live dev-side)

A MIRROR of the truth stack for the anti-truth universe (the per-camera
"known glitch" families the measures can't judge by score alone):

- **The statuses:** `gclaim` (the review — the full editor, the categories
  FIXED to 'unclassified') -> `gstage` (the final QA, the g-verse target
  chosen like the stage gate's version) -> `glitched` (the verdict
  terminal, the anti_truth row's class/version/batch on the queue face).
  The reverses carry the S058c naming law (un-g-claim / un-g-stage /
  un-g-publish); the release from glitched KEEPS anti_truth (the training
  corpus survives; only the workflow ref + the file doctrine run).
- **The stores (BOTH):** `g_universes`/`g_batches`/`anti_truth` (the
  mirrors of ground_truth_versions/gt_batches/ground_truth — training
  reads anti_truth directly) AND the `glitch:<cam>` banded tag family in
  assessment_tags (the meters + the filters + the tagmap).
- **The editor's g-namespace:** the layers `gclaimed`/`gstaged`
  (properties keys; live overrides welcome), the origins
  `human-gclaimed`/`human-gstaged`/`g-export` (the xml_docs CHECK widened),
  and THE COERCION: every label flattens to 'unclassified' on save (the
  normalise-then-flatten ORDER — a box-shape dict coerced via
  dict(b, label=) gains BOTH label and name and _norm then misses the
  box; the battery's KeyError).
- **The anti-truth row lands BEFORE the flip** (the publish law's shape:
  the truth row first, the status move rides it); the g-export xml moves
  to g_universes/<verse>/<batch>/.
- **The viewer's editable g states:** gclaim (claimant/adoptable like
  claimed) + gstage (the QA edits like staged); the categories list is
  [unclassified] when the image is in the glitch namespace.
- **The toolbar/dash:** the ICONS ⚑ (g-claim) / ⚠ (g-stage) / ⊘
  (g-publish); the dash.html mirror spine east of the main flow (the
  lavender FILL.glitch family; the assessed->gclaim diagonal clears
  reserved's rim — zero crossings); the /action/gclaim|gstage|glitched
  verbs in routes' POST map (ACTION_ORDER grew but the POST map is
  separate — the 404 the browser caught live).

## THE UNIVERSE LAW (the measures head; the user's simplification — the S095b model-kind gate is GONE)

- **One head per camera, tied to its configured universe, trained ONLY on
  that universe's verified rows, WHATEVER model produced the detections.**
  A garden camera may run COCO (or any model) and trains the same way.
  What protects the corpus is the universe tie + the human review — NOT
  the model's kind.
- **The corruption mechanism (the reason the claim holds):** the
  position-density feature is SCENE-local (another camera = another
  scene), the score/area/aspect statistics are MODEL-local (another graph
  = another calibration); cross-universe rows teach the head another
  scene as if it were this one's.
- **The class-balance gate:** the corpus is the camera's OWN verified
  truth, both classes — positives = the anti_truth rows (the mini flow's
  landed verdicts), negatives = the camera's PUBLISHED rows (humans saw
  and kept those). Either class under `glitchhead.MIN_PER_CLASS` (5) —
  the trainer refuses honestly (a thin class teaches a bias).
- **glitchhead.train_one / score_one** are camera-scoped by construction
  (the corpus queries bind the camera; scan_state keys are the per-camera
  weights — no model file, no restart). The features are label-free:
  score/area/aspect/focus-rank/light-rank/POSITION-DENSITY (the pond-rim
  family's own signature — a FP family tagged "sharp" is geometry's catch,
  not focus's).
- **`glitch:<cam>` plugin family** (assess_plugins.plugin_of extension):
  banded clean/unsure/likely/glitch off the tags.glitch.bands knob; an
  untrained camera refuses honestly; verdicts ride the run-modal like
  every banded plugin.
- **The and_glitch=1 route-variant** (POST /action/assess-selected): the
  rows whose verdict sits in the plugin's TOP band transition to **gclaim**
  after the tag write — THE REVIEW FIRST, never straight to glitched
  (nothing becomes anti-truth without a human walking the mini flow).
  Gates: the plugin must be banded + the ladder must resolve + the held
  roles may g-claim.
- **ANTI-TRUTH IS A HUMAN ACT:** a walkthrough probe never publishes truth
  (or anti-truth) — the rehearsal's probes walk the fixture-recorded rows
  only.

## THE REHEARSAL (the production hold; BUILT S097 — the runbook's run)

**Production stays HELD until the user's UAT + the live deploy.** The
rehearsal is now BUILT and runs clean ×3 (S097): the fixtures file
`debug/glitch_rehearsal_sets.json` (CHECKED IN: CAM-0102's five
compelling-band positives = the mesh-chair family + six negatives walked
through the MAIN flow), the driver `debug/glitch_rehearsal.py` (RUN:
fixtures -> the candidate face -> the g-verse seed -> the positives walk
-> the negatives walk -> the train -> the plugin face -> the opt-in
--with-walk -> the report; every check a hard assert through the REAL
routes). `--collect` for more samples stays as designed (parked until
the fixture grows). The user's UAT + the live release line:
pi_deploy -> pi_migrate (the addenda's idempotent edge rows; NO CHECK
movers needed) -> by-PID restart -> the live QA (v2026-11-15.002).
-- 

## S097's additions to this addendum (the state of the mini flow)

- **THE CANDIDATE LANE:** the assessor's no-detection discard stamps
  `glitchcand` (the camera graph's top score banded
  background/faint/ambiguous/compelling; MODELLED universes only — the
  no-assessor junk lane stays stamp-free). The mine = junk filtered on
  the band (the widget rides payload-scoping). JUNK->G-CLAIM is a NEW
  DIRECT edge (not a walk through assessed); the g-claim copy seeds the
  camera boxes from junk too.
- **THE METERS' WORDS LAW:** the tag-WORDS (the meters' ladder words)
  ride EVERY queue's payload — the WIDGETS stay plugin-scoped. A queue
  can DISPLAY a tag it cannot filter on.
- **THE INGEST STAMP:** focus + light_level stamp at the bite's MOVE
  (the ordering law: a stamp before the move measures a nonexistent
  path — the battery caught it). Image-fixed; never re-decoded
  (skip-if-measured); no glitchcand at ingest (the disagreement does
  not exist yet).
- **THE HEAD'S JUNK FALLBACK:** glitchhead.image_scores falls back to
  the CAMERA GRAPH's top box when no assessment doc exists (junk rows
  have none BY CONSTRUCTION — the assessor saw nothing; that is why
  they are junk). Without it the --with-walk moves zero.
- **THE DASHBOARD'S TRAINERS LINE:** one chip per trained head
  (camera | n imgs; hover = the corpus split + the trained-at stamp)
  — read from scan_state, fresh on the 5 s refresh.

## S102's additions (the slice-2 era, 2026-11-15)

- **THE QUERY-SELECTION SLICER:** `dev-slice/make-slice2.py` — the live
  estate sliced by QUERY SELECTION (assessed + camera + classes LIKE over
  the S079 union + the day block) instead of camera+day windows; the
  manifest carries the FILEPATH LIST + the config rows;
  `rebuild-devbank.py --zip <slice2.zip>` re-points the standard rebuild.
  A dry-run MUST assert the expected count (the user's filter number) —
  the complement typo produced 8,286 vs 1,787 and the check caught it.
- **GFAM'S DENSITY LAW:** `gfam._density_for(con, image_id, det)` binds
  the IMAGE (glitchhead.features_for reads the row's own tags + the
  camera's historical mass); `fit_one(con, verse, image_id, det)`. A
  feature that degrades to neutral 0s needs a probe that reads the
  FEATURE (the per-row fit prints), not the verdict.
- **THE TRAIN-ORDER LAW:** the density feature reads the camera's
  HISTORICAL MASS — a corpus trained BEFORE the walk rows exist sees a
  mass without them; train after a batch (the envelope holds the mass
  the rows trained into).
- **THE VERDICT GATE (the family join):** a g-publish must name a verse
  the row has JOINED (the fit in the family band) AND the verse TRAINED
  (>= MIN_HOLOS) — the sorting hat READS, NEVER WRITES (live inference
  per render, never stored veracity). A per-row gate runs AFTER its
  targets resolve. The BOOT GAP: a cold verse cannot take its own seed —
  the first 3 holotypes of a new family need an explicit boot mechanism.
- **THE RESOURCING SCORE:** a verse's size read (0 below the floor, the
  log climb, the plateau at ~12) + the corpus's mean self-fit + cohesion
  (1 - the mean per-axis CV) ride the gclaim payload's g_analysis + the
  bar's family line. The g-claim EDITOR renders per-verse fit analysis.
- **THE EPISTEMIC METRES:** the tag meters' colour carries the ROW's
  state (static green, candidate blue, LANDED anti-truth red), the bank
  order is static -> dyn -> anti, and the inline strips align by
  box-bottom (vertical-align:bottom — the inline-BASELINE trap), verified
  by a browser remeasure, not an eyeball.
- **THE SEED WINDOW (the boot gap closed):** `g_universes.seed_mode` (1 =
  the boot window open). A cold verse admits its OWN first holotypes
  (the family-join gate stands down); `gfam.train` AUTO-CLOSES the window
  when the corpus lands (the gate re-arms itself). The train re-arm is
  the one close event — no stale flags.
- **THE CAPTURE DOCTRINE:** the user's recorded decisions live in
  `debug/glitch_slice2_sets.json` (the triples + the walk filters + the
  fit fingerprints); `debug/glitch_slice2_replay.py` replays them
  (rerun-honest). A holotype fitting its own envelope LOW is a review
  flag (split or re-look), never an auto-evict (g-publish is the record).

## The seam notes (S095 era)

- pi_migrate's two CHECK movers (workflow.status + xml_docs.origin): the
  S052 junk pattern twice — row-copy in one transaction, the debris law
  generalised (an empty mover table drops on entry, debris WITH rows
  refuses), and THE VIEW TAIL: SQLite's rename checks view integrity, so
  the movers drop the v_queue_* set and a FINAL loop re-creates the whole
  set (a mover that drops the views cannot be the last word).
- workflow.g_target_version (the FK to g_universes) is the mirror gate's
  column — the mini flow does NOT overload gt_target_version (the GT FK
  refused the g-verse names; the battery caught it).
- The whole-N streams (S094/S095): ONE keying law — **the X-Stream
  header only**; a live-handler-alone key streams the buffered small
  selection POSTs by mistake (the browser caught it). The stream branch
  REPLACES the buffered loop (a stream branch AFTER it runs every
  transition twice — the battery's caught double-walk).
