# S105 addendum — the junk-compound, the retrain lane, the map's full
# inventory, the three-site ident (2026-11-16, the small hours)

## THE JUNK-COMPOUND (the user's Option A — the REAL walk)

- The DIRECT `assessed→gclaim` edge is RETIRED. schema-v2.sql + the
  addenda re-cut carry the fresh build; `pi_migrate.py` DELETEs the edge
  + its role row on a live DB (the 2026-10-31 assessed→staged
  retirement's shape). The g-family edge COUNT is now 6 (was 7).
- `queues.COMPOUNDS["gclaim"]["assessed"] = ["rejected","junk"]` — THREE
  REAL leg transitions per row: reject → discard → g-claim. The claim
  copy seeds on the junk leg (workflow.transition's gclaim arm reads
  fr in ("assessed","junk") — unchanged). NO new schema rows: the legs
  are existing edges.
- ROW_ACTION_EDGES drops the direct gclaim on assessed; the COMPOUNDS
  generator renders the button; the client's SEL_WALKS maps
  `assessed|gclaim` → "assessed → rejected → junk → gclaim".
- THE ONE SEAM THAT NEEDED A CUT: the glitch:<cam> walk's top-band
  lane (assess-selected's walk_target). Its gate now tests
  may_perform OR the COMPOUND legs (`_compound_ok`); the walk lands
  every leg per row (a mid-walk refusal leaves the row at its last
  good step; the outcome reports the row's final truth, "to" = where
  it LANDED, not the intent).
- THE BATTERIES RE-CUT HONESTLY (never silenced): test_glitch_flow's
  "assessed->gclaim legal" became NOT-legal + the exact route assert
  ([("reject","rejected"),("discard","junk"),("g-claim","gclaim")]) +
  the event-trail [route assessed>rejected>junk>gclaim] assert;
  test_generic_select's actions list drops gclaim (it is a row-action
  compound now, not a batch action); test_glitch_migrate's edge count
  7→6 + the raw-CHECK probe rides junk→gclaim; test_gjunk's walk
  asserts the compound door.

## THE MAP'S FULL INVENTORY

- `/api/queue/glitched/family-map` reads each root's LANDED rows
  DIRECTLY (workflow ⋈ anti_truth ⋈ g_universes, WHERE status=
  'glitched' AND u.name IN (root's verses)) — NO page slice, no day/
  camera filter ride. A map that inventories a root is not the
  table's echo. The live catch that named it: the chip promised 78
  landed, the map drew 50 + TITLED it "50 landed".
- test_family_map.py pins it (the fixture lands 55 rows in one root —
  MORE than a page of 50; the paged caller still reads the full mass).

## THE RETRAIN LANE (the S104e finding's fix)

- `gfam.corpus_size_of(con, verse)` — the usable-corpus size WITHOUT
  training (the same corpus_of filters, a count).
- The payload's verses carry `n_ct` + `stale` (n_ct > n = the corpus
  grew past the train-time envelope).
- The gclaim bar's ⚗ train renders for `!ready || stale`; the modal's
  option names the top-up ("starling-1017 (8 landed) — re-train: the
  corpus grew 8 → 10"); `gfam.train` re-fits honestly from the fresh
  corpus (the envelope is NOT pinned; the S104e "envelope lag" was
  inaccessibility, not a law).

## THE THREE-SITE IDENT (the user's law)

- The header's ident names the estate: imagebank-live (bank in BLUE,
  the as-now accent var(--acc)) / imagebank-dev (RED #c24a1e) /
  imagebank-qa (GREEN #1a7f37). ONLY the word 'bank' changes colour —
  the suffix (-live/-dev/-qa) rides the plain ink.
- ONE knob: `site.brand` in properties.DEFAULTS = "live" (an
  UNCONFIGURED estate IS the live artifact). dev-bank's machine-local
  override carries site.brand=dev; the qa-bank's staged override
  carries site.brand=qa (make-qa-bank.py writes it).
- ONE server impl: pages.brand_html() (BRANDS map + the em's inline
  colour). The STATIC shells (queue/dash/queues/cameras.html) carry NO
  hard-coded suffix any more: app.py serves site-version.js
  DYNAMICALLY, injecting the knob's word into the SITE_BRAND
  constant; the hook paints the em + appends the suffix span.
- queue.js's document.title rides SITE_BRAND (guard: typeof check —
  site-version.js loads AFTER queue.js in queue.html).
- VERIFIED in-browser: dev red (rgb(194,74,30) on the em), qa green on
  the :8451 login, live blue by default (the brand_probe drove all
  three words + the bogus-word fallback live→blue).

## THE WORKBENCH RULE (the user's ruling, 2026-11-16)

- qp: **development happens on DEV-BANK** (the tools live there + new
  tools THERE); qa-bank is the release-rehearsal stand-up
  (make-qa-bank.py's stage/destroy lifecycle), NOT a workbench.
- THE SUITES RUN FROM BOTH HOMES: every battery's path head resolves
  db.py from the WSL staged shape (debug/ beside the code) OR the repo
  checkout (qa-bank/tests + the imagebank sibling via ART); the honest
  S105 sweep: 50/50 green from the REPO paths; the qa stage 348 files
  md5-verified + the full WSL suite green.

## THE NEXT SESSION (the user's words)

- QA PLAYWRIGHT SUITES in qa-bank: the browser-face instruments. The
  one prototype rides in qa-bank/tests/qa_browser_faces.js (S091):
  playwright-core + the system's Chrome (no browser download),
  --headed / --shot dir, IBV2_BASE env so ONE file drives dev :8443
  AND qa :8451. THE CATCH (live-proven): from the WINDOWS browser the
  :8451 qa server answers on the WSL IP (172.30.x.x), NOT
  127.0.0.1/localhost — IBV2_BASE rides the IP.
- The natural first faces: the fammap (dot-count-vs-chip promise, the
  tooltips, dot-click→fam=), the de-glitch lozenge's scope law, the
  compound's confirm+flash+counts, the retrain button's
  appear/train/hide cycle, the ident face (red/green/blue per estate).
