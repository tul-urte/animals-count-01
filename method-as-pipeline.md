# method-as-pipeline.md — THE META-SYSTEM SEEN AS ITS OWN PIPELINE

**What this is**: an essay, not a law doc. The estate's pipeline
(reception → ingest → assess → workflow → triage → publish) set beside
the methodology we've been building to develop it. The analogy earns
its keep where it makes *predictions*; it is marked honestly where it
breaks. Written as a riff after S113 (2026-11-18) — the user's opening
moved, the assistant's cut, per the usual division.

Status: EXPLORATION. Nothing here binds a future session unless a
ruling adopts it (the same standing rule as any plan: a plan is not
authorization; a doc is not a law).

---

## 1. The mapping table

| the animals-count system | the methodology |
|---|---|
| an **image** (a data block) | a session's raw experience: the incidents, the catches, the questions asked |
| its **detection** (metadata: boxes, scores, labels) | the assistant's first-cut interpretation: "what happened → what rule follows" |
| **reception** (the pool) | the world mid-session: broken things, un-read docs, open questions — arriving faster than they're processed |
| **ingest** (register + move; idempotent by natural key) | the session ritual: queue read → notes read → work → the three-write close (the register commits BEFORE the move: the record survives a crash mid-session) |
| **assess** (per-universe assessor) | the interpretation pass at close: which laws did this session mint, violate, or re-verify? |
| **the workflow statuses** | see §3 — the mapping is tight |
| **triage/editor** (human claims, publishes) | THE USER: rulings, verbatim commands, the over-build catches ("what else is involved?") |
| **GT versions** (ground truth, sticky, terminal) | the skills + their addenda: the doctrine, versioned |
| **GT universes** (which rows assess where; publish routing) | the path-scoped instructions + the skill split: which laws apply to which part of the estate |
| **reports** (http surfaces: doorways, counts) | the ledger, the README, the plan's finish-lines: the authoritative read for a newcomer |
| **junk** (EMPTY; the evicted bytes) | the scratch: probes, `__*.py` files, shot dirs — cleaned by the hygiene law, unmissed |
| **glitched lanes** (gclaim → review → glitched \| gjunk) | the misfiring laws: re-examined ("fired a THIRD time" = re-fit again), re-cut, or cut entirely |

## 2. The two trees, side by side

```
THE ESTATE                                          THE RECORD
reception/ pool (<cam>/<day> pairs)                 the world's incidents (unbound)
  └─ ingest-v2.bite (register→move)                   └─ the session (S#; the pass)
      └─ images tree (/cam/<day>/)                        └─ notes/ (one file per session)
          └─ assess (per model, per universe)                 ├─ ledger row (the index)
              └─ workflow row (one per pair)                  ├─ observations (minted rules)
                  ├─ new → assessed                           ├─ queue (NOW → DONE/RETIRED)
                  │    ├─ claimed → editor                    ├─ plans (legs, finish-lines)
                  │    │    └─ published|rejected             └─ skills (addenda = versions)
                  │    ├─ staged → reserved                       └─ GT = the doctrine
                  │    ├─ junk (EMPTY; eviction)
                  │    └─ auto-reject / auto-repeat
                  └─ g-claim → gclaim → glitched|gjunk        batteries (the teeth)
      GT versions ← publish (per universe)                    shares ← animals-count-01
      reports ← the http surfaces                             reports ← README + ledger
```

Both trees are real trees: the estate by camera/day/universe, the
record by date/session/law. Both have ONE entry gate (the pool / the
incident) and one idempotency law (the natural key / the session
number).

## 3. The statuses, mapped tightly

| estate status | the methodology's | the moment |
|---|---|---|
| `new` | an unexamined incident | the thing breaks / the question is asked; value unknown |
| `assessed` | the observations entry | interpreted: observed → the rule that follows (tags: none carried beyond the words — see §6, the honest gap) |
| `claimed` | a plan leg claimed | the work locked to a session (the queue's NOW top) |
| `staged` | built + rehearsed on dev | the commit exists, the gate (qa-bank / the user's call) hasn't moved |
| `published` | the commit that lands + the law that enters a skill | verified, evidenced; sticky |
| `rejected` | the cut | the over-build catch (satisfies()/roles()); the retired design (estates dropped) |
| `junk` | the scratch | probes, temp shots, `__*.py` — unmissed after the commit |
| `published + is GT` | an addendum | the doctrine versioned; "absorbed into the ground truth" ↔ "absorbed into the skill" |
| `gclaim/glitched` | the misfiring law | "the S080 alias trap fired a THIRD time" — the re-fit walk; convergence = a session passing with no re-fire |
| `gjunk` | the cut law | re-examined and dropped (from= retired; the {k} grammar retired cleanly BECAUSE it had a refusal-shaped exit) |

## 4. The internal outputs (to self)

- **The observations log** is the methodology's glitch-lane cleaner.
  An incident enters (the same bug a third time), the entry converts
  it to a rule, and future sessions inherit the prior. That IS "cleans
  the queue": the rule removes the failure class from future
  processing.
- **The plans' legs + finish-lines** are the published blocks feeding
  new GT: each leg's commit + evidence is a ground-truth version of
  the design; the next plan builds on the versioned truth (the
  validate_pages "UNKNOWN stamp = re-adopt it" loop is the re-admission
  face: the estate grows, the design doc must follow).
- **The queue** is the workflow's own read: what's open, in what order,
  what each item's evidence-finish is. Its DONE/RETIRED tail is the
  published log's spine.

## 5. The external outputs

- **`animals-count-01`** — the published record: verified (redacted,
  audited, byte-checked), exactly as a GT publish is gated (the stage
  gate rode first: the redaction pass = the privacy version of "the
  publish law").
- The **README + ledger** as the report surface: the doorways-analogy
  (the authoritative read a newcomer trusts).
- **The skills as portable doctrine** — the external output with a
  future: another project's meta-system could load the same laws and
  addenda. (The closest thing the methodology has to a deployable
  model artifact.)

## 6. What matches (the predictive mappings)

1. **The skills are the weights; the notes are the training data.**
   The estate improves by retraining — it forgets the data, keeps the
   model. The methodology improves the other way: sessions are
   STATELESS (the forgetfulness law is user-observed fact), so it
   keeps ALL the data and rebuilds the "model" each session from
   skills + AGENTS.md. The addenda are the epoch markers.
2. **The convergence walk is law-stability.** "done: 537 of 537
   re-fitted … pass 2 carries NO move leg" ↔ a law whose failure class
   stops re-firing. The observations log's repeats ("fired a THIRD
   live time") are not embarrassments — they are the envelope's
   remaining members, honestly reported at each re-fit.
3. **MIN_PER_CLASS / MIN_HOLOS ↔ the single-incident floor.** The
   estate refuses to train a head on too small a corpus. The
   methodology has the analogous discipline only informally: a LAW
   minted from ONE catch is under-trained. §9 proposes the floor
   explicitly.
4. **Anti-truth is a human act** — the walkthrough law ("a probe
   never publishes truth") ↔ the USERROLE law's origin (the assistant
   invented satisfies(); the user cut it). Both systems: the model
   interprets, the human certifies. The meta-system's "model" is the
   assistant writing law-prose mid-flow — always a hypothesis until
   ruled.
5. **The whole-queue gate ↔ the redaction pass.** "Date (and camera)
   filters go on BEFORE any whole-slice action on the live estate" was
   re-learned today at the public-repo boundary: the privacy filters
   went on BEFORE the publish.
6. **The stale-server trap ↔ the handoff-drift law.** The estate: a
   restarted process serving old code — the claim says X, the artifact
   says Y. The record: a session claiming X while disk says Y. Same
   defect class, same cure: verify against the artifact (the rendered
   page / the filesystem), never the claim.
7. **The doorways' honesty law ↔ the ledger's row honesty.** "A page's
   payload word lying under a chip" = a queue row claiming
   last-confirmed S### that never touched the work. Both systems lie
   in their SUMMARY surfaces when the summary is written by a
   different pass than the work.

## 7. What doesn't match (the honest gaps)

- **No model sits in the meta-system.** The "assessor" is attention.
  There is no loss, no epochs, no confidence — a law is adopted or
  cut, never a 0.62. The closest thing to a confidence surface is the
  battery (a law's teeth = the check that fires). Do not mistake the
  analogy for an architecture: it should NOT tempt an "auto-law"
  feature (the anti-truth law again); it SHOULD tempt mechanical
  *premise checks* (§9).
- **The meta-system cannot crash — it can only forget.** The estate's
  failure mode: stateful services dying under load, needing
  keep-alives and stop-first discipline. The record's failure mode:
  a fresh session that follows a stale claim, needing re-read
  discipline. They are mirror-image failure modes: the estate carries
  state forward dangerously; the meta derives state dangerously. Both
  laws exist for exactly this mirror.
- **Serial, one-tenant.** The estate is multi-tenant with parallel
  queues; the record processes one session at a time (an agent in
  background is the rare fork). The analogy should not suggest
  fan-out "workers" for law-writing.
- **No throughput.** "Largely self-maintaining" is true of the
  estate's pipeline (cron keep-alives; the pool self-heals; errors
  never move files) and NOT true of the record: the three-write close
  is a discipline, not a daemon. The honest phrasing: the meta-system
  is *self-maintaining through ritual*, and the ritual is the
  keep-alive.

## 8. What informs / what disinform

- **INFORMS**: treating session closes as ingest (commit-before-move;
  the evidence-before-claim idempotency); treating repeated failures
  as re-fit events (the envelope honesty, never shame); treating
  skills as versioned GT (the addendum discipline already right);
  treating the public copy as a publish (gated, staged, filtered
  first).
- **DISINFORMS**: "automate the assessments" (the record's assessing
  is the human+AI pair by design); "fan out workers" (one reader, one
  writer at a time); "schema the markdown" (the queue already carries
  the statuses; the notes stay prose until a consumer needs them —
  the d:-face admission law, applied to the record itself); "keep a
  model" (the meta-system's model IS the corpus; freezing any of it
  as weights would be a loss, not a gain).

## 9. What the analogy suggests next (proposals, not rulings)

1. **The law-premise check (the stage-1.5 of methods).** At session
   start, the S113 pattern applied to itself: read the open plan legs
   and the observations log together; refuse a leg that rides a law
   it violates. Cheap, mechanical, and it is exactly how the fresh-
   premise refusal caught the poisoned estate.
2. **The single-incident floor, made law.** One live catch = an
   observation (a hypothesis, banded "HINT"); the second live catch =
   promotable to RULE; a mechanical refusal or the user's verbatim
   ruling = LAW (the never-bend band). The band tags ride the entry
   itself (`[LAW]`/`[RULE]`/`[HINT]`), the strength read at a glance —
   the band decision, applied to the record, with the same
   retune-at-the-panel rule.
3. **The session-close smoke.** The close is the meta-system's cron; a
   smoke would be its curl: did the three writes land (the notes file
   exists, the ledger row appended, the queue touched)? The same
   "restart is not verified by its HTTP 200" law, self-applied.
4. **The addendum discipline as versioned GT — kept, and named as
   such.** It is the closest thing to the model-version loop and
   costs nothing extra.

## 10. The verdict line

The analogy is not decoration: it says the record's improvement loop
is a *reader loop* where the estate's is a *training loop*, and it
predicts the meta-system's best next moves are mechanical gates (a
premise check, a close-smoke, a banded law strength), not learned
ones. Where the analogy whispers "automate the assessments", the
estate's own law answers from the S106 walkthrough: a probe never
publishes truth — and in this system, neither do I.
