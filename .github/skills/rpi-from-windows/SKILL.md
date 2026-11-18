name: rpi-from-windows
description: 'RPi camera node / Pi from Windows without repeating the same mistakes: connect over SSH via PuTTY plink/pscp (the password-based path), handle host-key changes on a fresh card, bootstrap passwordless sudo once, and ship files LF-clean (never inline ssh "bash ..."). Use when: talking to a Raspberry Pi from Windows — fresh card, reflash, re-add after a new host key, install packages or deploy a script, or any task that would otherwise mean re-learning the SSH/SCP/psudo dance.'
keywords:
  - raspberry pi
  - ssh
  - scp
  - plink
  - pscp
  - pi
  - linux node
  - windows
  - deploy
  - commissioning
---

# RPi from Windows — getting on without re-deriving everything

## LAWS THAT NEVER BEND (all work from Windows onto remote estates)

- **NO MOJIBAKE — NEVER, EVER.** Every file written this session and
  every file edited MUST keep its encoding bytes intact: the repo's
  docs/code carry UTF-8 (—, →, ✓) and some carry pre-existing mojibake
  (read those bytes EXACTLY as-is when matching old_str, never "fix"
  them incidentally). The PowerShell string-replace trap has eaten
  whole files before (2026-11-15: both schema SQLs emptied) — a blind
  bulk replace is not a verified step; check the file still
  parses/loads + its size is sane before anything builds on it. When
  PowerShell mangles a non-ASCII literal: stop and use a different
  mechanism (a python script, byte-level operations), never a re-typed
  lookalike char.

Raspberry Pi Zero 2 W + IMX500 camera nodes (and the imagebank Pi) live on the
LAN. From the Windows box, the reliable path is **PuTTY tooling + password
auth**, not OpenSSH and not the UHTRED key (that key/cert is not available in
most working sessions — fall back to the password and never waste time hunting
for a cert that isn't there).

This skill exists because every session re-derives these basics from scratch,
and the re-derivation itself is where the bugs come from. If a step here looks
familiar and annoying, that's the point.

## Who / how to reach (do not guess)

| Host | What | User | Where the answer is |
|------|------|------|---------------------|
| RPi OS Lite node (`picods100`/`101`/`102`) | bare node | `pi` | `06-deployment/picods/picods10X.md` node record |
| imagebank | bank Pi 5 | `pi` | the session index `documents/session_notes.md` + `notes/` |

- The node record lists: IP (`.52`/`.55`/`.54`), the **pi password**, the
  passwordless-sudo setup, host-key fingerprint, and the bank's `picods_1XX`
  identity. **Read the specific node record first** — never type an IP or
  password from memory.
- **Passwords are case-sensitive and wrong-case looks like "Access denied"
  with no hint that case was the problem.** picods101's `acd101-pi` fails as
  `ACD101-PI`. Type the record's password verbatim.
- The password is short and follows the hostname (`pi`, `acd100-pi`, etc.) —
  but the bank's NAS share uses the **bank** account (`acd001-pi`), which is a
  different thing. Don't mix them.
- **Never commit secrets.** Passwords and key fingerprints stay in node records
  and this skill — not in the repo.

## Connect with plink (PuTTY's command-line SSH)

`plink` is at `C:\Program Files\PuTTY\plink.exe`; `pscp` at
`C:\Program Files\PuTTY\pscp.exe`. OpenSSH's `ssh.exe` also works but its
quoting is the same trap as bash (see below), so prefer plink for remote work.

```powershell
# One line on the node — host-key is cached on first sight.
& "C:\Program Files\PuTTY\plink.exe" -pw 'ACD101-PI' pi@192.168.68.55 'hostname; uname -r'

# First sight (or after a reflash / new host key): pass it explicitly so plink
# in -batch mode doesn't abort on "cannot confirm a host key".
& "C:\Program Files\PuTTY\plink.exe" -pw 'ACD101-PI' `
    -hostkey '255 SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX' `
    pi@192.168.68.55 'hostname'
```

`255 SHA256:...` is the fingerprint format plink prints on first sight — copy
it verbatim.

## Fresh card or reflash? The host key has changed

A reflash regenerates the SSH host key. Windows keeps a `known_hosts` (OpenSSH)
or PuTTY session cache that now references the **old** key. Two distinct
symptoms:

- **plink** — `FATAL ERROR: Cannot confirm a host key in batch mode` (host key
  not cached) OR the old key is cached and mismatches.
- **OpenSSH** — `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` with
  `StrictHostKeyChecking` blocking.

Fix (both are safe here: we already confirmed the node is reachable and the
card is the one we reflashed):

- **plink**: add `-hostkey '<fingerprint>'` on the connect (above), or let it
  prompt once in interactive mode.
- **OpenSSH**: remove the stale line from `~/.ssh/known_hosts`
  (`~/.ssh/known_hosts` is line-addressed by IP — delete the line for that IP),
  then connect again. `StrictHostKeyChecking=accept-new` re-keys automatically
  but you must delete the offending line first.

This is the normal consequence of a reflash, not a threat. Confirm the new
fingerprint matches what the node record shows, then update the record.

## After a failed-attempt burst, plink/pscp can hang

Repeated failed logins (a wrong password, a killed transfer) can leave
subsequent **plink/pscp invocations hanging 30+ s with no output** even for a
command that worked minutes earlier. This is sshd rate-limiting on the node's
side. **Do not hammer**: stop the hung call, wait ~45s, retry once cleanly
with the exact recorded (lowercase) password. If it still hangs, stop and
come back later — more retries deepen the lockout. (Confirmed live 2026-11-12:
the *cause* was mixed-case passwords in a retry loop, not a rate limit —
fix the password first, then only hammering becomes the problem.)

## Two host-key stores exist on Windows — clear BOTH on a reflash

1. **OpenSSH `~/.ssh/known_hosts`**: line-addressed; delete every line for
   the node's IP (the same IP can have MULTIPLE stale key-type lines —
   ed25519 + RSA + ECDSA; the strict-check error names one at a time. Delete
   all of them, then reconnect).
2. **PuTTY's session cache** (registry): plink `-batch` will NOT confirm a
   new key even with the fingerprint printed in the error. Pass it:
   `-hostkey '<the printed "255 SHA256:...">'` — plink then stores it.

`StrictHostKeyChecking=accept-new` only fixes the OpenSSH store; it does
nothing for PuTTY's. (S073, third reflash of picods101: both stores needed
clearing; the error messages differ per store and both look like
"something is terribly wrong" when it is just the reflash.)

## The one-time password-to-sudo bootstrap

A **fresh** card has no passwordless sudo — the first `sudo` still wants the pi
password. Install the NOPASSWD rule once, then everything after uses `sudo -n`.
See `06-deployment/picods/picods100.md` and the node record for the exact
sudoers line. Do this via a shipped script (below), not inline.

## SHIP A SCRIPT FILE. Never inline `ssh "bash ..."`.

This is the single biggest cost of this environment and the source of most
bugs. PowerShell re-parses `$()`, parentheses, quotes, and
`sed 's/\r$//'` as PowerShell; it also **adds CRLF** to any piped bytes. The
remote bash then fails on `$'\r'` or on `set -e` breaking — and the failure
looks like a bash bug, not a Windows bug.

**Rule:** normalize once locally, ship the file, run it with `bash /path/file`.
Re-affirmed live ~6× during the S073 commissioning (2026-11-12); every
failure traced back to it.

1. Write the script with **LF** newlines (the editor/tool writes LF by default;
   never pipe through PowerShell which injects CRLF).
2. **Transfer with `pscp`** (PuTTY), not the node's `scp` binary:
   ```powershell
   & "C:\Program Files\PuTTY\pscp.exe" -pw 'ACD101-PI' `
       "C:\path\to\file.sh" pi@192.168.68.55:/tmp/file.sh
   ```
   `pscp` may still prompt on first sight if the PuTTY session cache has a
   conflicting key — use `-hostkey` as above.
3. **Verify both sides have the same bytes** before trusting the transfer:
   md5 the local file, md5 the remote file, and **compare with carriage returns
   stripped** (`md5sum` then `tr -d '\r'` before hashing, or compare the raw
   files on each side with CRLF removed). Windows CRLF and Unix LF files can
   have different raw md5s but be byte-identical once CRLF is stripped — the
   CR-stripped md5 is the real equality check.
4. Run it:
   ```powershell
   & "C:\Program Files\PuTTY\plink.exe" -pw 'ACD101-PI' pi@192.168.68.55 'bash /tmp/file.sh'
   ```

## dos2unix and the deploy-after-transfer sequence

After a script (or a whole deploy) is on the node:

- Run `dos2unix` on it (and on every file you ship) — fresh files from a
  Windows working tree carry CRLF, which chokes bash `set -e`.
- When deploying a set of files, copy them, `dos2unix` the lot, then run the
  deploy script. This is the runbook standard, not optional.
- **Verify the crontab against the raw spool bytes** (`od -c` on the crontab
  spool), never against `crontab -l` output — that's how a CRLF line hides.

## Quick reference cheatsheet

```powershell
# reachable?
Test-NetConnection 192.168.68.55 -Port 22 | Select-Object RemoteAddress, TcpTestSucceeded

# first sight / after reflash
& "C:\Program Files\PuTTY\plink.exe" -pw 'ACD101-PI' `
    -hostkey '255 SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX' `
    pi@192.168.68.55 'hostname'

# ship a file
& "C:\Program Files\PuTTY\pscp.exe" -pw 'ACD101-PI' `
    "C:\path\to\file.sh" pi@192.168.68.55:/tmp/file.sh

# verify LF-clean on both sides
& "C:\Program Files\PuTTY\plink.exe" -pw 'ACD101-PI' pi@192.168.68.55 'md5sum /tmp/file.sh'
# then CR-strip-compare against the local md5

# run it
& "C:\Program Files\PuTTY\plink.exe" -pw 'ACD101-PI' pi@192.168.68.55 'bash /tmp/file.sh'
```

## Anti-patterns (learned the hard way)

- **Do not** type inline `ssh "bash ..."` strings. Ship the file.
- **Do not** feed `$()` or parens through PowerShell to the node — it re-parses.
- **Do not** `printf 'password\n' | sudo -S` from a PowerShell here-string for
  more than the first sudo — CRLF in that pipe corrupts the password. Do the
  first `sudo` with the password, install the NOPASSWD rule, then use `-n`.
- **Do not** hunt for the UHTRED key/cert to save time. Fall back to the
  password — it's how this estate is actually reached in these sessions.
- **Do not** assume the old host key is valid after a reflash. Delete/override
  it.
