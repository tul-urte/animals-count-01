# Image Bank Plan: Harvest & Serve

Companion to [plan.md](./plan.md). Authoritative plan for the imagebank initiative.
Host record & inventory: [06-deployment/imagebank/imagebank.md](../06-deployment/imagebank/imagebank.md).
Created 2026-10-24.

## Objective

Turn the image bank RPi (`imagebank`, <bank-ip> Ã¢â‚¬â€ Pi 5, 4-core, 16 GB, idle) from a passive
SMB/HTTP dump into an active harvester: catalog the historical detections, re-assess them with
the current model, and surface ground-truth candidates for human triage Ã¢â‚¬â€ feeding the next
dataset build.

**Attack order**: harvest first (the hard part, and we want a feel for what's there).
The HTTP server overhaul comes later, done **together** with the picods nodes' HTTP servers
(one design, deployed to both imagebank and the kit).

## Scope

- **In**: CAM-02 + CAM-05 historical image+XML pairs (~180K pairs, 2021-era eb_v15Ã¢â‚¬â€œv19 graphs).
- **Out for now**: CAM-0100/CAM-0101 (current nodes, boxed jpgs only Ã¢â‚¬â€ revisit once the demo's
  CSV/XML counting path is wired), `/xenophon` archives, unclassified XML-less images
  (potential later trawl, see Phase F).
- **Web visibility during harvest**: static pages written to `/yermo/animals-count/harvest/` Ã¢â‚¬â€
  served immediately by the existing crude http.server. No server work needed up front.

## Phases

### Phase A Ã¢â‚¬â€ Catalog & first look *(start here)*
Get a feel for what's actually in the bank.
- [ ] `harvest/` scaffold on imagebank: `/home/pi/harvest/` (scripts, venv, `harvest.db`),
  web output dir `/yermo/animals-count/harvest/`
- [ ] **Indexer**: walk CAM-02/CAM-05 `*-CLASSIFIED/<date>/<graph>/*.xml`, parse VOC XMLs
  (label, score, box, graph, light, frame), pair with `CAM-XX/<date>/<basename>.jpg`
  - SQLite schema: `detections` (camera, day, filename, graph, label, score, box, light, frame),
    `images` (camera, day, filename, path, size, mtime, paired), `scan_state` (watermarks)
  - One-time full pass (~180K XMLs, minutes), then incremental by watermark
- [ ] **Stats pages** (static HTML, served by the crude server): per-camera/day counts,
  per-class totals, score histograms per graph, date coverage, pairing rate
- [ ] Review with user: is the shape of the data what we expected?

### Phase B Ã¢â‚¬â€ Old-era candidate ranking (no new model yet)
- [ ] Rank XML-only candidates: score bands per class, multi-graph days (same image assessed
  by several graphs Ã¢â‚¬â€ cross-graph agreement is itself informative), sample grids per band
- [ ] "First look" candidate page: thumbnail grids (lazy thumbnail cache) with old label+score,
  filterable by camera/date/class/band
- [ ] Estimate: how many strong candidates exist before any re-assessment?

### Phase C Ã¢â‚¬â€ Model re-assessment
- [x] Deploy the current ssd-mbv3 ONNX to imagebank (`/home/pi/harvest/model/`), runner via
  **onnxruntime** in a venv (fallback: OpenCV DNN via apt) Ã¢â‚¬â€ Pi 5 CPU, ~4 workers
- [x] Worker: batch by day, run the model on the **detection frame** (the XMLs' `frame`
  attribute = the region the old model watched), decode with the same
  `postprocess_ssd` contract as the live nodes
- [x] Write back to `assessments` (camera, day, filename, new_label, new_score, new_box,
  model_version, assessed_at); checkpoint every N; nice-level so SMB/CIFS is never disturbed
- [x] Measure real throughput on the Pi 5, then let the full 180K pass run overnight
  (7.3 img/s at 4 threads Ã¢â€ â€™ brownout crash at 1500; **3 threads = 6.7 img/s**,
  ETA ~7.5h, watchdog @reboot+hourly resumes after any crash)
- [x] Candidate pages gain the **new-model column** and the ranking views:
  - **Agreement**: old == new label, both Ã¢â€°Â¥ 0.5 Ã¢â€ â€™ quick-confirm queue
  - **Disagreement**: labels differ, both Ã¢â€°Â¥ 0.5 Ã¢â€ â€™ the most valuable triage (ground truth resolves these)
  - **New-only**: new finds with box-IoU < 0.2 vs all old boxes Ã¢â€ â€™ possible missed species
- [x] **XML export** (decided 2026-10-24): new detections mirrored to
  `/yermo/CAM-XX-CLASSIFIED/<day>/<model_version>/` as VOC XMLs Ã¢â‚¬â€ only Ã¢â€°Â¥1 detection,
  rewrite-always, full-image coords, assessment time in `<date>/<time>`, graph =
  model version; indexer excludes these graphs from old-era stats (verified 0 cataloged)

### Phase D Ã¢â‚¬â€ Triage & export  *(DONE 2026-10-24: claim/edit/confirm + conflict-aware export)*
Design (supersedes the original bullet list) Ã¢â‚¬â€ a **Layer model** with **per-image locking**:
- **Layer 0 (immutable)**: `/yermo/animals-count/cameras/CAM-XX(-CLASSIFIED)/...` Ã¢â‚¬â€ the bank;
  never written by triage.
- **Layer 1 (mutable, per-user)**: `/yermo/animals-count/edit/<camera>/<day>/<user>/<base>.xml`
  Ã¢â‚¬â€ a human's edited "pseudo-model". Created empty on claim; boxes seeded from any
  graph / the current model; fully editable (label, box, score, attributes).
- **Layer 2 (product)**: `/yermo/animals-count/export/<camera>/<day>/<base>.xml` Ã¢â‚¬â€ confirmed
  edits only.
- **Locking**: `active_edits` (per image key `camera|day|filename`) Ã¢â‚¬â€ a user *claims* an image
  (locks it + creates their empty Layer 1 file), *releases* (unlocks + deletes their file).
  While locked, other users see "Locked by <user>" and cannot claim.
- **Outcomes** (`image_status` per image): `confirmed` Ã¢â€ â€™ exported; `rejected` Ã¢â€ â€™ not exported;
  `released` Ã¢â€ â€™ no outcome (implicit). A non-human pseudo-model file is never modified Ã¢â‚¬â€
  human edits live in their own Layer 1 file.
- **Editor** (`/editor/<cam>/<day>/<file>`): read Layer 0 graph XMLs + the current-model
  assessment as seedable sources; seed appends boxes into the working set; save/publish/reject/release.
  *(Shipped 2026-10-24/25 S018cÃ¢â‚¬â€œf: claim pre-populates the edit copy with **all** sources' boxes,
  each tagged `source_graph` Ã¢â‚¬â€ the "delete the wrong ones" workflow; the editor is image-centre-stage Ã¢â‚¬â€
  a thin control bar (multi-select, delete, reload, +box, Save, Save+Publish, Save+Reject, Release)
  and a box panel that opens via a Ã¢Å“Å½ icon on the selected box; per-source box colours (stable
  palette hash of `source_graph`); category dropdown for labels (catalog Ã¢Ë†Âª GT classes);
  per-box attributes (selection-gated); locked image attrs (graph=user, source_day, server-enforced);
  zoom (ctrl+wheel, anchored) + pan (space/middle-drag) in a clipped frame; click-outside deselects
  instantly; save() stamps provenance; publish/reject redirect back to the originating queue Ã¢â‚¬â€
  the triage loop: click Ã¢â€ â€™ Ã¢Å“Å½ Ã¢â€ â€™ dropdown-retype Ã¢â€ â€™ delete wrongs Ã¢â€ â€™ Save+Publish Ã¢â€ â€™ next row.
  S019: the panel is a **tiny floating popup anchored at its box** Ã¢â‚¬â€ category dropdown + score +
  Ã¢Å“â€œ (apply+close), Ã¢Å“â€¢ (close, discards the popup's edits), Ã°Å¸â€”â€˜ (remove box); outside-click closes;
  attrs tucked behind an "attrs (N)" link; the selected box carries a smaller Ã¢Å“Å½ + Ã°Å¸â€”â€˜ badge pair.)*
- **Export engine**: scans all Layer 1 files; if a single source image was edited by **more than
  one user it hard-fails** (recorded in `export_conflicts`, nothing copied) Ã¢â‚¬â€ otherwise copies each
  confirmed image's single Layer 1 XML to Layer 2 (skipping + reporting images already in the
  Windows GT tree). Run from `/editor/export`.
- Server kit gained `do_POST` (backwards compatible); imagebank routes wired + verified end-to-end.

Remaining (open):
- [ ] **S062 Ã¢â‚¬â€ THE STAGED-PILE REVIEW + PUBLISH + dataset v4 (updated
  2026-10-30, S061)**: the local dev bank now holds **9,475 eb_v20 pairs
  STAGED @ eb_v20** (the replay; `pi_rehearsal_full.py`; export md5 =
  xml_hash Ã¢Ë†â‚¬ verified). The shape: (1) the USER's review in the browser Ã¢â‚¬â€
  `http://127.0.0.1:8443/q/staged` (filters work; the hover-peek draws the
  truth; classes+time columns populated); (2) the batch publish per
  approved slice (the bar adopts the assessment doc; producer stays the
  model); (3) the local bank = the eb_v20 GT edition Ã¢â€ â€™ dataset v4 (the
  pull: `export/eb_v20/run-rehearsal/` + the ground_truth manifest) Ã¢â€ â€™
  retrain ssd-mbv3 (+ the yololite line), the assessment round on the
  fresh model. The naming + library decisions live in
  [plan-model-library.md](plan-model-library.md); the live-bank mass
  cutover + the `network` hard-rename migrate with it (the lane details:
  workflow-transitions.md Ã‚Â§the-import-lane).
- [ ] **S061 (2026-10-30) DONE-threads**: the gate retirement (publish
  adopts the assessment doc), the direct-stage edge born+retired (the
  worker lane), the model-library registry + plan, the layers selector +
  floors, the QA round (the classes filter, the publish counters, the
  row-link contexts). See notes/2026-10-30 S061.md + plan-box-layers.md (BUILT)
  + plan-model-library.md (the decisions settled).
- [x] **THE 5A PSU (S054Ã¢â€ â€™S059-prep, UPGRADED priority)**: 13 undervoltage
  flags on THIS boot's quiet morning (06:44Ã¢â€ â€™10:29) Ã¢â‚¬â€ the cadence is worse
  than the surveyed brown-out cluster. **DONE: fitted 2026-10-30; verified
  2026-10-31** Ã¢â‚¬â€ zero dmesg undervoltage lines across the post-swap boot,
  `throttled=0x0`, EXT5V 5.11 V idle / 5.08 V under a 90 s 4-core burn
  (the v2 server parked for the burn window, then revived by its cron).
- [ ] **THE STAGED-NAMING FIX (S059-prep's bug)**: workflow.py's publish
  move expects src = staged/<cam>/<day>/<filename> but the staged layer
  carries BOTH namings (the pour's `<stem>.xml`, the editor's
  `<filename>`). The historical set is repaired (pi_batch_xml_repair.py);
  harden the move: try both, fail LOUDLY (a repair event) when neither.
- [ ] **PICODS101 KNOWN_HOSTS (S059-prep note)**: the Windows known_hosts'
  ED25519 is stale since the S040 rebuild (strict checking blocks ssh;
  `StrictHostKeyChecking=accept-new` re-keys; the fix is one line).
  *(RESOLVED 2026-10-31 S065b: the real key = SHA256:peJdO6ZujBpFDxx0
  (= root@raspberrypi, the S040 rebuild's key); the <bank-ip-2> IP
  entries already carried it, the 'picods101' NAMED entry was the stale
  pre-S040 holdout (SHA256:pKT5rW) Ã¢â‚¬â€ replaced with the true key fetched
  over the trusted channel; strict-check ssh through the name verified
  working. The bank's /root known_hosts holds nothing for the nodes
  (the S049 relay cleaned it) Ã¢â‚¬â€ nothing to fix there.
  RE-KEYED AGAIN 2026-11-12 S073: the card's 3rd reflash regenerated the
  host key (now SHA256:cHRAV2VSG5Gos2D5us3YVuOGK4D0/4wu5nN6iP2PC68); BOTH
  Windows stores (OpenSSH known_hosts IP entries AND PuTTY's session
  cache) had to be cleared Ã¢â‚¬â€ the rpi-from-windows skill now documents the
  two-store dance.)*
- [ ] **THE AUTO-REJECT EXPERIMENTS (S059, CONTINUING)**: the framework is
  in + LIVE-TRIED (the box/hybrid gate: honest stops at moved birds; the
  hash experiment: the pond frames swept; the knobs: containment
  0.80 / hash_distance 24 / cap 10 / budget 5s; the remaining trial
  drivers in debug/: ar_health.py + ar_health_run.py (the health/run
  pair), ar_lastrec.py, ar_restore.py (the gate flip pair with
  ar_hash_trial removed Ã¢â‚¬â€ recreate from the notes' one-liner when
  needed), ar_verify_states.py). EXPERIMENT list: containment
  sensitivity, the hash distance sweep, `match_empties=1` on the empty
  bursts, cross-day bursts (the window), and the auto-reject STOPS at
  junk rows (the regime interplay). The knobs: /admin/config, no deploys.
- [ ] **THE AUTO-TOOLBAR (S058 Ã¢â€ â€™ plan settled 2026-10-29 by user Q&A; timing = the user's
  GO Ã¢â‚¬â€ lands before S060, which itself may be delayed a while)**: `route_for()` stays the
  kernel; the work adds the multi-role base and generates the controls from
  `allowed_transitions` + `transition_roles`. **LARGELY LANDED 2026-10-29** (see
  items 2-4 + the viewer-direct-filter; only live browser review + any follow-ups
  remain). User-set decisions:
  roles = a `user_roles` join table with `users.role` DROPPED (pure join, every consumer
  migrates); **NO DERIVATION anywhere (user: "why derive anything?")** Ã¢â‚¬â€ the S057
  super-editor append in `roles_for()` DIES; the matrix is the complete law; powers come
  only from edges granted to roles the user HOLDS (a super user holds editor+admin);
  batch scope = the WHOLE filtered queue (the publish-bar doctrine generalised); row cap
  500. **RECREATE THE ORIGINAL ACCESS (user, 2026-10-29)**: the deploy's data grants are
  REQUIRED not optional Ã¢â‚¬â€ every current user's powers must be at least what they had
  before (Admin: admin-page gate + full editor/publisher powers via user_roles rows;
  Alaric: editor; Copilot: admin + the matrix row it was granted for the walkthrough) Ã¢â‚¬â€
  verified by a post-deploy toolbar/action comparison, no silently-narrowed   estate.
  **ONE ROLE PER TRANSITION (user, 2026-10-29)**: each edge may hold at most one
  `transition_roles` row; users can hold multiple roles instead. Chosen roles:
  `newÃ¢â€ â€™junk=editor`, `junkÃ¢â€ â€™assessed=editor`, `stagedÃ¢â€ â€™published=publisher`.
  The assessor keeps its trusted internal path for the first two edges, outside
  the user policy matrix. The policy page replaces the edge's role on save.
  The literal `admin` role stays the /admin/* page gate; queue/page role access
  (per-queue + per-page visibility) is DEFERRED deliberately (note 2026-10-29).
  1. **MULTI-ROLE FIRST** (the kernel's input becomes a role SET): addenda
     `user_roles(username, role)` (PK + FKs, ON DELETE CASCADE); pi_migrate probe-then-add:
     CREATE IF NOT EXISTS Ã¢â€ â€™ backfill `INSERT INTO user_roles SELECT username, role FROM
     users` while the column exists Ã¢â€ â€™ `DROP COLUMN` (sqlite Ã¢â€°Â¥3.35; the Pi's 3.40 ok);
     schema-v2.sql updated. `session_user()` returns `{username, roles:[Ã¢â‚¬Â¦]}`; every
     consumer migrates: workflow's `roles_for` (matrix rows ONLY Ã¢â‚¬â€ the S057 append
     removed)/`may_perform`/`_edges_from`/`findRoute` (any-of semantics), routes.py's
     guest gates, viewer's `EDIT_ROLES` (dies Ã¢â‚¬â€ the claim edge's matrix rows become the
     edit gate), admin.py's `_admin_only` + the users page (per-role checkboxes;
     create/update write the join rows; pi_seed_users.py's insert follows), api.py
     `me_json` + `staged_summary`'s may_publish (any-of), the auth-bridge payload
     (`roles` list joined the `role` string for the nodes, which only test ok), pages.py
     who-line, the three shells' `me.role === 'admin'` Ã¢â€ â€™ `me.roles.includes('admin')`.
     `workflow_events.actor_role` records the session's comma-joined roles (the audit
     keeps working). DEPLOY-DAY DATA GRANTS: the recreate-the-original-access set
     (Admin editor+publisher etc.) applied by pi_migrate/seed as visible rows, then
     VERIFIED on live (the post-deploy comparison).
  2. [x] **THE ACTIONS KERNEL**: `workflow.actions_for(con, fr, roles)` Ã¢â€ â€™
     `[{to, via, direct, needs}]` for every reachable target (direct edge or compound
     route); `needs` serves the extra form fields (stage Ã¢â€ â€™ the version list from
     `ground_truth_versions`). ONE implementation feeds the image page's
     `actions_html` (refactors onto it) and the queue API payload Ã¢â‚¬â€ the toolbar gate
     logic lives entirely in the data + this function. *(DONE 2026-10-29: implemented;
     the viewer and queue API use the shared descriptors; `test_actions.py` passes 6/6.)*
  3. [x] **QUEUE TOOLBARS FROM DATA**: `/api/queue/<id>` carries the queue's batch actions;
     queue.js renders a bar per DIRECT role-permitted edge on the queue's own status
     (queue id == status name Ã¢â‚¬â€ pin it with an explicit `status` field in QUEUES;
     compounds stay per-image on the editor page). Act on the whole filtered set;
     POST `/action/batch/<to>` = per-row `transition()` loop with a per-row outcome
     report (publish_batch_post is the template; the junk bar and the publish bar become
     instances, publish keeping its bespoke batch-name ask). CAP 500: above it the
     control refuses with the filtered size + the slice-down hint; client confirm gate
     (the junk-bar precedent) states the count + direction. *(DONE 2026-10-29:
     generated actions, stage-version selector, publish batch-name suggestion, direct
     per-row transitions, and filtered-queue cap/error handling implemented.
     The generated controls now use the compact image-page toolbar style.
     Queue paging remains on its own strip; merging it into the toolbar is a
     follow-up (user 2026-10-29).*
  4. **DEPLOY (one window)**: after local browser review, pi_migrate.py FIRST
     (backfill + drop, plus the single-role policy migration), then pi_deploy
     the code Ã¢â‚¬â€ `session_lookup` changes shape, the two sides must land together.
     *(DONE 2026-10-29: live window executed Ã¢â‚¬â€ backup
     /home/pi/imagebank-dev/backups/harvest-v2-20261029T181500Z.db (237 MB,
     integrity ok) + pre-deploy crontab copies; worker + server-keep-alive
     crons paused; `pi_deploy --migration-prereqs` then pi_migrate (user_roles
     already backfilled = 8 rows recreate-access, users.role absent,
     single-role matrix + uq index verified, views rebuilt, assessed_pairs
     178,117); full pi_deploy; crons restored; run-v2-server revive; live
     checks read-only via debug/verify_live_multirole.py ALL PASS Ã¢â‚¬â€ Admin +
     Alaric me.roles={admin,editor,publisher}; queue actions per status:
     assessed claim/reject/reserve, rejected unreject/discard, staged
     unstage/publish, junk assess/reject, new discard, claimed unclaim/stage;
     /admin 200 for held-admin vs 403 guest; workflow + GT counts identical
     to the backup (no live transition applied). Rollback: the two deploy
     backups under .deploy-backups/20261029T171{3,5}Z + the DB/cron backups.)*
  Tests: `test_multirole` (55/55), `test_actions` (9/9), `test_batch_actions` (8/8),
    `test_route` (21/21), `test_publish_gt` (60/60), `test_junk` (30/30),
    `test_autoreject` (46/46), and `test_editor_actions` pass. Local
    browser QA on the synthetic DB as admin verified the compact assessed/claimed/staged/junk
    action bars, stage-version selector, publish batch-name suggestion, count/direction
    confirmation, and the rejected-image viewer showing only its direct
    assess/junk controls (no compounds, no auto-reject there). The viewer
    toolbar filters to DIRECT edges only; the toolbar gate is presentation-only.
- [ ] **QUEUE/PAGE ROLE ACCESS (deferred deliberately, user 2026-10-29)**: after the
  auto-toolbar lands Ã¢â‚¬â€ per-queue and per-page visibility/permission driven by roles in
  data (today the /admin/* pages gate on the literal `admin` role and every queue page is
  visible to any session). The auto-toolbar's actions kernel is this item's foundation:
  the same actions/policy data generalises from controls to page access.
- [x] **SCHEMA TIDY (the relic drops, 2026-10-29, DONE)**: the user's table
  census settled the essential set Ã¢â‚¬â€ `workflow` (the state + its trigger
  backstops), `ground_truth` (the export registry, version+batch), and
  `assessed_pairs` (the view's 1.1sÃ¢â€ â€™0s cache) essential; **`schema_version`**
  (never wired up Ã¢â‚¬â€ pi_migrate is the migration mechanism) and **`categories`**
  (superseded by the scan_state `editor.categories.<universe>` cache) dropped.
  Both schema-v2.sql copies deleted the DDL, schema-addenda.sql now carries
  idempotent `DROP TABLE IF EXISTS` for both (a live DB that still holds them
  loses them on the next apply), test_drop_relics.py covers fresh +
  legacy-shaped paths; the follow-up window applied it live (backup
  harvest-v2-20261029T174000Z.db first; verified relics_absent, integrity ok,
  fk 0, workflow/pairs/gt counts unchanged) + deployed the schema files
  (pi_migrate re-run idempotent, no-op).

- [x] **THE QUEUE SHELL'S COLUMNS PANEL + PAGER MERGE (S060c, 2026-10-29, DONE Ã¢â‚¬â€
  deployed live the same day, see S060d)**: user_filters gained **type**
  ('filters' | 'columns', PK (username, queue, type); the table/cols names
  unchanged, user-set); pi_migrate probe-then-row-copy rebuilds old-shape
  tables (stored filter rows -> type='filters'). The columns store:
  [columns.py](C:/projects/animals-count/06-deployment/imagebank/columns.py)
  (available = the queue's own specs; upsert validated against ALL known
  columns = mode-switch-proof; all-blank = reset) + `/api/columns/<qid>`
  GET/POST/DELETE. `/api/queue/<id>`'s payload gained the columns object
  {visible, available, saved}; queue.js renders visible (spec-ordered) and
  the panel's checkboxes (filename FORCED visible Ã¢â‚¬â€ the peek/editor link
  ride it, user call). The toolbar row settled as user-specified:
  right-justified `<queue-actions> <paging-controls>` (actions butted
  against the paging group; the standalone strip div is gone) and the
  queue bars in the image-page toolbar's order (queues.ACTION_ORDER, one
  law with viewer.actions_html Ã¢â‚¬â€ "the same actions with multiplicity",
  user confirmed). The simple spec gained a `date` (ISO, sortable via the
  day expr) + `time` (capture_time, the picods filename parse shared with
  the viewer breadcrumb) column pair. Tests: test_columns.py 25/25; the
  full 15-battery set green pre-deploy. The fillColumns stray-quote bug
  was caught by the user's live QA (unticks re-checking) and fixed.

  *(DEPLOYED LIVE 2026-10-29 as S060d Ã¢â‚¬â€ one window: backup
  harvest-v2-20261029T184224Z.db + crontab copies; worker + keep-alive
  crons paused; server stopped; migration (user_filters rebuilt, 2 rows
  kept as type='filters'); full deploy; crons restored; keep-alive
  revived. ONE in-window incident: pi_deploy's FILES omitted columns.py,
  the live queue route 404-crashed; the list was fixed, the file pushed,
  and the verification re-run ALL PASS (the columns round-trip, the
  narrowed payload, the unified action order, the merged pager strip,
  workflow/GT counts equal to the backup, relics still absent,
  properties untouched). The omission is structurally closed: FILES is
  the single deploy manifest and it now names the module.)*

- [ ] **PICODS POWER-SAVING REVISIT (S054, user-set)**: there is already an
  attempt at power-saving in the nodes' startup Ã¢â‚¬â€ revisit it (what it
  does, what it actually saves) and see if we can improve it. Pairs with
  the imagebank PSU item: the estate's power posture matters after the
  S054 brown-out.
- [ ] **THE IMAGEBANK PSU (S054Ã¢â€ â€™S059-prep, the morning finds it WORSE)**: 32
  undervoltage flags across the S054-era boot + **13 flags on THIS morning's
  quiet boot** (06:44Ã¢â€ â€™10:29, `throttled=0x50000`, no throttle events) Ã¢â‚¬â€ the
  cadence climbs even with the workers idle. **Fit an official 5A/25W
  Pi 5 PSU (or equivalent) FIRST Ã¢â‚¬â€ before any bulk training pull/worker.**
- [ ] **NETWORK + SECURITY ANALYSIS (S050, user-set)**: the estate's network shape is now
  clear (ColchFibre_0F15 = the street's main router; 3 Deco units = OliveGrove, the
  estate's home; imagebank wired into a Deco; the laptop must be ON OliveGrove to reach the
  nodes). The analysis should include: the Deco/main-router separation (guest isolation,
  the 0.xÃ¢â€ â€68.x routing), device password rotation (all the RPis + the Decos + the NAS),
  the CIFS credentials on the nodes, and the TLS CA's custody. **Password changes on all
  the devices ride this analysis.**
- [x] ~~**AUTO-REJECTED status (S048f note, user set)**~~ Ã¢â‚¬â€ superseded: S052's
  `junk` absorbs the serial no-detection pool (the assessor's auto-discard),
  and S059's auto-reject covers the "similar images rejected in bulk" need
  (the cascade over the queue-forward, not a new status).
- [ ] Transform flat Layer 2 (`<cam>/<day>/<base>.xml`) into the **eb_v20 class-tree** shape
  (`(pending)` Ã¢â€ â€™ class folders) that [01-ground-truth/eb_v20](../../01-ground-truth/eb_v20) consumes Ã¢â‚¬â€
  the export currently writes the flat shape; binning by primary label is a follow-up.
  *(First confirmed edits exist now Ã¢â‚¬â€ e.g. the CAM-05 cat Ã¢â‚¬â€ so this is becoming live, not theoretical.)*
- [x] Wire export/triage counts into the stats landing page (home).
  *(DONE 2026-10-24 S018: confirmed / rejected / locked / already-ground-truth cards, all via
  the SQL library.)*
- [x] Queues as *named, user-manageable queries* over the triage state.
  *(DONE 2026-10-24 S018: server-side JSON SQL libraries Ã¢â‚¬â€ `ib-server/sql/{queries,views,queues}.json`
  + `querylib.py`. S019 superseded the label-only agree/disagree pair with the **workflow state
  machine** (below): every queue is now backed by a `workflow_state.status` (mutually exclusive
  by construction) Ã¢â‚¬â€ see "Workflow state machine (S019)".)*
- [x] Extract all static SQL from the server kit + speed up home.
  *(DONE 2026-10-24 S018: every query from ib_server.py + harvest_manager.py lives in
  `queries.json`; shared logic became SQLite views in `views.json` (old-graph exclusion via
  a `model_versions` mirror table Ã¢â‚¬â€ kills the per-row `SELECT DISTINCT` scan class; single-object
  filter; pairing; latest-assessment). `assessments` gained normalised `top_label`/`top_score`
  (backfilled by `harvest/migrate.py`, maintained by assess.py) so home's agree/disagree tally
  is one set-based query Ã¢â‚¬â€ the 200K-row Python loop and its 200K LIMIT cap are gone.)*
- [x] Reconcile the Windows ground truth with the bank (the S018 "NB").
  *(DONE 2026-10-24 S018: `harvest/gt_reconcile.py` Ã¢â‚¬â€ manifest (Windows) walks eb_v20 VOCs
  (12,125 mappable + 5,567 old-era/unmappable set aside), load (Pi) upserts the
  `ground_truth` table keyed `camera|day|filename`, diff reports coverage + class mismatches.
  Editor shows a GT banner; export **skips and reports** GT images. S019: GT is also a
  *workflow status* (sticky + terminal); the `exclude_gt` queue params are retired Ã¢â‚¬â€ the
  status field encodes it. Found en passant: 2 images duplicated across GT class folders.)*

### Workflow state machine (S019 Ã¢â‚¬â€ DONE 2026-10-25)

One materialized table, `workflow_state` (one row per indexed image+XML pair), written by a
single refresh step (`harvest/workflow_state.py`, nightly after assess) + the server POSTs
(claim/release/publish/reject update `status` in-transaction). **Every queue reads
`status`/`bucket` directly** Ã¢â‚¬â€ mutually exclusive by construction, no per-request view
evaluation, sorting/filters are free:

| status | meaning | queue |
|---|---|---|
| `new` | image+XML indexed, no assessment yet | `/queues/new` |
| `assessed` | current model assessed it (bucket computed) | `/queues/assessed` + 4 bucket queues |
| `claimed` | a human holds the edit lock | `/queues/claimed` |
| `rejected` | human outcome: not truth (terminal) | `/queues/rejected` |
| `published` | human outcome: truth (exported; terminal) | `/queues/published` |
| `GT` | absorbed into the Windows GT tree (sticky + terminal) | `/queues/gt` |

Transitions: indexÃ¢â€ â€™new Ã‚Â· assessÃ¢â€ â€™assessed Ã‚Â· claimÃ¢â€ â€™claimed (gate: only from new/assessed/GT) Ã‚Â·
releaseÃ¢â€ â€™assessed|new Ã‚Â· publishÃ¢â€ â€™published Ã‚Â· rejectÃ¢â€ â€™rejected Ã‚Â· gt-loadÃ¢â€ â€™GT.
**Outcome wins**: a model re-run recomputes sides/overlap/bucket but never re-files a
published/rejected/GT image into triage.

Buckets (the slicing of `assessed`; NULL when either side is missing Ã¢â‚¬â€ multi-object old XMLs
aren't in `v_single_obj`, model found nothing, etc.):
`expected-correct` (overlap Ã¢â€°Â¥ threshold, same label Ã¢â‚¬â€ *replaced agree*),
`expected-mis-categorised` (overlap Ã¢â€°Â¥ threshold, different label Ã¢â‚¬â€ *replaced disagree*),
`spatial-disagree` (overlap < threshold, same label Ã¢â‚¬â€ the S018c observation),
`unrelated-find` (overlap < threshold, different label). Overlap threshold default 0.5
(`workflow_state.py --iou 0.4` re-buckets; a threshold change forces a full rebuild; the
configured value lives in `scan_state`). **Overlap = intersection-over-smaller
(containment), not IoU** Ã¢â‚¬â€ measured on the live bank, old boxes are a median 5.4Ãƒâ€” the
area of the current model's tight boxes, so a concentric pair caps at IoU Ã¢â€°Ë† 0.19 while
containment reads Ã¢â€°Ë† 1.0 ("similar box" = the smaller box sits mostly inside the larger).
With plain IoU the buckets found 143/32 rows at 0.5; with containment, 4,551/3,905.

Old side = strongest old single-object find (full-image VOC px). New side = latest
assessment's top find mapped through the detection frame to full-image px
(`write_xmls.box_to_full` semantics Ã¢â‚¬â€ same as the editor seeds).

Queue pages (S019): **paged sortable tables** (click any column header; sort comes via a
whitelisted `?sort=`), a filter form per queue, **hover-peek** Ã¢â‚¬â€ a Ã°Å¸â€˜Â per row loads ONE image
in a floating preview with both boxes drawn (old red, new green; no row thumbnails Ã¢â‚¬â€ a
180K-image bank can't pre-generate them all), a count-aware pager, and **session-remembered
filters/sort/page** (one cookie per queue, `Path=/queues/`, session-scoped). `agree` /
`disagree` / `triaged` retired; `newtop` / `gt-mismatch` / `conflicts` stay as diagnostics
(newtop opts out of the count: `"count": false`).

### Workflow state machine v2 (planned 2026-10-27 Ã¢â‚¬â€ supersedes S019 semantics where noted)

The sketch + corrections of 27 Oct 2026. Changes vs the S019 machine:

- **`staged` status is new** (between `claimed` and the outcomes): the editor marks an image
  ready-for-publish; the full outgoing set is `rejected | reserved | claimed | assessed | published`.
- **`reserved` status is new** Ã¢â‚¬â€ a holding area to extend the workflow later, not a verdict.
  Out: `un-reserve Ã¢â€ â€™ assessed` only.
- **`published` stays a real workflow status** (terminal, **no unpublish**) Ã¢â‚¬â€ it is what stops
  images re-entering the workflow, protecting the ground truth; published images are view-only.
- **`GT` status is RETIRED.** The GT concept is simply "what's already GT, i.e. what's been
  published" Ã¢â‚¬â€ an **image-level attribute** (`is_gt Ã¢â€¡â€ status='published'`), not a state. The
  process that marked 532 images up as `GT` in the DB was wrong; the fix is part of this plan
  (migrate `GT` rows to `published`, drop the status, retire `/queues/gt`).
- **All imagebank images identifiable in the ground truth must end in `published`** (reconcile
  GT-known pairs whose status isn't published; they become view-only).

Transition table (v2):

| from | Ã¢â€ â€™ | via |
|---|---|---|
| new | assessed | assess |
| assessed | claimed | claim |
| claimed | assessed | release |
| claimed | staged | stage |
| claimed | reserved | reserve |
| staged | claimed | claim |
| staged | assessed | release |
| staged | reserved | reserve |
| staged | rejected | reject |
| staged | published | publish |
| reserved | assessed | un-reserve |

Terminals: `published` (into the GT, no exit), `rejected` (dead end). **Outcome wins** is
unchanged: re-assessment never re-files a published/rejected image into triage; the claim gate
widens to accept from `staged` (return to editor without passing through assessed).

### Phase E Ã¢â‚¬â€ HTTP server overhaul (imagebank + picods together) *(deferred until harvest is in hand)*
- [x] One design for both: stdlib `http.server` subclass, breadcrumbs, thumbnails, camera/date
  awareness, harvest landing at `/`; picods version serves the RAM drive live view (8080) and
  imagebank version serves the bank
  *(DONE 2026-10-24: shared kit Ã¢â‚¬â€ imagebank home/categories/calendar/day-detail on :80;
  picods live overlay/stats/config on :8080, both nodes; see session notes S015)*
- [x] picods kit `lib/http_server.py` and imagebank's server converge on shared page templates
  (kit `site/` assets already cover bootstrap/jquery/datatables)
  *(superseded: the shared pages.py chrome replaces both; the old lib/http_server.py is retired)*
- [x] Same cron keep-alive idiom on both; atomic deploy (temp-file + `mv`)
  *(imagebank: run-server.sh marker swap; picods: run-from-cron.sh keeps the server alive
  every minute, independent of detector sessions Ã¢â‚¬â€ survives night light-gate cycling)*
- [ ] Thumbnails: lazy-generated cache (phase 1 ships full images, LAN-tolerable)
- [x] picods live view: raw frame + overlay replacing the burned-in boxed image (drop boxed
  images later per user Ã¢â‚¬â€ the XML sidecar makes them redundant)
  *(DONE 2026-10-25 S022b: the demo stores raw images only (timestamp burned in, boxes NOT);
  the boxed pipeline is deleted; the live view draws boxes client-side.)*
- [x] Box overlays correct (single-SVG: image + boxes in one viewBox Ã¢â‚¬â€ aligned by
  construction; per-source colors for future multi-model swap/combine; foundation for
  box editing Ã¢â€ â€™ new XMLs).
  *(DONE + corrected S023/S024: per-category colours (crc32 Ã¢â€ â€™ kit palette, shipped as a
  `color` attribute); geometry now correct Ã¢â‚¬â€ see the S024 bullet below. Verified with the
  right invariants: 30/30 boxes in-image, 0 degenerate, 0 outside the ROI.)*
- [x] **IMX500 box-coordinate fix** (S023/S024, live-proven on picods101): with an
  inference ROI set, the firmware emits **ROI-relative fractions** Ã¢â‚¬â€ picamera2's
  `convert_inference_coords` maps them against the full sensor and clips (boxes piled at
  the ROI edge, degenerate zero-size rects, the old bank's out-of-bounds symptom class).
  And the ISP output is the **ScalerCrop band** of the sensor (1920Ãƒâ€”1080 from 4056Ãƒâ€”3040
  shows sensor rows 380Ã¢â‚¬â€œ2660), not the full 4:3 Ã¢â‚¬â€ full-sensor proportional math sits
  everything ~115 px too low. The demo's `inference_box()` now maps fractions Ã¢â€ â€™ sensor
  px via the ROI Ã¢â€ â€™ image px via the live ScalerCrop, returns **corners**, publishes
  `scaler_crop` in latest.xml + sidecars; server + live-page scripts share the mapping.
  Ã¢Å¡Â  imagebank follow-up: `frame_geometry.sensor_roi_to_image_box` still assumes
  full-sensor proportions for the 2021-era XMLs Ã¢â‚¬â€ check an old frame against a known
  object; if displaced, add the scaler-crop correction.
- [ ] Box editing UI (user as "master" model: adjust/add boxes, save as new XML) Ã¢â‚¬â€
  explicitly NOT yet a goal; noted so the overlay facility keeps it in reach.

### Phase EÃ‚Â½ Ã¢â‚¬â€ picods live node roadmap (2026-10-25, user's list)
- [x] **Box tidy**: drop inner boxes of the same category (containment dedup per label at
  the parse gate Ã¢â‚¬â€ the firmware NMSes overlaps, not nestings).
  *(DONE 2026-10-25 S025: `boxes.box_contains` + `drop_contained_same_label` gate in
  `parse_detections` behind hot key `DETECTION_DROP_CONTAINED` (default True); outer box
  survives, mutual containment (identical boxes) keeps the first. Live A/B-proven on
  picods101: 0/40 frames with containment vs 7/30 baseline, negative control 7/25 with
  the gate hot-off. Offline diagnostic: `diagnostics/check_box_tidy.py` +
  `sweep_containment.py`.)*
- [ ] **Auto-level**: auto-tune `MIN_LIGHT_LEVEL` from observed scene-light history
  (census substrate from the S022 auto-cal sketch; its own topic like auto-tuning).
- [ ] **Auto-track**: centre the detection frame on detections Ã¢â‚¬â€ re-emit
  `DETECTION_FRAME` toward the detection centroid (hot-applied by the demo loop; the
  dormant `maybe_move()`/`box_centre_move()` in cam.py is the ancestor).
- [x] **Suspend/restart from the config page**: `SUSPENDED=1/0` writes to
  service.properties (demo exits within a cycle; cron revives on resume).
  *(DONE 2026-10-25 S027: the config page's restart button runs the handshake
  server-side (root) and streams progress Ã¢â‚¬â€ suspend Ã¢â€ â€™ verify stopped Ã¢â€ â€™ resume Ã¢â€ â€™
  wait for revival; restart-required values (SENSOR_MODE dropdown) auto-run it
  on save. Live-proven on both nodes; picods100 switched itself to
  SENSOR_MODE=2028x1520 through it.)*
- [x] **Storage frame** (movable, like the detection frame): today storage = the entire
  aperture frame (no separate crop). Design: capture at a larger aperture, crop at save
  time around a draggable sensor-space rectangle; sidecar boxes offset accordingly;
  moving it needs no restart.
  *(DONE 2026-10-25 S026 as the **capture frame** Ã¢â‚¬â€ superseded by the frame hierarchy:
  the main stream now runs at full-sensor SENSOR_MODE and storage IS the CAPTURE_FRAME
  crop of it, 16:9-constrained inside the sensor, draggable live in the full view,
  sidecars carry `capture_frame`/`sensor_mode`; the demo re-constrains + persists every
  cycle so no restart is ever needed to move it.)*
- [x] **Constrained image sizes on /config**: enumerate Picamera2 sensor modes + the
  model's input shape (`NetworkIntrinsics`/`get_input_size`), offer only valid aperture
  choices; applying needs a detector restart.
  *(DONE 2026-10-25 S026 for the sensor modes: the IMX500 exposes exactly 4056x3040 and
  2028x1520 Ã¢â‚¬â€ `SENSOR_MODE` on /config, validated, restart to apply. Aperture choices
  ride with model switching. **S028 follow-up**: the binned mode's main is the full
  sensor *downscaled*, not a crop Ã¢â‚¬â€ `mode_scale` now applies at the image boundary
  (frames stay full-sensor px everywhere); both modes live-verified, both nodes run
  the binned mode by choice. S027: the mode dropdown + restart handshake are the
  restart-required section of /config.)*
- [x] **Model switching on /config**: pick from the imagebank bundle catalogue +
  stock COCO; writes `GRAPH`/labels/filter/per-model thresholds; restart to apply.
  Bundle layout + the `{GT version}_{model type}_{epoch}.rpk` filename law live in
  [plan-model-library.md](plan-model-library.md). *(DONE 2026-11-12 on picods102:
  animals -> stock COCO -> animals, real COCO detections, healthy restored live
  page; S070. Deployed and UAT-verified on picods102 + picods100; picods101
  remains the outstanding node.)*

### Phase F Ã¢â‚¬â€ Extensions (later, optional)
- [x] **Spatial-disagree queue** (user observation, S018c): old and new agree on the *label* but the boxes sit in entirely different places Ã¢â‚¬â€ the old graph's detection frame meant it never saw the full picture, so its box can be beside the object the new model found.
  *(DONE 2026-10-25 S019: `workflow_state.bucket = spatial-disagree` Ã¢â‚¬â€ one of the four bucket queues, computed by `harvest/workflow_state.py` from the old find vs the frame-mapped new top box with an IoU threshold. The sibling bucket `unrelated-find` (low IoU + different label) covers the "two different objects" family.)*
- [x] **Check the picods frame mechanism** (user, S018e; resolved 2026-10-25): IMX500 `frame` is a full-sensor ROI in `(left, top, width, height)` coordinates (sensor 4056Ãƒâ€”3040), not output-image `(x0,y0,x1,y1)`. Old detection boxes are already in full output-image pixels; do not remap them. The GT example ROI `[680,692,2600,1772]` maps to `[322,246,1553,875]` on its 1920Ãƒâ€”1080 image, which contains the old sparrow box `[544,370,804,539]`. Imagebank had clamped the sensor tuple as image corners, making the dashed frame appear at bottom-right and re-assessing the wrong crop. Shared sensor-to-image conversion now drives assessment, workflow boxes, editor overlays, and XML exports. The target image's corrected overlay confirms the remaining current-model box disagreement is a model prediction error. The target was reassessed with `frame-v2`; the corrected pass runs under the 3-thread supervisor (in progress 2026-10-25, ~32.5K/169K, ETA ~19:30 GMT; the hourly cron freshens workflow + XML mirror while it runs Ã¢â‚¬â€ see S021).
  *(Ã¢Å¡Â  S024 follow-up: the live probes proved the 2021-era camera mode was very likely the 16:9 ScalerCrop band of the 4:3 sensor Ã¢â‚¬â€ imagebank's `sensor_roi_to_image_box` still assumes full-sensor proportions. Verify an old XML's frame against a known object before trusting spatial queues; add the scaler-crop correction if displaced.)*
- [x] **Realtime node frame controls** (2026-10-25): the existing config page exposed score/NMS controls, but not frame or scene-light threshold; the live demo also never applied `DETECTION_FRAME` because its ROI calls were commented out. The updated server/demo/config are deployed on picods101: the authenticated config page validates and hot-applies full-sensor ROI, score, and light thresholds, and the live view overlays the ROI. Verified by changing to full-sensor ROI and score 0.25, observing the new frame in `latest.xml` and threshold in live stats, then restoring the original ROI `(800, 200, 2496, 2496)`, score `0.5`, and light floor `3`. picods100 was observed read-only and left untouched.
- [ ] CAM-0100/0101 inclusion Ã¢â‚¬â€ the XML path is wired (S021+): CAM-0101 pairs (jpg + VOC XML) land on the NAS daily and were verified live (S029); the remaining gap is the imagebank indexer's `--cameras` list (`harvest.py index` defaults to `CAM-02 CAM-05`). When the list is extended, the incremental indexer picks the day folders up from the NAS on its next pass Ã¢â‚¬â€ CAM-0101's 2026-10-25 detection pairs are already sitting there. Note the purge-list eras (S023/S024/S028/S029) should be purged first.
- [ ] Unclassified-image trawl: run the current model over XML-less days for new-only finds
- [ ] Re-assessment re-runs on model upgrades (`assessed_with != model_version`)

### Phase G Ã¢â‚¬â€ the v2 build (imagebank-dev) *(DONE 2026-10-27 S043: built + deployed; run-in continues)*
- [x] Build the parallel v2 in `06-deployment/imagebank-dev/` (blueprint-v2.md's scope:
  new schema `xml_docs`+`detections`+`workflow`+events, the 14-edge diagram-as-data,
  roles/sessions/auth, admin pages, queues as views, same-URL cutover shape)
  *(checkpoints 1Ã¢â‚¬â€œ5 + workers built and verified through HTTP: db/auth/queues/editor/admin
  + ingest-v2/assess-v2; S043)*
- [x] Deploy to imagebank (the pour cutover): v1 frozen cold (cron commented, rollback
  intact), 17 files pushed md5-verified, harvest-v2.db poured from a fresh .backup
  (188,463 workflow rows = v1), users seeded, HTTPS :443 via the tiny CA + plain-HTTP
  redirect, v2 workers on cron
  *(S043; runbook: `06-deployment/dev-bank/cutover-runbook.md`, gotchas 1Ã¢â‚¬â€œ9)*
- [x] Editor polish to parity-or-better vs v1: same-origin images (/raw + HEAD),
  materialised assessed_pairs + indexes + category cache (nav 2.9s Ã¢â€ â€™ 0.53s),
  deterministic ordering + window-function rank, walk-on-action lands on the next
  queue item, sticky edit mode, claim-gating (+ poured-orphan adoption), the
  claimedÃ¢â€ â€™rejected edge, Ã¢â€ Â» reclaim (re-seed from assessment, dedupe by label+box),
  box/background context menus, dblclick focus (box Ã¢â€°Â¤ 3/4 a dimension, centred),
  bounds-clamped pan
  *(S043; all browser-verified)*
- [x] **Data issues pass** *(S066, 2026-10-31 Ã¢â‚¬â€ all items settled against the live DB, read-only): claimed orphans = 0 (S048c repair held); `legacy-GT` rows = 0 (the S049 fix-up held; 661 GT rows, 18 real labels); assessed_pairs rebuild-vs-view ZERO drift across 178,240 rows Ãƒâ€” 22 columns (the S043 sample drift cannot reproduce Ã¢â‚¬â€ the bucket slice was retired from the view 2026-10-27); `<assessment-json>` marker paths audited audit-only Ã¢â‚¬â€ no code path opens them as files (all read sites exists/isfile-guarded or det-row-only), 661 published rows all carry real hash+batch_dir; the from-scratch populate compare SKIPPED (user decision Ã¢â‚¬â€ audits clean, assessor-heavy run unjustified; the runbook plan stays on file); CAM-0101/CocoTest GT-home decision PARKED again (rides the deferred S062 round).
- [x] **Node-cam purge + provenance** *(S067, 2026-10-31 Ã¢â‚¬â€ user-approved reset of CAM-0100/0101: the node-cam trees + junk residue parked to `/yermo/animals-count/purge/20261031T175028Z/`, 11,362 image rows + docs/dets/events/workflow deleted camera-scoped, DB backup pre-reset; the frame/box UNIT FIX landed with it Ã¢â‚¬â€ `detections.capture_frame`+`sensor_mode` provenance columns, `ingest-v2` records them, `frame_geometry.sensor_roi_to_image_box` decodes the ROI band-relative when provenance exists (legacy whole-sensor math unchanged), the viewer/editor/assess-v2/db.pour consume it; the S063 rename's 702 dead-producer docs re-seated (the FK trap pi_migrate's check caught); the superseded bare-ep29 assessment generation purged (98,183 docs, user-approved; frame-v2 now the sole assessor layer). picods101's first post-reset pairs carry full provenance; browser-verified ROI/box units match. picods100 at its light gate Ã¢â‚¬â€ pairs land at dawn.)*
- [ ] **Cutover formalities once run-in is agreed**: retire the `#IBDEV-FROZEN` comments (the root-cron line was parked-with-a-note 2026-10-31; delete the parked line + import the TLS root on client machines). *(V1 dirs/processes: DONE 2026-10-31 Ã¢â‚¬â€ `/home/pi/{harvest,imagebank,server}` + `run-server.sh` REMOVED after archiving to `/yermo/animals-count/old-build-process/pi-v1-decommission-20261031T114201Z.tar.gz`; v2 self-hosted off v1: `/home/pi/v2venv`, postprocess_ssd.py + frame_geometry.py in imagebank-dev.)*
- [ ] **S0XX Ã¢â‚¬â€ RSYNC INGEST (replacing the picodsÃ¢â€ â€imagebank CIFS mounts)** *(user-set
  2026-10-31: feasible+sensible; PARKED Ã¢â‚¬â€ but see plan-replace-samba-with-rsync.md:
  **EXECUTED COMPLETE 2026-10-31 S065** Ã¢â‚¬â€ both nodes on rsync-as-identity
  (picods_100/picods_101), the callhome authenticated per node, the CIFS
  mounts parked + unmounted, smbd stopped, the /cameras page live)* Ã¢â‚¬â€ the full
  plan lives in [plan-replace-samba-with-rsync.md](./plan-replace-samba-with-rsync.md):
  why (no persistent mount to rot, real atomicity kills the settle-window guess,
  per-node identity, per-file integrity), the R0 identity/flag-set/mtime/soak
  decisions, imagebank prep (R1), the ingest dotfile-skip hardening + harness
  assertions (R2), one-node-first cutover with a live full-cycle watch (R3),
  soak + second node (R4), and the CIFS teardown + doctrine updates (R5),
  with rollback at every stage.

## Decisions & defaults

| # | Decision | Default (unless overridden) |
|---|---|---|
| 1 | Triage marking | Static pages first (Phase B/C), marking arrives with Phase D |
| 2 | Export target | eb_v20-style `(pending)` tree on `/yermo`, not the Windows ground-truth tree |
| 3 | Scope | CAM-02 + CAM-05 pairs only for now (CAM-0100/0101 excluded per user, 2026-10-24) |
| 4 | Inference engine | onnxruntime in a venv; OpenCV DNN as fallback |
| 5 | Web auth | None for now (LAN-only; SMB is guest-writable anyway) |
| 6 | Old-style listing | Keep a plain `/raw/`-style route when the new server lands |
| 7 | `published` semantics | Real workflow status; terminal, no unpublish; protects the GT (2026-10-27) |
| 8 | GT semantics | Derived image-level attribute: `is_gt Ã¢â€¡â€ status='published'`; the `GT` status is retired (2026-10-27) |
| 9 | `reserved` semantics | Holding area for future workflow extension, not a verdict; exits only via un-reserve (2026-10-27) |

## Risks / notes

- NTFS via ntfs-3g: fine locally, but SQLite lives on the **SD system disk** (`/home/pi/harvest/`), never on the NTFS share; web outputs to `/yermo` are plain files.
- The old XMLs' boxes are full-image coords (1920Ãƒâ€”1080); re-assessment runs on the detection frame, not the old box alone.
- One image may carry several XMLs (one per graph dir) Ã¢â‚¬â€ the catalog is keyed (camera, day, filename, graph).
- Everything cron-supervised follows the estate idiom (keep-alive every minute, rate-limited, atomic deploys, no in-place scp of running scripts).