# plan-model-library — the model producers' naming law + the library

## Camera bundle contract (2026-11-12, user-set; S072 extension)

New trained-camera artifact and bundle directory name:

```
{GT version}_{model type}_{epoch}_{decode}_{serial}/
  {GT version}_{model type}_{epoch}_{decode}_{serial}.rpk
  labels.txt
  label-filter.properties
  thresholds.json
  metadata.json                 # optional; future model metadata
```

Example: `eb_20_ssd-mbv3_ep29_ssd_001/eb_20_ssd-mbv3_ep29_ssd_001.rpk`.

**The S072 extensions (2026-11-12, user-set):**
- **`{decode}`** mirrors the intrinsics `cpu.post_processing` value exactly —
  the token that selects the node's host-side decode branch (`ssd`,
  `yololite`; stock on-sensor-NMS models use `pp` inside the reserved stock
  form). Two bundles differing only in decode never collide on a name.
- **`{serial}`** — three digits (`001`), guarantees name uniqueness for
  same-weights rebuilds (e.g. re-quantization fixes) without inventing a new
  epoch or model-type token.

This supersedes the 2026-11-12 four-segment form (`eb_20_ssd-mbv3_ep29`) and
the meaningless `network.rpk`. The rpk basename becomes new XML graph folders.
Historical producer ids retain their conversion variant and are not rewritten;
the pre-law bundle row `eb_20_ssd-mbv3_ep29` is retired (`available=0`) with
its payload archived at `model/rpk-archive/eb_20_ssd-mbv3_ep29-20261112/`.
Stock models use the reserved `stock_{model type}_builtin` form (its builtin
epoch token makes the serial/decode segments inapplicable).

Imagebank's `model_bundles` table catalogues identity/availability;
`model_bundle_files` references every payload file with SHA-256 and size. The
bundle directory holds the payloads, so additional model/category files do not
require new catalogue columns. `pi_models_refresh.py` registers complete
bundles from `model/rpk/`; `GET /api/models` returns available manifests and
`GET /api/models/<bundle>/files/<filename>` serves an authenticated, referenced
payload to a camera-role node. The `producers` table remains the historical
XML-producer ledger, not the installable-model catalogue.

On the node, installed bundles live in `/home/pi/picods/models/<bundle>/`.
`service.properties` points GRAPH, labels, filter, and THRESHOLDS_PATH into the
selected bundle. Download every file and verify its manifest hash before
activation; switching then performs the server-side suspend/verify/resume
handshake. The imagebank CA is pinned at `server/imagebank-ca.crt`; model
installation verifies TLS as well as payload hashes.

*Status: DONE (2026-10-30). Design doc; the three decisions below are
SETTLED (user, 2026-10-30). Registry skeleton landed in
[universes.py](../06-deployment/imagebank-dev/universes.py) (the dict shape;
the S054 tuple form superseded).*

## The brief (user, 2026-10-30)

The model named `network` should be named `eb_v20_ssd-mbv3-ep29-imx320` —
**GT version + model type**. The producers table should name the model that
generated a detection, not the actor who clicked publish. The rehearsal
(proved on the local bank: 9,475 pairs through the real stage→publish path)
gave the naming law its first mass-scale proof + this design's decisions.

## The naming law

```
producer id = <GT version>_<model type>[-<variant>]
eb_v20_ssd-mbv3-ep29-imx320    # the ep29 SSD-MobileNetV3 assessor,
                               # 320px IMX conversion, built on eb_v20
```

- **GT version** — the ground truth edition the model was BUILT from
  (`eb_v20`; the next generation trains on it and becomes `eb_v20_…` of a new
  type; a model built on eb_v21 would be `eb_v21_…`).
- **model type** — the family + checkpoint (`ssd-mbv3-ep29`,
  `yololite-ep<N>`; training runs live in `04-training/runs/<family>/`).
- **variant** — the input-size/device conversion (`imx320` = the IMX500 320px;
  `float320` = the float ONNX the Pi-side assessor runs; a batch size or
  frame-hierarchy revision would be another `-variant`).
- **The camera graphs' ids follow the same law going forward**: a picods
  node's live graph produced by the eb_v20-trained ssd is
  `eb_v20_ssd-mbv3-ep29-imx320` as a camera too (kind distinguishes what it
  did: `camera` = the node's own detections, `assessment` = our re-assessment).

## The registry (universes.py `MODELS`, extended 2026-10-30)

One dict per model the estate can RUN or HAS RUN; both workers + server import
it (stdlib-only module). Shape: `version -> dict(kind, labels, onnx, note,
alias_for)`.

- `kind`: `camera` (the node's own graph) | `assessment` (our assessor).
- `labels`: the model's class space (labels ride the model).
- `onnx`: file under `imagebank-dev/model/` (assessment entries; camera
  entries are rpk-only → None → not offered in the /admin/config dropdown).
- `alias_for`: a legacy/foreign id this entry covers — `network` aliases the
  eb_v20 SSD; `ssd-mbv3-ep29-imx320-frame-v2` (the pre-law era id of the same
  weights) aliases the law id. The registry is the alias DISPLAY face; the DB
  keeps historical ids unless the rename migrates them (decided per row).

Live consumers updated to the dict shape: `configured_models()` (the
assignment seam), `admin.assess_assignment` (the dropdown: assessment kind +
onnx-present only), `api.dashboard_json`'s `available` map, and
`assess-v2`'s loader.

## Decision 1 — SETTLED: the library lives in universes.py

(a) extend `MODELS`. Rationale: it is already the single import both the
assessor and the server read; stdlib-only preserved; no new config class. A
future DB mirror (rows for the UI) would be a view OVER this registry, not a
second source. `ground_truth_versions`/`gt_batches` remain the GT-side
catalogue; the library is the MODEL-side one.

## Decision 2 — SETTLED: hard rename `network`

One-time migration, both surfaces:

1. **DB**: `UPDATE xml_docs SET producer='eb_v20_ssd-mbv3-ep29-imx320' WHERE
   producer='network'`; delete the `network` producers row; ensure the target
   row exists (kind='camera' or the dual-kind note; the FK is by name, so the
   camera docs' original identity must stay distinguishable — see the
   camera-docs note below). `workflow_events`/`assessed_pairs` carry no
   producer refs (checked) — no other tables to touch.
2. **On disk**: the bank's `cameras/CAM-XXXX-CLASSIFIED/<day>/network/`
   folders rename to `eb_v20_ssd-mbv3-ep29-imx320/` (the ingest derives the
   graph name from the folder; post-rename the derived name IS the law).
   Order: folders first, then DB, in the same maintenance window (the
   keep-alive-cron's 5-minute ingest is `#IBDEV-FROZEN`-paused; run pi_migrate
   in the deploy window's pause).

**The camera-docs note (the wrinkle to hold)**: `xml_docs` is
`UNIQUE(image_id, origin, producer)`. Old images can carry BOTH a `network`
camera doc AND an `eb_v20_ssd-mbv3-ep29-imx320` assessment doc — renaming
camera→the same name is fine (origin already differs). BUT an image whose
camera doc's producer becomes the SAME `(origin, producer)` as an existing
camera doc would collide on merge. The migrate must merge detections in that
case (or suffix -keep) — synthetic test first.

## Decision 3 — SETTLED: the actor rows die

`producers` rows whose name is a USERNAME (`Alaric`, `Admin`, `claim-copy`):
delete + relabel their `xml_docs` refs:

- `human-*` docs: producer = the ACTOR who wrote them (the xml_docs comment
  says "human = username" — the editor's save_boxes mints these; actor rows
  for real people are correct and stay: relabel `claim-copy`-producer docs to
  the actor who made the copy — workflow_events' claim rows say who).
- The `Alaric`/`Admin` producers rows: delete; their xml_docs rows relabel to
  the actual writer (per-image from workflow_events; fallback = the doc's own
  claim/save event actor).
- `editor.save_boxes` keeps minting actor producer rows for human docs (the
  design is right: a human's xml is authored by the human). The LAW gap:
  schema comment `CHECK (kind IN ('camera','assessment'))` stays — humans ride
  kind='assessment' with a NOTE. Future polish: kind='human' deliberate when
  the queue views want it.

## The full-cutover sequence (the mass run, after the review)

1. The dev-bank's 9,475 sit **staged @ eb_v20** for the user's review (DONE;
   staged via the IMPORT LANE — the 2026-10-31 edge retirement keeps the
   users' diagram claim-gated; the driver stages by
   `transition_worker(..., token=wf.worker_token(), gt_target=…)`).
2. Review verdicts: the batch publish bar takes the approved set (adoption
   path; producer stays the model); rejects go to rejected; junk stands.
3. THEN the local bank is the eb_v20 GT edition, and the pull/dataset build
   (dataset v4) reads it — the puller is a plain file tree walk of
   `export/<version>/<batch>/` + the ground_truth table for the manifest.
4. The LIVE bank picks the same shape at its next mass cutover (the eb_v20
   import batch flow), with the renaming done.

## Test hooks

- The registry shape: assert `MODELS[v]["onnx"]`/`["labels"]`/`["alias_for"]`
  and the /admin/config dropdown contents (assessment + onnx-present only).
- The rename migrate: run on a COPY of the local bank (which holds
  network-era camera docs? — no: the rehearsal mirrored xmls under node names, NOT
  `network/`; the LIVE bank holds the network folders — test the migrate on a
  synthetic bank with a `network/` folder + refs before the live window).
- The actor relabel: same synthetic-first discipline.