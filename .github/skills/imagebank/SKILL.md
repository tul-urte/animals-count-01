---
name: imagebank
description: 'Imagebank (Pi 5 at 192.168.68.56): the reception ingest pipeline, harvest.db catalog (images/xml_files/detections/assessments/workflow_state), GT universes (AnimalsCount/CocoTest) and per-universe assessors, triage queues with filters, the editor claim/publish/reject flow, export layers. Use when: working on ingestion/assess/queue/editor code under 06-deployment/imagebank, reception or mover flow, workflow statuses or buckets, GT-universe tagging or pollution, ib_server routes, harvest.py/ingest.py/assess.py/workflow_state.py, or triaging/publishing ground truth.'
---

# Imagebank — harvest, ingest, triage

## LAWS THAT NEVER BEND (all work in this estate)

- **NO MOJIBAKE — NEVER, EVER.** Every file written this session and
  every file edited MUST keep its encoding bytes intact: the repo's
  docs/code carry UTF-8 (—, →, ✓) and some carry pre-existing mojibake
  (read those bytes EXACTLY as-is when matching old_str, never "fix"
  them incidentally). Before declaring any write step done: the
  PowerShell string-replace trap has eaten whole files before (the
  2026-11-15 schema-SQL emptying) — a blind bulk replace is not a
  verified step; check the file still parses/loads and its size is
  sane before anything builds on it. When PowerShell mangles a
  non-ASCII literal: stop and use a different mechanism (a python
  script, byte-level operations), never a re-typed lookalike char.

The bank: `/yermo/animals-count/cameras/` (local USB disk on imagebank,
SMB-shared to the nodes as /YERMO) + `/home/pi/harvest/harvest.db`
(SQLite, WAL). Server `ib_server.py` on :80. Current state lives in the
docs; this file holds the structure.

## Authoritative state (read these, do not trust memory)

- **Pipeline + routes**: `06-deployment/imagebank/imagebank.md`
- **Recent work / decisions**: the session index
  `documents/session_notes.md` (one file per session under `notes/`,
  named `YYYY-MM-DD S###[.suffix].md`)
- **Phase plan**: `documents/plan-image-bank.md`
- **Addenda**: S059, S062, S081, S096, S104 (the restructure + the qa-bank
  + the multi-family root, 2026-11-15); S105 (the junk-compound + the
  retrain lane + the map inventory + the three-site ident, 2026-11-16);
  S106 (THE QA USE-CASE LIVES IN THE BROWSER — the UAT law + the
  slice-2 qa estate + the exhaust/fire/bank-guard laws + the selector
  map, 2026-11-16)

## The pipeline

```
picods mover (cron, rsync-over-ssh AS the node's own identity —
  S0XX 2026-10-31: account picods_1XX on the bank, restricted key
  [from=<node ip>] + command=rrsync /yermo/animals-count; the OLD
  CIFS mover + the /YERMO mounts are PARKED/retired, smbd stopped)
  -> /yermo/.../reception/CAM-0XXX/<day>/ + -CLASSIFIED/<day>/<graph>/
ingest-v2.py (cron */5, flock): scan pool (rsync dot-temp files
  SKIPPED; settle window retired — rsync's temp+rename means a file
  under its real name is COMPLETE) -> register in harvest-v2.db
  (idempotent by natural keys; paths store the FINAL resting place)
  -> os.replace to cameras/... -> workflow refresh
assess-v2.py (cron, venv python): assesses new pairs PER GT UNIVERSE
  (universes gate via cameras.gt_universe — v1's image_status table
  does NOT exist in v2); a universe with a NULL model is skipped
  entirely (its pairs stay 'new' forever - node detections their only
  annotations)
```

The OTHER node-channel, the callhome (`callhome.sh` →
`POST /node-report`): since S0XX the nodes post AS THEIR OWN IDENTITY —
users rows `picods_100`/`picods_101` (role `camera`, zero workflow
powers), HTTP Basic from each node's root-600
`/home/pi/picods/nodehome.creds`; the event vocabulary is whitelisted
(starting/stopping/no-go/ping), `nodehome.require_auth=1` closes the
anonymous POST (the flip is a properties key), denials audit to
`picods_auth_log`. The nodes' rsync dot-temp files NEVER enter the pool
(the ingest skips leading-dot names) — a half-copied pair cannot
register.

Crash-safety: the ingest registers + COMMITS before moving, so a
mid-bite crash leaves a registered file the next bite just moves.
Errors never move files either - the pool self-heals.

## GT universes (the S035 seam)

`images.gt_universe` is stamped at ingest (camera -> universe mapping
in `harvest/universes.py`: CAM-02/05/06/0100 -> AnimalsCount,
CAM-0101 -> CocoTest). `universes.py` also holds UNIVERSE_MODELS
(universe -> assessor model or NULL) - stdlib-only on purpose: both
the worker and the server import it (assess.py pulls onnxruntime).

Why: CAM-0101 runs a COCO model on a kitchen scene - its output is
NOT for the animals GT. Universes gate: which images get assessed
(assess.py), which categories the editor's dropdown offers (per-
universe), and how publish outcomes are routed (image_status.
gt_universe, stamped at outcome; export/GT reconciliation per
universe is still future work - publishing a CocoTest image is
visible but its GT home is undecided).

## The triage workflow (workflow_state)

One row per image+XML pair. Statuses: `new -> assessed -> claimed ->
published|rejected`, plus `GT` (absorbed into the Windows ground
truth; sticky + terminal). Buckets slice 'assessed': expected-correct /
expected-mis-categorised / spatial-disagree / unrelated-find
(overlap = intersection-over-SMALLER - old boxes are ~5x looser than
the model's). Outcome wins: re-assessment never re-files
published/rejected/GT.

The editor: claim (locks, active_edits) -> edit boxes/categories ->
publish (confirmed) or reject -> release. The queues
(`/queues/<id>`) carry filters (camera/label/scores + the universal
universe/day_from/day_to applied by querylib transform to every
workflow queue), and rows link into the editor with the queue
context: the editor shows a prev/next bar (position within the SAME
filtered queue, across page boundaries) and offers reject-direct
(auto-claim + reject + land on the NEXT image).

Form convention trap: POST form values are plain strings (parse_qsl)
while the query-string convention is lists - never index POST form
values with [0] (the S038 bug: a ctx string truncated to '/').

## Where the code lives (repo -> deploy)

**THE S104G RESTRUCTURE (2026-11-15): the four siblings** under
`06-deployment/`: `imagebank/` = THE deployable artifact (what `pi_deploy.py`
stages — the app code, web/, sql/, model/, the sh's, run-v2-server.sh, the
shipped pi_* tools + pi_deploy.py itself; the machine-local `properties`
override, uncommitted); `dev-bank/` = ALL development artefacts (the estate
in `bank/`, the dev db + baks, logs/, __shots*, the dev-only pi_* tools in
`pi-devtools/`, the runbooks); `dev-slice/` = the slice zips + slicers;
`qa-bank/` = the QA spec + the suites (`make-qa-bank.py` — build/destroy
the WSL qa-bank, `rebuild-devbank.py`, `README.md`, `tests/`). History
(S076): the v1 kit is GONE — `06-deployment/imagebank/` (the OLD v1 dir)
REMOVED 2026-11-13 (its 307MB v1 harvest.db snapshot archived to
`C:\projects\archive\`); references to `imagebank/deploy.py`,
`imagebank/harvest/*`, `imagebank/ib-server/*` are historical narrative.
The workers remain ingest-v2.py / assess-v2.py; the server api.py / app.py
(the artifact root, deployed to /home/pi/imagebank-dev).

### Deploying ib-server changes (the straight line)

**One command:** `python 06-deployment/imagebank/pi_deploy.py`

It scp's all ib-server files to a staging dir on the Pi, dos2unix + backup
+ atomic `mv`, md5-verifies both sides (CR-stripped), `sudo pkill`s the
server, waits for the root-cron keep-alive (`run-server.sh`, every minute)
to revive it, then `curl`s the three key routes (must all be 200).

Options:
```
deploy.py --check                      # SAFE: diff local vs live, change nothing
deploy.py --target harvest             # deploy the worker scripts (no restart)
deploy.py --target both                # both targets
deploy.py --files ib_server.py pages.py  # only specific files
deploy.py --dry-run                    # show exactly what would happen
```

**The traps this tool sidesteps (documented so you don't re-hit them):**
1. **PowerShell mangles inline `ssh 'bash …'`** by stripping `\r`, so
   `tr -d "\r"` silently becomes `tr -d "r"` and deletes every `r` from the
   script. The tool drives scp/ssh via `subprocess` with explicit argv.
2. **`run-server.sh` runs the server in the foreground** — invoking it from
   your own ssh session hangs that session. The tool pkill's and lets cron
   revive it (up to 90s poll).
3. **`__pycache__` may hold root-owned `.pyc`** files (the server runs as
   root) — `rm -rf` on them needs `sudo` or a `|| true` guard.
4. **CRLF in the finalize script** breaks `set -euo pipefail` — the tool
   writes the remote script with `newline="\n"` (Windows default is CRLF).
5. **`sql/*.json` changes** take effect after the ib_server restart; views
   are materialised by migrate.py.

**Transfer doctrine** (manual fallback, if the tool is not available):
temp-file + `mv` (atomic), dos2unix (required: CRLF chokes bash `set -e`),
md5-verify both sides with CR stripped, `cp -p` backup before replace.
imagebank has **passwordless sudo** (`sudo -n true` is OK). Uhtred key
auth is permanent (plink was only for the one-time password stage).

### BROWSER QA IS MANDATORY FOR UI CHANGES (user law, 2026-10-30)

Any UI change to the imagebank surfaces (image page, queue shells, peeks,
dashboard, editor chrome) is **not done until it has been QA'd in a real
browser pointed at the live server** — the HTTPS surface at
`https://imagebank/` (or `https://192.168.68.56/`). Python-driven probes
(curl/ssh renders, API fetches, response-string greps) are fine for
debugging and for server-side assertions, but they are **not** a substitute:
a page that "renders 200 with the right strings" can still be visually
broken (layout, popover/z-order, CSS, JS event wiring, port sizing).

The QA pass in a browser must at least: open the changed surface on a real
image/queue, exercise the new control (click/toggle/hover), confirm the
visual result (boxes, colours, layout) and take a **screenshot as the
record of evidence**. Do this before reporting a UI change complete.

### QA IS USER ACCEPTANCE TESTING — THE BROWSER IS THE TEST SUBJECT (user law, 2026-11-16)

**WHY the laws below hold: the browser session ITSELF is what's under
test. A QA use-case is USER ACCEPTANCE TESTING (UAT) — a human's journey
performed on the real product, in the real browser, the way the user
would run it. Everything the product does for the user must happen on
that surface, through that surface's own gestures.** This is the reason
linking the rules together:
- **The browser is the ONLY execution surface for state changes.** A QA
  use-case is performed in the shared VISIBLE browser, click by click,
  stage by stage — the way a human would: filters applied from the
  panel, rows ticked, buttons clicked, dialogs confirmed, toasts and
  doorway counts watched. HTTP POST scripts, direct DB writes, and
  hand-built form payloads CANNOT perform a UAT use-case: what a user
  cannot do by hand does not exist in UAT. (Headless Playwright is a
  display-device error of the same kind: the human must be able to
  WATCH the journey run — headed by default; `--headless` is an
  explicit exception, never the default.)
- **Every stage is visible as it happens.** The tab MOVES to the surface
  each stage lives on (the source queue -> the filter face -> the
  review queue -> the editor/publish face -> the confirming toast ->
  the moved counts -> a representative image page), screenshotting each
  stop. The stages are the PROCESS, not snapshots of outcome my scripts
  made true invisibly and then "showed". URL navigation happens ONCE at
  the story's start; after that the tab is driven by the UI's own
  gestures (links, clicks) — a URL jump straight to a result page is a
  smuggled stage, not a human journey.
- **Console/DB-side steps are OPERATOR steps — announced, not smuggled.**
  When a stage has no human lever (e.g. opening a cold verse's seed
  window, a first gfam.train), it's run openly as an operator step with
  before/after shown in the tab, AND reported as a FINDING ("the product
  offers no human path for this step") for the user to rule on - that
  finding is how the product grows its missing UI faces.
- **The playwright suite is the replayable encoding of the human
  session** — headed, UI gestures only, waiting on real UI feedback
  (toasts, counts, rendered rows). NOT the thing the UAT is: the suite
  is so the journey can re-run tomorrow without me driving it live.
  Route-level API batteries (the replay drivers) live on as regression
  checks - their green output is NEVER presented as "the work, shown".
- **Evidence = the human's view**: screenshots from the SHARED visible
  tab (navigatePage/screenshotPage on the user's listed page), not a
  private headless browser's shot dir. "Show it in the browser" means
  the user's shared tab.

**Root cause this law exists (recorded so it stops recurring): I treated
speed/determinism (checked-in replay drivers, invisible POSTs, headless
shots) as serving QA; the user was watching for the work to HAPPEN in
the browser like their own S102 hand-driven holotype pass. "Show the
work" is not performance-after-the-fact; it is THE TEST METHOD.**

**Restart trap live-proven the same day**: a server "restart" via
`pkill -f 'python3 app.py'` matches NOTHING (the real cmdline is
`python3 /home/pi/imagebank-dev/app.py`) and the ssh session that ran it
can kill ITSELF (the pattern matches the ssh argv). The old process keeps
serving pre-deploy code — a UI QA then validates stale code and "finding
bugs" that don't exist. The restart idiom: kill by PID from `ps aux`
(flock wrapper too), then run `run-v2-server.sh`, then verify the NEW pid.
When in doubt on a UI mismatch between what you deployed and what you see:
`ps aux | grep '[a]pp.py'` and check the process START TIME vs deploy time.

### v2 surfaces + deploy

The current estate is **imagebank (v2)** deployed at `/home/pi/imagebank-dev`
(API + static shells, TLS :443, root-cron keep-alive `run-v2-server.sh`):
- `viewer.py` (the image page), `api.py` (JSON API), `web/queue.html` +
  `queue.js` (queue shells), `peek.py`/`api.peek_json` (hover peek).
- Deploy: `python 06-deployment/imagebank/pi_deploy.py`
  (same traps doctrine as deploy.py above; restart = kill by PID from
  `ps aux | grep '[a]pp.py'` — flock wrapper TOO — then let the keep-alive
  revive, poll `https://127.0.0.1:443/login` up to 90s).
  **pi_deploy's flags are ONLY `--check` / `--migration-prereqs` /
  `--live`** (the `--files/--target/--dry-run` set belongs to the v1
  `deploy.py`; do not guess flags — deploy is all-file, and pre-existing
  live drift is pushed along with the change). **THE LIVE GATE
  (2026-11-17, learned the hard way): a bare `pi_deploy.py` REFUSES —
  the push runs ONLY on `--live`, the user's own explicit decision.
  The assistant NEVER launches the push (including "the plan said
  deploy"); the user issues the push command; `--check` is the
  assistant's tool.** pi_deploy does NOT restart the server; the
  restart is the by-PID idiom above. Schema rides `pi_migrate.py`
  AFTER the deploy (idempotent; rebuilds the pairs cache).
- Synthetic tests: the QA batteries live in `qa-bank/tests/` (run locally —
  temp DBs — and ON THE WSL qa-bank as the release rehearsal; see the
  qa-bank README). The manifest ships ONLY the two live-host diagnostics
  (`test_filterpost`, `test_housekeep_check`) + `pi_s056_cron.py` to the
  Pi (S104g ruling: the suites are qa-bank's).
- **THE PAGE MODEL ERA (S110c→h, 2026-11-18)**: `design/pages.xml` is
  THE ONE MASTER; the emitted `web/page-model.json` + `web/estates.json`
  ride the artifact; the model carries its own verbs
  (design-model.inject) + the EL grammar (`${k}` + tpl; the param
  elements are documentation — the runtime value is the caller's) +
  **THE USERROLE LAW (S110h)**: `userRole="admin"` rides verbatim
  ($userRole); NO expressions/model machinery; the CONSUMING CALLER
  reads it and acts (a page renderer hides/shows as it wishes).
  THE MASTER IS EIGHT PAGES (S110h: login, dashboard, queue, viewer +
  cameras, queues, admin, site); `types/nav.xml` is the ONE CONTAINER
  TYPE (h:group anchor=[data-test-id='nav'] + h:a equals-href entries —
  every nav in every page includes it; the product carries
  data-test-id='nav' on every face). An empty locator chain is the
  DOC face (a documentary node), not a runtime miss. Consumers: the
  suites (pages.js's playwright surface), `/admin/design` (the TEST
  perspective: the diagnostic dialog over an iframe workspace — the
  live census), `/static/sitemap.html` (the DESIGN perspective:
  documentary tree + descs, 8 cards). Re-cut the master →
  `python 06-deployment/imagebank/design/make-design-json.py` (the
  compound refuses a non-canonical load; fail fast — its refusals are
  the parse laws: key= first, hints-on-leafs, tag-carriers-are-leaves,
  xml= refs relative to the referencing file), then the batteries
  (`test_page_xml.js` + `test_h_interceptor.js` +
  `test_model_surface.js`), then the dev rehearsal in the browser.
  THE LADDER: dev (:8443) → qa-bank (NOT yet run for this era) →
  live (ONLY the user's `--live`, verbatim).

### THE LOCAL DEV SERVER (dev-bank) — UI/route rehearsal before live (S075/S076)

Any queue/UI change is rehearsed on the LOCAL dev server before the live
deploy (the live browser-QA law still applies after the deploy — dev-bank
QA is the dress rehearsal that makes live QA boring):

- **Run**: `cd 06-deployment/imagebank && python app.py --port 8443`
  (plain HTTP, stdlib-only, Windows Python fine). No TLS, no args.
- **The DB is `harvest-v2-dev.db` at the dev-bank SIBLING**
  (`../dev-bank/harvest-v2-dev.db` — db.py's DEV_DB default rides the
  restructure; `IBV2_DB` wins when set).
- **The bank tree** (peeks, /raw, the junk-evict move target) = the
  `properties` file's `bank_root` (default points at the dev-bank sibling's
  `bank/`; the local `properties` file is git-ignored and OVERRIDES
  `properties.py` DEFAULTS for every key it names — **check for stale
  test-appended lines, they win**
  (S076 live-proven: a test's Temp-dir bank_root line starved the ingest to
  0 moves; last-line-wins in properties.read)).
- **THE SLICE + REBUILD (S076, replaces the pour): dev is built from a
  slice of LIVE, and the QA drives it through the real pipeline.**
  1. `dev-slice/make-slice.py` runs ON the Pi (read-only): slices the
     live estate by camera+day (CAM-0100/0101/0102 full + latest-2 days
     of CAM-02/05, walked back to >=50 imgs per window; **today excluded**
     — the live system is churning it), copies each pair jpg+xml
     back-to-back into RECEPTION shape (+ the manifest: counts, windows,
     the config rows = cameras/edges/roles/producers) and zips to
     /yermo/animals-count/devbank-slice.zip.
     TRAP (S076 live-proven): pool_paths sorts by mtime and bite()'s todo
     window is mtime-capped — copy pairs interleaved or the window sees
     one leg only (the XML-FIRST batch copy produced a 1000-wide window
     of lone XMLs: 0 moves).
     CAMERA SOURCES ONLY: no workflow rows, no assessment XMLs — pairs
     enter dev as 'new' with camera-origin detections, exactly as
     production (user law).
  2. scp the zip back -> `06-deployment/dev-slice/devbank-slice.zip`.
  3. `python 06-deployment/qa-bank/rebuild-devbank.py` (STOPS the dev
     server first; it refuses while :8443 answers): fresh DB via
     db.bootstrap (schema + addenda: triggers included), config rows from
     the manifest, GT ladder repointed into the estate's export (the
     addenda seeds REPO paths — the rebuild UPDATEs them), the qa user,
     **the pair stamps** (both legs of every pair get the SAME
     past-anchored mtime — the zip round-trip loses the rsync
     pair-adjacency the ingest's mtime window needs; a future-stamp bug
     produced the geometric-decay 167→90→49 stall; the stamps walk the JPG
     side ACROSS the sibling trees — the qa-bank's own build caught the
     per-dir variant pairing nothing), then **the REAL ingest_v2.bite**
     until the pool drains (settle 0), pairs cache rebuilt.
  4. `python 06-deployment/qa-bank/tests/qa-devbank.py --server
     --qa-password qa` (start the server first): A=drift asserts,
     B=distribution vs the manifest,
     C=happy edge walks (drives the NEW pairs through: new->junk,
     junk->rejected/assessed, claim/unclaim (the S075b claim-409 is
     GONE — frame_of reads provenance that now exists), stage (version
     gate), unstage, batch publish, reserve/unreserve),
     D=refusals (unknown queue, non-direct edge, bad/empty ids, >500
     backstop, 409 per-row, 207 mixed). Module mode (--db) runs A+B+C
     without the walk segments.
- The dev DB is git-ignored (`harvest-v2-dev.db*`), throwaway and CHEAP
  to rebuild now (the slice zip + one command); NEVER trust a "dev DB
  wiped" claim — check BOTH candidate paths before rebuilding.
- A QA login is created by the rebuild (`qa` / --qa-password, default
  `qa`; admin+editor+publisher).
- Fixtures must walk LEGAL edges — the DB trigger refuses direct status
  UPDATEs (`transition not allowed`); note: a direct UPDATE between two
  ALLOWED pairs SUCCEEDS silently (new->junk is legal!) — probe the
  trigger with an ILLEGAL pair that MATCHES ROWS (S080b: an UPDATE
  matching nothing succeeds silently without firing anything — the
  assess era leaves 0 new rows, so the probe uses
  assessed->published). An UPDATE matching nothing is a
  PHANTOM VERIFICATION.
- THE REBUILD NOW ASSESSES (S080b, the user's ask): rebuild-devbank.py
  runs ingest THEN the assess-v2 pass LOOPED until the new pool drains
  (the default per-pass limit 50; bounded 40 passes; onnxruntime needed
  locally — pip install onnxruntime, the model file is in-tree) — the
  estate lands assessed/junk exactly as production leaves it and the
  reception is EMPTY. An import error skips assess honestly (the pairs
  stay new; the pool still drains).

## THE MINOR ASSESSMENTS (phase 2, S080/S080b — the code-side doctrine)

A minor assessment = a function that reads a selected row-set and writes
an attribute, NO transition. Seam facts the next session inherits:

- assessment_tags: (camera, day, filename, tag_id) PK, value TEXT, FK
  ->images ON DELETE CASCADE, upsert (last assessment wins). The tags
  column on EVERY view (+ assessed_pairs via pi_migrate's
  probe-then-rebuild): db.tags_expr() is the ONE implementation, the
  {TAGS} token — the substitution law is now BOTH {CLASSES} and {TAGS}
  (every raw reader of schema-v2.sql). The correlation binds IMAGES's
  triple (at.camera = i.camera in the views), never w (workflow has no
  triple columns). idx_tags_image rides the join.
- assess_plugins.py: PLUGINS (the list = the buttons' order), plugin_of,
  for_status(scopes-filter), evaluate_one (an unknown plugin refuses
  honestly), write_tags (None-skip upsert), tags_of. focus stores the
  NUMERIC edge-variance measure (matchers.blur_measure — the blur LAW
  lives in matchers.py, NOT similar.py; a threshold change re-FILTERS,
  never re-runs); light_level takes one of the fixed 4
  (dark|mid|bright|overcast) and stamps every selected row.
- POST /action/assess-selected: streaming NDJSON (progress per
  ASSESS_PROGRESS_EVERY=25, ONE final done line, end-of-stream = the
  run-modal's self-close). Gates mirror /action/selected EXCEPT the
  role-edge gate (a tag write is not a transition; the gate is
  _may_assess = the held roles may perform one direct edge out of the
  queue). Abort semantics BOTH pinned: a mid-run (progress-line) break
  tags nothing; a final-line break keeps the completed tags.
- The api payload's `plugins` array drives the purple selection-bar
  buttons AND the filter panel's tag widgets (light: the 4-choice
  dropdown; focus: any/in-focus/blurry — the verdict binds at QUERY time
  against the blur.threshold knob, properties-read fresh per query).
  The filter params (tag_focus/tag_light) ride the stored-filters row
  and EVERY strip link.
- THE TAGGED-ROW CACHE REFRESH (S080b): the run route refreshes the
  assessed_pairs row per tagged image (a write without the refresh left
  the page's tags column empty — the browser flow caught it; the
  page's own render is the probe that matters).
- An EMPTY view's cursor.description can carry NULL names — never check
  a view's columns from cursor.description on an empty set; probe the
  DDL (db._view_columns, the WHERE-0 cursor).
- window.prompt() is BLOCKED in the integrated browser: an in-modal
  select (rmAskInput) is the input seam, the veil ON while asking,
  Cancel resolves null + closes.
- Dev-bank state + the poison law: the dev server is the by-PID idiom
  (a TWIN process caught S080: two app.py pids, the 09:49 one holding
  the socket). A probe that properties.set()s must redirect PATH first;
  blur.threshold IS in DEFAULTS now (500).

### Testing the server locally (the harness)

`harvest/ibtest_harness.py` (stdlib only) builds a synthetic `harvest.db`
+ bank tree, runs the real pipeline + `ib_server.py` on a local port, and
probes every route. It also asserts new feature behaviour (dropdown
contents, census, date validation) - add assertions there when you change a
route or a queue param. **The synthetic schema must mirror production**
(both `images` and `image_status` need `gt_universe`) or new queries fail.
Run: `python 06-deployment/imagebank/harvest/ibtest_harness.py` (expect
`probe failures: 0`).

## Data semantics that keep biting

- `image_path`/`xml_path` must point at the FINAL resting place
  (assess.py opens them after the move).
- `image_exists=1` is required by the workflow's new-pair join.
- Assessment graphs are excluded from 'old detection' sources via
  model_versions (harvest.py's skip_graphs rule).
- Old XML `frame` attributes are full-sensor ROIs (left, top, width,
  height) in 4056x3040 sensor coordinates; VOC boxes are
  output-image px. The 2021-era ScalerCrop mapping (old cameras were
  16:9 bands) is still an open correctness item.
- THE QUEUE SELECTION DOCTRINE (S075/S075b, 2026-11-13): multi-select is
  THE way to run transitions on many rows — every queue's direct edges are
  generated as row_actions on the ONE generic route, `POST /action/selected`
  (queue, to, ids form field = JSON array of {camera,day,filename}; optional
  gt_target_version for stage; per-row outcomes 200/207/409). The
  whole-filtered-queue TRANSITION bars are removed; the surviving whole-
  queue bars: junk empty (physical move; route kept, its bar retired) +
  publish (the GT-batch law — per-image publish stayed retired). 500 is a
  WARN-then-proceed guidance everywhere (the routes' >500 refusals exist
  only on the selection routes as backstops). Default queue order (D1):
  `day DESC, filename DESC` (picods filenames BEGIN with the capture time —
  ordering by filename IS time ordering; `idx_ap_order` matches). The
  doorways/queue links run in-place (pushState + render): QID/API are
  re-derived PER RENDER, and the selection set clears when the queue id
  changes — never assume module-load constants track the URL.