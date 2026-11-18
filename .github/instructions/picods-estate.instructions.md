---
description: PICODS camera nodes and imagebank estate — geometry/deploy/debug doctrine
applyTo: '06-deployment/**'
---

Work in this tree touches live, cron-supervised services on real hosts.
Before changing or deploying anything here, use the domain skills:

- `picods` skill — the frame hierarchy/coordinate-system invariants
  (sensor px, mode_scale at the image boundary, box mapping order),
  the deploy doctrine (temp-file + mv, dos2unix from the Windows tree,
  md5-verify both sides), and the debugging doctrine (live pages as
  the evidence surface). A wrong coordinate assumption here has
  twice produced silent zero-detection or black-image bugs.
- `imagebank` skill — the reception ingest pipeline, harvest.db
  semantics (paths must be final, image_exists gates the workflow),
  GT universes (per-universe assessors, editor categories, publish
  routing), and the triage workflow.

Current state is authoritative in the docs, not in memory: read the
session index `documents/session_notes.md` (one file per session under
`notes/`, named `YYYY-MM-DD S###[.suffix].md`), the kit README, and the
node records (`06-deployment/picods/picods*.md`) before acting on live
nodes.
Never commit secrets (tokens, passwords) — node records hold access
details outside the repo's committed files where possible, and any
secret that must exist in a file stays out of git.