# plan-auto-repeat — the unified repeat-action framework

*Status: S060b LANDED (2026-10-31, dev). The engine (autorepeat.py), the
matcher library (matchers.py), the launder route + the sweep delegation,
THE CHUG + THE CANCEL, and /admin/autorepeat (the typed-knob config page)
are built, browser-QA'd, and battery-green. The session's live proof: 389
rows un-staged in one click (the unclassified experiment; the chug's
OR-walk, 16.1s). Pack-down restored the dev-bank to the pour's shape
(9,475 staged). See notes/2026-10-28 S064.md.

## The brief (user, 2026-10-30)

"how to configure and improve the mechanism... deal with 178,074 images in the assessed queue" and "the review of the 9,475 staged images".

The goal is a unified framework for "auto-actions" that process a list based on a manual seed.

## The Unified Pattern

An auto-repeat action follows a consistent lifecycle:
1. **Trigger**: A manual transition on a "Seed" image (e.g., ✖ Reject or ⤓ Unstage).
2. **Activation**: Only runs if the user has the "Auto-Repeat" toggle ON for the current queue.
3. **The Walk**: Walks the queue forward in the order clicking NEXT produces (the queue's own sort + filter context).
4. **The Pipeline**: For each candidate, it applies a sequence of boolean match functions.
    - **Seed-Relative**: `f(seed, candidate) -> bool` (e.g., "Is this similar to the seed?")
    - **Absolute**: `f(candidate) -> bool` (e.g., "Is this too dark?")
5. **Halt**: The walk stops immediately at the first `False` result (unless the composition logic dictates otherwise).
6. **Execution**: All candidates that passed the pipeline are transitioned using the same action as the seed.

## What exists (The Auto-Reject implementation)

**The doctrine** ([similar.py](c:/projects/animals-count/06-deployment/imagebank-dev/similar.py)):
Implemented for the "Reject" action. It uses a **Fingerprint** (deduped boxes + dHash) and a **Gate** (signature-box, hash, or hybrid).

**The route**: `POST /action/autorejected` ([routes.py autoreject_post](c:/projects/animals-count/06-deployment/imagebank-dev/routes.py)).

**The knobs**: `/admin/config` keys (`autoreject.*`) for tuning the gates (containment, score tolerance, etc.).

## The Road Ahead: The General Framework

To support "Auto-Repeat-Unstage" and future actions, the framework must be generalized.

### 1. The Function Library — LANDED (matchers.py)

`six`... the library exists: `similarity` (the S059 seam, relative),
`darkness` (mean grey ≤ threshold; absolute), `blur` (PIL FIND_EDGES
variance ≤ threshold; absolute). Relative matchers receive the seed;
absolute ones don't (the launder's seed only marks the start).

### 2. Admin Configuration — LANDED (/admin/autorepeat)

One panel per (queue, action): the ordered sequence (↑↓✕ reorder,
append/remove), the composition (and/or dropdown), the arm state, the run
knobs, and TYPED WIDGETS — sliders constrained to valid ranges (with live
readouts), checkboxes for booleans, dropdowns for choices; the same spec
validates server-side. Config persists to the properties file, read fresh
per run. OPEN: per-instance step parameters (the same matcher twice with
different thresholds) — today one live value is shared per matcher name.

### 3. Specific Application & Logic

#### Auto-Repeat-Unstage (The launder) — LANDED (/action/autorepeat-unstaged)
- **Context**: The queue is expected to have high-quality detections. We are hunting for anomalies (mistakes that slipped through).
- **Logic**: Since we are looking for "broken" images, **Similarity is not used**. The seed image simply marks the start point. Any single match (e.g., "too dark" OR "too blurred") is sufficient to trigger the unstage.
- **Goal**: Rapidly clear the 9,475 staged images by isolating these anomalies.
- **The transition**: the DIRECT staged→claimed edge ('un-stage') — no invented statuses (the transition law held).
- **The chug + cancel**: the OR-walk pages the WHOLE queue-forward (clean frames step over); live tally + ⚠ cancel via /api/autorepeat + /action/autorepeat-cancel.

#### Auto-Repeat-Reject (The sweep) — route delegated, queue-shell wiring OPEN
- **Context**: The queue is expected to have poor-quality detections. We are looking for reasons to reject.
- **Logic**: Because failures often occur in contiguous scene sequences, **Similarity is highly effective**.
- **Goal**: Process the 178K assessed queue by sweeping similar "junk" bursts.
- **Status**: /action/autorejected runs the engine (AND-composition, the S059 stop-at-first-miss); its live tuning reads the autoreject.* keys as before. The remaining work is the queue-shell's per-row wiring + its battery pass.

## The 178,074-image arithmetic (The a-priori need)

The "Auto-Reject" side remains critical for the massive assessed queue. The general framework's ability to sequence "cheaper" checks (Darkness -> Blur -> Similarity) before expensive ones (Model Inference) is key to processing 178K images without crashing the browser or server.