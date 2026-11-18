# The QA triage runbook Ã¢â‚¬â€ a fresh dev-bank rebuild Ã¢â€ â€™ the full-lifecycle replay (S091)

The repeatable use-case this captures (the user's 2026-11-15 ask): a fresh
`rebuild-devbank.py` run, then **one comprehensive, repeatable battery** that
drives the ingested slice through the whole system Ã¢â‚¬â€ triage cohorts, editor
flow, publish Ã¢â‚¬â€ and proves the guards refuse what they should. Run it ad
nauseam; every failure names a contract that regressed.

## The three instruments (run in this order)

| # | Instrument | Drives | Dependency |
|---|---|---|---|
| 1 | `qa-bank/tests/qa-devbank.py --server` | the transition routes (edges + refusals) | none (stdlib) |
| 2 | `debug/qa_triage_lifecycle.py` | the S091 triage policy + the editor flow through the routes | none (stdlib) |
| 3 | `debug/qa_browser_faces.js` | the real UI faces (hydration, scopes, confirm dialog, toast, image ports) | `playwright-core` + the system Chrome/Edge |

The stdlib pair proves BEHAVIOUR (statuses, events, refusals, DB truth);
the browser pair proves FACES (what a human actually clicks and sees).
Neither substitutes the other Ã¢â‚¬â€ the S091 session's lesson: both layers
caught bugs the other couldn't.

## The loop

```
cd 06-deployment/imagebank

python qa-bank/rebuild-devbank.py --qa-password qa      # fresh estate (ingest + assess)
python app.py --port 8443                       # window 2: the dev server (Ctl-C when done)

python qa-bank/tests/qa-devbank.py --server --qa-password qa      # battery 1 (26-32 checks)
python debug/qa_triage_lifecycle.py                       # battery 2 (21-33 checks)
node debug/qa_browser_faces.js --shot __shots             # battery 3 (13-15 checks)

# re-run battery 2+3: the skips assert the guards instead of the cohort
# actions Ã¢â‚¬â€ a GREEN rerun is the rerun-honesty proof, not a false pass.
```

The rebuild leaves `assessed ~340 / junk ~1165 / 0 new` (the assess-era
rebuild: the model's opinions are stamped, the no-detection pile junked).
If onnxruntime is missing the rebuild says so honestly and the pairs stay
new (battery 1 then exercises the newÃ¢â€ â€™junk paths again).

## Battery 2 Ã¢â‚¬â€ the recorded triage policy (COHORTS in the file)

The S091 pass, replayed exactly. The cohort rule is (camera, tag_focus,
tag_light Ã¢â€ â€™ action), applied **whole-filtered-queue** (the same
`whole_queue=1 POST /action/selected` the UI's filtered-queue scope posts):

| Cohort | Filter | Action | The S091 evidence (the record, not a judgment) |
|---|---|---|---|
| cam02-dark | CAM-02 + light=dark | reject | 2023-02-01 dusk + 02-02 dawn: 0.61/0.63 "starling" on shadowed bare ground, no bird |
| cam05-lit | CAM-05 + light=lit | reject | the pond-rim FP family: 4 visual samples (0.36-0.88) all bare rim + box-position centroid (1148,700) Ã‚Â±2ÃÆ’ Ã¢â€°Ë†300px over 150 frames |
| cam02-lit | CAM-02 + light=lit | claim | blue tit (0.96/0.82 agree), textbook robin (0.95/0.79) Ã¢â‚¬â€ the pattern the user's claimed sparrow run matches |
| cam0100-blur-dark | CAM-0100 + focus=blurry + light=dark | reject | 07-03-56: dark, 0.10 box on stones |
| cam0100-rest | CAM-0100 | claim | the garden set: robin triplets, dog/person dusk sets, lit pigeons |

LEAVES (never acted on): CAM-0101 (CocoTest Ã¢â‚¬â€ GT home undecided) and
CAM-0102 (the in-tree model finds birds there where the live pipeline
junks them Ã¢â‚¬â€ the user's call to make). The battery asserts both survive.

E's skip law: a cohort the fresh build didn't produce (or an earlier run
consumed) is a SKIP + the 409-refusal assert Ã¢â‚¬â€ never a false failure. F's
editor flow takes ONE claim through releaseÃ¢â€ â€™re-claimÃ¢â€ â€™stage (version gate,
bad version 400)Ã¢â€ â€™unstageÃ¢â€ â€™re-stageÃ¢â€ â€™**batch publish** (`POST
/action/batch/published`, the JSON twin of the form's `/gt/publish-batch`)
and asserts the DB row + the per-row payload. G asserts the post-policy
no-ops: an empty resolved slice 409s; an action-word (not a status) 403s.

## Battery 3 Ã¢â‚¬â€ the browser faces + the traps it re-proved

- **The empty-slice face**: on a 0-row filtered queue the toolbar never
  hydrates (`hasCol` needs rows) Ã¢â‚¬â€ no scope radios, no action bar, NO
  filter form. The driver falls through to the junk queue for the
  filter-face assertions. A fresh-rebuild + battery-2-first run WILL hit
  this on `/q/assessed` (222-ish rows Ã¢â€ â€™ fine) Ã¢â‚¬â€ the fallthrough exists so
  a post-policy estate still yields a green, honest run.
- **The peek-switch trap**: the selection column's FIRST checkbox is the
  header's hover-preview toggle, NOT a row tick Ã¢â‚¬â€ a driver (or a hurried
  human) ticking "the first checkbox" flips the preview and acts on the
  wrong row. The probe binds to ONE row: its own `<tr>`'s checkbox.
- **The counter face**: `#sel-count` reads "N selected" (selection scope)
  or the filtered total (whole scope) Ã¢â‚¬â€ and NOTHING in selection scope
  with no ticks. Assert the scope you switched to, not the one you left.
- The confirm dialog names the scope + the count ("Apply reject to the 1
  selected image(s)?" / "Apply reject (WHOLE filtered queue) to the 86
  image(s)?") Ã¢â‚¬â€ the count is the last-line screen before a bulk act.
- The dialog probe is a **one-row reversible junkÃ¢â€ â€™rejectedÃ¢â€ â€™junk walk** Ã¢â‚¬â€
  never a leaf queue, never whole-queue scope (that would fire the
  recorded policy against the leaves).

Run `--headed` to watch, `--shot <dir>` to keep the six evidence shots
(login, queue, filtered, after-action, image-fill, image-boxes).

## The exogenous visual pass (NOT automated, by design)

The recorded policy encodes the S091 *decisions*; the visual JUDGMENT that
produced them is a human/agent step the harness can only document. When a
FRESH slice changes (a new make-slice cut, a model swap) the cohorts'
ground truth may shift Ã¢â‚¬â€ before trusting battery 2's green on a NEW slice:

1. Open one `cam05-lit` row in fill port: is the best-detection box on the
   pond rim (the S091 FP family's signature), or on an actual bird?
2. Open one `cam02-lit` row in boxes port: does the green (assessor) +
   purple (node) pair hold a bird?
3. Spot-check the lowest-scoring member of each claim cohort (the FP tail
   lives at the bottom: the S091 samples at 0.40-0.88 were all rim).

The mechanical proxy (best-det box clustering Ã¢â‚¬â€ `best_det_stats()` in the
battery) PRINTS the centroid + spread per reject cohort; a centroid that
jumps between rebuilds on the same camera+filter is the smell that the
cohort's ground truth moved and steps 1-3 are mandatory.

## What "green" means

- battery 1: every legal edge walked + every refusal fired (the fresh
  estate routes the refusals through assessed when new is drained).
- battery 2 first run: the cohorts drained, the leaves intact, claims +
  rejects created, one publish landed.
- battery 2 re-run: the guards (409/403) instead of the actions.
- battery 3: the faces hydrated, the dialog named the scope, the probe
  walked back, the image page rendered both ports.

Any red = a contract regression, name it in the session notes with the
battery line.
