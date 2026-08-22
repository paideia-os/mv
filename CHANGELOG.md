# CHANGELOG

All notable changes to `mv` are documented here. Version numbers
follow semver. Release entries name the paideia-os wave that hosts
them (`design/tooling/r49-r50-plan.md` §5.7 in the paideia-os repo)
and the issues each entry closes.

---

## 1.0.0 — 2026-08-22 (R50 Wave 2 close)

First dual-signed release. mv is source-complete + audit-complete
+ undo-complete for the four move shapes (same-dir rename,
cross-directory move, cross-device fallback, cross-user handoff).
Every Pdxfs::* trampoline is an M2 STUB at this release; live
TXN dispatch flips to real syscall trampolines when the R42 PdxFS
v1 substrate lands in the paideia-os kernel. The full QEMU
acceptance smoke (mv a b; pdx-undo; ls) waits on R42 + shell.M5
+ pdx-undo (R51) per `tests/README.md`.

### M5 — signed release
- `mv.M5-001` (#15): dual-signed `manifest.pdxsig` for mv-1.0.0,
  `deps.list` for the five shared-library deps, `doc/mv.pdxdoc`
  for `doc mv`, this CHANGELOG entry, and the RELEASE.md runbook
  for the SHAPE-PENDING → LIVE handoff at signing-bot standup.
- `mv.M5-002` (#16): `MIRROR-PUSH.md` runbook for the
  `pkgs.paideia-os` mirror push and the `v1.0.0` git tag on
  `github.com/paideia-os/mv`.

### M4 — tests + smoke
- `mv.M4-001` (#12): `tests/test_txn_abort.pdx` fixture verifying
  TXN-abort shape (argv-gate rejects, direct abort dispatch,
  happy-path abort/error-leak invariant). 11 codes in band
  0xFFFFEBCx.
- `mv.M4-002` (#13): `tests/test_undo_replay.pdx` fixture with
  three-iteration swap-correctness matrix (aliasing-detection
  pointer pairs 0x1111/0x2222, 0x3333/0x4444, 0xDEAD/0xBEEF).
  8 codes in band 0xFFFFEBDx.
- `mv.M4-003` (#14): `tests/test_smoke.pdx` shape-smoke fixture
  + `tests/README.md` live-harness runbook for the R42-gated
  end-to-end QEMU smoke. 16 codes in band 0xFFFFEBEx.

### M3 — semantic-pipe / audit integration
- `mv.M3-001` (#8): MoveRecord[] schema bind (schema id 0x4D56,
  64-byte record, was_rename + was_cross_device + was_cross_user
  + elevated flags) + emit at Schema::mv_schema_emit.
- `mv.M3-002` (#9): MoveRecord via libpdx-audit at
  Audit::mv_audit_write_move -- audit-first (D3) so an mv that
  cannot log itself aborts the TXN.
- `mv.M3-003` (#10): PdxFS v1 undo record at Undo::mv_undo_write
  (magic 0x4D566E52, replay opcode 0x6D76, swap-source-and-dst).
- `mv.M3-004` (#11): libpdx-elevate hop at
  Elevate::mv_elevate_cross_user for cross-user-boundary dst;
  best-effort with graceful degrade on refusal.

### M2 — core implementation
- `mv.M2-001` (#4): cross-directory same-device move via
  link-unlink threaded through a single TXN (Move::move_dispatch).
- `mv.M2-002` (#5): cross-device move via cp+rm internal fallback
  at Move::move_cross_dev_body, still atomic at commit boundary.
- `mv.M2-003` (#6): was_cross_device diagnostic on --verbose at
  Verbose::mv_verbose_line.
- `mv.M2-004` (#7): signed-inode preservation for same-user
  moves + cross-user graceful degrade at Inode::mv_inode_preserve.

### M1 — design + skeleton
- `mv.M1-001` (#1): repo scaffold (README, LICENSE, caps.decl,
  design/architecture.md), Mv module (KIND ordinal mirrors,
  MV_* return-code band, 16-slot _mv_stats table, mv_reset /
  mv_note / mv_stat).
- `mv.M1-002` (#2): argv surface via libpdx-argv at
  MvArgv::mv_argv_parse (flags -v -i --dry-run, two positionals).
- `mv.M1-003` (#3): first runnable -- Rename::rename_same_dir
  same-dir rename skeleton (SKELETON: argument gating + counter
  dispatch returns MV_RN_STUB; live TXN dispatch lands with R42).

### Upstream substrate at 1.0.0 landing (paideia-os HEAD 2026-08-22)
- `KIND_USER = 0x190` (R48.M1)
- `KIND_IPC_ENDPOINT = 5` (R20b base kind)
- `KIND_PDXFS_FILE = 0x195` (R48b substrate-prep)
- `KIND_PDXFS_TXN = 0x196` (R48b substrate-prep)
- `KIND_ELEVATE_CHANNEL = 0x191` (R48.M7 elevate broker)

### Known deferred work
- Live TXN dispatch: gated on R42 PdxFS v1 substrate in
  paideia-os kernel; every Pdxfs::* trampoline is an M2 STUB
  in this release. Round-2 amendment lands when R42 closes.
- QEMU acceptance smoke: gated on R42 + shell.M5 + pdx-undo
  (R51). Runbook in `tests/README.md`.
- Signing-bot live re-sign: `manifest.pdxsig` in this release
  is SHAPE-PENDING (author + root signature slots reserved,
  bytes marked with the PENDING pattern per `RELEASE.md`).
  Real ML-DSA-65 signatures land when T-INFRA-002 (signing
  bot host + paideia_root_pk policy) reaches standup.
- `pkgs.paideia-os` mirror push: gated on T-INFRA-001 (dual-sign
  package repository infrastructure) reaching standup. Runbook
  in `MIRROR-PUSH.md`; the git tag `v1.0.0` at
  `github.com/paideia-os/mv` is the interim distribution point
  a user can `pkg install --from-source mv` against today.
