# Queue processing re-imagined — design draft (for discussion)

Draft, 2026-11-13; **phases 1 + 1.5 BUILT, TESTED, DEPLOYED, LIVE (2026-11-13
evening — see §5's ledger + the S075/S075b session notes).** Phase 2 (minor
assessments) remains built-to-order. The goal is **one design** the queues
share, with the queue-specific stuff kept at the seams.

## 0. What we have today (and what's in the way)

- **The queue is a view**; `06-deployment/imagebank-dev/queues.py` renders it
  as a paged (50/page), sortable table. `api.py::queue_data` mirrors the same
  render to JSON for the `web/queue.js` shell. Both are the two heads of one
  body — a queue now has a **server template** (`queues.queue_page`) and a
  **JS client** (`queue.js`).
- **The toolbar** is the auto-toolbar (S058): `/api/queue/<id>` carries the
  queue's *batch actions*, one bar per direct, role-permitted edge on the
  queue's status. `queue.js::setupActionBars` renders them; each bar is a
  filtered-queue batch action with a confirm gate + 500-row `batch_limit`
  cap (a "narrow your filters" warning).
- **The junk queue** already has a physical "🗑 empty the filtered junk queue"
  bar (`#junkbar`, `queue.js::setupJunkBar`, route `POST /junk/empty` in
  `routes.py:734`). It moves **ALL** XMLs of the filtered queue, but the route
  refuses to touch more than `batch_limit` (500) — so it must be narrowed.
- **The assessed queue** has reject-only (per-image in the editor; the
  toolbar's batch reject is filtered-queue-wide, no 500 cap, but no
  per-row selection).
- **The precedent for phase 2** is `similar.auto_reject` (`/action/autorejected`)
  — one workflow transition, a per-image `note`, decided by a matching
  function. That's the plugin shape we generalise.

**What's in the way of selection:**
1. There is **no per-row identity** in the rendered table today — rows have no
   key; the only stable key is `(camera, day, filename)`, which the table
   already emits (camera is plain text, filename is the peek link). So the key
   **exists**, it's just not surfaced as a row key.
2. Selection needs a **row-scoped** action (the toolbar is filtered-scope by
   design). That's a new kind of action.
3. The junk route needs a *subset* variant; the assessed route has none.

## 1. The common design (both queues, then shared later)

### 1.1 Row selection (phase 1)

- Every rendered row gets a **checkbox** in a new leading column. The row's
  `(camera, day, filename)` is carried in the checkbox's `value` (JSON-encoded
  the same `{"camera","day","filename"}` the table already emits as `_img`).
- A **selection bar** appears (above the table, replacing the empty filtermsg
  row when there are selections). It is a small inline toolbar:
  - select-all toggle (ticks/unticks the currently *visible* page's checkboxes)
  - "N selected" counter
  - the queue's **row-scoped action buttons**, enabled only when N > 0
- The row-scoped action button (the important bit): it advertises what it does
  to *selected* rows and calls the same route as the filtered-queue action,
  but with the selected identity set posted instead of a fresh filter.

```
select all  ☐   3 selected
            → reject selected items  (disabled until N > 0)
```

**The fork I need you to decide** — how the selected set is shipped to the
server:

- **(A) Post the identity set.** A POST with `{"ids": [{camera,day,filename},...]}`.
  Server loops the existing per-image transition for each. Simple, matches
  how the editor rejects one at a time. **Limit problem for junk:** 500 rows
  as a form body is 500 small JSON blobs — fine over HTTP, but it's a POST
  whose payload equals the operation.
- **(B) Post a filter that equals the selection.** When the user selects the
  *whole filtered queue*, post the filter (cheap). When they select a *subset*,
  fall back to (A). Best of both, more code.
- **(C) Selection is a server-side temp set.** Selecting is a POST `/queue/<id>/select`
  that stores the id-set in the session keyed by (queue, page-hash); the row
  action posts `?sel=<session-key>` and the route reads the set back. Survives
  across a page reload, survives the whole queue, not just the page. This is the
  richer shape and what makes selection "across pages" cheap — but it needs a
  decision (session table? a temp cookie?).

**Where it lives (the honest answer to "do we store the selection in the DB?"):**
under **(A), no** — nothing selection-related is stored server-side. The
checkboxes are the state (keyed on `(camera, day, filename)` in the page's DOM)
and the set is shipped once, with the action POST. Consequence: navigating to
another page or reloading clears the selection — it is **page-scoped**
(decided 2026-11-13, D3): turning the page or reloading clears it, and there
is deliberately no server-side selection store. If cross-page selection ever
becomes painful in practice, the (C)-lite working set (session-keyed, dropped
when the filters change) is the fallback — not phase 1. What (A) does give:
the route accepts an id-set from day one, so phase 2's minor-assessment
tagging posts the same shape.

### 1.2 The two queues, specifically

- **Junk** → add **"✋ empty selected junk"** to the selection bar. Same route
  path semantics as `POST /junk/empty` (`routes.py:734`), but operating on the
  posted id-set, not the whole filtered queue. The 500-row route limit becomes
  "you selected N ≤ 500". This resolves the junk problem exactly: you no longer
  need to narrow the *filters*, you narrow your *selection*.
- **Assessed** → add **"🚫 reject selected items"** to the selection bar. Same
  route as the toolbar's per-image reject, looped over the posted id-set. The
  500-row question is now a *warning*, not a hard route limit: if N > 500 the
  row action shows a warning ("you selected N, the batch limit is 500") and the
  user can **proceed or cancel** — the existing hard route cap stays as the
  safety ceiling, not the UX. This matches the junk queue: the 500 limit becomes
  "you selected N (≤ 500), proceed or cancel" rather than a hard refusal.
- **The cap is warn-then-proceed everywhere (decided 2026-11-13, D2):** the
  selected-row actions above AND the existing filtered-queue bars — the
  toolbar's batch actions and the whole-queue junk empty (`routes.py:771`) —
  all convert to the same UX: a warning naming the count, then proceed or
  cancel. No route hard-refuses a batch on 500 any more; the ceiling becomes
  the user's explicit proceed. (For the junk empty specifically — a mass file
  move — the confirm gate keeps doing double duty: it names the count *and*
  that files will move.)

### 1.2b Multi-select replaces the batch bars (phase 1.5, decided 2026-11-13) — DONE + LIVE 2026-11-13

**Selection becomes the ONE way to run transitions on many rows.** The
generic route `POST /action/selected` (fields: `queue`, `to`, `ids`, optional
`gt_target_version`) validates the edge is direct + role-permitted from the
queue's status, then loops `wf.transition` per posted id — every queue's
edges for free. The per-queue coverage (from the live-matrix audit):

| queue | selection-bar buttons | notes |
|---|---|---|
| assessed | claim-selected / reject-selected / reserve-selected | reject-selected retires into the generic route |
| claimed | unclaim-selected / stage-selected | stage shows the GT-version dropdown (needs.gt_target_version) |
| junk | assess-selected / reject-selected / ✋ empty-selected | the empty stays whole-queue-only? NO — empty-selected exists; empty also stays as the whole-queue bar (the physical-move LAW) |
| new | discard-selected | (new→assess is system-only) |
| rejected | unreject-selected / discard-selected | |
| reserved | unreserve-selected | |
| staged | unstage-selected | |
| published | no selection column (no outgoing edges) | |

**Decided (2026-11-13, phase-1.5 review):**

- **PUBLISH stays the whole-queue bar (option A)** — the one seam. Per-image
  publish was retired 2026-10-28 (the GT-batch law: version + batch name,
  GT rows written, XMLs to the batch layer); a selection-batch of it adds
  nothing the whole-filtered-queue bar doesn't already do. Staged therefore
  gets `unstage-selected` multi-select, and the publish bar remains as-is
  (with the D2 warn gate).
- **The whole-filtered-queue TRANSITION BARS are removed** on every queue —
  claim/reserve/reject/assess/discard/unclaim/unstage/unreserve/unreject all
  become selection-bar actions (`{via} selected`). Acting on many rows = tick
  (or select-page), run, flip page, repeat. The two whole-queue bars that
  survive: **junk empty** (physical file move, not a transition) and
  **publish** (option A).
- The D2 warn-then-proceed gate applies to the new selection actions exactly
  as it does to the surviving bars.
- The old `/action/batch/<to>` route + `addBatchResult` machinery stays (it
  serves the two surviving whole-queue bars: publish via
  /gt/publish-batch, and any future filtered-queue ask); `setupActionBars`
  renders only the surviving bars.

### 1.3 Paging (the shared ask, both queues, all queues)

- **Sort:** correction (this replaces my previous wording, which was wrong in
  two ways): the picods/legacy filenames *begin* with the zero-padded capture
  time (`17-43-13_785-picods_02.jpg`), so **ordering by `filename` IS ordering
  by capture time** — time is not a separate key (and `time` stays a
  display-only column, computed from the filename at render). What is actually
  off today: `day DESC, camera, filename` (a) sorts each day **oldest-first
  within itself** (00:00 → 23:59 inside a newest-first day) and (b) reaches
  time only *after* camera. Neither reads as "descending by datetime". The fix
  is one of:
  - **(i)** keep the camera blocks, flip the within-day direction:
    `day DESC, camera, filename DESC`;
  - **(ii)** pure reverse-chronological, camera dropped as a sort key:
    `day DESC, filename DESC` — matches "temporal order completely, date and
    time".
  → decided (2026-11-13, D1): **(ii) pure reverse-chronological — `day DESC,
  filename DESC`** as the default for every image-shaped queue; camera stops
  being a sort key. Mechanical fallout: `assessed_pairs` carries an index on
  the old order (`idx_ap_order`), so the change comes with a small migration.
  The rest of this section stands: every queue is already DESC-by-default via
  its `order` string (claimed/staged/reserved/published/rejected sort on
  claimed_at/updated_at DESC).
- **Page-number control:** the current strip is *only* relative jumps (`<-`,
  `->`, `<y-`, `<m-`, `<d-` and their forward twins) + a `(n–n of N)` count.
  Add a **number input** (or a small "go to page ▸" box). Submitting it sets
  `?page=<n>`. Keep the relative strip; this is additive, not a replacement.
  For a queue with few pages the relative strip is fine; the number box wins for
  a queue with hundreds.

## 2. Minor assessments (phase 2)

### 2.1 The plugin model

A minor assessment = **a function that reads a set of queue rows and writes an
attribute onto them, without a workflow transition**. The attribute is stored
as a parallel row keyed on the same identity, not on the workflow row — so a
re-assessment, a reclaim, etc. can't clear it, and it's queryable by queue via
the view. The schema seam is one new table:

```
CREATE TABLE assessment_tags (   -- phase 2 addenda; not a workflow row
  tag_id      TEXT PRIMARY KEY,  -- plugin name, e.g. 'light_level'
  ...
);
```

Each assessment is a **plugin registered on the server** (extend the existing
action/transition registry, reuse the same "register a function" pattern that
`workflow.py::actions_for` already uses). A plugin declares:

- `id` + human name
- **input**: what it needs (a parameter to set — e.g. a light-level dropdown;
  or nothing — "auto" assessments like blur/darkness)
- **`evaluate(con, identity_set)`**: returns, per row, the value to tag
- **scopes**: which queue statuses it may run on (assessed; junk; claimed; …)
- **tag columns it writes**: one or two columns on a small tag table

Reused precedent = `similar.auto_reject` (one transition + a decision function
+ a per-image note). We're swapping "transition" for "tag write" and
"one seed" for "a set".

The three you named map to three plugins:
- **light-level** — needs a parameter (dark/mid/bright/overcast); the run-modal's
  dropdown sets ONE value for the whole selection (the tag write is mechanical)
- **focus** — auto: the blur gate is `matchers.py::match_blur` (NOT similar.py —
  similar.py carries the dHash/box SIMILARITY gates only; the edge-variance blur
  test + its `blur.threshold` knob (default 500, admin /admin/config range)
  live in matchers.py, wired via autorepeat's launder pipeline). The plugin
  reuses the edge-variance measure extracted from that matcher
- **by-example** — auto/near-duplicate: reuse the same-fingerprint walk
  `similar.py` already does

**THE BAND DECISION (2026-11-14, settled by the user + Copilot by form;
supersedes parts of the plugin-shape list above; the DDL sketch above is a
sketch, the real addendum DDL keys on the triple).** The auto-stampers
write BAND WORDS, and the table gains a score column so the measure is
never lost:

- `assessment_tags` gains `score REAL` (pi_migrate probe-then-add). Storage
  law: **value = the band word, score = the measure.** A band-edge change
  re-bands via SQL CASE over scores - re-run only when re-MEASURING. And
  every row debugs itself ("why is this soft?" - the score sits just under
  the edge, in the row).
- **`light_level` becomes AUTO** (`input: None`): each row measured (mean
  0..255 L - `match_darkness`'s measure) and banded: pitch-dark <16 /
  dark <50 / dim <100 / lit <170 / bright. The human-choice dropdown
  RETIRES (it was the phase-2a stopgap). The LIGHT MEASURE lands in
  the score column too - the same store as focus's edge variance:
  EVERY banded tag keeps its number (the band-decision storage law
  generalised to both measures: one shared REAL column, not per-tag
  columns). Honest caveat, kept from the
  discussion: mean-L says what the SENSOR saw - an IR-flash night frame
  reads bright; if IR-separated truth is ever needed it is a SECOND
  measure (a flash flag), not a re-litigated luminance ladder.
- **`focus` bands on the existing edge-variance measure** (the
  `blur.threshold` 500 law survives as the first edge): blurry <=500 /
  soft <=1200 / fair <=1800 / sharp <=2800 / very-sharp above. Ladder
  provisional - to be tuned after the first estate-wide stamping (today's
  n=50 dev window spans only 1685..2446).
- Band edges live in properties knobs (like blur.threshold); re-banding
  rewrites value from score against the new knobs (a small re-band pass),
  re-running re-measures. Two different operations, never conflated.
- **THE ORDER LAW (the user caught it, 2026-11-14): within one tag,
  ascending score = same-or-higher band — the ladder is monotone by
  construction (battery-pinned across a measure sweep). ORDER BY value is
  ALPHABETICAL (idx_tags_tag) — never an order slice; ORDER BY score is
  the rank. The words' order lives in ladder.py DEFAULT_LADDERS (code,
  not config — assessor outputs cannot drift), and the band dropdowns will
  render the ladder's order, not the alphabet.
- **`sim:<key>`** (band+score): a
  per-set ladder (default twin/near/like/faint/far) with score = the
  distance to the set. **BUILT (S082, 2026-11-14 — dev-bank, browser-QA'd,
  NOT deployed):** the ladder is ONE for the whole sim:* family
  (ladder.DEFAULT_LADDERS["sim"], the tags.sim.bands knob 0.05/0.15/0.35/0.6
 — the set's key rides the TAG NAME sim:<key>, not a per-set ladder;
  reband('sim') re-words the whole family on a knob flip). The DISTANCE is
  the DETECTION PATTERN primary (the user's settled call: our cameras are
  static — a pixel hash calls every dusk-lit office frame 'twin', but the
  GLITCH is a repeating detection pattern): similar.py's box-match shapes
  graded into 0..1 (examples.py::box_distance — identical 0, drift small,
  unmatched/count-volatile/empty-vs-content 1.0), with the dHash judging
  only DETECTION-FREE pairs (the S059 empties seam). SET DISTANCE = MIN
  over the seeds. The STORE: example_sets + set_members (schema-addenda,
  FK on the triple — the tags die with the image). THE GESTURE: tick a
  small sample → ✚ by example (green lozenge) → name it in-modal →
  POST /action/byexample-create stores the seeds then SWEEPS the whole
  filtered queue (streamed, the tagmap's whole-queue law). Each stored set
  renders a `sim <title>` lozenge (whole-queue re-run via /action/
  assess-selected whole_queue=1, no ticks needed) + a similar-to filter
  dropdown (value = <key>:<band>, or <key> = any band) riding the stored
  filters as tag_sim. Battery: debug/test_byexample.py 50/0 (A distance
  primitives, B sim ladder + fold, C the store, D the sweep, E the
  dynamic sim:* plugin via plugin_of, F the create route + refusals,
  G the tag_sim fragment, H the reband-over-the-family).
  **THE S082b/c/d ADDENDA (same day, the user's play findings, all
  committed d9f38bd + 5279ce1 + f070bb0):** the TWO SELECTION MODES
  (page | whole queue radio on the selection bar; whole mode posts
  whole_queue=1 to /action/selected and the SERVER resolves the slice
  from the user's STORED FILTER ROW — the tag params ride transitions,
  drop for the sim sweeps; battery I/J extended to 63/0 incl. the
  tags-survive-the-move law) — "sim-tag a filtered view and shift the
  lot by tag" is now: filter → whole → discard. THE RETIRE SEAM (POST
  /action/byexample-retire + the red ✕ pill inside each lozenge): 3
  DELETEs + the cache refresh, the rows never move. THE SIM LOZENGES
  FOLLOW THE MODE (page = the ticked sample; whole = the whole filtered
  queue; the confirm names it). ✚ by example precedes its children; the
  filter columns carry their set's title (unlabelled selects are
  indistinguishable). THE ONE OPEN GAP (pinned, S082d): after a
  streamed create the bar/tags sometimes stay STALE until a manual
  reload (the modal-close's render likely raced the final write) —
  reproduce, pin, fix next session.
- **Auto-banders RIDE THE ASSESS-V2 PASS** - every completing pair is
  stamped with light+focus bands with no selection needed; the
  selection-run route stays as the re-measure tool. By-example stays
  selection-run only (S082 honoured: it NEVER rides assess-v2 — a set's
  scores make sense only over the queue you curated it on).
- **By-example seeds live in the DB** (`example_sets`; the set's key
  doubles as the `sim:<key>` tag namespace) - chosen over a properties
  JSON blob.
- **Filters become ONE EQUALITY DROPDOWN per band tag** (options from
  each tag's band ladder); the light-choices dropdown and the focus
  yes/no widget RETIRE - the uniform band widget replaces them.
**CLASSES vs TAGS (the user's observation, 2026-11-14):** same assertion
shape (word + score + writer), two stores on purpose. detections are
per-object (boxes), document-mirrored (xml_docs + VOC files), replaced as
a whole document on re-assess/release, open vocabulary, score = model
confidence 0..1; assessment_tags are per-image, upsert-independently,
survive every transition, closed ordered ladders, score = a physical
measure in tag-specific units. Merging would break cardinality, the
document lifecycle, and the units. The harvestable convergence: a
read-only ASSERTIONS view (UNION ALL of both stores' word+score+writer)
when the tag-map/anomaly work wants every fact on one image; and the
projection runs det->tag only (an image-level model detection could
mirror down as a pred:<label> tag family later - never the reverse; tags
never carry geometry).

### 2.2 Where the "run assessment" lives

The selection bar (1.1) already has enabled buttons when N > 0. In phase 2 the
selected-set buttons **grow to include the available minor assessments** (only
those whose `scopes` include the queue's status). Running one on N rows posts
the id-set + the chosen plugin id + its params; the plugin writes tags on the
tag table. No workflow row moves.

**The run-modal (user thought, 2026-11-13):** while a minor assessment is
running, a **modal dialog pops up over the queue page** (the main page is
disabled underneath — no scrolling, no queue re-render, no nav-strip jump). The
dialog shows **regular feedback** as the run progresses — at minimum a
"running / done / failed" state per row (or per batch) and a "N of M done"
counter. Once the run finishes, the dialog **closes itself** and the queue
re-renders with the new tags (or errors, with the failure noted in the dialog —
never silently).

**The feedback transport (2026-11-13, "can't we use a websocket?"):** the
verified shape of the server is a stdlib `http.server` handler on a threaded
TCP server (app.py) — no push channel exists anywhere today. A websocket would
mean hand-building handshake + frame codec + origin checks + a pinned thread
per connection, for a need that is strictly **one-shot, one-way** (one run
command, then events toward the modal, then done). So the run uses a
**streaming POST response** instead: the modal posts the run and the server
writes progress lines down that same response as it works (NDJSON — one JSON
object per line: `{"n_done":12,"n_total":340}`, …, final
`{"done":true,"summary":…}`); the client reads them via fetch +
ReadableStream, updates the modal per line, and the **end of the stream is the
self-close signal**. No polling, no new protocol, ~20 lines of server code.
Failures mid-stream break the pipe — the modal reports the failure. A
websocket earns its cost only for genuinely bidirectional / live-multi-user
needs (two editors watching the same queue), which this estate does not have;
if one ever arises, a GET-only EventSource (SSE) is still lighter than WS.

Two consequences for the seam:

- The run is **not fire-and-forget**: the client holds the request open for the
  plugin's whole `evaluate()`, so the modal's lifetime = the response's
  lifetime. The page being blocked is the point — that's the modal doing its
  job. Tag *writes* are table rows (seconds to a few minutes for the largest
  plausible selection), so a held-open response is proportionate; a plugin
  that would hold it much longer is a plugin that needs a different shape,
  not streaming with the page locked.
- The close/refresh handoff is **one route shape**: the streaming response
  ends with a final line carrying the per-row results and the new tag state,
  so the client re-renders from the response rather than refetching the whole
  queue.

### 2.3 The filter panel

- Add the assessments as **filter controls** under the existing filter panel,
  scoped to the current queue (so junk doesn't offer assessed-only assessments).
- Each assessment gets a filter widget matching its input: a dropdown for
  light-level, toggles for the booleans (in-focus, by-example), and a slider or
  "blur ≥ X" field for the numeric ones.
- The query joins the tag table (a row must exist with the matching value) —
  the same per-filter fragment code (`filter_fragments` in `api.py`) that
  handles camera/classes/version/by. Filtering "by focus" then reads the
  `assessment_tags` rows directly, no SQL change per plugin.

### 2.5 The tag-space map (user idea, 2026-11-14)

A 2D (later 3D) view over a queue in TAG space. It composes on the band
decision, it reuses the existing filters, and it reopens none of them:

- **The band axes make the space discrete**: light = 5 ordered bands,
  focus = 5 ordered bands => the 2D tag space is a 5x5 grid of cells.
  Continuous scores stay available for a later scatter view (they ride
  the score column); the primary map is the band grid.
- **Aggregate endpoint**: GET /api/queue/<id>/tagmap (the queue's
  existing filters apply) -> {axes: (tag_id + ordered band ladder),
  cells: [(light, focus, n)]}. Computed over the WHOLE filtered queue
  with the same filter fragments; rows are never fetched - a cell click
  NAVIGATES (sets the two tag-filter params on the queue's filter row;
  the URL carries them, like every filter today). Images missing a tag
  sit in an explicit untagged bucket per axis - the map shows coverage.
- **Coordinates are data; the client owns placement**: the server emits
  band INDICES (+ the ladders); no projection/camera logic server-side,
  ever. Only when >2-3 tags need squeezing would a transform appear -
  stdlib small-d PCA (power iteration, ~50 lines), never t-SNE/UMAP
  (platform law); a learned embedding, if ever wanted, would be a
  cached, versioned coords table - decided then, not now.
- **KIT (the user's existing hand-rolled JS+SVG)**:
  C:\projects\index-expressions\js\svg-viewport.js - the seam:
  createSvgViewport(state, {onDiagramRender}) with caller-owned
  {rotX, rotY, zoom}, wheel-zoom, drag, createZSortedBatch (painter's
  sort - the 3D fly-through reuses exactly this), AbortController
  dispose. It is COPIED into imagebank-dev/web/ with attribution
  (deploys are self-contained units - no cross-repo imports). 2D uses
  zoom only; drag pans (rotate would fight cell clicks); the 3D ride
  keeps rot.
- **Render: choropleth cells** (heat fill + count), dots only when a
  cell is small (<=~10); hover could ride the peek seam later.
- **Lives as a table<->map mode toggle on the queue shell** - another
  head on the one body (section 0).
- **BUILT (2026-11-14, bands-4 + map):** the §2.5 map lives - GET
  /api/queue/<id>/tagmap (aggregate cells = [light_idx, focus_idx, n] +
  ladders + untagged; refusals honest) + the queue shell's map mode (the
  vendored svg-viewport kit at C:\projects\index-expressions\js - zoom +
  the click-to-filter cell). Evidence: test_tagmap 15/0 + the full suite +
  the browser QA pass (cell click landed tag_light=lit&tag_focus=very-sharp
  in the URL + the panel's dropdowns; wheel zoom scale(1->1.1); mode
  toggles). The 3D fly-through keeps this feed + the kit's rot/z-batch.
- **THE CELL-CLICK GESTURE (user call, same evening): a cell click OPENS
  the filtered queue** - filter + flip to the TABLE (a one-cell map of
  itself reads as "nothing happened"; the rows are what a click asks
  for). Found + fixed live: the slice CTE carried a VIEW ALIAS, but the
  tag-join fragments qualify the BARE VIEW NAME (the S080 scoping law) -
  an alias hid it ("no such column: v_queue_junk.camera"), so the junk
  map 500'd (empty response). Law: the tagmap's slice CTE is UNALIASED;
  the endpoint crash-guards into a 500 JSON (never a killed connection).
  test_tagmap 19/0 with the junk repro.
- **THE WHOLE-QUEUE LAW + THE DB-SIDE GROUPING (user QA, 2026-11-14):**
  the lattice slices by the queue's OTHER filters only - it IGNORES the
  tag_light/tag_focus params (they drive the TABLE the cell links to; a
  map that sliced by them could only redraw its one clicked cell - the
  live probe showed exactly that). And the user's call "push the grouping
  to the DB query" stands: the cells ride ONE joint GROUP BY (join light x
  focus over the slice, grouped in SQL); the untagged buckets ride TWO
  per-axis GROUP BYs (a light-ONLY row needs the light axis's own
  aggregate - the joint inner join cannot see it, battery-pinned).
  test_tagmap 21/0.
- **Build order (the user set, 2026-11-14): bands first, then the map,
  then by-example.** The map eats band tags; sim:* sets add axes later;
  the user explicitly wants to FLY THROUGH the data in the 3D evolution
  (the kit's rotX/rotY + z-batch are the seam for it).
## 3. The seam where "common" ends

The design above is the common body. The per-queue differences are all at the
seams — which is exactly the shape you asked for:

- **Parallel XML (assessment detections)**: the assessed queue already reads
  `assessed_pairs`; minor assessments add a *parallel* tag table that the view
  can `LEFT JOIN`. Junk's `misses`/`classes` columns stay as-is.
- **Actions plugged in**: junk gets `empty-selected`; assessed gets
  `reject-selected`; both get the minor-assessment buttons. Other queues add
  their own seam.
- **The identity**: `(camera, day, filename)` is the single key the tables
  already emit; the selection layer just exposes it.

## 4. Build order (suggested; you steer)

1. **Selection bar + identity key** — phase 1, both queues. Adds the leading
   checkbox column, the selection bar, and (A) the id-set POST. This is the
   whole "can I select rows and act on them" ask.
2. **Junk: empty-selected** and **Assessed: reject-selected** — the two
   routes, already using the id-set from (1). This *is* phase 1 done.
3. **Page-number control** — additive to the strip (any queue, done once).
4. **Minor-assessment plugin registry + `assessment_tags` schema** — phase 2a.
5. **light-level / focus / by-example plugins** — phase 2b.
6. **Filter-panel assessment controls** — phase 2c.

## 5. Decisions and open items

**Decided (2026-11-13 review):**

1. **Selection transport: (A)** — post the id-set (`ids` form field, JSON
   array); nothing selection-related is stored server-side; the checkboxes
   are the state.
2. **The 500 batch cap is a warning with proceed/cancel, everywhere** —
   warn-then-proceed on every batch surface; no route hard-refuses on 500
   any more (backstops remain on the two selected-row routes).
3. **Default sort: pure reverse-chronological** — `day DESC, filename DESC`
   for every image-shaped queue (camera drops out of the sort); migration of
   `idx_ap_order` rode the live deploy.
4. **Run-modal for minor assessments** with regular progress feedback, over a
   **streaming POST (NDJSON)** response; end-of-stream is the self-close.
5. **Selection is page-scoped** — checkboxes are the only state; navigating,
   QUEUE-HOPPING (in-place lozenge links) or reloading clears it.
6. **Sequencing: phase 1 first** — done.
7. **Multi-select replaces the transition batch bars (phase 1.5)** on every
   queue via the ONE generic route `/action/selected`; publish stays
   whole-queue (option A); the whole-queue junk-empty bar retired (user,
   close-out #2) — eviction is "empty selected junk" only.

**Shipped 2026-11-13 (the evidence trail):** phase 1 (S075: selection bar,
junk/assessed selected routes, warn gate, page box, D1 sort) + phase 1.5
(S075b: generic route, generated row_actions, lozenge-fix, selection-clear
on queue hop, junk-bar retirement, the label-collision fix) — 3 live deploys
(backups 20261113T{192034,194127,194924,195433}Z), live browser-QA'd per the
user law, test batteries new (test_selection 22/22, test_generic_select
20/20) + updated (test_junk 32/32, test_batch_actions 8/8,
test_autoreject 47/47) + full suite ALL GREEN.

**Open (phase-2 plugin specifics):**

- light-level choices; focus gate thresholds; by-example seeds/parameters.
- Whether any plugin ever needs the session-backed selection set (D3 revisit).
- ~~S076 (first item next session)~~: DONE (S076, the dev-bank standard).

**NEXT SESSION (the user-set, S080b close, 2026-11-14) — two items:**

1. ~~Reconsider the tags data~~ **SETTLED same day by form - the band
   decision above: auto banders ride the assess pass; value = band word +
   score = measure; the uniform band dropdowns; the tag-map design is
   2.5 below.**

2. **The plugins on MORE queues**: today both scope to assessed + junk
   (the settled #6). The queue statuses without plugins: new, claimed,
   staged, reserved, rejected, published (published has no selection
   column at all — the option-A seam). The natural asks: rejected (a
   light_level audit before unreject?), new (a focus pre-screen before
   assess?). Whatever widens: the scopes list in assess_plugins.py IS
   the whole change (the route + the advertisement + the panel all read
   for_status), so this is a data edit + a battery update — the UI
   pieces come free.

**Interim landed 2026-11-14 (S078/S079, all dev-bank-verified then deployed
live in the S079 close):** S078's reserve compounds (staged/claimed/rejected/
junk → reserved via declared walks, `queues.py::COMPOUNDS` + the one generic
route), the bare-verb lozenges + the SEL_WALKS tooltips; S079's four: the
imagebank version marker (`web/site-version.js`, v<date>.<serial> on all four
shells), the classes UNION (every `v_queue_*` view binds `db.classes_expr` —
labels from every xml origin; published keeps `class_label`; the
`assessed_pairs` cache carries `classes` too), the classes SEARCH (the filter
panel's text box + plain-DOM suggestion list — the datalist law holds),
and the heading-checkbox as the HOVER-PREVIEW switch (localStorage "qpeek",
default OFF; select-page stays bar-only). The filters' classes option set
needs no cache UNION any more (the views carry every origin). These do not
change phase 2's shape: the plugin registry, `assessment_tags`, the NDJSON
run-modal and the tag-filter panel are unchanged; the only new seam notes
are (a) `filter_fragments`/`nav_strip_filtered` now resolve the classes bind
by the view's actual columns (per-column awareness phase 2c's tag joins
should reuse), and (b) `assessed_pairs` gained a column — cache redeploys
now ride `pi_migrate`'s add_col pattern, see the S079 close in the notes.

**Defaults taken unless overruled:** progress per 25 rows in the modal;
page-number control = a plain number input (Enter or Go submits it); after a
run closes, the table data is refetched and previously selected rows are
re-ticked (so one selection can chain several assessments); `time` stays a
display-only column.

## 6. Implementation plan (where it's built, tested, documented, and live)

All code changes land in the repo tree `06-deployment/imagebank-dev/` — the
same tree that deploys to the Pi; there is no separate "staging branch" (the
synthetic tests + the deploy's md5 gate are the staging). The phase-1 steps
map to §4's build order 1–3.

### 6.1 Where the code changes are made

- **Selection bar + identity key (step 1)** — three files, one feature:
  - `web/queue.js` (with `web/queue.html` for the bar/checkbox host markup):
    the checkbox column render, the select-all toggle, the "N selected"
    counter, the warn-500-proceed confirm gate, and the id-set POST body
    builder. Phase 1's only new client code — the queues are static shells +
    JSON API, so this is where row-scoped UI lives.
  - `api.py::queue_data`: emit the row key it already builds (`_img` =
    `{camera, day, filename}`) into the JSON rows' checkbox payload, and the
    row-scoped actions a queue offers (junk → `empty-selected`; assessed →
    `reject-selected`).
  - `queues.py`: the QUEUES specs gain the row-action declarations next to
    the existing batch bars.
- **The two selected-row routes (step 2)** — `routes.py`:
  - `POST /junk/empty-selected` — the subset variant of `POST /junk/empty`
    (routes.py:734): resolve each posted `(camera, day, filename)` to its
    image_id via the existing `image_by_triple` statement, move exactly those;
    same event log, same note shape.
  - `POST /action/reject-selected` — loop the existing per-image reject for
    each posted id, one event each.
  - Both routes accept `{"ids":[...]}` from day one (the (A) transport), and
    both re-check the row cap server-side as the backstop.
- **Warn-then-proceed everywhere (D2)** — `web/queue.js` converts the
  client-facing gate for the *existing* batch bars + whole-queue junk empty
  (the count is already in the API response, so the warning is client-side,
  pre-POST). **The route-side BATCH_ACTION_LIMIT refusals (routes.py:531, 678,
  771) are REMOVED per the D2 decision ("everywhere = warn-then-proceed
  instead of refusing"): the confirm gate is the safety mechanism — it names
  the count, the file-move (for junk) and the over-guidance warning, and
  nothing proceeds without an explicit OK.** The 500 constant survives as the
  *warning threshold* (`BATCH_ACTION_LIMIT` stays defined). The tests that
  pinned the 413 refusals are rewritten to pin the new contract: a >500 batch
  is ALLOWED server-side (202), the confirm is the only gate.
- **Page-number control (step 3)** — `api.py::nav_strip` gains one more piece
  of strip data (`page_no`, `pages_n`) and `web/queue.js` renders the number
  box from it; `queues.py`'s retired HTML renderer is NOT touched (retired —
  see README).
- **The sort (D1)** — `queues.py`'s order strings for the image-shaped queues
  become `day DESC, filename DESC`; the `assessed_pairs` index
  (`idx_ap_order`, schema-v2.sql:497) is recreated in the new order via
  `schema-addenda.sql` (drop + recreate, idempotent) so `pi_migrate.py` picks
  it up on live.
- **Phase 2 later:** the `assessment_tags` table + plugin registry
  (`workflow.py`'s registry seam), the streaming NDJSON run route, and the
  modal in `web/queue.js` — nothing in this plan blocks on them.

### 6.2 Where the changes are tested

- **Synthetic tests (run locally on Windows, temp DBs, stdlib-only)** — new
  `debug/test_selection.py` for the id-set transport (POST shape, unknown
  triples rejected, cap backstop) and the two selected-row routes; extend
  `debug/test_batch_actions.py` and `debug/test_junk.py` for D2's client-gate
  contract (the API response fields the JS warns from) and for the *changed*
  live expectation in `test_junk.py` (its routes.py:210 assertion pins the 413
  refusal — kept, as the backstop, but the test must still pass with the
  client-gate change being JS-only); extend `debug/test_route.py` (or the
  filters test) for the strip's new page-data piece and for the new default
  `ORDER BY day DESC, filename DESC` (assert the rendered row order on a
  synthetic cross-camera, same-day set). Run: the local suite —
  `debug/test_*.py` excluding the two live-host tests
  (`test_filterpost`, `test_housekeep_check`) per the doctrine.
- **BROWSER QA IS MANDATORY for the UI half** (user law 2026-10-30): every
  queue-surface change (checkboxes, selection bar, warn gate, number box, and
  in phase 2 the modal) is QA'd in a real browser against the LIVE server
  (`https://imagebank/`), exercising the control on a real queue, confirming
  the visual result, screenshot as the record. The integrated browser in this
  workspace is the tool; a 200-with-right-strings probe does not substitute.
  QA happens *after* the deploy below and before the step is reported done —
  and, per the restart trap live-proven 2026-10-30, only after confirming the
  serving process start-time is post-deploy (`ps aux | grep '[a]pp.py'`).
- **Unit/step evidence convention**: each step ends with the test run's real
  output shown, per the working agreement (no narrated dones).

### 6.3 Where the changes are documented

- **`06-deployment/imagebank-dev/README.md`** — the layout/API section gains
  the selection bar, the selected-row routes (now the ONE generic route),
  the warn-then-proceed gate, the strip's page box, and the LOCAL DEV SERVER
  runbook (the DB path trap, the bank_root source, the QA login, S076's
  rebuild procedure). This is the doc-of-record for the v2 surface shape.
- **The domain skill (`.github/skills/imagebank/SKILL.md`)** — carries the
  doctrine for the next sessions: the dev-bank section (run/rehearse, DB
  path trap, fixture gaps), the queue-selection doctrine (the generic
  route, D1 order, the in-place navigation law), and the pi_deploy flag
  correction. Skill over session note when the rule is standing, not
  event-shaped.
- **`documents/session_notes.md`** (the index) + **`notes/2026-11-13 S075.md`**, **`notes/2026-11-13 S075b.md`**
  recording what was built, the decisions (D1–D3 + the two close-out asks),
  the deploys + QA evidence, and the traps worth the next session's
  attention. The draft doc (this file) stays the *design* record; the
  session note is the *what actually happened* record.
- **In-code comments** — only where the shape needs a why (the id-set's
  page-scoped lifetime, the generic-route gate, the via-vs-special label
  rule), following the estate's comment doctrine.
- **No doc change**: `workflow-transitions.md` — phase 1/1.5 introduce no
  new edge (the selection actions reuse the existing transitions;
  empty-selected is a physical move, not a status).

**PHASE 2 is the next session (the user-set, S079 close, 2026-11-14)**:
the minor-assessment build starts at §4's step 4 (the plugin registry +
`assessment_tags` schema), rehearsed on the S076-standard dev-bank
(rebuild-devbank.py + the slice zip + the qa login), QA'd per §6.2's
browser law, deployed by §6.4's line (pi_deploy → pi_migrate → restart by
PID + sudo → behaviour probe → live browser QA). S079's interim additions
that phase 2 inherits: the union classes column (any tag-filter join binds
alongside it in `filter_fragments`, which is now per-view-column aware),
and the S079 deploy's exec-bit/chmod law for any file push.

**Phase 2: BUILT + DEV-QA'd (S080, 2026-11-14)** — §2 landed as decided:
the six open items settled by form (the registry shape, the separate
streaming route, the triple-keyed tags, the fixed 4 light choices, the
NUMERIC focus measure, the scope assessed+junk); 2a built (the schema +
assess_plugins.py + the matchers.blur_measure extraction — THIS FILE's §2.1
"blur gate in similar.py" was WRONG, similar.py carries only the
similarity gates), 2b built (POST /action/assess-selected, the streaming
NDJSON + the run-modal, the in-modal input select after the integrated
browser's prompt() refusal), 2c built (the tag-join filters + the panel
widgets + the strip params). Evidence: the three new batteries (33+26+17)
+ the full suite 33/33 + the dev-bank QA suite 35/35 + the browser QA (the
screenshots). NOT deployed — the same gate as S078's batch (the user's dev
QA, then the §6.4 line; the version marker bumps v2026-11-14.002). The
full session record: notes/2026-11-14 S080.md.
### 6.4 How the code goes live

- **One command: `python 06-deployment/imagebank-dev/pi_deploy.py`** — scp to
  a staging dir on the Pi, dos2unix, backup the live files, atomic `mv`,
  md5-verify both sides with CR stripped, purge `__pycache__`. NOTE (correction
  from the live run, 2026-11-13): this v2 deployer's flags are ONLY
  `--check` / `--migration-prereqs` — there is NO `--files`/`--dry-run`
  (that was the v1 `deploy.py`'s option set); deploys are all-file, and
  pre-existing live drift (e.g. universes.py) is pushed along with the
  change. It also does NOT restart the server — the restart is manual (by
  PID — the proven pkill-pattern trap — then cron's `run-v2-server.sh`
  revives it; verify the new pid + start time post-deploy). `--check` first
  (diff local vs live).
- **Schema rides the deploy, not the scp**: the D1 index change goes through
  `pi_migrate.py` (applies `schema-addenda.sql` idempotently + rebuilds the
  pairs cache) *after* `pi_deploy.py`, then the pairs-cache rebuild is
  verified (row count unchanged). Phase 1 touches no other table.
- **Order on the wire**: deploy → migrate → verify the new server
  (`curl https://127.0.0.1:443/login`, the process start-time check) → the
  browser QA pass (6.2) on the live queues → only then report the step done.
- **Rollback**: the deploy's `cp -p` backups are the rollback path (v1 stays
  frozen-cold but is not the rollback for v2 UI work); a failed step rolls
  back by restoring the backed-up files and re-killing the server. The sort
  index migrates down cleanly (recreate the old index name/order).
- **Phase 2's live step** is the same line — no separate release process; the
  `assessment_tags` table arrives via `schema-addenda.sql` + `pi_migrate.py`
  like every addendum before it.
