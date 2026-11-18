# Plan — replace the picods↔imagebank Samba with rsync + per-node accounts

Created 2026-10-31 (S062-prep session, user-set). Companion to
[plan-image-bank.md](./plan-image-bank.md) (Phase G carries the pointer)
and [session_notes.md](./session_notes.md) (the index) /
[notes/2026-10-31 S062-prep.md](../notes/2026-10-31%20S062-prep.md). The mover side
lives in the picods estate: [../06-deployment/picods/kit/move-calendar-files.sh](../06-deployment/picods/kit/move-calendar-files.sh);
the ingest side in imagebank-dev's `ingest-v2.py`.

**Status: COMPLETE (S0XX, 2026-10-31, 12:11→13:47) — all stages R0–R5
executed and verified; the CIFS era is closed.** The R0–R5 sections
below carry the per-step verification trail (the as-run record).
Scope widened 12:43 (user): the cutover REPLACES BOTH node→bank
mechanisms — the CIFS file path AND the anonymous callhome — with ONE
per-node identity on imagebank (`picods_100` / `picods_101`), used by
both. The R0 section below carries the full decision trail.

## Verdict (recorded 2026-10-31)

**Feasible: yes. Sensible: yes.** The trade is worth it, with clear eyes
on the costs:

**What rsync-over-ssh buys**
1. **No persistent mount to rot.** The CIFS mounts are the nodes' most
   fragile dependency (reconnect-after-outage pain, the S037-era
   quarantine mess). rsync is stateless: every cron minute is an
   independent, self-healing attempt; a Wi-Fi drop costs one minute.
2. **Real atomicity kills the settle-window hack.** rsync writes a temp
   dotfile then renames into place — any file seen under its real name is
   COMPLETE. Ingest's 20-second "CIFS copy in flight?" guess (`smb settle
   window`) becomes unnecessary; registration can fire immediately.
3. **Per-file integrity** (size+mtime, or `--checksum` for md5-grade
   equality) instead of trusting the SMB dialect.
4. **Per-node identity + audit**: CIFS rides one shared `username=pi`;
   per-NODE identities (an ssh account + a users-row each) make every
   transferred file AND every callhome event attributable — pairs
   with the estate's actor-stamping doctrine (events say who did
   what). (Broadened 12:43: the identity serves BOTH mechanisms.)
5. **Smaller blast radius on imagebank**: smbd loses its highest-churn
   clients (stopped entirely at R5; the only other consumer is the
   Windows box) and the bank-pi password stops living on every node.

**Costs / risks to plan around**
- Node churn: the mover script is rewritten (the estate's deploy doctrine
  applies: temp-file+mv, dos2unix, md5-verify both sides).
- rsync temp names must be invisible to ingest (hardening below) or a
  half-copied file gets registered — the bug class the settle window exists
  to prevent.
- mtime semantics change meaning (arrival time → the node's copy time).
  That is BETTER for ordering, but it must be a decision, not an accident.
- Two live nodes produce continuously — the cutover is done ONE node at a
  time with a live watch, soak, then the second.

## The current path (what is being replaced)

```
picods node (cron every minute):
  move-calendar-files.sh  — RAM drive (/home/pi/cam-ram) pairs →
  CIFS mount /YERMO → /yermo/animals-count/reception/CAM-0XXX[-CLASSIFIED]/<day>/<…>/

imagebank (cron every 5 min):
  ingest-v2.py — scans reception, settles (20 s), registers (idempotent
  by natural keys; image_path/xml_path = FINAL resting place), moves to
  cameras/…, refreshes workflow rows.
```

Consumers of the CIFS mount on the nodes: the mover only (verified in
S062-prep planning; re-verify at R0 with a mount-point grep — `lsof |
grep YERMO` + the kit's `.sh` greps).

## R0 — decisions (settling in the RSYNC-INGEST session, 2026-10-31)

1. **Identity.** RESOLVED FINAL (user, 2026-10-31 12:43): **one account
   PER NODE — `picods_100` + `picods_101`** (matching the nodes'
   NODE_IDs) — carrying BOTH node→bank mechanisms:
     * **the rsync push**: a one-key authorized_keys on the account
       (`from="<node ip>"`, `command="/usr/bin/rrsync
       /yermo/animals-count"`, `no-port-forwarding,no-X11-forwarding,
       no-agent-forwarding,no-pty`); the private half lives ONLY in
       the node's /root/.ssh. sshd's log names the account per node =
       attribution.
     * **the callhome (`POST /node-report`)**: the same-named row in
       the bank's users table (role `camera` — the schema's
       node-service role, present since the schema was drawn) via
       **HTTP Basic**; the form's node value must equal the
       authenticated principal. The ANONYMOUS era ends at this
       cutover: bank accepts no-header POSTs while
       `nodehome.require_auth=0`, flips to 1 once both nodes onboard
       (properties key = reversible, fresh-read). Denials log to
       picods_auth_log (the S050 audit table, reused).
   REJECTED shapes (the honest trail): reusing the nodes' local `pi`
   (nothing reusable — the nodes hold pub key halves ONLY; today they
   cannot initiate anything); a single shared account (the 12:24
   shape, SUPERSEDED 12:43 per the user — the API identity is per
   node too); and the CIFS status quo (the BANK-pi password in every
   node's fstab — every node holds a sudoer credential worth far more
   than its job; the flaw this cutover exists to remove).
2. **The flag set.** Baseline: `rsync -a --mkpath --timeout=60
   --remove-source-files --chmod=D0777,F0644`. NO `--inplace` (temp+rename
   IS the point). `--partial-dir` rejected: incomplete files must stay
   hidden, not in `--partial`. `--checksum` optional per-run cost (~180K
   historical N/A, live pairs are dozens/hour: affordable if wanted).
   The `--chmod` is NOT optional — it is the cutover's only real perms
   seam: rsync creates the reception day dirs under the `picods_1XX`
   accounts, while the ingest (pi's cron) must `os.replace` the files OUT
   of them later — rename needs write on the containing DIR, so
   mover-made dirs arriving 755 would wedge the drain. D0777 matches the
   CIFS-era posture exactly (today's reception tree is root:root 777 —
   probe-verified), files land F0644 (world-readable: parse_xml/os.stat
   in the register step need read). Refined at R1's touch-test.
   **Granularity**: one rsync per day-dir (same cadence — the mover's
   cron fires every minute either way — but one handshake per day-dir
   vs per-file handshakes; a night's burst of hundreds of files would
   cost minutes of pure handshake overhead per-file). `--remove-source-
   files` still removes per-file after each transfer succeeds.
3. **mtime.** RESOLVED (user: "choose the easiest"): **preserve** (`-a`
   brings the node's file mtime across). Not only easiest — zero semantic
   change: the CIFS-era `mv` already preserved the node's mtime and the
   ingest registers exactly that st_mtime; the XML's snap time stays the
   precise capture clock. (No-RTC clock skew only disturbs pool ORDER;
   it gates nothing once R2 retires `ingest_settle_s`.)
4. **Soak policy.** RESOLVED (user): CIFS stays mounted-but-unused for
   **~one night (~12 h)** before picods100's cutover; the full 24–48 h
   posture before the fstab teardown (R5).

## R1 — imagebank prep

- The TWO Linux accounts: `useradd -r -m -d /home/picods_1XX -s
  /bin/bash picods_1XX` (system users, no password login). TRAP
  (recorded): forced-command ssh runs via the user's SHELL — **no
  `nologin` shell** or the rrsync pin never executes; confinement
  comes from `command=`, not the shell.
- Each authorized_keys = the node's ONE pub key with the from=/command=
  restriction line + key comment naming the node (as R0.1).
- The users-table rows `picods_100` / `picods_101` (role `camera`,
  long random passwords; created by an idempotent driver,
  debug/pi_picods_accounts.py, via the bank's auth.hash_password).
  Password convention: generated at cutover, stored in the node's
  root-600 creds file + the node record's identity table in-repo (the
  estate's existing node-record convention). Verify
  roles_of(picods_1XX) == ['camera'] (NO workflow powers: not one
  transition edge grants camera).
- The enforcement key `nodehome.require_auth` lands with R2's deploy
  (default 0 → flip 1 in R4, after both nodes onboard).
- Bank host-key pre-seed per node: `ssh-keyscan -T 8 <bank-ip>`
  **from each node as root** (the mover's vantage; the bank need not
  know the nodes) + the fingerprint sanity check, into
  `/root/.ssh/known_hosts` — BatchMode must not die on the first-use
  prompt. **rrsync PATH TRAP (proven 12:53): with
  `command=rrsync /yermo/animals-count`, remote paths are RELATIVE to
  that root — an absolute remote path silently creates
  `<root>/<abs-path>` nesting (found via a test file at
  `animals-count/yermo/animals-count/reception/…`; cleaned + re-run).
  The mover's remote spec is therefore plain
  `picods_1XX@<bank-ip>:reception/<CAM-…>/<day>` (no leading
  slash) — recorded here because it is invisible in a plain success
  and only shows when the bank tree is walked.
- `rsync` on the nodes verified (picods101: 3.5.0, picods100 expected
  equal — check at its cutover; both negotiate to the bank's 3.2.7).
- Perms: the reception tree must be writable by the service identity
  (`/yermo/animals-count/reception/` is drwxrwxrwx root — the account
  inherits; verify by touch-test from the node).
- sshd: no changes needed (probed 2026-10-31: pubkey-only; no
  ClientAliveInterval; no Allow/Deny lists to fight).
- Trap (recorded): `from=` pins the DHCP-stable node IPs (.52/.55). If a
  lease ever drifts, the mover fails visibly (its log + sshd's "refused
  for picods from <ip>") and the fix is one authorized_keys line.

**R0 probes (run 2026-10-31, read-only):**
- imagebank: rsync 3.2.7 / protocol 32; **rrsync present at
  /usr/bin/rrsync** (the `command=` scoping base); sshd pubkey-only,
  no ClientAliveInterval, no Allow/Deny lists → NO sshd changes needed;
  reception `drwxrwxrwx root` (any service account inherits write);
  pool depth 0.
- picods101: rsync 3.5.0 / protocol 32 (negotiates down to the bank's
  3.2.7 fine); YERMO mount ACTIVE; `username=pi` + password INLINE in a
  644 /etc/fstab (the exposure R0.1 removes, confirmed live).
- Flag set gains `--mkpath` (rsync ≥3.2.3 both sides ✓): rsync does NOT
  create missing destination dirs for plain per-day pushes; --mkpath
  makes each day's dir self-serve — no helper ssh rights needed.
- Kit CIFS consumers beyond the mover: `run-from-cron.sh`'s
  `sudo mount -a` (leave during soak; harmless no-op after R5) and
  `callhome.sh`'s `cifs=` fact (flip to an rsync-era fact in R5).

## R2 — ingest hardening (imagebank-dev)

- `ingest-v2.pool_paths()`: skip rsync temp names — leading `.` (the
  rsync default `.name.random` form) and any hidden entries. Cheap:
  one `fn.startswith('.')` check on the file name at both loop levels.
- `nodehome.report()`: the auth gate — read `Authorization: Basic` /
  user+pass via auth.verify_password; the principal MUST be a user
  row with role `camera` AND `disabled=0`, the form's `node` value
  MUST equal the principal's own node id (`picods_100`'s credentials
  cannot file as `picods_101`); failures → 403 + one rate-bounded
  picods_auth_log row (the S050 pattern, reused verbatim). While
  `nodehome.require_auth=0` (the new properties key, default 0), a
  headerless POST is accepted as today — the two-nodes-onboard flip
  to 1 in R4 closes the anon path. (No token files, no new secret
  plumbing: Basic over the existing HTTPS.)
- callhome tests: the battery's existing nodehome assertions stay
  green PLUS new ones — a camera-row POST passes; wrong password /
  not-camera / node-mismatch → 403; while require_auth=0 headerless
  passes (compat), after flip → 403. *(DONE 12:57: test_nodehome 20/0
  including all ten new gate assertions; the ingest dotfile battery
  test_ingest_dotfile.py 7/0 (hidden skipped, settle-0 registers at
  the FINAL path, idempotent second bite); the full local suite
  24 files / 0 failed; the admin config page lists
  nodehome.require_auth; everything compiles.)*
- `ingest_settle_s` → 0 NOW with the rsync move (temp+rename atomicity
  makes the file-under-its-real-name-complete guarantee exact; the
  properties key stays for reversibility — restore `20` if ever needed).
  *Timing correction (12:58, self-review): the flip happens WITH R3's
  cutover, not before — until then the CIFS pool is the live path and
  the settle guess is still load-bearing for it. R2 only READIES the
  default; the deploy sets it the same minute the first node flips.*
- `ibtest_harness` assertions: (a) a dot-named file in the pool is
  skipped and later processed under its real name; (b) a settled file
  with settle_s=0 registers on the first bite. *(PROVEN 12:55 in an
  isolated scratch bank: bite with settle 0 = scanned 2 / registered 1
  / hidden file skipped / real pair moved + registered at its FINAL
  path. Perma-test to add below the existing debug batteries.)*
- Deploy via pi_deploy + harness `probe failures: 0`; then set
  `ingest_settle_s=0` via the admin config (properties, fresh-read) and
  watch ONE live bite register a fresh pair immediately.

## R2b — the /cameras page (the focused node view) — user-set 12:50

A login-protected, server-rendered page: ONE CARD PER CAMERA (all six:
CAM-02/05/06 historical + CAM-0100/CAM-0101 on the nodes), the focused
detail the dashboard's node panel compresses — and AFTER it exists, the
user may de-clutter the dashboard (decision at the R4 QA, his call).

Each card:
- **identity**: camera id, GT universe (universes.py), the owning node
  (CAM-0100→picods_100, CAM-0101→picods_101) with the node's LAST WORD
  (node_state: event/reason/facts — parsed: disk_use, throttled, wifi,
  demo/server) + `at`; historical cameras show "node: harvest-era (none)".
- **pair health**: that camera's workflow counts (new/assessed/claimed/
  staged/published/rejected/junk) + queue doorways (`/queues/<status>?
  camera=CAM-…` — the existing filter convention, no new query shape).
- **ingest pulse**: registered count + newest arrival over 24 h (the
  images rows), the newest day dir + its pair count on disk.
- **the node's recent callhome trail** (node_events for its node id,
  newest-first, bounded) — now AUTHENTICATED per R0.1's users rows.
- **auth audit**: the camera row's recent picods_auth_log verdicts.

Build notes: follow the estate (queries.json entries, not inline SQL;
server-rendered pages.py pattern; the nav's link row gains "cameras").
Test: ibtest_harness asserts the route for a synthetic camera + node
rows (the synthetic schema mirrors production — node tables included).
**BROWSER QA IS MANDATORY** (the standing law) before this is "done":
open /cameras live, exercise a queue doorway, screenshot as evidence.
DASHBOARD DECLUTTER: deferred to the R4 QA (remove the node panel only
once /cameras carries more; the user decides).
*(BUILT+DEPLOYED 13:36–13:39: `api.cameras_json` (the per-camera merge:
identity/universe/node + last word + parsed facts + per-status counts
+ the 24 h pulse + the callhome trail + auth verdicts; node-only rows
for never-seen nodes) + `web/cameras.html` (the static-shell idiom,
5 s refresh, EVERY status count a doorway into
`/q/<status>?camera=<cam>`) + `/cameras` + `/api/cameras` routes +
the nav link on all three shells (dash/queues/queue). Local synthetic
sanity passed before deploy; the live feed verified server-side
(200, real rows). The BROWSER QA pass is the owner's step (the R4
law) — the page is live at https://imagebank/cameras for it.)*

## R3 — node cutover (ONE node first — picods101, the test bench)

- Order note (user decision 2026-10-31): picods101 goes first — CAM-0101
  is the CocoTest universe (NO assessor: `UNIVERSE_MODELS["CocoTest"] =
  (None, None)`; pairs take the no-assessor junk path), so the first
  live cutover carries no GT/triage risk. picods100 (the animals
  pipeline) is the SECOND node, in R4.
- New mover script `move-calendar-files-rsync.sh` in the kit:
  same RAM-drive scan, same reception layout
  (`CAM-0XXX/<day>/*.jpg` + `CAM-0XXX-CLASSIFIED/<day>/<graph>/*.xml`),
  per-day-dir `rsync -a --remove-source-files` to
  `picods_1XX@<bank-ip>:…/reception/<CAM-…>/…` (the node's OWN
  account; -i /root/.ssh/id_ed25519_picods, the R1-known host key);
  failure → files stay on the RAM drive (the cron retries next minute
  — the pool self-heals, the estate's existing doctrine). The mover
  runs from ROOT's cron (the demo writes the RAM drive as root), so
  the key lives in /root/.ssh; a simple flock guards the cron-overlap
  the old lock-dir+ps pattern covered. The old mover's trailing CSV cp
  is DROPPED (picods100.md: the ACTIVE demo action writes no CSVs — the
  cp was a no-op; the R0 probe confirms the detections tree holds none).
- callhome.sh gains the Basic header: `-u picods_1XX:<pw>` sourced from
  a root-600 creds file (`/home/pi/picods/nodehome.creds`,
  user:password) — the crons run as root, so the file is readable
  there and unreadable to the pi-level services. Deploy-order safe:
  the header arrives with require_auth still 0; the flip to 1 (R4)
  turns enforcement on without touching the nodes again.
  *(DEPLOYED + creds installed 13:05: the node's forced ping passed
  AS `picods_101` (verified: node_state updated 13:07:49, NO audit
  denial = authenticated). My mid-probe "denied" rows at 13:04-13:06
  were MY OWN probe-harness CR-trap (PowerShell-fed $(sudo cat)
  strings) — the estate code was right; the S062 lesson bites the
  prober too. The 13:05:01 legacy-anon was the creds-not-yet-landed
  window, by design.)*
  *(DEPLOYED to picods101 13:04: temp+mv, CR-stripped, +x, .bak-s0xx
  copies kept. The creds file installs at the cutover proper.)*
- Key generation + install; verified live (2026-10-31 12:53): the
  restricted round-trip WORKS for picods101 — `sudo rsync -a --mkpath`
  with `-i /root/.ssh/id_ed25519_picods` into
  `picods_101@<bank-ip>:reception/…` lands the file (777, root
  owner on the NTFS disk), the shell attempt through the same key is
  REFUSED (`rrsync error: SSH_ORIGINAL_COMMAND does not run rsync`),
  and picods_100's key is generated + installed at ITS cutover (R4).
  Host key comes from the R1 pre-seed (root's known_hosts).
- Deploy doctrine: temp-file+mv, dos2unix, md5 both sides; the OLD mover
  stays in place (not the primary path) until R4 passes.
- The new mover: same RAM-drive scan; ONLY existing, non-empty day dirs
  are pushed (the old one created empty bank dirs on every pass —
  probe-verified litter on the CAM-0101 tree); one flock (the old
  lock-dir + `ps | grep` double-check pattern retires); a LOG FILE this
  time (`mover.log` — the old one's echo-to-cron was discarded, no MTA);
  the trailing CSV cp is DROPPED (picods100.md: the ACTIVE demo action
  writes no CSVs — a no-op; the R0 probe confirms the detections tree
  holds none).
- Cron flip on the NODE follows the spool-byte doctrine: park the old
  mover's line with a marker comment (never delete), add the new line,
  verify the FINAL root-crontab spool with `od -c` (the estate's two
  poisonings), then prove firing via the mover's log stamps.
- LIVE WATCH across a full cycle: burst lands → rsync minute fires →
  reception shapes correct → ingest bite registers/moves → pairs appear
  in the queues → workflow_events stamped with the right actors → and
  the node's callhome `ping` lands in node_state/node_events AS
  `picods_101` with the picods_auth_log `ok` verdict (the new identity
  proven end-to-end on BOTH mechanisms in one watch).
  *(RUN 13:09–13:16: the first cron pass 13:09 fired clean; the 13:12
  pass pushed 46 jpg + 47 xml with the --chmod dirs 777/644 ✓; the
  ingest bite at 13:15 registered 65 pairs, settle=0 live ✓; today's
  CAM-0101 count 71 and climbing; the RAM drive drains — by 13:15 the
  demo's duty cycle ended so the tree emptied naturally. ONE REAL BUG
  FIXED IN THE MOVER (13:11, caught by the doctrine's LIVE WATCH):
  ssh options passed as bare rsync args (`-o BatchMode=yes -i …`) are
  rsync's OWN unrelated legacy flags — silently dropped, ssh fell back
  to a password prompt (auth.log: "Failed password"), rc=255. The
  proven `-e "ssh -o …"` transport form is the fix (the 12:53 probe
  had used it; the script draft hadn't). CRON SPOOL BYTE-VERIFIED
  (0 CR bytes, 0 literal `\r` text, 9 clean lines; the old mover's
  line parked with `# CIFS-PARKED-S0XX-RSYNC` marker + a pre-cutover
  crontab backup at /root/crontab.pre-s0xx-cutover).)*
- md5 spot-check 3 pairs end-to-end (node file vs bank file).
  *(DONE 13:25 — via the honest form available at this hour (the RAM
  push empties its source on success, so a same-name both-sides pair
  no longer exists to compare; the demo was between duty cycles): a
  known test file pushed through the REAL identity + flags arrived
  byte-identical (md5 `13969a99db…` node == bank), and the pipeline's
  93 real pairs (46+47) all transferred + registered with zero
  reception backlog.)*

## R4 — soak + the animals node (picods100) — *(RUN SHORT at the user's call,
## 13:27: he raised picods101's detection thresholds to flood it with
## detections and watched the whole chain drain live — "we can cut the soak
## short - I'm happy". Agreed: the soak's evidence was already in hand.)*

- Soak picods101 **~12 h (user-set) → CUT SHORT (user 13:27)**: the
  soak's real evidence (continuous cron-driven pushes 13:09→13:30 with
  zero failures after the transport fix, 65+37+36-pair ingest bites
  with errors:0, authenticated pings landing, no orphan half-files,
  the RAM drive draining on schedule) was already the 12 h posture's
  content. CAM-0101's queue risk stayed structurally zero throughout.
- picods100 CUTOVER (13:28): crontab flipped — old mover line parked
  with the same `# CIFS-PARKED-S0XX-RSYNC` marker, rsync mover added,
  **the 04:00 reboot line preserved** (the flip's recorded trap),
  spool byte-verified (0 CR, 0 literal `\r`, 36 clean lines);
  LIVE: the 13:29 pass pushed 5 jpg + 6 xml as `picods_100` (the
  animals node's real graph name `network` rides under
  -CLASSIFIED/<day>/<graph>/ as always); both nodes NOW run rsync.
- **Enforcement flip (13:30)**: `nodehome.require_auth=1` (fresh-read,
  no restart) — the anonymous callhome era ENDS. Proven live: a
  headerless POST from the bank → 403 + one audited denial row; BOTH
  nodes' forced pings landed (13:30:17 / 13:30:19) with zero denial
  rows = authenticated, per-node, as designed.
- picods100's rsync pairs register on the same path (13:30 bite: 36
  more, errors 0; CAM-0100 today: 18 and climbing).
- The user's browser QA: new pairs visible on the queues, hover-peek
  boxes correct (the S058-era lesson: visual check, not just 200s);
  the /cameras page (R2b) once built.
  *(DONE 13:39–13:41 — the real-browser pass, screenshots taken:
  /cameras on the live estate (4 camera cards — CAM-0100/CAM-0101/
  CAM-02/CAM-05; node last words pinging in real time; the parse of
  the facts chips; the auth-verdict trail with the honest denied/
  legacy-anon history); a QUEUE DOORWAY clicked (CAM-0100's new 11 →
  /q/new?camera=CAM-0100 rendered the 11 new pairs (13-28→13-31
  arrivals — wood pigeon / mouse / blue tit / dunnock / goldfinch,
  all today's rsync-arrived); the IMAGE VIEWER on one pair's overlay
  (mouse 0.01 box drawn client-side, geometry sane). Two small page
  bugs found + fixed during the pass: the /v/<...> route guess 404'd
  (the viewer is /img/...), and /api/autorepeat returned EMPTY once
  (transient). The nav's cameras link live on all three shells.)*

## R5 — cleanup — *(RUN 13:43–13:45, all steps)*

- **The nodes' CIFS mounts DISABLED (user-set)**: both fstab lines
  commented (parked: `# S0XX-RSYNC-ERA-PARKED` prefix, backup
  `/etc/fstab.rsync-era` per node) + `umount /YERMO` clean on both;
  fstab parse verified (mount -a --fake). picods100's 04:00 reboot
  survivability checked: the root ssh key + known_hosts persist in
  /root, `run-from-cron.sh`'s `mount -a` no-ops harmlessly.
- **callhome's `cifs=` fact → `mover-key=`** (the node's rsync
  credential present): deployed to BOTH nodes (md5-identical
  `0d54f49a…`), pings fired + the new facts verified landing
  (`mover-key=ok` on both).
- **smbd + nmbd STOPPED and disabled (user decision)**: zero active
  SMB sessions at the stop (smbstatus empty — the gate passed); no
  SMB listeners remain (`ss` clean). The estate's last anonymous
  writable surface is gone.
- The kit's README (+ the mover section), `picods100.md`,
  `picods101.md` updated (identity tables, parked-mount notes, the
  retired-CIFS notes); BOTH skills updated (the picods skill's
  handoff section + the imagebank skill's pipeline diagram — the
  rsync-identity era's shape, with the traps recorded).
- session_notes entry (next session: one file in `notes/`, e.g.
  `notes/2026-11-01 S###.md`, plus a ledger row in session_notes.md — this
  is the plan file's
  own trail; the notes entry rides the session wrap-up).

## Rollback (any stage)

- The old mover + its cron line are parked, not deleted (both nodes:
  `# CIFS-PARKED-S0XX-RSYNC` in the spool); the fstab lines likewise
  (`# S0XX-RSYNC-ERA-PARKED`, backups `/etc/fstab.rsync-era`).
  Rollback = remove the parked prefixes + `sudo mount -a` + restart
  smbd — the files the RAM drive holds are source-stable; rsync's
  `--remove-source-files` only removes what it transferred.
- The ingest's settle window is a properties key: restore `20` and
  restart nothing.
- The callhome gate: `nodehome.require_auth=0` reopens the anonymous
  POST immediately (fresh-read) — the nodes never need touching for it.

## S065b — the page adjustments (same session, 14:00–14:45; user-set)

The user DEFERRED S059/S062/S064 (the auto-tools + the staged-pile
review wait for eb_20+eb_21 needs) and set three page changes; all
built, deployed, browser-QA'd (screenshots as the record):

1. **THE PORT-MODE RESPEC** (the /img port control): `fill` now truly
   COVERS the port (`xMidYMid slice` — aspect kept, edges crop; no dead
   dark space on any aspect mismatch); `natural`/`boxes` MEET-fit AND
   the port shrinks to the rendered content (`flex: 0 0 auto` + px
   sizing — the pure-% chain was Chrome-indefinite and silently fell
   back to the svg's ratio: caught by the DOM probe in the QA).
   [viewer.py](../06-deployment/imagebank-dev/viewer.py)
   PORT_MODE_HTML rewritten; [pages.py](../06-deployment/imagebank-dev/pages.py)
   gains the `bx`/`frame-outline` rect classes (the boxes-union selector
   the modes need). test_portmode 21/0.
2. **THE STATUS DOORWAYS on EVERY queue shell**: `api._status_doorways()`
   (`{status: {total, camera}}` per queue's view; the camera filter
   applied where present) + `queue.js renderDoorways()` — the
   cross-queue pill row under every queue title (current status
   highlighted static; a doorway drops the status-specific filters
   honestly, keeps the camera one). Verified on the rejected + assessed
   queues (CAM-02's pills: new 0 / assessed 33,886 / … / junk 371).
3. **THE DASHBOARD SLIM-DOWN**: the rationalised cameras table + the
   picods trail panel retired to /cameras; the dashboard keeps the
   workflow diagram + the two worker cards + a compact node-alive chip
   rail (one chip per node: green/warn per its event, linking /cameras)
   + the estate links card. Verified live (both nodes' freshest pings).
4. **THE FRAME-DECODE FIX** (the user's catch): the legacy
   `<attribute key="frame">` carries TWO conventions by era — the
   2021-2023 harvest XMLs the capture BAND as exact-1920x1080 CORNERS
   (229,136 rows), the 2026 demo XMLs the detection ROI as LTWH squares
   (24,734 rows); the viewer drew both as corners (the off-image
   dashed rect). [frame_geometry.py](../06-deployment/imagebank-dev/frame_geometry.py):
   `is_band_corners()` discriminates on the band signature,
   `normalize_frame_roi()` → LTWH, `sensor_roi_to_image_box()` accepts
   both (the v1 copy synced; the harness assertion updated); the
   viewer's outline draws the ROI in IMAGE px now (live-verified:
   x 602..1402 / y 412..1012 on CAM-0100's 13-28-47 pair = exactly the
   unit arithmetic). The pour's corners-consumption is correct as-is
   (the 2021 assessments ran ON the band; no data migration needed).
   The remaining frame-geometry open item stays the 2026-10-30 one
   (the ScalerCrop/ROI-era mapping precision — display-only).

## Traps carried from S062-prep (the lessons this plan must respect)

- Deploy to the nodes with temp-file+mv + dos2unix + md5 (the CRLF
  doctrine killed whole sessions before).
- Cron changes on the NODES follow the spool-byte rule too (`od -c` the
  spool, worker-log stamps prove firing; no MTA = silent death).
- The ingest's dotfile-skip must cover BOTH rsync's leading-dot temp files
  AND any partial-dir style remnants the flag set might produce.
- Keep the estate's "files never deleted on suspicion" streak: `--remove-source-files`
  only removes after a successful transfer handshake (rsync guarantee;
  still verify with an end-to-end md5 pair at R3).
- **The R2 additions carry their own traps**: nodehome's gate runs in
  the SERVER process — no new imports heavier than the stdlib it
  already carries (auth/nodehome are already imported there); the
  gate must read properties FRESH (the estate's read-per-access law)
  so the R4 flip needs no restart; and the Basic parse must tolerate
  curl's exact header shapes only (no `base64` edge cases beyond
  b64decode-tolerant failure → 401-not-500).
- **Never** put the node passwords in the repo's committed files —
  the node records hold them outside git (the estate's convention),
  and the repo never gains a `nodehome.creds` file.