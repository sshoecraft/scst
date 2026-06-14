---
name: scst-caw-leak-investigation
description: "SCST CAW/teardown command-blocking leak — corrected diagnosis, repro status, tooling"
metadata: 
  node_type: memory
  type: project
  originSessionId: 98012e9b-9a21-4040-bd7a-859ad55957ea
---

Investigating a command-blocking leak that wedges clyde's iSCSI target stack under
mass initiator teardown with CAW in flight (host: clyde, SCST 3.11.0-pre at /src/scst,
build 9319, DEBUG/EXTRACHECKS/TRACING, module srcversion 1CF27B5E18657A09E7A0383).

**Correction to the original hypothesis:** CAW (COMPARE AND WRITE, 0x89) is
`SCST_SCSI_ATOMIC` only (scst_lib.c:1269-1274) — it does NOT touch `block_count` or
the strictly-serialized path. CAW blocks only LBA-*overlapping* commands via the
SCSI-atomic mechanism (`dev_scsi_atomic_cmd_active`, `scsi_atomic_blockers`,
`scsi_atomic_blocked_cmds[]`); these are NOT on `blocked_cmd_list` and are NOT walked
by `__scst_unblock_aborted_cmds`. PR IN/OUT (0x5e/0x5f) and RESERVE are `SCST_SERIALIZED`
→ they use `block_count`/`unblock_dev`. So the prompt's "block_count leak" framing (made
on a wedged host with non-working drgn) is likely imprecise; the CAW leak counter is
the atomic one. The wedge is a command left **parked (not freed)** with its unblock
trigger lost — freeing leaves a guard (scst_free_cmd EXTRACHECKS_BUG_ON on
`on_dev_exec_list`) that would panic, and production wedges rather than panics.

**Proven so far:** orderly NEXUS_LOSS teardown (iscsi-scst per-session `force_close`
sysfs attr) does NOT leak any counter — tested extensively on isolated scratch targets
(pure CAW; mixed read+CAW; full production-shape CAW+PR-churn+reads; and with a
dm-delay 6ms-latency backend). force_close and a real TCP RST both call
`scst_rx_mgmt_fn(SCST_NEXUS_LOSS_SESS)` (nthread.c close_conn:261), but the
connection-ERROR path additionally runs `req_cmnd_release_force` /
`free_orphaned_pending_commands`.

**UPDATE (sess this run): the TCP-RST connection-error path also does NOT reproduce.**
Minimal-footprint repro (3 sessions to disk2, distinct conns via ephemeral-port diff,
`ss -K` abrupt RST mid-flight under CAW+PR+read load, 30 rounds) drained clean every
time — `close_conn:241` fires but still funnels to the same `NEXUS_LOSS_SESS` TM abort
(nthread.c:261), so the SCST-side teardown is identical to force_close. CONCLUSION:
the leak is NOT synthetically reproducible on a scratch fileio device with either
teardown path. The real trigger is more specific — likely real mxfs CAW/PR interleaving
+ real LVM latency on disk1 + simultaneous loss of 16 full initiator stacks (virsh
destroy), not a loopback RST. RECOMMENDED next move: arm the working tooling to capture
state LIVE on the next natural wedge (which counter actually leaks: block_count vs
dev_scsi_atomic_cmd_active vs the scsi_atomic_blockers chain) — that pins the path that
static analysis + synthetic repro could not.

**Tooling (persisted in /src/mxfs/scripts/):**
- `scst_block_diag.py` — read-only live per-device block state via /proc/kcore. drgn
  can't relocate scst.ko without vmlinux debuginfo, so it uses drgn only as a kcore VA
  reader + struct offsets pulled live from scst.ko DWARF via gdb. Run:
  `sudo PYTHONPATH=/home/steve/.local/lib/python3.12/site-packages python3 scst_block_diag.py`
- `scst_mon.py <dev> [seconds]` — fast (~100k samples/s) continuous monitor of one
  device's block_count / dev_scsi_atomic_cmd_active / on_dev_cmd_count / sscw /
  blocked_cmds; flags a persistent leak (counter held nonzero with on_dev=0 >1.5s).
- `scst_caw_repro.sh` — N-session mass-teardown repro (env: T, MONDEV, IFP, INIP).
- `scst_delay_setup.sh` — dm-delay-backed high-latency scratch target (note: multipathd
  auto-grabs multiple same-LUN sessions into an mpath device — flush with `multipath -f`).

**Isolation rules used:** disk2 (fileio) and delaydisk are separate scst_devices from
production disk1; clyde itself holds 17 loopback sessions to disk1* — never RST port
3260 broadly. The cluster (16 test VMs) runs on disk1; coordinate before any module reload.
Related: [[[scst-host-clyde]]]
