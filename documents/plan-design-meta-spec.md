# plan-design-meta-spec.md — THE DESIGN MODEL'S META-SPEC (the keys + the prose vocabulary)

**Status**: staged (S113, 2026-11-18). The review rode the S112 session's
aftermath (the consistency re-cut); the rulings below are the user's,
verbatim where quoted.

**The umbrella ruling**: enforcement is **BATTERY ONLY** — the laws check
in the suites (test_page_xml / test_h_interceptor / the consistency
family), never at parse. The keeper's refusals stay what they are (the
compound's own load laws: key= first, the grammar's name law, duplicate
keys, the reference laws). A meta-law that is "nice to have" lives as a
check, not a refusal. *(S113, the user's ruling: "battery-enforcement
only.")*

## Part A — THE KEY LAW (the canonical names)

The review's inventory: **115 distinct keys** (pages.xml + types/*.xml),
audited against the estate's established vocabularies (the 11 workflow
statuses, the product's data-test-id stamps, the DOM ids, the payload
fields). The findings and the laws they become:

1. **The grammar law**: a key is lowerCamelCase, `[a-z][a-zA-Z0-9]*`
   (title=-camelisation already enforces it at parse — the machinery's
   own law; no new check needed).
2. **The noun law**: name the CONCEPT the face carries, not the widget
   or the position. The type names the widget (`h:button`); the key
   names its meaning. Casualties in the current tree: `uni` (the
   universe chip — a camera's GT-universe membership mark), the trap
   keys `t1/t2/t3`, the cell index `n`.
3. **The verb law**: action faces carry verb-keys (`apply, submit,
   clear, reset, analyse, seed, train, tick, allocate, cancel, close,
   go`); state/label faces carry noun-keys. A toggle that switches a
   face is `xxxToggle` (the product's own stamps already speak it:
   `mapToggle`, `tableToggle`).
4. **The scope law**: a name is free within its path, but the
   globally-semantic words are RESERVED once named elsewhere: the 11
   workflow statuses, the payload's canonical fields (`total`, `rows`,
   `columns`, `filters`), the reserved `$` faces. The audit's seed
   list: `assess.new` (a status word as a count label),
   `assess.state` (shadows the reserved `state=` attribute).
5. **The self-reference law**: a child never repeats its parent's
   singular where both name the same concept (the audit's instances:
   `selection.selection`, `familyMaps.root.root`, `bands.band.tick`
   — the last rides because `band` there names the PARAM, allowed).
6. **The rename law**: keys are canonical. A rename is a RECORDED
   decision (this file's Part A ledger, below) + the suites' call
   sites re-pointed in the same change (the S111 precedent:
   fammapToggle's model-home re-point).

**Enforcement (battery)**: a new check in test_page_xml (B-block) walks
the emitted model and asserts the grammar + the reserved names. The
self-reference check walks depth-2 name pairs. RENAMES remain manual
(each one's suites re-pointed; the battery pins the NEW names).

**The staged renames** (each: master + suites + the battery in ONE
change; none started yet):
- `assess.new` → `assess.newCount`; `assess.state` → `assess.runState`
  (the reserved-word clean-up).
- `uni` → `universeChip`.
- `t1/t2/t3` → the traps' own names (`confirmNamesTheWholeStory`,
  `batchAddsWarning`, `scopeEmptyFlash`) — or traps ride positional
  keys by LAW (decide at touch time; the trap slot is a value-leaf
  list today and no consumer names them beyond position).
- `selection.selection` → `scope.selection` radio stays `selection`
  (it names the SCOPE's value; the GROUP carries the name) — the
  honest fix is the GROUP's rename (`selection.scopes` is already the
  group's name; the collision was `queue.selection` (container) vs
  `selmode=selection` (a value) — ALLOWED by law 5 as-is). Re-read at
  touch time.

## Part B — THE PROSE VOCABULARY (the d: faces)

**The duty split** (the desc/readme constitution, now explicit):
- `$desc` — the face's PURPOSE + its key rationale ("why this name").
- `$readme` — the model's constitution (root-only today; unchanged).
- `$state` — the visibility/existence condition (SITE; the asserting
  consumer is the S112 consistency mirror's successor, a battery that
  DERIVES its gates from `$state`).
- `$anatomy` — **NEW (leg 1)**. The DOM's shape facts that today
  muddy desc: what is a sibling not a child, what renders dynamically,
  what carries the stamp. The S091a class. Consumers: the sitemap +
  the dialog render it as its own face; the suites' SKIP maps cite it.
- `$rules` — **NEW (leg 3, prose-only)**. The group's constraints
  (scope/enable/visibility combinators) — prose lines, one per rule.
  Consumer: the consistency battery derives the both-ways gate from
  the lines (replacing `consistency_states`' hardcoded mirrors).
  NOT a mini-language (the S110h over-build trap; the structure
  arrives when a consumer needs it).
- `<d:algorithm url="...">` — **HELD** (not built). It needs an
  attribute on a prose face (a carve-out, a new kind) and has only
  ~3 honest instances today (gfam's fit, the picods box mapping, the
  ladder banding). Opens with the SECOND demand or the first suite
  that wants the URL as data.

**The d:-face admission law** (the meta-spec's own): a new face exists
when (a) a NAMED consumer renders or asserts it, (b) its content muddies
desc/readme by living there, (c) it is prose-first; structure arrives
only when a consumer needs it. Declined on this ruling: d:provenance
(the session notes are the record of record), d:why (desc+readme
cover it; vocabulary sprawl).

## Part C — THE ENFORCEMENT SHAPE

- **The battery is the teeth**: test_page_xml grows the key-law checks
  (Part A) + the prose-face checks (Part B); the consistency family
  grows the derived-gates check (Part B's $rules consumer).
- **The parse stays lawless beyond its own**: no new keeper refusals.

## Part D — THE DB OUTSIDE THE CODE DIR (the deploy-safety note, S113)

**The user's note**: move the REAL imagebank's db out of the code dir —
all the banks (live `/home/pi/imagebank-dev/harvest-v2.db`, the qa
estate's `/home/qa/imagebank/harvest-v2.db`, the dev bank's sibling) —
so a deploy/stage wipe can never ride a data path. *(It was live-proven
the hard way twice this session: the S112 'no such table: users' and
the S113 `--skip-estate` wipe — both the same defect.)*

- The shape: follow the bank dir's precedent (its own location,
  `properties`-keyed, db.py's search order names it, the deploy
  manifest carries code only).
- The work: properties.py DEFAULTS (`data_root` or a `db_path` key) +
  db.py's connect/bootstrap paths + make-qa-bank.py's paths (line 74's
  WSL_DB) + pi_deploy's FILES (confirm the db never stages) + the
  rebuild scripts' explicit paths + ONE live migration (move the db,
  repoint, verify) — the LIVE gate applies (the user's verbatim
  command).
- Until it lands, make-qa-bank's stage_code rides the preserve dance
  (the KEEP/RESTORE aside) — the interim guard, not the fix.

## The legs (each its own commit + evidence)

1. **Leg 1 — d:anatomy** (this session): the FACES entry + the A5b
   battery check + the master's 3 splits + the two consumers' render
   + the re-emit + the full sweep.
2. **Leg 2 — the key law's battery** (Part A's checks; the names ride
   their renames case-by-case as suites touch the regions).
3. **Leg 3 — d:rules + the consistency battery's derived gates**
   (replacing the S112 hardcoded mirrors; the $state-derived consumer
   rides here too).
4. **Leg 4 — the db outside the code dir** (Part D; the real fix for
   this session's two live catches; the interim preserve-dance lands
   WITH the --skip-estate fix).

## The ledger (renames + face admissions, appended here)
- (none yet — leg 1 opens it with `$anatomy`.)
