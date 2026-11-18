# The animals-count project — the methodology record

**What this is**: the working record of a wildlife-camera estate —
the design decisions, the operating laws, the session-by-session
history, and the plans — shared as METHOD, not as a running codebase.
The repository carries **no code and no git history**: the deliverable
here is *how the project reasons about itself*.

## The project in one paragraph

 Raspberry Pi Zero 2 W camera nodes (IMX500 AI sensors) photograph a
garden/camera-trap estate, detect animals on-device (SSD-MobileNet and
stock-COCO models), and push pairs (image + detection XML) to a central
"imagebank" (a Pi 5 with a USB disk) over rsync. The bank ingests them
into a SQLite workflow (harvest-v2.db: new → assessed → claimed →
published/rejected/junk, plus the glitch-family lanes), humans triage
them in browser queue/editor surfaces, ground-truth versions publish
out, and the per-universe assessors improve the nodes' models in a
loop. Small fleet (3 camera nodes + 1 bank), one human + one AI
assistant pair, browser-QA'd every step.

## The folders

| folder | what it holds |
|---|---|
| `documents/` | the plans (plan-*.md), the two big design drafts, the session LEDGER+INDEX (session_notes.md), the PLAN front door (session_queue.md), misc research notes |
| `notes/` | **the record**: one file per session (S001–S113), dated, each an entry with what was done + the evidence |
| `.github/skills/` | the distilled operating doctrine: `imagebank` (the bank's pipeline + QA laws), `picods` (the camera nodes), `rpi-from-windows` (the deploy/debug discipline) |
| `.github/instructions/` | the path-scoped working rules |

## The reading order (how a newcomer should tour this)

1. **`AGENTS.md` at the repo root** — the working agreement: how the
   pair works, the step-by-step law, the known failure modes, and the
   OBSERVATIONS LOG (the lessons each incident minted).
2. **`documents/session_queue.md`** — the PLAN: what is open, in what
   order, what each item's finish-line of evidence is.
3. **Any `plan-*.md` that catches the eye** — the big designs each got
   their own document (the image bank, the design model's meta-spec,
   rsync cutover, the materializer…).
4. **The skills** (`.github/skills/imagebank/SKILL.md`,
   `.github/skills/picods/SKILL.md`) — the doctrine that survived the
   sessions, including the "laws that never bend" sections.
5. **`notes/`** — the session record itself, earliest to latest; every
   claim in the plans and skills traces back to an entry here.

## The conventions the record itself follows

- Sessions are numbered S001…; a same-day follow-up rides its
  parent's file (the rider law) or carries a suffix.
- Every session closes with THREE writes: the notes file, the ledger
  row, and the queue update.
- Claims carry provenance: "the doc says" is distinct from "I
  verified"; handoff claims are re-verified against the filesystem
  when they matter (the handoff-drift law).
- The observations log at the bottom of AGENTS.md is append-only:
  each entry is date, what was observed, and the working rule that
  follows from it.

## Privacy note

All local network addresses (192.168.68.0/22 hosts and subnets) have
been replaced with stable placeholders (`<bank-ip>`, `<node-ip>`,
`<lan-subnet>`, …) in this copy; the QA-estate's throwaway logins
render as their placeholders too. The original estate's real
addresses and credentials deliberately live outside any repository.
