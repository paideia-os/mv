# mv — status

**Wave:** R50 (Wave 2)
**Current milestone:** M3 (semantic-pipe / audit integration) — in progress

See `design/tooling/r49-r50-plan.md` §5.7 in paideia-os for the full
breakdown.

## M3 — semantic-pipe / audit integration (in progress)

- `src/schema.pdx` (issue #8): Schema module — MoveRecord[] fixed
  layout (80 bytes; ten u64 fields at documented offsets) with
  MOVE_RECORD_MAGIC schema id 0xFFFFEB5000000001. `mv_schema_reset`
  clears the 80-byte .bss slot; `mv_schema_populate(src, dst)` fills
  it from the Move module's per-invocation .bss + the two path
  pointers (was_rename derived from src_parent == dst_parent);
  `mv_schema_emit` writes it to the semantic-pipe fd (fd 1 stub at
  M3; a libpdx-semantic-pipe endpoint when the crate lands at R50).
  Return-code band 0xFFFFEB5x (MV_SCH_EMIT_FAIL / MV_SCH_SHORT_WRITE).
  Adds MV_ST_RECORDS_EMITTED (slot 11).
- `src/move.pdx` (issue #8): Move::move_dispatch calls
  mv_schema_reset + mv_schema_populate + mv_schema_emit AFTER
  mv_inode_preserve but BEFORE pdxfs_txn_commit so the schema
  emission satisfies D3 audit-first (the record is observable even
  if the commit fails mid-drain; the emitted txn_handle field lets
  a consumer correlate to PdxFS TXN state).
- `src/audit.pdx` (issue #9): Audit module — svc_lookup +
  ipc_send trampolines against R20b substrate sysno 43 / 42
  (live at HEAD). `mv_audit_write_move` looks up the
  "svc.audit-journal" endpoint (cached in mv_audit_endpoint on
  first call) then sends the 80-byte MoveRecord as an IPC payload.
  Return-code band 0xFFFFEB6x (MV_AUD_LOOKUP_FAIL /
  MV_AUD_SEND_FAIL). Adds MV_ST_AUDIT_WRITES (slot 12).
- `src/move.pdx` (issue #9): Move::move_dispatch calls
  mv_audit_write_move AFTER schema emit and BEFORE
  pdxfs_txn_commit. On failure branches to mv_md_audit_fail --
  aborts the TXN via pdxfs_txn_abort (source dirent stays intact,
  no destination artifact appears), bumps MV_ST_TXN_ABORTS +
  MV_ST_ERRORS, and returns the specific MV_AUD_* code via the
  mv_move_return_code stash slot (nested calls clobber rax). This
  is the D3 audit-first gate: a mv that cannot log its own
  operation is refused.

## M2 — core implementation (complete)

- `src/pdxfs.pdx` (issue #4): Pdxfs module — eight userspace
  syscall trampolines for the PdxFS v1 txn substrate
  (pdxfs_txn_open / pdxfs_txn_commit / pdxfs_txn_abort /
  pdxfs_unlink / pdxfs_link / pdxfs_resolve_parent /
  pdxfs_inode_of / pdxfs_device_of). All bodies are M2 stubs
  returning 0 (or 1 for handle-valued returns) pending the R42
  substrate landing of sys_pdxfs_* dispatch entries; each is a
  straight `mov rax, SYSNO; syscall; ret` body-edit away from
  R42.
- `src/move.pdx` (issue #4): Move module — `move_dispatch(src,
  dst)` threads pdxfs_resolve_parent x 2 + pdxfs_inode_of +
  pdxfs_txn_open + pdxfs_unlink + pdxfs_link +
  pdxfs_txn_commit under one TXN, handling BOTH same-directory
  rename (src_parent == dst_parent) AND cross-directory move
  (src_parent != dst_parent). Bumps MV_ST_CROSS_DIR when the
  parents differ. Error paths call pdxfs_txn_abort to unwind a
  mid-thread TXN and return a distinct MV_MV_* code from the
  0xFFFFEB3x band.
- `src/move.pdx` (issue #5): Move::move_cross_dev_body — cp+rm
  internal fallback for the EXDEV case. Byte-copies through a
  4 KiB scratch (mv_move_xdev_buf) via real sys_open/read/write/
  close; on EOF closes both fds and unlinks the source dirent
  inside the OUTER TXN (still-single-TXN invariant). move_dispatch
  detects EXDEV by `mov rcx, 0xFFFFFFFFFFFFFFEE; cmp rax, rcx`
  and dispatches to the fallback; on success bumps MV_ST_CROSS_DEV
  and joins the commit path; on failure preserves the specific
  MV_MV_XDEV_* code across pdxfs_txn_abort via a .bss stash slot
  (mv_move_return_code).
- `src/pdxfs.pdx` (issue #5): Pdxfs::pdxfs_open / pdxfs_read /
  pdxfs_write / pdxfs_close — four real sys_open/read/write/close
  trampolines (matching cp's discipline) added for the fallback
  path.
- `src/verbose.pdx` (issue #6): Verbose module — `mv_verbose_diag`
  reads MvArgv::mv_argv_verbose + Move::mv_move_was_cross_device
  and prints "mv: crossed device (cp+rm fallback, not O(1))\n" to
  stderr (fd 2) when both flags are set. Called from Move::
  move_dispatch's success arm as a best-effort emission (the
  operation is already committed; a fd-2 write refusal does not
  invalidate mv's return code).
- `src/inode.pdx` (issue #7): Inode module —
  `mv_inode_preserve(src_inode, dst_parent_inode)` queries owner
  keys via pdxfs_inode_owner, dispatches: same-user -> bump
  MV_ST_INODE_PRESERVED; cross-user -> set mv_move_was_cross_user,
  try pdxfs_inode_rebind_owner under invoker's key, bump either
  MV_ST_INODE_REKEYED (rebind succeeded) or MV_ST_CROSS_USER
  (graceful degrade -- inode keeps old owner). Returns 0 for every
  outcome (metadata refinement, not required step).
- `src/pdxfs.pdx` (issue #7): three trampolines added --
  pdxfs_inode_owner (SYS 520), pdxfs_inode_rebind_owner (SYS 521),
  sys_user_self (SYS 528). All M2 stubs; return 1 (user_key) or 0
  (rebind OK).
- `src/verbose.pdx` (issue #7): extended with MV_VERBOSE_XUSER_MSG
  ("mv: crossed user (inode kept old owner)\n") emission gated on
  Move::mv_move_was_cross_user. The two advisories are independent
  -- an operation can be cross-device + cross-user simultaneously
  and both fire.
- `src/move.pdx` (issue #7): move_dispatch calls mv_inode_preserve
  after successful pdxfs_link (or cross-dev unlink) but BEFORE
  pdxfs_txn_commit -- so a preservation-side abort could unwind
  the whole TXN (currently mv_inode_preserve never returns non-
  zero; the ordering is future-proofing).

## M1 — design + skeleton (complete)

- `src/mv.pdx` (issue #1): top-level `Mv` module — KIND ordinal
  mirrors (MV_KIND_USER / MV_KIND_IPC_ENDPOINT / MV_KIND_PDXFS_FILE /
  MV_KIND_PDXFS_TXN), the 0xFFFFEBxx return-code band, and the
  eight-slot `_mv_stats` counter table with `mv_reset` / `mv_note` /
  `mv_stat`.
- `caps.decl` (issue #1): the five caps mv holds at exec
  (KIND_USER self, KIND_IPC_ENDPOINT invoke, KIND_PDXFS_FILE write on
  src-parent + dst-parent, KIND_PDXFS_TXN invoke+mint).
- `design/architecture.md` (issue #1): full M1 spec covering all three
  modules plus the 0xFFFFEBxx band, the paideia-as conformance
  checklist, and the M4 test matrix.
- `src/argv.pdx` (issue #2): `MvArgv` module — argv surface wrapping
  libpdx-argv's `parse_argv`. Accepts flags `-v`, `-i`, `--dry-run`;
  rejects unknown flags with `MV_ARG_UNKNOWN_FLAG` and wrong
  positional counts with `MV_ARG_TOO_FEW_POS`. Stores parsed results
  in mv-owned .bss slots (mv_argv_src / mv_argv_dst /
  mv_argv_verbose / mv_argv_interactive / mv_argv_dry_run).
- `src/rename.pdx` (issue #3): `Rename` module — the same-directory
  rename skeleton. `rename_same_dir(src, dst)` gates the args and
  returns `MV_RN_STUB` (0xFFFFEB20) on the happy path; `MV_RN_BAD_SRC`
  / `MV_RN_BAD_DST` on reject. Bumps `MV_ST_INVOCATIONS` /
  `MV_ST_TXN_OPENS` / `MV_ST_ERRORS`. M2 replaces the MV_RN_STUB tail
  with the real sys_pdxfs_txn_open + sys_pdxfs_unlink +
  sys_pdxfs_link + sys_pdxfs_txn_commit dispatch.
- `tests/README.md`: placeholder describing the M4 fixture matrix
  (same-dir rename, cross-dir/cross-dev/cross-user, TXN-abort, undo
  replay).

## Return-code band 0xFFFFEBxx

| Code       | Name                | Meaning                                        |
|------------|---------------------|------------------------------------------------|
| 0xFFFFEB00 | MV_OK               | General success (unused at M1)                 |
| 0xFFFFEB10 | MV_ARG_STUB         | Reserved; unused at M1                         |
| 0xFFFFEB11 | MV_ARG_BAD_ARGC     | argc == 0                                      |
| 0xFFFFEB12 | MV_ARG_BAD_ARGV     | argv == 0                                      |
| 0xFFFFEB13 | MV_ARG_TOO_FEW_POS  | positional count != 2 (need <src> <dst>)       |
| 0xFFFFEB14 | MV_ARG_UNKNOWN_FLAG | flag not in {v, i, dry-run}                    |
| 0xFFFFEB15 | MV_ARG_PARSE_ERR    | libpdx-argv reported an ERR_* code             |
| 0xFFFFEB20 | MV_RN_STUB          | Rename.M1: validated, no live TXN yet          |
| 0xFFFFEB21 | MV_RN_BAD_SRC       | src == 0                                       |
| 0xFFFFEB22 | MV_RN_BAD_DST       | dst == 0                                       |
| 0xFFFFEB23 | MV_RN_TXN_OPEN_FAIL | M2+: sys_pdxfs_txn_open refused                |
| 0xFFFFEB24 | MV_RN_UNLINK_FAIL   | M2+: sys_pdxfs_unlink refused inside TXN       |
| 0xFFFFEB25 | MV_RN_LINK_FAIL     | M2+: sys_pdxfs_link refused inside TXN         |
| 0xFFFFEB26 | MV_RN_COMMIT_FAIL   | M2+: sys_pdxfs_txn_commit refused              |
| 0xFFFFEB30 | MV_MV_STUB          | reserved                                        |
| 0xFFFFEB31 | MV_MV_RESOLVE_SRC_FAIL | pdxfs_resolve_parent(src) failed             |
| 0xFFFFEB32 | MV_MV_RESOLVE_DST_FAIL | pdxfs_resolve_parent(dst) failed             |
| 0xFFFFEB33 | MV_MV_INODE_FAIL    | pdxfs_inode_of(src) failed                     |
| 0xFFFFEB34 | MV_MV_TXN_OPEN_FAIL | pdxfs_txn_open refused                         |
| 0xFFFFEB35 | MV_MV_UNLINK_FAIL   | pdxfs_unlink refused inside TXN                |
| 0xFFFFEB36 | MV_MV_LINK_FAIL     | pdxfs_link refused inside TXN                  |
| 0xFFFFEB37 | MV_MV_COMMIT_FAIL   | pdxfs_txn_commit refused                       |
| 0xFFFFEB38 | MV_MV_XDEV_OPEN_SRC_FAIL | cross-dev fallback: sys_open(src) failed  |
| 0xFFFFEB39 | MV_MV_XDEV_OPEN_DST_FAIL | cross-dev fallback: sys_open(dst) failed  |
| 0xFFFFEB3A | MV_MV_XDEV_READ_FAIL     | cross-dev fallback: sys_read failed       |
| 0xFFFFEB3B | MV_MV_XDEV_WRITE_FAIL    | cross-dev fallback: sys_write failed      |
| 0xFFFFEB3C | MV_MV_XDEV_SHORT_WRITE   | cross-dev fallback: short write           |
| 0xFFFFEB3D | MV_MV_XDEV_UNLINK_FAIL   | cross-dev fallback: pdxfs_unlink refused  |
| 0xFFFFEB50 | MV_SCH_STUB              | Schema.M3-001: reserved                    |
| 0xFFFFEB51 | MV_SCH_EMIT_FAIL         | Schema.M3-001: pdxfs_write neg errno       |
| 0xFFFFEB52 | MV_SCH_SHORT_WRITE       | Schema.M3-001: pdxfs_write short           |
| 0xFFFFEB60 | MV_AUD_STUB              | Audit.M3-002: reserved                     |
| 0xFFFFEB61 | MV_AUD_LOOKUP_FAIL       | Audit.M3-002: sys_svc_lookup neg errno     |
| 0xFFFFEB62 | MV_AUD_SEND_FAIL         | Audit.M3-002: sys_ipc_send neg errno       |

## Milestone rollup

| ID              | Title                                                         | State  |
|-----------------|---------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + caps.decl (src-parent + dst-parent write + TXN)    | LANDED |
| M1-002 (#2)     | argv surface via libpdx-argv (mv [-v|-i|--dry-run])           | LANDED |
| M1-003 (#3)     | first runnable: same-dir rename in single TXN                 | LANDED |
| M2-001 (#4)     | cross-dir same-device move (link-unlink atomic in TXN)        | LANDED |
| M2-002 (#5)     | cross-device move via cp+rm internal fallback                 | LANDED |
| M2-003 (#6)     | was_cross_device diagnostic on --verbose                      | LANDED |
| M2-004 (#7)     | signed-inode preservation + cross-user graceful degrade       | LANDED |
| M3-001 (#8)     | MoveRecord[] schema bind (was_rename, was_cross_device flags) | LANDED |
| M3-002 (#9)     | MoveRecord via libpdx-audit before commit                     | LANDED |

## Upstream substrate (paideia-os, at HEAD 2026-08-21)

- `KIND_USER = 0x190` — landed in R48.M1 at
  `src/kernel/core/cap/kind_user.pdx`.
- `KIND_IPC_ENDPOINT = 5` — R20b substrate at
  `src/kernel/core/cap/kind.pdx:72`.
- `KIND_PDXFS_FILE = 0x195` — R48b substrate-prep at
  `src/kernel/core/cap/kind_pdxfs_file.pdx`.
- `KIND_PDXFS_TXN = 0x196` — R48b substrate-prep at
  `src/kernel/core/cap/kind_pdxfs_txn.pdx`.
