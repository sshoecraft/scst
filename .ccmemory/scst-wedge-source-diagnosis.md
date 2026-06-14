---
name: scst-wedge-source-diagnosis
description: Source-grounded SCST wedge diagnosis: abort-reclaim gap for atomic-blocked cmds; SCST_PROBLEM.md doc corrections
metadata:
  type: project
---

Source-level diagnosis done in /src/scst (SCST 3.11.0-pre) against the wedge described in
/src/mxfs/SCST_PROBLEM.md. Verified by reading scst_lib.c + scst_targ.c, not a live host.

## Corrections to SCST_PROBLEM.md (its mechanism claims are wrong)
- **CAW (0x89) is NOT serialized.** opcode table scst_lib.c:1269-1274 = SCST_LOCAL_CMD |
  SCST_WRITE_MEDIUM | **SCST_SCSI_ATOMIC** only. It does NOT block the whole device or
  "drain all outstanding". It blocks only LBA-*overlapping* cmds via the SCSI-atomic
  mechanism (scsi_atomic_blockers / scsi_atomic_blocked_cmds[] / dev_scsi_atomic_cmd_active).
- **WRITE SAME (0x41 scst_lib.c:1001-1007, 0x93 :1343-1349) is NOT serialized/atomic** —
  plain SCST_WRITE_MEDIUM. No special blocking at all. Doc's "WRITE SAME strictly-serialized"
  is false.
- So doc defect #1/#3 ("CAW & WRITE SAME run strictly-serialized → block+drain device →
  exceeds 60s timeout") rests on a wrong premise. The device-block that parks READs comes
  from **PERSISTENT RESERVE OUT (PR churn)** which IS SCST_SERIALIZED (block_count/unblock_dev).
- **Doc defect #2 (CAW↔READ A↔B cycle) is impossible by construction.** SCSI-atomic blocking
  is strictly arrival-ordered: scst_check_scsi_atomicity (scst_targ.c:259) only ever makes a
  NEWLY-arriving cmd block on OLDER cmds already on dev_exec_cmd_list. New-blocked-by-old is a
  DAG; you cannot get both CAW-blocked-by-READ and READ-blocked-by-CAW for the same pair. The
  gdb walk on the wedged host almost certainly conflated CAW-blocked-by-READ#1 with
  CAW-blocking-READ#2. (Memory already flagged that analysis as imprecise.)

## The real defect (one fixable SCST bug, matches all symptoms)
**Abort-reclaim gap for SCSI-atomic-blocked commands.**
- __scst_unblock_aborted_cmds (scst_targ.c:5357) walks ONLY dev->blocked_cmd_list and
  order_data->deferred_cmd_list. It does **NOT** walk dev->dev_exec_cmd_list, and nothing
  else does on teardown (dev_exec_cmd_list is referenced only in the blocking code +
  init/BUG_ON at scst_lib.c:4317/4382).
- A cmd parked atomic-blocked (scsi_atomic_blockers>0, on_dev_exec_list=1, !sent_for_exec)
  sits ONLY on dev_exec_cmd_list. If aborted (ABORT_TASK→LUN_RESET→NEXUS_LOSS), it is marked
  SCST_CMD_ABORTED but is NEVER put back on the active list by the abort. Its only reactivation
  path is scst_check_unblock_scsi_atomic_cmds when its blocker completes.
- In scst_abort_cmd (scst_targ.c:5118) a parked (!sent_for_exec) cmd takes finish_counted →
  mcmd->cmd_finish_wait_count++ ("deferring ABORT" log is scst_targ.c:5245). The TM never
  completes until the cmd finishes; the cmd never finishes until its blocker completes.
- The orphaned cmd holds its SCST cmd ref → iscsi conn_ref_cnt → close_conn (nthread.c:222,
  the iscsi_conn_cleanup kthread) spins forever in its `while (conn_ref_cnt != 0)` msleep loop
  at nthread.c:286 = the observed 33-49 D-state kthreads.
- Freeing-while-still-on-list can't silently leak: scst_free_cmd EXTRACHECKS_BUG_ON(on_dev_exec_list)
  at scst_lib.c:7849 would PANIC. Build has EXTRACHECKS on → it WEDGES (parked-alive), consistent.
- Scope: wedge is **LBA-region-scoped** to the orphan's overlap range (every later CAW/READ to
  that range queues behind it forever). Since the mxfs mkdir storm hammers one CAW metadata-lock
  LBA, region-scoped == effectively total for this workload. (dev_scsi_atomic_cmd_active stuck >0
  — used only at scst_targ.c:373 — does NOT whole-device wedge; doc's "even on idle LUN" is an
  over-generalization.)

## Why synthetic teardown repro never triggered it (prior memory)
force_close/RST tests used a local fileio backend where the blocker ALWAYS completes, so atomic
blockees were always reactivated → transient, not permanent. A PERMANENT orphan needs a blocker
that never completes (a parked chain rooted on a never-draining block_count/PR ext-block), which
the real PR+CAW interleaving supplies. Pin the exact root chain from a LIVE wedge via
/src/mxfs/scripts/scst_atomic_wedge_diag.py.

## Fix direction
Make NEXUS_LOSS/abort reclaim atomic-parked cmds: in __scst_unblock_aborted_cmds also walk
dev_exec_cmd_list and force-reactivate ABORTED cmds with scsi_atomic_blockers>0 (safe: aborted
cmd won't execute, only finish). Must also drop the back-references in each blocker's
scsi_atomic_blocked_cmds[] so scst_check_unblock_scsi_atomic_cmds doesn't later touch a freed
acmd. Subtle — design before coding. Related: [[scst-caw-leak-investigation]]
</body>
