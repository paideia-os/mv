# CHANGELOG

All notable changes to `mv` are documented here. Version numbers
follow semver. Release entries name the paideia-os wave that hosts
them (`design/tooling/r49-r50-plan.md` §5.7 in the paideia-os repo)
and the issues each entry closes.

---

## 1.2.0 — 2026-09-13 (Wave-C consolidation)

Real-body entry point, POSIX multi-source form, cwd-relative operand
resolve, and `-i` interactive confirm.  Retires the last M1-001 STUB
reachable from `_start`: the process entry now dispatches through
real syscalls end-to-end.

- `mv.v1.2.0-A` (mv#30 flagship real-body extraction).  New
  `src/entry.pdx` (`Entry` module) provides the process `_start`
  entry frame.  Reads argc/argv from the frozen execve ABI
  (rdi=argc, rsi=argv per `paideia-os design/user/execve-abi.md`),
  parses the mv flag surface inline (short + long forms of `-i`,
  `-v`, `-f`, `--dry-run`, `--verbose`, `--force`, `--interactive`),
  and dispatches through `Rename::rename_same_dir` for every source.
  Every helper on the hot path calls a real kernel syscall
  (`sys_getcwd` SC+ ID 86, `sys_stat` SC+ ID 77, `sys_read` SC+
  ID 0, `sys_write` SC+ ID 1, `sys_rename` SC+ ID 82, `sys_unlink`
  SC+ ID 81, `sys_exit` SC+ ID 60); the M1-001 stubbed
  Pdxfs::pdxfs_txn_* substrate is unreachable from `_start` at
  v1.2.0.
- `mv.ENH-002` (mv#18): `_start` entry frame + I4 exit-code
  mapping.  0 = all requested moves succeeded (or user-declined via
  `-i`); 1 = missing operand / usage; 2 = one-or-more-moves failed
  (i/o, target not a directory, resolved-path overflow, unwritten
  source).  Every terminal path in `_start` ends in `sys_exit`;
  `@no_frame` reflects the never-return shape.
- `mv.ENH-003` (mv#19): userspace cwd-relative operand resolution.
  Both source and destination arguments are resolved against the
  task cwd via `sys_getcwd` (SC+ ID 86, R86.M1-003 #1956 kernel
  body at `src/kernel/core/syscall/sys_getcwd.pdx`) BEFORE the
  `sys_rename` call site.  Matches the mkdir v1.1-B (#25) and
  cp v1.1-B (#21) precedents: absolute paths pass through
  unchanged; a cwd of exactly `/` skips the separator to avoid
  `//foo`; overflow (>= 511 bytes) folds to -ENAMETOOLONG.  Two
  new 512-byte scratches (`entry_src_resolved`,
  `entry_dst_resolved`) plus one 256-byte cwd scratch
  (`entry_cwd_scratch`).
- `mv.ENH-005` (mv#21): `-i` interactive confirm via KIND_TTY.
  `Entry::entry_confirm(dst)` emits `mv: overwrite '<dst>'? ` to
  fd 2 (stderr) then reads a single byte from fd 0 via
  `sys_read` (KIND_TTY read).  Only 'y' (0x79) or 'Y' (0x59)
  proceeds; every other byte (including EOF, error, whitespace)
  causes `entry_move_one` to skip that source cleanly (skip is
  not counted as an error -- had_error stays 0).  The `-i` /
  `--interactive` long form is recognised by the inline flag
  scan in `_start`.
- `mv.ENH-011` (mv#27): multi-source form `mv f1 f2 ... dir/`.
  `_start` now accepts `pos_count >= 2` (previously
  `MvArgv::mv_argv_parse` pinned it at exactly 2).  When
  `pos_count > 2` the destination MUST resolve to a directory
  (via `Entry::entry_probe_dir` -- sys_stat + S_IFDIR mask check,
  with a trailing-'/' short-circuit for callers that spell
  directory intent explicitly); a non-directory destination emits
  `mv: target is not a directory (need dir/)` on stderr and exits
  2.  Per source, `Entry::entry_move_multi` composes
  `<dst_dir>/<basename(src)>` into a new 512-byte scratch
  (`entry_dst_composed`) via `Entry::entry_compose_dir_slash_base`
  (basename walker + slash-idempotent join, same shape as cp
  `copy_join_dir_basename`) and then delegates to
  `Entry::entry_move_one`.  A per-source failure sets `rbx = 1`
  and continues the loop; a subsequent success does not clear
  the flag (POSIX "any failure -> exit 2" convention).
- `PDX_TOOL_NAME` extern (`"mv\0"`) added per the Wave-6
  libpdx-argv v1.1.3 contract.  Stable NUL-terminated .rodata
  byte array consumed by libpdx-argv's `diag_render` and by any
  future `mv --help` renderer wanting to spell the tool's own
  name without hard-coding it at the call site.  Length is *not*
  exported -- consumers strlen the array.
- Manifest bumped to 1.2.0.  `src/entry.pdx` added to the
  `[artifacts]` stanza; `MoveRecord` schema stanza + `[undo]`
  magic unchanged (Move::move_dispatch remains the substrate for
  the pending R42 TXN-mediated path, which v1.2.0 does not
  exercise from `_start`).

---

## Unreleased (Enhancement v1.x)

- `mv.ENH-004` (#20): destination-clobber guard. `Move::move_dispatch`
  (`src/move.pdx`) now probes the destination path via a new
  `Pdxfs::pdxfs_dst_exists` trampoline BEFORE opening the PdxFS TXN
  and refuses the move with `MV_MV_DST_EXISTS` (0xFFFFEB3E) unless
  the caller passed `-f` / `--force`. The refuse path emits the
  stderr diagnostic `mv: cannot move '<src>' to '<dst>': destination
  exists (use -f to overwrite)\n`, populates + emits the MoveRecord
  with `dst_existed = 1`, bumps the new `MV_ST_DEST_REFUSED` (slot 16)
  and `MV_ST_ERRORS` counters, and returns without ever opening a
  TXN -- the source dirent stays intact and the destination is byte-
  identical to its prior contents. The `-f`-consented overwrite path
  still sets `mv_move_dst_existed = 1` so the emitted MoveRecord
  distinguishes a fresh-create from a clobber-consented move. New
  `MvArgv::mv_argv_force` slot + `-f` / `--force` flag whitelist
  entry in `src/argv.pdx` (long form byte-walks against `"force\0"`).
  New `Move::mv_move_dst_existed` .bss slot cleared in `move_reset`.
  New `Move::mv_dst_exists_diag(src, dst)` best-effort emitter uses
  five `pdxfs_write` calls with `Rename::rn_strlen` for the path
  bodies. MoveRecord widened from 80 to 88 bytes with `dst_existed`
  at offset 80; schema magic bumps to `0xFFFFEB5000000002`
  (SCHEMA_LABEL to `"MoveRecord@0.2"`); `Mv::_mv_stats` widened to
  17 live slots (24 reserved) with `MV_ST_MAX` = 17. New
  `tests/test_dst_exists.pdx` fixture (band `0xFFFFEBFx`, entry
  `TestDstExists::test_dst_exists_run`) witnesses the guard pieces
  (flag storage, schema field, diag helper, stat slot) that must be
  present for the guard to fire correctly when the R42 `sys_stat`
  substrate body-swap lands. At HEAD the M2 `pdxfs_dst_exists` stub
  returns 0 (does-not-exist) so the M4 fixtures (test_smoke /
  test_txn_abort) continue to reach their success arms unchanged.
- `mv.v1.1-A`: retired the M1-003 `MV_RN_STUB` (0xFFFFEB20) that
  deferred `Rename::rename_same_dir` (`src/rename.pdx`) to the
  R42 PdxFS v1 substrate. The rename dispatcher now consumes the
  path-based syscalls that landed with R56.M3-005 in paideia-os
  (`sys_rename` sysno 82, `sys_unlink` sysno 81 -- rows 82 + 81
  in `design/user/syscall-table.md`): tries `sys_rename` first
  for the same-filesystem atomic fast path, and on -EXDEV falls
  through to a 4 KiB byte-copy scratch + `sys_close` + `sys_unlink`
  fallback. New `Rename::rn_strlen` walker sizes the caller's
  paths for the walker-CAP field the two syscalls consume. New
  `Rename::rn_xdev_copy_unlink` body owns the fallback (mirrors
  `Move::move_cross_dev_body`'s shape but without the outer TXN
  wrap -- the v1.1-A single-file mv path takes the strictly
  weaker no-TXN guarantee documented in the module header).
  New error codes `MV_RN_RENAME_FAIL` / `MV_RN_XDEV_OPEN_SRC_FAIL`
  / `_OPEN_DST_FAIL` / `_READ_FAIL` / `_WRITE_FAIL` / `_SHORT_WRITE`
  / `_UNLINK_FAIL` in the 0xFFFFEB27..0xFFFFEB2D slice. New
  `Pdxfs::sys_rename` + `Pdxfs::sys_unlink_path` trampolines
  (arity-4 and arity-2; the arity-4 uses the same SysV-rcx-to-
  syscall-r10 shuffle `pdxfs_link` uses). v1.1-A carries no
  `-i` / `-n` / `-f` flags -- the dispatch is unconditional; the
  three flags land in a later v1.x milestone. `tests/test_txn_
  abort.pdx` block A.3 (which asserted the retired STUB return)
  is dropped from the fixture; its slot in the return-code band
  (`MV_TEST_ABT_STUB_RC`, 0xFFFFEBC3) stays reserved.
- `mv.ENH-001` (#17): fixed a stack-frame corruption in
  `Move::move_dispatch`'s commit-fail epilogue (`src/move.pdx`) --
  a stray `add rsp, 8` broke the 5-push/5-pop parity, corrupting
  the `ret` target and four callee-saves the instant
  `pdxfs_txn_commit` stops being a stub. The branch was unreachable
  at 1.0.0 (the stub always returns 0), so it shipped green.
- `mv.ENH-008` (#24): `Audit::mv_audit_write_move` now reports every
  move through libpdx-audit's `AuditClient` three-call lifecycle
  (`audit_begin` / `audit_record_output` / `audit_commit`) instead of
  hand-rolling raw `sys_svc_lookup` + `sys_ipc_send` against
  `svc.audit-journal` with a wire shape that disagreed with
  `AuditBroker`'s real dispatch. `deps.list` / `manifest.pdxsig`
  corrected to the two libraries actually linked (`libpdx-argv`,
  `libpdx-audit`); `libpdx-cap`, `libpdx-semantic-pipe`, and
  `libpdx-elevate` were claimed but never called and are removed.
- `mv.ENH-006` (#22): `MvArgv::mv_argv_parse` now accepts `--verbose`
  as a long spelling of `-v`. `doc/mv.pdxdoc` corrected to drop the
  false claims that `-v` prints a generic `mv <src> <dst>` line and
  that POSIX `-f`/`-n` are "covered by `-i`" (neither is true of the
  source); `--interactive` stays undocumented as a working flag
  since `-i` still has no consumer (mv.ENH-005 blocks on `KIND_TTY`).
- `mv.ENH-009` (#25): `doc/mv.pdxdoc`, `CHANGELOG.md`, and
  `manifest.pdxsig` corrected to the MoveRecord layout `src/schema.pdx`
  and `src/undo.pdx` actually build -- 80-byte record under schema id
  `0xFFFFEB5000000001` (not the pre-M3 64-byte / `0x4D56` fossil), and
  undo magic `0xFFFFEB7000000001` (not `0x4D566E52`). See the erratum
  under M3 below.
- `mv.ENH-007` (#23): `--dry-run` now actually gates `Move::move_
  dispatch` -- after resolving both parents and the source inode
  (the "parse + validate + resolve" contract the doc always claimed),
  a dry-run returns success without opening a TXN, emitting a
  MoveRecord, writing the audit journal, or threading an undo record.
  Previously `mv_argv_dry_run` was parsed and read by nothing, so
  `mv --dry-run a b` performed a real, committed move.

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

> **Erratum (mv.ENH-009, #25):** the schema id / record size and undo
> magic named below were pre-M3 placeholders that were never updated
> to match the M3-001/M3-003 implementation, and M3-002's "via
> libpdx-audit" claim was not actually true until mv.ENH-008 (#24).
> The record `src/schema.pdx` builds is 80 bytes under schema id
> `0xFFFFEB5000000001`; the undo magic `src/undo.pdx` builds is
> `0xFFFFEB7000000001`. See the Unreleased section above.

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
