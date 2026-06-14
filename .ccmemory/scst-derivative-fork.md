---
name: scst-derivative-fork
description: Derivative SCST fork + branch carrying the CAW abort-reclaim wedge fix
metadata:
  type: reference
---

Local /src/scst is a clone of upstream **SCST-project/scst** (official repo, remote
`origin`). As of 2026-06-13 fast-forwarded to upstream master `83745c0a2`.

Our derivative fork: **https://github.com/sshoecraft/scst** (remote `fork`).
Fix branch: **caw-abort-reclaim** (commit `488704520`), pushed to the fork.
Module version marker: `3.11.0-pre+caw-abort-reclaim.1` (set via
SCST_VERSION_STRING_SUFFIX in scst/include/scst_const.h; check with
`modinfo -F version scst.ko`).

The branch carries the fix for the atomic-blocked-cmd abort-reclaim wedge - see
[[scst-wedge-source-diagnosis]] for the root cause, and docs/caw-abort-reclaim-fix.md
in the tree. Built clean on the upstream base; MXFS session to test via
`tests/criteria/zero_silent_loss.sh --iters 3 --dpn 100 --mode 1`.

Note: .ccmemory/ was intentionally NOT pushed to the public fork (internal notes /
host names). Backup of the raw diff also at /tmp/scst-caw-abort-reclaim.patch.
</body>
