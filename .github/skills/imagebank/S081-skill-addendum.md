# S081 addendum — the bands law + the tag map (draft for the next skill refresh)

*Not yet merged into SKILL.md — the next session's skill review decides the
wording; kept alongside the skill while it's fresh (2026-11-14). The band
work's doctrine below supersedes phase-2a's paragraphs where they conflict.*

## THE BAND LAW (assessment_tags since S081)

- **Storage: value = the BAND word, score = the measure.** One shared
  `score REAL` column carries EVERY banded tag's measure (focus edge-variance,
  light mean-L; a future sim distance rides it too). Band edges live in
  properties knobs (`tags.focus.bands` / `tags.light.bands`),
  /admin/config-validated (positive ascending; a flip REBANDS from the stored
  scores immediately — count reported). Re-run re-measures; a flip re-words;
  never conflate them.
- **ladder.py is the ONE implementation**: DEFAULT_LADDERS (words in rank
  order, code — not config), edges_of fresh-reads the knob, band_of maps
  score->word (boundary joins the UPPER band: band i = score < edges[i];
  more names than edges folds the surplus into the final band). The ORDER
  LAW: within a tag ascending score = same-or-higher band (battery-pinned);
  ORDER BY value is ALPHABETICAL (idx_tags_tag) — order slices go by score;
  dropdowns render the ladder, not the alphabet.
- **Banding happens in write_tags(banded=True) ONLY.** The plugins return
  the MEASURES (matchers.blur_measure / matchers.light_measure — one gate,
  one truth; light_measure is the match_darkness extraction). light_level
  is AUTO (the phase-2a human-choice dropdown retired: input None, no
  choice gate). Ladders: pitch-dark<16/dark<50/dim<100/lit<170/bright;
  blurry<=500/soft<=1200/fair<=1800/sharp<=2800/very-sharp — provisional
  until tuned on the live estate (the dev window spanned lit..dark only).
- **THE IR CAVEAT (live-proven):** mean-L reads the SENSOR's light — the
  IR-flash night mass bands lit (1018 rows), never dark. "Dark shots of the
  site" needs a SECOND measure (a flash flag), not a re-litigated ladder.
- **THE BANDS-RIDE:** workflow.transition's tail + assess-v2's flip stamp
  every completing pair (light+focus) via assess_plugins.stamp_tags —
  skip-if-measured (score IS NOT NULL) so batch transitions never re-decode;
  a scoreless legacy word row re-measures on its first transition;
  force=True re-measures. Failures are per-key and best-effort (a stamping
  failure never fails the transition).
- **The filters:** tag_light/tag_focus accept band words (EXACT equality —
  a stale word after a reband filters to nothing, never a wrong slice);
  the focus yes/no THRESHOLD verdicts remain as server-accepted queries
  (score preferred, legacy value-as-measure via COALESCE). The panel
  widgets = uniform band dropdowns fed from options.tags (the ladders).

## THE TAG MAP (§2.5, live)

- **Feed:** GET /api/queue/<qid>/tagmap — aggregate cells
  [light_idx, focus_idx, n] + ladders + untagged, never rows; refusals
  honest (404 unknown queue, 400 queue without the band plugins).
- **THE WHOLE-QUEUE LAW:** the lattice IGNORES the tag params (they drive
  the TABLE a cell links to); the OTHER filters shape it. A map that
  sliced by its own link-filters would only redraw one cell.
- **THE GROUPING IS IN THE QUERY:** one joint GROUP BY (light x focus over
  the slice) for the cells + TWO per-axis GROUP BYs for the untagged
  buckets (a one-sided row is invisible to the joint inner join).
- **The slice CTE is UNALIASED:** the tag-join fragments qualify the BARE
  VIEW NAME (the S080 law) — an alias hides it and every non-cache view's
  map 500s ("no such column: v_queue_junk.camera", found live on junk).
  The endpoint crash-guards into a 500 JSON (never a killed connection).
- **Front-end:** svg-viewport.js VENDORED into web/ from
  C:/projects/index-expressions/js (attribution banner; deploys are
  self-contained — no cross-repo imports). The kit's render callback must
  be wired AFTER the group it paints exists (a TDZ throw kills zoom
  silently — found in QA). Map mode = a queue-shell toggle; coordinates
  are INDICES (the server owns no projection — the 3D fly-through rides
  the same feed + the kit's rot/z-batch).
- **The cell-click gesture:** filter + FLIP TO THE TABLE (a one-cell map
  of itself reads as "nothing happened").
- **Chunk the runs:** the assess-selected 413 backstop holds >500 ids;
  a whole-queue run chunks client-side (3x450 covered junk's 1162).
  SUPERSEDED S082: whole_queue=1 (the server resolves the slice) is THE
  whole-run affordance now — the sim sweeps + the whole-queue mode
  transitions ride it; the client never chunks.

## THE BY-EXAMPLE LAW (S082, the sim:* family)

- **The distance is the DETECTION PATTERN** (similar.py's box-match shapes
  graded 0..1 in examples.py::box_distance — identical 0.0, score-drift
  a PRICE, unmatched/count-volatile/empty-vs-content 1.0 — the S059 volatile
  rule carried); the dHash judges only DETECTION-FREE pairs. A static
  camera's pixel hash would call every dusk-lit office frame 'twin', but
  the GLITCH is a repeating detection pattern. SET DISTANCE = min over
  the seeds (twin to any seed = twin). **S083 DRIFT LAW (2026-11-15,
  live-proven):** the score drift is priced, never gated — the S082
  0.10-tolerance gate (similar.cfg_of's autoreject score_tolerance)
  compressed the whole ladder to {<=0.1, ~0.5, 1.0}: 'like' UNREACHABLE
  on 421 rows and visually-identical chairs banded far at drift 0.14
  while drift-0.33 chairs banded twin via a darker seed. box_distance
  now matches on geometry+label only (a drift-blind cfg COPY —
  auto-reject's gate is untouched) and prices the matched drifts:
  (Σ|drift| over matched + unmatched) / n_seed. Same-estate re-run:
  twin 108 / near 118 / like 36 / faint 25 / far 744 — the 0.10–0.20
  mass exists and 'like' fills.
- **The sim ladder is ONE for the whole family** (ladder.DEFAULT_LADDERS
  ["sim"]: twin/near/like/faint/far; the tags.sim.bands knob, edges
  0.05/0.15/0.35/0.6 — provisional). The SET KEY rides the TAG NAME
  (sim:<key>); reband('sim') re-words every sim:* tag from its score. An
  EXACT-0 distance (a seed itself) is 'twin' under ANY positive edges.
- **The store:** example_sets + set_members (schema-addenda; FK on the
  triple — the tags die with the image, they SURVIVE every transition).
  create_set is an UPSERT (re-creating a key REPLACES membership); the
  key law: [a-z0-9_-]{1,32}.
- **The gestures (queue shell):** tick a sample → ✚ by example (the
  create ask = the name + THE GLYPH PALETTE ROW — the author's meters
  badge, S089b) → the streamed create-sweep of the whole filtered queue;
  per stored set: a `sim <title>` lozenge whose scope FOLLOWS THE SCOPE
  RADIO (selection = the ticked sample; page = the page's rows implicit;
  filtered queue = whole_queue=1; the confirm names the scope) + an
  integrated retire ✕ (a stop-propagating pill INSIDE the lozenge; the
  rows never move) + the GLYPH-EDIT ✎ beside it (S090c: the badge
  repaint — one UPDATE on example_sets, the tags/membership/rows never
  move, no cache refresh; the palette opens with the current badge
  pre-highlighted; ✦ = the default, empty = the reset). ✚ by example
  STAYS SELECTION-scope (the seeds ARE the ticks) and precedes its
  children.
- **The filter:** tag_sim = '<key>:<band>' (equality) or a bare '<key>'
  (= any band, EXISTS); rides _effective_filters/the stored row/the strip
  links (labelled per-set columns in the panel — unlabelled dropdowns are
  indistinguishable: the S082 finding).
- **The routes:** POST /action/byexample-create (seeds + glyph, then the
  sweep; streaming NDJSON), /action/assess-selected whole_queue=1 (the
  whole-run affordance), POST /action/byexample-retire (3 DELETEs + the
  cache refresh; NO events — the rows never move), POST
  /action/byexample-glyph (S090c: ONE UPDATE — the badge repaint; an
  off-palette glyph degrades via _glyph_ok).
- **THE GLYPH (S089b/090c, the author badge):** example_sets.glyph
  (schema-addenda + pi_migrate add_col); the server list examples.GLYPHS
  is the truth (a 20-glyph monochrome palette; the client's picker
  mirrors it — keep in step); NULL = the default ✦; a glyph-less
  re-create KEEPS the badge (the upsert's membership replace, not the
  badge); sets_of carries it to the meters.
- **THE MULTIMETER (S089/090b/090c, the tags-cell face):** every banded
  pair = ONE vertical bar of STACKED segments (one 2px segment per ladder
  word, floor = the FIRST word, column-reverse; the rank lights the first
  N upward; the greys above stay visible — the ladder's shape + the
  position both read); the bars stand ADJACENT as ONE bank (the strips
  are two-row columns: meter on top, glyph centred UNDER its tower — the
  glyph row must not space the bars apart); sim bars tint the set hue +
  the author's badge; the WORDS ride the tooltip ("rank N/M of the
  ladder"); unknown/stale words render the plain pair (honest). The SIM
  PAIR SPLIT is the LAST colon (sim:<key>:<word> — the first colon made
  'sim' the tag). td.tagscell nowrap keeps the bank on one line (the
  SELECTOR was the bug: `td .tagscell` never matched the td — the
  computed style caught it).
- **THE SCOPES (S090, the 2-mode law RETIRED):** selection | page |
  filtered queue — three radios on the selection bar (localStorage
  qselmode; the OLD value 'page' MIGRATES to 'selection'). SELECTION =
  the ticked rows (the checkbox column + the peek-switch header + the
  ⚗ measures + the ✚ create live ONLY here; "N selected"); PAGE = this
  page's rows IMPLICITLY (LAST_ROWS captured per render, no ticks, the
  checkbox column hides, "N on this page"; selIds() resolves LAST_ROWS);
  FILTERED QUEUE = whole_queue=1 (the unchanged server contract; "N in
  the filtered queue"). THE ENABLE RULE: transitions + the sim re-runs
  resolve per scope (page: ≥1 row; whole: always); the measures + the
  create need SELECTION (+ticks; the seeds) — a bulk scope is not a
  sampler. The select-page checkbox is GONE (the scope IS the
  select-all); the confirm names the scope.
- **THE BAND SORT + the rank (S089, the order-fundamental law):**
  assessment_tags.band INTEGER = the ladder word's RANK (dual-with the
  words; write_tags dual-writes; reband re-ranks on a knob flip;
  backfill score-first, in-ladder-word otherwise, neither = NULL honest;
  idx_tags_band). sort=band:<tag> on every queue (the bare sim key
  upgrades to sim:<key>): ORDER BY COALESCE(band, 999) — the untagged
  sort LAST both directions. THE ALIAS LAW, third live proof: the ORDER
  BY subquery binds the WRAPPER's alias q, not the bare view name (the
  first cut 500'd "no such column"). The tags HEADER cycles tags →
  band:focus → band:light_level. ladder.py: _banded_names is the ONE
  name-shape (band_of + band_index_of + rank_of cannot desync under the
  fold); The token law consolidated: db.substitute_tokens (bootstrap +
  pi_migrate + the tests one implementation).
- **The modes (S082b — SUPERSEDED by S090's scopes):** the old page |
  whole radio and its battery semantics retired; the stored-row slice +
  the tags-survive-the-move laws HOLD (the whole-queue scope is their
  heir).

## The workflow seam notes (S081)

- pi_migrate's backfill runs INSIDE the schema step (add_col + the legacy
  value-as-measure rows -> score + banded words; scoreless word rows stay
  scoreless honestly). A dev-DB migration IS the same three steps on a
  COPY first (the prebands backup pattern) + cache rebuild after.
- The batteries: test_ladder (the ladder law + monotone sweep + the S089
  rank law), test_bands_ride (the transition ride + skip-if-measured +
  force), test_tagmap (the endpoint laws), test_byexample (the sim laws
  + the glyph cases + the byexample-glyph route, 83/83). All temp-DB, no
  server; the dev-server console now tails to
  dev-server.log/dev-server.err.log (Start-Process -RedirectStandard*).
- **THE TOKEN LAW (S080/S089):** every raw reader of schema-v2.sql /
  schema-addenda.sql substitutes via db.substitute_tokens ({CLASSES} +
  {TAGS}); an un-substituted '{' in a CREATE is an "unrecognized token"
  error (live-proven by 34 tests at once when the addenda gained a token
  while bootstrap executed it raw). The schema SQL files are TOKEN files
  now — NEVER bulk-string-replace them in a shell one-liner (the S090
  incident: a replace target that had already been reverted turned BOTH
  files 0 bytes; restore = git checkout + re-apply, verified by loading
  through db._read_schema before building on).
- **The S089/S090 era's one open seam:** the tagmap still converts GROUPED
  WORDS to indices in Python (api.py ~811) — the band column makes a
  straight rank read possible; NOT OWED, a cleanup for a future session.
