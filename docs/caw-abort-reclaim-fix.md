# Fix: reclaim aborted SCSI-atomic-blocked commands on NEXUS_LOSS/abort

Derivative patch on top of upstream SCST `3.11.0-pre`. Module version string:
`3.11.0-pre+caw-abort-reclaim.1` (verify with `modinfo -F version scst.ko`).

## Symptom

Under a heavy concurrent COMPARE AND WRITE (CAW, 0x89) workload across many
initiators (the MXFS 16-node shared-directory `mkdir` storm), the iSCSI target
permanently wedges:

- `iscsi_conn_cleanup` / `close_conn` kthreads pile up in **D state** (33-49
  observed) and never exit.
- Every subsequent command to the hot LBA region blocks forever in
  `EXEC_CHECK_BLOCKING`, even on an otherwise-idle LUN.
- `systemctl stop scst` hangs "deactivating"; the module cannot be unloaded.
- Only recovery is the out-of-tree `scst_unwedge` kmod or a host reboot.

Guest-side downstream effect: the dropped I_T nexus loses its Persistent
Reservation registration, so its next WRITE returns RESERVATION CONFLICT →
`XFS log I/O error -52` → filesystem shutdown.

## Root cause

CAW is `SCST_SCSI_ATOMIC` (`scst_lib.c`, opcode table entry for 0x89). It does
**not** block the whole device; it blocks only LBA-*overlapping* commands via the
SCSI-atomic mechanism:

- `scst_check_scsi_atomicity()` (`scst_targ.c`) parks an overlapping command on
  `dev->dev_exec_cmd_list` with `scsi_atomic_blockers > 0`, and records the
  back-reference in each blocker's `scsi_atomic_blocked_cmds[]` array.
- Such a parked command is reactivated **only** by
  `scst_check_unblock_scsi_atomic_cmds()` when its blocker completes.

The teardown/abort path `__scst_unblock_aborted_cmds()` (`scst_targ.c`) walks
only `dev->blocked_cmd_list` and `order_data->deferred_cmd_list`. It does **not**
walk `dev_exec_cmd_list`, and nothing else does on teardown.

So when ABORT_TASK → LUN_RESET → NEXUS_LOSS aborts a command that is parked
atomic-blocked, the command is flagged `SCST_CMD_ABORTED` but is **never
re-queued**. Its only wakeup is its blocker completing. If the blocker is itself
parked/aborted in the same mass-teardown event, the command is orphaned forever.

The orphaned command keeps its `tgt_dev`/connection refcounts, so:

- `close_conn()` spins forever in its `while (atomic_read(&conn->conn_ref_cnt)
  != 0)` msleep loop (`iscsi-scst/kernel/nthread.c`) — the D-state kthread leak.
- The device-side blocking state it pins never drains.

(On an EXTRACHECKS build the alternative — freeing it while still on
`dev_exec_cmd_list` — would trip `EXTRACHECKS_BUG_ON(cmd->on_dev_exec_list)` in
`scst_free_cmd()`, i.e. a panic, not a silent leak. So the observed behaviour is
a parked-alive orphan, consistent with this diagnosis.)

## Fix

`scst/src/scst_targ.c`:

1. New helper `__scst_check_unblock_aborted_scsi_atomic_cmd()` — for an aborted
   command parked with `scsi_atomic_blockers > 0`, detaches it from every
   blocker's `scsi_atomic_blocked_cmds[]` array (preserving the array invariant
   `non-NULL <=> count > 0`, freeing the array when it empties so a later
   `scst_check_unblock_scsi_atomic_cmds()` cannot dereference the freed command),
   then re-activates it onto its `cmd_threads->active_cmd_list`.

2. `__scst_unblock_aborted_cmds()` — after the existing `blocked_cmd_list` walk
   and inside the same `dev_lock` + IRQ-disabled region, add a walk of
   `dev_exec_cmd_list` (same `tgt`/`sess` filter) that calls the new helper.

Once reactivated, the aborted command runs normally: `__scst_check_blocked_dev()`
returns immediately for `SCST_CMD_ABORTED`, so it does not re-block, proceeds to
abnormal completion, and `scst_check_unblock_dev()` cleans up all the device
state it held (including releasing any commands it was itself blocking).

It is safe to force-reactivate an aborted atomic-blocked command before its
blocker completes because an aborted command never executes against the backend —
it only finishes.

## Scope / correctness notes

- All `scsi_atomic_*` fields are protected by `dev->dev_lock`; the new code runs
  under that lock, matching the existing producers/consumers.
- The wedge is LBA-region-scoped (to the orphan's overlap range). It presents as
  a whole-device wedge for the MXFS workload because that workload concentrates
  CAW on a single metadata-lock LBA.
- Not reproducible by synthetic session teardown on a local fileio backend,
  because there the blocker always completes and blockees drain transiently. A
  permanent orphan needs a blocker that never completes (a parked chain rooted on
  a never-draining device block under real PR + CAW interleaving).

## Not this fix (related but separate)

- Upstream issue #328 "out-of-bounds read in `scst_cmd_overlap_atomic`" — a
  buffer over-read in the UNMAP-descriptor overlap path; different defect.
- The doc that motivated this (`/src/mxfs/SCST_PROBLEM.md`) described CAW and
  WRITE SAME as "strictly-serialized"; that is incorrect (CAW is `SCSI_ATOMIC`,
  WRITE SAME 0x41/0x93 is plain `WRITE_MEDIUM`), and its claimed CAW↔READ A↔B
  blocking cycle is impossible by construction (atomic blocking is strictly
  arrival-ordered: new-blocked-by-old, a DAG).

## How to test (MXFS session)

1. Build: `cd /src/scst/scst && make`
2. Install + reload SCST (requires the current wedge to be cleared first), then
   confirm: `modinfo -F version /lib/modules/$(uname -r)/extra/scst.ko`
   → `3.11.0-pre+caw-abort-reclaim.1`.
3. Run the reproducer:
   `bash tests/criteria/zero_silent_loss.sh --iters 3 --dpn 100 --mode 1`
4. Pass = the 16-node CAW storm completes with no `iscsi_conn_cleanup` D-state
   threads, no permanent `EXEC_CHECK_BLOCKING` blocks, and no reservation
   conflicts on the guests.
