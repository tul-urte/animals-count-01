## S076 STARTING BRIEF (written 2026-11-13, the S075/S075b close) - the dev-bank rebuild + the standard procedure

**The user's brief for the next session**: (1) rebuild dev-bank if it is not
an accurate model of imagebank; (2) a STANDARD procedure for that (make one
if not exists). 

**Verified state to start from**:
- The dev server reads the ROOT database: `06-deployment/imagebank-dev/harvest-v2-dev.db`
  (db.py DEV_DB default; a dev-bank/-nested file is a decoy). The bank TREE
  (peeks/raw/move target) = the `properties` `bank_root` (default dev-bank/).
- The current dev DB works but its POUR is stale vs live schema: the
  detections rows lack the camera-XML provenance columns
  (`frame_sensor/capture_frame/sensor_mode` exist in the LIVE detections
  schema, verified read-only 2026-11-13) - the claim edge's
  save_boxes->frame_of crashes `no such column: d.capture_frame` in dev only.
  Live carries the column.
- The dev pour also has NO camera-origin XMLs (imagebank-dev/dev-bank
  assessment/ XMLs only): the empty-selected move skips them (by design the
  route tolerates absent XML) but the fixture cannot exercise the xml-move
  leg faithfully.
- The v1 pour source: 06-deployment/imagebank/db/harvest.db (307MB, found on
  disk 2026-11-13). The db.py `pour()` + sql/migration.json do the
  transformation; `IBV2_HARVEST_DIR` needs frame_geometry on the path.
- Publishing must NOT be exercised on dev until gt_batches.directory rows
  are repointed inside dev-bank (they name 01-ground-truth dirs).
- The dev-bank rebuild procedure does NOT exist as a script yet - that is
  the deliverable (rebuild-devbank.py: stop server, fresh DB via pour +
  pi_migrate's addenda, schema caught up to live, fixture rows via LEGAL
  edge walks only (the trigger refuses direct UPDATEs), the qa user,
  properties sanity-check, server up + a smoke route probe). The skill now
  carries the doctrine; the README carries the runbook skeleton.

**Also parked from S075b**: the dev-only claim-409 becomes the rebuild's
first acceptance test (claim-selected must succeed on the rebuilt bank).
Phase 2 (minor assessments + the NDJSON run-modal) follows after S076 per
the design draft (sections 2.x + build order 4-6).

---

