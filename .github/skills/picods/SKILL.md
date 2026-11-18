---
name: picods
description: 'PICODS wildlife-camera nodes (picods100/picods101/picods102, Raspberry Pi Zero 2 W + IMX500 AI camera): the sensor-mode/frame-hierarchy geometry, box-mapping invariants, demo + kit architecture, deploy doctrine (temp-file+mv, dos2unix, md5, cron revival), debugging with the live pages, and the scripted commissioning ladder. Use when: working on picods nodes or the kit under 06-deployment/picods, IMX500 coordinates/boxes/sensor modes, detection/capture frames, mode_scale, no-detections or wrong-box-geometry bugs, thresholds/config pages, deploying to or SSHing the nodes, service.log, cron supervision, commissioning/rebuilding a node.'
---

# PICODS camera nodes

## LAWS THAT NEVER BEND (all work in this estate)

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

Three live nodes (picods100 = animals model, picods101 = COCO test bench,
picods102 = garden camera), a master kit in the repo, and an imagebank
ingest pipeline. This skill holds the slow-moving structural truth; for
current state ALWAYS read the authoritative docs named below.

## Authoritative state (read these, do not trust memory)

- **Current values / recent work**: the session index
  `documents/session_notes.md` (one file per session under `notes/`,
  named `YYYY-MM-DD S###[.suffix].md`)
- **Kit layout + hot keys + version marker**: `06-deployment/picods/kit/README.md`
- **Per-node facts** (IP, SSH, model, quirks, backups on the Pi):
  `06-deployment/picods/picods100.md`, `picods101.md`, `picods102.md`
- **Commissioning a node / the golden image**: `06-deployment/picods/commission-new-node.md`
  (run-book, validated) + `06-deployment/picods/commissioning/` (the scripts:
  install-base.sh → set-node-identity.sh → register-node-on-bank.sh →
  set-node-identity-finalize.sh; proven end-to-end on picods101, S073
  2026-11-12). Verify with `commissioning/verify-base.sh`-style ladders —
  the ladder caught a clone hazard (the kit template ships picods100's
  identity) and three script bugs on their first live run.
- **Node resilience / outage review**: `06-deployment/picods/node-resilience-review.md`
  (runbook, S074) — failure-pattern diagnosis from the bank's
  `node_state`/`node_events` trail FIRST, then DHCP/SD/power/powersave/
  journald checks. Start outage triage here.
- Never commit secrets; node records hold access details.

## Topology

| Host | Role | Serves |
|---|---|---|
| picods100, picods101, picods102 | camera nodes (Pi Zero 2 W + IMX500 4056x3040) | demo (detections) + `picods_server.py` on :8080 |
| imagebank | Pi 5, the bank disks + ingest/assess workers | `ib_server.py` on :80 |
| Windows box | repo, training, deploy source | - |

Node kit on the Pi: `/home/pi/picods/` = `lib/` (demo `imx500_object_detection_demo.py`), `server/`, `cam.properties` (hot, root-owned: append with `sudo tee -a`), `service.properties` (SUSPENDED etc.), `label-thresholds.json` (per-class gates).

Cron (root, every minute): `run-from-cron.sh` (supervisor: revives the demo if down, archives/rotates service.log on a committed start) and `move-calendar-files-rsync.sh` (the S0XX mover: RAM drive pairs -> imagebank's reception over rsync AS the node's own account; see "imagebank handoff" below — the CIFS-era mover is parked). Server keep-alive revives a killed HTTP server within a minute.

## The frame hierarchy and its coordinate systems (the invariant set)

```
full sensor 4056x3040  -- SENSOR_MODE (4056x3040 | 2028x1520), restart to apply
  +-- CAPTURE_FRAME     16:9 band in SENSOR px = what image_raw.jpg shows
       +-- DETECTION_FRAME  square in SENSOR px = the IMX500 inference ROI
```

Rules that have each caused a real bug when violated:

1. **Frames are full-sensor px everywhere** - cam.properties, the UI,
   `set_inference_roi_abs` (mode-independent). The UI speaks sensor px.
2. **mode_scale (mode/4056x3040; 1.0 full, 0.5 binned) applies exactly at
   the image boundary**: aperture crop corners, the scaler_crop band
   published for box mapping, the full-view drag math. The binned 2028
   main IS the full sensor downscaled 2x, not a crop (ScalerCrop stays
   (0,0,4056,3040) at both modes - probe-verified).
3. **Box mapping order** (firmware emits ROI-RELATIVE y0,x0,y1,x1
   fractions): fractions -> SENSOR px via DETECTION_FRAME -> mode_scale
   -> image px via the capture band -> aperture px. Missing the
   mode_scale step collapses every box to the bottom-right 1x1 corner
   (area=1, rejected by DETECTION_MIN_AREA = zero detections at ANY
   threshold; the S029 bug, invisible at full mode).
4. **Sidecar boxes are aperture-image px** (1920x1080), NOT sensor px;
   boxes drawn client-side from latest.xml.
5. Wrong rule 2 black-pads the aperture image's bottom (the S028 bug);
   wrong rule 3 kills detections silently.

Hot keys live in cam.properties; the demo re-reads them every cycle
(~1.5 s). SENSOR_MODE is restart-required (config page's suspend ->
verify -> resume handshake). Per-class thresholds live in
label-thresholds.json (the live page's panel; label=all = slave mode).

## Deploy doctrine (both nodes; breaks have all been violations)

1. Deploy scripts with **temp-file + mv** (atomic), never scp-over-
   in-place - a running bash re-reads by byte offset.
2. **dos2unix** everything from the Windows working tree (CRLF lurks in
   fresh files).
3. **md5-verify** both sides after every deploy (CRLF: compare
   LF-stripped hashes).
4. Servers/demos revive via cron within a minute of a kill; `kill-
   http-server.sh` greps picods_server.py; expect the pkill-then-wait
   pattern.
5. Verify on the live pages (the nodes' :8080): stats chips (frames/
   dets ticking), LOG_REJECTIONS for gate rejections, boxes on the
   image, not just HTTP 200.
6. The kit in the repo is the master; deployed copies drift at your
   peril - the version marker (pages.VERSION, v<date>.<serial>) bumps
   per release.

## Debugging doctrine

Probe-first, one variable at a time (see kit `diagnostics/` - probe3
no-model, probe4 with-model, probe5 = the demo's exact sequence). The
live pages are the evidence surface: stats.json (frames/dets/light/
threshold), latest.xml (boxes + frame attributes + scaler_crop),
service.log (Detection lines with box= + raw fractions; area=1 +
box=[1919,1079,1920,1080] = the mode_scale bug). Rejections with
LOG_REJECTIONS=True show which gate fired. Light gate: below
MIN_LIGHT_LEVEL the demo exits (night); cron restarts at dawn.

## imagebank handoff (see the imagebank skill for the ingest side)

**S0XX (2026-10-31): the CIFS era is over — the nodes push via
rsync-over-ssh AS THEIR OWN ACCOUNTS.** Root's cron runs
`move-calendar-files-rsync.sh` (the CIFS mover's line is parked with a
`# CIFS-PARKED-S0XX-RSYNC` marker, `/etc/fstab`'s CIFS line parked +
unmounted, imagebank's smbd STOPPED). The identity: the node's private
key `/root/.ssh/id_ed25519_picods` + `/home/pi/picods/nodehome.creds`
(both root-only); the bank side: account `picods_1XX` (one per node,
`from=<node ip>`, forced `command=rrsync /yermo/animals-count`) +
the same-named users row (role `camera`) for the now-authenticated
callhome. TRAPS: remote rsync paths are RELATIVE to the rrsync root
(`reception/...`, never a leading slash — an absolute path silently
nests); ssh options ride `-e "ssh -o …"`, NEVER bare rsync args (rsync's
own `-o/-i` are unrelated legacy flags — they get dropped, ssh falls to
a password prompt, rc=255; the log shows it, the next pass self-heals).
The mover pushes pairs to `picods_1XX@imagebank:reception/CAM-0XXX/<day>/`
(+ -CLASSIFIED/<day>/<graph>/); imagebank's ingest registers + moves
them; the NAS purge/quarantine areas and node records list what is
excluded. Rollback ANY stage: uncomment the parked cron + fstab lines
(both backed up: /root/crontab.pre-s0xx-cutover, /etc/fstab.rsync-era),
restart smbd, restore the settle key — nothing was deleted.