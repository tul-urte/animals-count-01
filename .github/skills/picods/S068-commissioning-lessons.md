# S068 addendum — the picods102 commissioning: process mistakes + the runbook the next node gets

*Draft for the next skill review — kept alongside the skill while picods102 is
being commissioned (2026-10-31). **The commission is now COMPLETE and live
(2026-11-01)** — see the checklist below and the node record
(`06-deployment/picods/picods102.md`). This captures the PROCESS LESSONS (the
mistakes that burned the session) so the NEXT commission is copy-paste.*

## The one sentence

**picods102 is a garden camera (the AnimalsCount universe, the animals-count
`network.rpk` model) — i.e. it follows picods100, not picods101. picods101
stays the in-house COCO test bench.** So the reference node for EVERY per-node
choice (identity, model, sensor mode, thresholds) is **picods100**.

## What the runbook should have been (read this FIRST next time)

The estate ALREADY has the answers; the mistakes below were mostly *re-deriving
them by hand* and *fighting the Windows→Linux toolchain* instead of going
through the proven path. The correct order:

1. **Read, in this order, before touching anything:**
   - `06-deployment/picods/picods101.md` (the two-commissioned node record —
     its "re-commissioning from a bare image (S040)" section is the *validated
     bare-flash ladder*; its "rebuild confirmed" list is the corrections to the
     S012 runbook).
   - `06-deployment/picods/kit/README.md` §"Deploying to a new RPi" (the layer
     list) + §"the rsync-era mover" (the identity shape).
   - `notes/2026-10-31 S065.md` **S065** (the rsync cutover — the EXACT
     bank-side identity shape: account, `rrsync` authorized_keys, `nodehome`
     creds, `users`+`user_roles` rows, and the two rrsync traps).
   - `06-deployment/imagebank/harvest/universes.py` (the `GT_UNIVERSES` map to
     extend for CAM-0102).
   - **Read picods100's LIVE state** (`service.properties`, `cam.properties`,
     `network.rpk` md5) — it is the reference node for a garden camera.

2. **Use the estate's tooling, don't hand-roll ssh:**
   - Bank + imagebank-dev changes: `deploy.py` / `pi_deploy.py` (they drive
     scp/ssh via `subprocess` with explicit argv, write remote scripts with
     `newline="\n"`, md5-verify CR-stripped, and sidestep every PowerShell
     quoting trap). For one-off node provisioning there is no single tool, so
     the DOCTRINE is: *ship the script LF-clean (base64 or `newline="\n"`),
     never an inline `ssh 'bash …'` string; verify the crontab against the
     SPOOL BYTES (`od -c`), never `crontab -l`; md5-verify both sides.*
   - PowerShell is the enemy here: it re-parses `$()`, parens, quotes, `sed 's/\r$//'`,
     and adds CRLF to piped bytes. Every failure in this session traced to one
     of those. **The S062-prep imagebank addendum §"cron-spool lessons" +
     "PowerShell-side stdin is itself a CRLF injection path" is the standing
     warning.** Route all multi-line work through a *file* that is normalized
     once and shipped.

3. **Copy the reference node, don't re-invent:**
   - The animals `network.rpk` is a **pipeline output that already lives on the
     bank / picods100** (`/home/pi/animals-count/network_ssd/network.rpk`) —
     pull it from there, don't hunt for a repo artifact.
   - The bank `users` row lives in `/home/pi/imagebank-dev/harvest-v2.db`
     (NOT `/home/pi/harvest/`, NOT `/yermo/harvest-v2.db`), columns
     `username` / `password_hash` / `disabled`, role in `user_roles`. Hash with
     `auth.py`'s `verify_password` (PBKDF2-sha256, `salt_hex$want_hex`).

## The mistakes, grouped (so the next session avoids the same traps)

1. **Inline ssh strings, over and over** — `ssh 'bash …'` with `$()`, parens,
   quotes; PowerShell re-parsed them as its own expressions (`set: -`,
   `syntax error near '('`, `Missing expression after ','`). 6+ hits.
   *Fix:* ship a script file; run `bash /tmp/script.sh`.
2. **`sed 's/\r$//'` eaten by PowerShell** → the shipped script kept CRLF and
   failed on *every* line (`$'\r': command not found`). *Fix:* normalize
   locally (`newline="\n"` / base64) BEFORE ship; md5-verify CR-stripped both
   sides (the local CR-stripped md5 == the remote md5 is the proof).
3. **CRLF in the sudo password pipe** — `echo pw | sudo -S` from PowerShell
   here-strings gave the wrong password 2×. *Fix:* `printf 'pw\n' | sudo -S`,
   and the first sudo only (write the NOPASSWD rule once, then everything is `-n`).
4. **Re-derived the bank identity from scratch** when picods100/101 had the
   exact shape; **looked for the bank DB in 3 wrong places**; **assumed the rpk
   was a repo artifact**. *Fix:* read the reference node + S065 first.
5. **Process, not technical:** spun up a pointless "scratchpad" sub-agent;
   **re-ran the same broken base script 3× instead of stop→diagnose→fix-once**;
   drafted a kit script that referenced files not yet shipped. *Fix:* on the
   FIRST failure, stop, read the log, fix the one cause.

## Node-specific facts (as established — verify before acting)

- **picods102 = 192.168.68.54** (OliveGrove), bare RPi OS Lite arm64, user `pi`,
  pw `acd102-pi`, Python 3.13.5, `hostname picods102`.
  Host key `ssh-ed25519 255 SHA256:tnE7aOi2uj/T1pnbLxzkfGeK6USP1dtOjuZWIZwyE60`.
- **Garden camera role** → AnimalsCount universe, animals `network.rpk`,
  `SENSOR_MODE` per picods100 (which runs `(2028,1520)` — confirm intent; the
  kit template ships `(4056,3040)`).
- **Bank identity for picods_102 DONE** (this session): account `picods_102`,
  `nodehome` creds at `/root/cutover-20261031/picods_102.nodehome` (root-600),
  `users`+`user_roles` rows in `harvest-v2.db` with `verify_password: True`.
  **STILL PENDING on the bank:** the restricted `authorized_keys` line
  (`from=192.168.68.54,command="/usr/bin/rrsync /yermo/animals-count"`) — it is
  filled in once the NODE's mover pubkey (`/root/.ssh/id_ed25519_picods`'s
  public half, generated on picods102) is shipped over.
- **Base layers** (sudoers, Uhtred key, apt, `imx500-all`, 5 python pkgs,
  dos2unix): in flight at time of writing (apt mid-`imx500-all`).

## Checklist — COMPLETE (commissioned live 2026-11-01)

- [x] bank identity (account + nodehome creds + users/user_roles rows)
- [x] base layers (verified live; see the `install ok unpacked` note in picods102.md)
- [x] kit → `/home/pi/picods` (82 files, identity=102, animals GRAPH, LF-clean, exec bits)
- [x] animals `network.rpk` → `/home/pi/animals-count/network_ssd/` (from picods100, md5-verified)
- [x] fstab tmpfs `/home/pi/cam-ram` (50M); **NO CIFS line**
- [x] root crontab: `run-from-cron.sh` + **`0 4 * * * shutdown -r now`** + `move-calendar-files-rsync.sh` (spool bytes LF-verified; reboot line added per user, mirrors picods100)
- [x] node: `id_ed25519_picods` generated; bank `picods_102` restricted line installed; `nodehome.creds` (root-600); root+pi known_hosts; mover dry-run rc 0
- [x] `universes.py`: `GT_UNIVERSES["CAM-0102"]="AnimalsCount"` (both repo copies, `pi_deploy.py`)
- [x] verify offline: `check_integration.py` ALL CHECKS PASS, `verify_rpk.py` VERIFY_OK
- [x] go live: SUSPENDED=0; detector + :8080 up; `stats.json` 47f/296detections on-content; mover push ok; landed in `reception/CAM-0102/…`
- [x] docs: `picods102.md` node record + S068 session note
- [x] callhome ping-as-picods_102 confirmed during the 2026-11-12 rebuild;
  authenticated ping recorded on imagebank at 10:42:56
- [x] runbook promoted to `06-deployment/picods/commission-new-node.md` and
  linked from the kit README, then validated during the 2026-11-12 rebuild

## 2026-11-12 rebuild — additional verified lessons

- The rebuilt node was at `.57`, not `.54`; mDNS found it. The user chose the
  new address and the bank's `from=` restriction was updated. Record DHCP
  reservations explicitly; do not assume the old address.
- New host key: `SHA256:OadvWz0RTPKV5l52CWAZzAYCJgxyI8vXnd09GKRqnFg`.
  New mover key: `SHA256:O5FKVN5cxpaMevMCl0p3m3/YnRowtqxro/CfYnEjPQQ`.
- Fresh rpicam tools do not imply `imx500-all` or the Python dependencies are
  installed. The explicit apt layer remains necessary.
- Never combine remote `ls` output with a credential stream. It corrupts the
  credential and produces an authenticated-looking shell command that the bank
  correctly denies. Transfer only the credential bytes; verify one line and the
  principal without printing the secret; confirm the bank event afterwards.
- Root's mover SSH identity does not give the node's `pi` user SSH access to
  imagebank. Do not add it just to retrieve a credential. A brief staging copy
  inside the rrsync root can be retrieved by the restricted key and removed.
- For a capture test, global `DETECTION_MIN_SCORE` does not override per-label
  gates. Use the live site's slave control (`label=all&value=0.00001`), save and
  restore `label-thresholds.json`, then verify real JPEG/XML pairs at reception.
