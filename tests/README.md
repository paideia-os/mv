# tests/

Empty at M1 by design. The correctness matrix — same-dir rename
(O(1) path), cross-dir same-device move (link-unlink atomicity),
cross-device degrade path, TXN-abort mid-move (source restored, no
dest), undo replay correctness across all four cases (same-dir,
cross-dir, cross-dev, cross-user), and the cross-user elevate flow —
lands with `mv.M4-001` through `mv.M4-003` per
`design/tooling/r49-r50-plan.md` §5.7 in paideia-os.

The M1 skeleton wired here (Mv / MvArgv / Rename) is validated
against its own return-code contract at M2, when the userspace
sys_pdxfs_* wrappers land and `rename_same_dir` flips from the
`MV_RN_STUB` sentinel to a real TXN dispatch. Every M1 entry point
already returns its documented `MV_ARG_*` / `MV_RN_STUB` / `MV_RN_*`
sentinels so the M2 tests can diff a live run against the M1
skeleton with a mechanical rule (`if rc == MV_RN_STUB: still on M1;
if rc == 0: M2 live and the TXN committed`).
