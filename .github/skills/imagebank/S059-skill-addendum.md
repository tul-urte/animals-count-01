# S059 addendum — the auto-reject (draft note for the next skill refresh)

*Not yet merged into SKILL.md — the next session's skill review decides
the wording; kept alongside the skill while the experiments run.*

## The auto-reject cascade (S059, 2026-10-28)

`similar.py` (imagebank-dev): the editor's reject cascades when the
?ar=1 toggle is armed — the seed fingerprinted once (deduped detection
boxes + 16x16 dHash), the queue-forward snapshot (rows AFTER the seed's
rank in the queue's order AS FILTERED, taken BEFORE the reject so the
ranks never shift), every row match() admits is rejected via a REAL
workflow.transition (events note 'auto-reject: similar to <seed>'), the
run stops at the first refusal or the cap (10) / the spend budget (5 s).
The landing: the row now AT the seed's old rank = the first survivor;
the resolved-ctx rule re-ranks against the row's OWN status queue when
the page's q= doesn't hold it (the stale-ctx hole, the user's live
catch). POST /action/autorejected; the knobs (autoreject.*) live on
/admin/config — retuning needs NO deploy. Reversible per image (the
rejected queue's unreject); never physically deletes (the junk EMPTY
stays the bank's only sanctioned mutation).

The trial drivers: debug/ar_health*.py (the estate health pair),
ar_verify_states.py, ar_lastrec.py, ar_restore.py + the batteries
(debug/test_autoreject.py 46/0 rides pi_deploy).

## The auto-repeat framework (S060/S060b, 2026-10-31)

The generalisation landed: `autorepeat.py` (the engine: per-(queue,action)
PIPELINE × COMPOSITION — 'and' = the sweep's stop-at-first-miss, 'or' =
the launder's step-over-and-keep-chugging; snapshots: AND keeps cap+8,
OR pages the whole queue-forward), `matchers.py` (similarity
[relative] + darkness/blur [absolute]; PIL only, no new wheels),
`viewer.py`-wired toggles ?ar= (S059) and ?au= (the launder twin,
direct-only gated on the staged→claimed edge), `/action/autorepeat-unstaged`
+ `/action/autorepeat-cancel` (the cooperative cancel flag on scan_state)
+ `/api/autorepeat` (the live record), and `/admin/autorepeat` — the
typed-knob pipeline page (sliders/checkboxes/dropdowns, server-validated;
sequence reorder live; similarity edits still write autoreject.*).
The user's live proof: 389 rows un-staged in ONE click (darkness=255
experiment on the eb_v20 unclassified replay rows). The dev-bank was
restored to the pour's shape at pack-down; the trail keeps every step.