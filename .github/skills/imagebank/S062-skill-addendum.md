# S062-prep addendum — the deploy/reliability lessons (draft for the next skill refresh)

*Not yet merged into SKILL.md — the next session's skill review decides the
wording; kept alongside the skill while they're fresh (2026-10-31). Each
lesson below was proven the hard way this session; the picods ones belong
to that skill's review too.*

## The cron-spool lessons (bit twice across S056 + S062; the 36 h silent outage)

1. **`crontab -l` is not proof.** It round-trips the same read path the
   poison travelled in. The truth is the **spool file**:
   `sudo -n od -c /var/spool/cron/crontabs/<user>` — and only a byte-level
   view (`od -c`, `cat -A`) distinguishes REAL CR bytes from
   **literal two-character `\r` TEXT**. Both have happened in this estate
   (the 2026-10-31 outage was literal `\r` TEXT on every line, from an old
   `tr -d '\r'` whose argument survived into the file).
2. **"Cron silence = cron health" is a lie.** Cron exec-failures emit NOTHING
   on a host with no MTA ("No MTA installed, discarding output") — a
   36-hour worker outage produced zero syslogs but stale worker logs. The
   health check: worker log STAMPS advancing (`ls -l --time-style=full-iso`)
   + `journalctl -t CRON` shows the `(pi) CMD (...)` lines clean (a trailing
   `^M` in the CMD = poison still live).
3. **Any crontab rewrite is verified against the spool bytes, not by
   re-running `crontab -l`.** And the fix for BOTH poison kinds in one
   pass: read-spool → `tr -d '\r'` (real CRs) AND a text-strip for literal
   `\r` → `crontab /tmp/cron.fixed`. A `cat -A` glance mid-fix can race an
   intermediate write; verify the FINAL spool with od.
4. PowerShell-side stdin (`ssh ... "crontab -"` fed from a Windows python)
   is itself a CRLF injection path — the estate pattern that DOES work:
   base64 the script, run it entirely remote-side (`echo <b64> | base64 -d
   | bash`), keep every byte-decision on the Pi.

## Deploy mechanics (the recurring trio, now in pi_deploy)

- **scp + mv loses the exec bit on `.sh`** — cron's direct exec dies 126,
  silently (no MTA). pi_deploy now chmods `.sh` in the replace loop; any
  future deploy tool gets the same rule: **mode changes belong in the
  deploy manifest, not as an afterthought.**
- Root-owned `__pycache__` (the server runs as root) still needs
  `sudo rm -rf` even for plain tree removal (re-bitten during the
  decommission).
- New load-bearing files (`postprocess_ssd.py`, `frame_geometry.py`, the
  cron wrappers) must ride the deploy manifest or they silently keep a
  stale copy on the next deploy — a drift the next session inherits.

## Code-tree/data-layout lessons (the publish-root migration)

- **Production roots never belong under the code tree.** `scratch_export`'s
  default `os.path.join(HERE, "dev-bank", "export")` is exactly how 661
  published files ended up on the SD card. The standing audit (add once,
  run always): after every deploy, resolve every scratch_* root from the
  LIVE properties, assert it exists + is on the expected filesystem, and
  assert `ground_truth.batch_dir`/`xml_docs.path` sample-verify against
  disk.
- **A cross-filesystem move is not atomic** — and `cp -a` to NTFS "dies"
  on preserve-permissions noise AFTER the content landed (rc≠0, `set -e`
  aborts). The doctrine that worked: `cp` (content), then **count +
  per-file md5 both sides = proof**, then delete; never delete on count
  alone, never trust one pass of anything.
- **Properties are read fresh per access** — scratch-root flips need no
  server restart (verify with an on-Pi python probe importing properties +
  calling `scratch_dir()`).
- **A live-tree grep beats `ps` for "who depends on this"**: the V1-v1-dep
  discovery (`/home/pi/harvest/venv` baked into v2's assess cron; an
  import from the helper mirror) came from grepping crontabs + code, not
  from process watching.

## Decommission discipline (the order that worked)

Inventory consumers (crontabs + code greps) → migrate the consumers
(venv, mirror imports, helper files) → prove the replacement runs (a
CRON-driven pass, not a manual one) → drop the cron lines → park (never
trust) the keep-alive → tar + `gzip -t` + md5 → THEN remove. Anything
removed gets archived first, on the big disk, with recorded checksums.

## The PSU-verification recipe (repeatable)

`dmesg` undervoltage count (whole boot) × `vcgencmd get_throttled` ×
`pmic_read_adc EXT5V_V`, with a **controlled burn**: park the target
service's keep-alive with a UNIQUE marker line (never delete it), kill the
service, prove it stays dead across one cron minute, burn (90 s all-core
was enough), restore-**last** (in a finally), verify revival with the
answer-200 loop + byte-identical crontab. "Zero flags at idle" is not a
verdict; "zero under load" is.