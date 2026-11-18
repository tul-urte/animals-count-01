# plan-box-layers — box-set selection for the image page + peek

*Status: BUILT (2026-10-30) on the image page — the ▤ layers popover, the
?show= layer grammar, the sticky lens, the floors retired (the user's purist
call). The peek keeps its own doctrine (top-1, un-floored triage camera);
the queue shells draw the same palette by side name. Deploy with
pi_deploy.py.*

## The brief (user, 2026-10-30)

"when/where detection boxes are drawn (queue popups/image pages) — from what
sets of available boxes; we probably need to offer a selection of all the
available sets of boxes for an image — but there will be some impact of the
workflow state — but what?"

## The box sets that exist (the five origins)

One `xml_docs` row per origin per image (most images carry 1-3); boxes in
`detections`, full-image px, `detections.coord_system='full-image-px'`
(v1-era rows keep `sensor-roi` + the frame mapping; the pour converted what
the transform covers):

| origin | producer | when it exists | colour (the settled scheme) |
|---|---|---|---|
| `camera` | the node's IMX500 graph (the S026 frame hierarchy) | every paired image | violet (`#c77dff`; the "graph" side) |
| `assessment` | the assessor's model output (assess-v2 write_voc_xml) — the GT import stages its pairs via the import lane (the workflow's worker allowlist; no user-visible edge) | every assessed image | green (`#00e676`; the "assessor" side) |
| `human-claimed` | the editor's xml in flight (the `claimed/` scratch tree) | while claimed (dies at release/reject) | blue (`#4da3ff`; the "work" side) |
| `human-staged` | the staged xml (the `staged/` tree) | stage → publish/reject (then becomes `export`) | blue (the "work" side) |
| `export` | the canonical published xml (the export tree) | permanently after publish | blue (the "published" side) |

`pages.overlay_image_svg(img_url, w, h, sources, frame)` renders them: ONE
svg holding the image + rects (aligned by construction, user's S016 design),
`sources = [(objs, side_index)]`, index → colour. The editor also draws a
dashed grey detection-frame outline where the old service watched.

## Where they are drawn TODAY (the live doctrine)

**The image page** — [viewer.overlay_sources](c:/projects/animals-count/06-deployment/imagebank-dev/viewer.py) (ONE implementation), status-driven:

- `published` → the export xml ONLY (default `?show=`), `assessment` +
  `camera` extras behind `?show=camera,assessment`; camera extras keep a
  0.5 score floor, top-1 each.
- `staged`/`claimed` → the human xml in flight (the "primary"; `working_boxes`),
  assessment + camera extras behind `?show=` (0.5 floor, top-1 each).
- everything else (new/assessed/rejected/reserved/junk) → THE TRIAGE VIEW:
  working set (rare: claimed leftovers) + **top-1** assessment det + **top-1**
  camera det. The camera side's 0.5 floor applies on the image page; `new`
  images' low-score graph boxes (0.02-0.05 wood pigeon) are INVISIBLE here.
- `edit_mode` → the working set ONLY (the editor's claim pre-populates it
  with source-tagged boxes from ALL sources — the S018 doctrine).

**The queue hover peek** — [api.peek_json](c:/projects/animals-count/06-deployment/imagebank-dev/api.py) → peek.py's fragment:
mirrors the viewer doctrine 1:1 (the state rule, 2026-10-28): published =
export ≤20 boxes; claimed/staged = work + `?show=` extras; else = BOTH tops
(the camera side UN-floored deliberately — the S048 fix: a low score is the
fact, the caption shows it). No UI toggle; the caption carries
`filename · camera · day · overlap n.nn · status`.

**The server-rendered page** (`queues.py` legacy surface — the `ⓘ`
overlay facility with seed buttons per source + counts, S017/S018): still
live on `/queues/<id>` but the shells have replaced its user-facing role;
any selector work targets the shell surfaces (queue.js + viewer.py) first.

## The gaps the item names

1. **The triage view is top-1-per-side** — a busy frame's 6 assessor + 12
   camera boxes draw as 2. No count control, no score floor control.
2. **`?show=` is URL-only and undocumented** — no control rides the toolbar;
   the peek has no toggle at all.
3. **No "all boxes of a set"** — both surfaces draw limit-1 (peek: ≤20 for
   export only).
4. **The state coupling is implicit in code** — what's "primary" flips with
   status (published→export, staged/claimed→human, else→opinions). A
   selector must carry the state's primary as the default and expose the
   rest as extras.

## What was BUILT (2026-10-30 — the resolution of those gaps)

**The layer unit is a DOC, not an origin.** `xml_docs` is
`UNIQUE(image_id, origin, producer)`: a re-assessed image genuinely carries
one assessment layer PER producer. Viewer code that queried an origin
blurred them together. [layers_of](viewer.py) is the one inventory query
(ordered human > export > assessment > camera), [layer_boxes](viewer.py)
fetches one doc floorless (top-1 only when the caller caps it as a default).
A layer key is `<origin>@<producer>`, e.g. `assessment@ssd-mbv3…-frame-v2`.

**The selector** ([layers_html](viewer.py) + LAYERS_JS): a `▤ layers`
`<details>` popover beside the port selector — one checkbox per doc,
colour chip by origin law, producer name, `(n boxes)` count. A claimed/
staged image's working set shows as a disabled (always-drawn) row. The
checkbox flip rewrites `?show=` from the checked keys and re-renders.
edit_mode has no selector (the working set is the artifact).

**The selection law** ([overlay_sources](viewer.py)):
- no `?show=` → the state's primary: published = its export doc WHOLE;
  staged/claimed = the human doc in flight; else (triage) = top-1 assessor
  + top-1 camera — the assessor side is the LATEST assessment doc, the
  graph the OLDEST camera doc (the queue shells' sides rule).
- `?show=` with the primary PINNED on published/staged/claimed: named
  extras join the truth, drawn WHOLE (drill-down); the primary can never
  be toggled off by a stray click.
- `?show=` on triage (opinions): the named layers REPLACE the default
  tops; bare `show=camera` = every camera doc whole;
  `show=assessment@<producer>` = exactly that doc.

**The sticky lens**: `show` rides FILTER_PARAMS, so every nav arrow, mode
switch, action-back and breadcrumb link carries the selection — walking a
queue keeps your lens set. `_ctx_params` carries it into the arrow URLs;
routes.py's filter plumbing ignores it (inert for the cascade).

**The floors are RETIRED (the user's purist call, 2026-10-30)** — the S048
law ("a low score is the fact") now covers the image page end to end:
`box_top` lost its `score >= 0.5`; the peek's staged/claimed camera extras
lost their floor; the triage default draws a 0.03-score graph box on the
page exactly as the peek already did. Every drawn box's label shows its
score. The palette is ONE vocabulary: pages.OVERLAY_COLORS + viewer's
SIDE_IDX + thin_legend + web/queue.js SIDE_COLOR all agree — human/export
`#4da3ff` blue, assessment `#00e676` green, camera `#c77dff` violet;
editor-red (`#ff4d6d`) stays the editor's live layer only.

**The peek** keeps its own doctrine unchanged (published = export ≤20;
staged/claimed = work + optional extras, now floorless; triage = both
tops un-floored). Its `?show=` grammar predates the selector and still
works; syncing it to the layer keys is parked until asked for.

**Tests**: [debug/test_layers.py](debug/test_layers.py) (22 checks — the
inventory, the purist floor, the pinned primary, multi-producer variance,
per-doc selection, the popover, the sticky param). Everything else green:
test_publish_gt (60), test_peekfix (9), the full synthetic suite.

**Browser QA (2026-10-30, LIVE, screenshots in the chat)** — the user law:
UI changes are QA'd in a browser on the live server before "done". Walked:
assessed CAM-0100 image (floorless 0.02 top drawn + legend chip), the
3-doc CAM-05 image (popover reveals the older ssd-mbv3 assessment doc the
triage glance hid; tick it -> both assess docs drawn whole + the 65-box
graph floorless; nav arrows carry the lens), uncheck flow (graph layer
removed from URL + all links), published image (the truth row is a PINNED
disabled checkbox; ticking graph = blue truth + violet extra drawn). Peek
QA'd via its hover: caption + 1 violet rect. **A live lesson folded into
the skill: the image page pkill pattern `python3 app.py` never matches the
real cmdline (`python3 /home/pi/imagebank-dev/app.py`) — a "restart" that
matches nothing serves STALE code from the old process (the boxless-page
bug was this, not the new code); kill by PID.**

## The workflow-state impact (the user's question, answered)

The state decides **what "the truth" is** on the page, not which sets exist:
- published: the boxes it was PUBLISHED with (the export doc) are primary —
  the model sides are history.
- staged/claimed: the human xml in flight is primary — the model sides are
  reference.
- assessed/new/rejected: NO human truth exists; both model sides are
  opinions (the triage view).

So a general selector's law: **the state's primary set stays
default-selected; every set that EXISTS becomes selectable; the extras
toggle without changing the primary.** The editor (edit_mode) stays
working-set-only (it's the artifact being edited, not a display).

## Decision points for the drill-down (settled 2026-10-30)

a. **Selector UI** → checkboxes riding the image toolbar as the `▤ layers`
   popover; URL-carried (`?show=`), no storage. DONE.
b. **Scope** → the image page only; the peek keeps its own doctrine and
   stays a cheap glance. DONE (its old `?show=` extras still work).
c. **Top-N vs all** → the default glance stays top-1-per-side; a selected
   layer draws ALL its boxes. On published the export truth was ALWAYS
   drawn whole and stays whole. DONE.
d. **Score floor** → RETIRED everywhere on the image page (`box_top`, the
   triage camera top, layer fetches). The peek's triage side was already
   un-floored; its staged extras lost the floor too. DONE (the purist call).
e. **Counts in the caption** → each layer row in the popover shows `(n)`;
   the summary title carries the total. DONE.

## Candidate design (the shape to react to)

An overlay-picker in the image toolbar (the `boxes` mode's popover, or a
`▤ layers` control beside the port selector): one checked item per
origin that exists on the image, the state's primary checked by default,
the URL's `?show=` carrying the selection (shareable; no storage yet).
Counts in the control's title. The peek inherits the same rule via
peek_json (its `?show=` already exists for staged/claimed; extend it to
the triage view + the count limit).

## The loose documentation bits absorbed

- S016: the one-svg facility (`overlay_image_svg`), the colour scheme, the
  "aligned by construction" lesson.
- S017/S018: the seed-buttons-with-counts pattern; the claim pre-populate
  tags boxes `source_graph`; the frame outline.
- S018c field note: the CAM-05 "disagree" was a CAT (both models wrong) —
  the boxes' provenance matters for the triage read, not just their count.
- S048: the peek's floors/facts doctrine; the boxless-caption convention.
- The S057 breadcrumb/caption conventions (overlap in the caption).
- plan-image-bank's containment analysis (overlap = intersection-over-smaller;
  old boxes a median 5.4× area) — why a "similar box" reads ≈1.0 while IoU
  caps at 0.19, the metric the AR gate also uses.