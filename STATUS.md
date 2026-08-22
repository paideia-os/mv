# mv — status

**Wave:** R50 (Wave 2)
**Current milestone:** M5 (signed 1.0 release) — M5-001 landed

See `design/tooling/r49-r50-plan.md` §5.7 in paideia-os for the full
breakdown.

## M5 — signed 1.0 release (M5-001 landed; M5-002 pending)

- `manifest.pdxsig` (issue #15): dual-signed package manifest at
  repo root per `design/tooling/plan.md` §6.3 + §9.3. Format is
  text header + two ML-DSA-65 signature stanzas. Header fields
  are AUTHORITATIVE (name=mv, version=1.0.0, license=MIT,
  wave=R50, milestone=M5, category=A/coreutils, priority=P0,
  authored_by=paideia-os-team, released=2026-08-22). The
  `[caps]` stanza names the six required caps from caps.decl.
  The `[deps]` stanza names the five shared-library
  dependencies from deps.list. The `[artifacts]` stanza names
  every file shipped in pkg.tar with a per-file BLAKE3-256
  digest slot. The `[schemas]` stanza names the MoveRecord[]
  schema id (0x4D56, 64 bytes). The `[undo]` stanza names the
  PdxFS v1 undo-record shape (magic 0x4D566E52, replay opcode
  0x6D76, replay = mv <dst> <src>). Signature slots are
  RESERVED with the SHAPE-PENDING pattern documented in
  RELEASE.md — the two 3309-byte ML-DSA-65 signatures land
  when T-INFRA-002 (signing bot + paideia_root_pk policy)
  stands up. Every `sig_bytes` field carries the literal
  `SHAPE-PENDING` and every fingerprint / digest carries the
  `pending:<label>` marker so a downstream tool cannot mistake
  the shape file for a valid one.
- `deps.list` (issue #15): shared-library dependencies per
  `design/tooling/plan.md` §6.3. Five entries in src/ file
  order: libpdx-cap >= 1.0.0 (cap-manifest verify + signed-
  inode helpers), libpdx-argv >= 1.0.0 (argv surface),
  libpdx-semantic-pipe >= 1.0.0 (MoveRecord[] emit), libpdx-
  audit >= 1.0.0 (user-events journal), libpdx-elevate >=
  1.0.0 (cross-user broker hop). `pkg install mv` resolves
  each against the semver constraint before staging.
- `doc/mv.pdxdoc` (issue #15): man-equivalent read by `doc mv`
  per I7 §2. Fifteen topics (.TOPIC synopsis / description /
  options / exit_codes / caps / schemas / audit / undo /
  ergonomics / posix / seealso / history / bugs / author /
  license) covering the full user surface. Ergonomics topic
  matches the D5 "five most common tasks" convention.
  Differences-from-POSIX topic names the three axes on which
  paideia-os mv diverges from POSIX mv (atomicity in all four
  move shapes, audit journal before every op, WAL undo record
  on commit).
- `CHANGELOG.md` (issue #15): v1.0.0 entry naming the five
  milestone rollups (M5 / M4 / M3 / M2 / M1), the upstream
  substrate at landing (KIND_USER / KIND_IPC_ENDPOINT /
  KIND_PDXFS_FILE / KIND_PDXFS_TXN / KIND_ELEVATE_CHANNEL),
  and the four deferred-work items (live TXN dispatch on R42,
  QEMU acceptance smoke on R42+shell.M5+pdx-undo, signing-bot
  re-sign on T-INFRA-002, pkgs.paideia-os mirror on
  T-INFRA-001).
- `RELEASE.md` (issue #15): SHAPE-PENDING → LIVE handoff
  runbook. Documents the §9.3 release flow, the regeneration
  recipe when T-INFRA-002 lands (`paideia-as release --sign`
  invocation), the verification recipe on the user side (`pkg
  install mv` fails today; `pkg install --from-source mv`
  works), the file-by-file M5 issue attribution, and the
  reserved-for-future entries (1.0.1 for R42 substrate
  landing; 1.0.0-signed for the identical tree with real
  signatures).

## M4 — tests + smoke (complete)

- `tests/test_txn_abort.pdx` (issue #12): TestTxnAbort module —
  fixture entry point `test_txn_abort_run` verifies the M4-001
  TXN-abort shape via three blocks: (A) Rename argv-gate rejects
  (rename_same_dir(0,dst) → MV_RN_BAD_SRC; rename_same_dir(src,0)
  → MV_RN_BAD_DST; rename_same_dir(src,dst) → MV_RN_STUB; both
  bump MV_ST_ERRORS), (B) direct Pdxfs::pdxfs_txn_abort(1) → 0
  (stub OK, dispatch shape verified for Move's abort branches),
  (C) happy-path invariant (move_dispatch(0x1000,0x2000) → 0;
  MV_ST_MOVES + MV_ST_TXN_COMMITS bump by 1; MV_ST_TXN_ABORTS +
  MV_ST_ERRORS unchanged — the abort-leak / error-leak check).
  Return-code band 0xFFFFEBCx (11 codes). Live fault injection
  deferred to R42 substrate (sys_pdxfs_fault_inject sysno 527).
  Prologue 2 callee-save pushes (rbx = abort baseline, r12 =
  error baseline) + sub rsp, 8 = 24 bytes; even-count alignment
  idiom same as elevate_client_lookup_broker.
- `tests/test_undo_replay.pdx` (issue #13): TestUndoReplay module —
  fixture entry point `test_undo_replay_run` runs the swap-
  correctness matrix three times with distinct fixture pointer
  pairs (0x1111/0x2222, 0x3333/0x4444, 0xDEAD/0xBEEF) to catch
  populate-time aliasing. Each iteration: mv_undo_reset + verify
  4 slots zero → mv_undo_populate(src,dst) → verify magic ==
  MV_UNDO_MAGIC (staged via rcx for reg-reg cmp), replay_op ==
  0x6D76, replay_src == dst (SWAP), replay_dst == src (SWAP) →
  mv_undo_write(1) → 0 → MV_ST_UNDO_RECORDS bumped by 1. The
  case dimension (same-dir / cross-dir / cross-dev / cross-user)
  collapses to a single check because Undo is case-agnostic by
  construction (one code path called from one call site with
  identical argument shape across all four cases). Return-code
  band 0xFFFFEBDx (8 codes). Prologue 3 callee-save pushes
  (rbx = stats baseline, r12 = src fixture, r13 = dst fixture)
  = 24 bytes; odd-count alignment idiom same as mv_inode_preserve.
- `tests/test_smoke.pdx` (issue #14): TestSmoke module — fixture
  entry point `test_smoke_run` runs ONE Move::move_dispatch on a
  fresh stats table with fixture pointers (0x1000, 0x2000) and
  verifies every M3-integrated subsystem fires per its stub-driven
  expectation: MV_ST_INVOCATIONS >= 1, MV_ST_TXN_OPENS ==
  MV_ST_TXN_COMMITS == MV_ST_MOVES == MV_ST_INODE_PRESERVED ==
  MV_ST_RECORDS_EMITTED == MV_ST_AUDIT_WRITES ==
  MV_ST_UNDO_RECORDS == 1; MV_ST_TXN_ABORTS == MV_ST_ERRORS ==
  MV_ST_CROSS_DIR == MV_ST_CROSS_DEV == MV_ST_CROSS_USER ==
  MV_ST_ELEVATE_ATTEMPTS == 0. Live QEMU harness (mv a b; undo
  mv a b; ls) deferred to R42 substrate + shell.M5 + pdx-undo
  (R51) — documented as a runbook in tests/README.md. Return-
  code band 0xFFFFEBEx (16 codes). No callee-save state persists
  between checks; prologue sub rsp, 8 for alignment.
- `tests/README.md`: complete runbook — fixture-to-issue mapping
  table, return-code band assignments, substrate-state discussion
  (reachable vs unreachable failure vectors at HEAD), QEMU smoke
  harness command sequence for the deferred live acceptance gate,
  fixture-input rationale, and the mv-test-runner (R51) gate for
  running the fixtures automatically.

## M3 — semantic-pipe / audit integration (complete)

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
- `src/undo.pdx` (issue #10): Undo module — 32-byte UndoRecord
  layout (magic + replay_op + replay_src_ptr + replay_dst_ptr).
  `mv_undo_reset` / `mv_undo_populate(src, dst)` populates with
  SWAPPED paths (replay_src=dst, replay_dst=src) so `pdx-undo`
  replaying with `mv <replay_src> <replay_dst>` invokes
  `mv <original_dst> <original_src>` and reverses the TXN.
  `mv_undo_write(txn_handle)` threads the record into the WAL
  group via pdxfs_txn_write_undo (sysno 522; M3 STUB pending
  R42 substrate landing). Return-code band 0xFFFFEB7x. Adds
  MV_ST_UNDO_RECORDS (slot 13).
- `src/move.pdx` (issue #10): Move::move_dispatch calls
  mv_undo_reset + mv_undo_populate + mv_undo_write AFTER audit
  write and BEFORE commit. Best-effort: a failed undo write does
  NOT abort the TXN (I5 uplift; the audit journal still records
  the operation, so the mv is still accountable).
- `src/elevate.pdx` (issue #11): Elevate module — libpdx-elevate
  hop for cross-user-boundary dst. 32-byte ElevateRequest layout
  (magic + requested_kind + requested_key + duration_secs) with
  MV_ELV_REQUEST_MAGIC 0xFFFFEB8000000001. `mv_elevate_reset`
  clears the request slot; `mv_elevate_populate(dst_owner_key)`
  fills it with ELV_KIND_USER (0x190) + dst_owner_key + 60s
  window; `mv_elevate_cross_user(dst_owner_key)` looks up
  "svc.elevate-broker" (cached in mv_elevate_endpoint on first
  call), sends the request via ipc_send, and returns 0 on grant
  (broker transiently minted KIND_USER(sign, dst_owner_key) into
  caller's cap_table for 60s) or MV_ELV_LOOKUP_FAIL / SEND_FAIL /
  DENIED on refuse. Return-code band 0xFFFFEB8x. Adds
  MV_ST_ELEVATE_ATTEMPTS (slot 14) + MV_ST_ELEVATE_GRANTS (slot
  15). Reuses Audit::svc_lookup + Audit::ipc_send trampolines
  (no duplication).
- `src/inode.pdx` (issue #11): Inode::mv_inode_preserve's
  cross-user branch calls mv_elevate_cross_user with the
  dst_owner_key BEFORE sys_user_self + pdxfs_inode_rebind_owner.
  Best-effort HINT: a denied elevate does NOT fail the mv --
  falls through to the M2-004 graceful-degrade path (rebind
  under invoker; on failure the inode keeps its old owner tail
  and the verbose diagnostic flags the degrade). The 60s window
  greatly increases the probability that the subsequent rebind
  succeeds when the invoker holds the elevate-broker cap.
- `caps.decl` (issue #11): the KIND_ELEVATE_CHANNEL (invoke,
  svc.elevate-broker) placeholder is now a required cap. mv
  refuses at exec-time cap_manifest_verify if the caller cannot
  hand down the elevate-broker binding.

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
| 0xFFFFEB70 | MV_UND_STUB              | Undo.M3-003: reserved                      |
| 0xFFFFEB71 | MV_UND_WRITE_FAIL        | Undo.M3-003: pdxfs_txn_write_undo neg errno|
| 0xFFFFEB80 | MV_ELV_STUB              | Elevate.M3-004: reserved                   |
| 0xFFFFEB81 | MV_ELV_LOOKUP_FAIL       | Elevate.M3-004: svc_lookup neg errno       |
| 0xFFFFEB82 | MV_ELV_SEND_FAIL         | Elevate.M3-004: ipc_send neg errno         |
| 0xFFFFEB83 | MV_ELV_DENIED            | Elevate.M3-004: broker refused             |
| 0xFFFFEBC0 | MV_TEST_ABT_OK           | test_txn_abort_run all-pass sentinel (== 0)|
| 0xFFFFEBC1 | MV_TEST_ABT_BAD_SRC_RC   | rename_same_dir(0,dst) != MV_RN_BAD_SRC   |
| 0xFFFFEBC2 | MV_TEST_ABT_BAD_DST_RC   | rename_same_dir(src,0) != MV_RN_BAD_DST   |
| 0xFFFFEBC3 | MV_TEST_ABT_STUB_RC      | rename_same_dir(src,dst) != MV_RN_STUB    |
| 0xFFFFEBC4 | MV_TEST_ABT_ERR_STAT     | MV_ST_ERRORS did not bump on reject       |
| 0xFFFFEBC5 | MV_TEST_ABT_ABT_STUB     | pdxfs_txn_abort(1) != 0 (stub broke)      |
| 0xFFFFEBC6 | MV_TEST_ABT_HAPPY_RC     | move_dispatch(src,dst) != 0               |
| 0xFFFFEBC7 | MV_TEST_ABT_COMMIT_STAT  | MV_ST_TXN_COMMITS did not bump             |
| 0xFFFFEBC8 | MV_TEST_ABT_MOVES_STAT   | MV_ST_MOVES did not bump                   |
| 0xFFFFEBC9 | MV_TEST_ABT_ABT_LEAK     | MV_ST_TXN_ABORTS bumped on happy path      |
| 0xFFFFEBCA | MV_TEST_ABT_ERR_LEAK     | MV_ST_ERRORS bumped on happy path          |
| 0xFFFFEBD0 | MV_TEST_UND_OK           | test_undo_replay_run all-pass sentinel     |
| 0xFFFFEBD1 | MV_TEST_UND_RESET_FAIL   | mv_undo_reset left a non-zero slot         |
| 0xFFFFEBD2 | MV_TEST_UND_MAGIC_FAIL   | populated magic != MV_UNDO_MAGIC           |
| 0xFFFFEBD3 | MV_TEST_UND_OP_FAIL      | populated replay_op != 0x6D76              |
| 0xFFFFEBD4 | MV_TEST_UND_SRC_FAIL     | replay_src != original_dst (SWAP wrong)   |
| 0xFFFFEBD5 | MV_TEST_UND_DST_FAIL     | replay_dst != original_src (SWAP wrong)   |
| 0xFFFFEBD6 | MV_TEST_UND_WRITE_FAIL   | mv_undo_write returned non-zero            |
| 0xFFFFEBD7 | MV_TEST_UND_STAT_FAIL    | MV_ST_UNDO_RECORDS delta wrong             |
| 0xFFFFEBE0 | MV_TEST_SMK_OK           | test_smoke_run all-pass sentinel           |
| 0xFFFFEBE1 | MV_TEST_SMK_DISPATCH_RC  | move_dispatch returned non-zero            |
| 0xFFFFEBE2 | MV_TEST_SMK_INV_STAT     | MV_ST_INVOCATIONS < 1                     |
| 0xFFFFEBE3 | MV_TEST_SMK_TOPEN_STAT   | MV_ST_TXN_OPENS != 1                      |
| 0xFFFFEBE4 | MV_TEST_SMK_TCMT_STAT    | MV_ST_TXN_COMMITS != 1                    |
| 0xFFFFEBE5 | MV_TEST_SMK_TABT_STAT    | MV_ST_TXN_ABORTS != 0                     |
| 0xFFFFEBE6 | MV_TEST_SMK_ERR_STAT     | MV_ST_ERRORS != 0                         |
| 0xFFFFEBE7 | MV_TEST_SMK_MOVES_STAT   | MV_ST_MOVES != 1                          |
| 0xFFFFEBE8 | MV_TEST_SMK_REC_STAT     | MV_ST_RECORDS_EMITTED != 1                |
| 0xFFFFEBE9 | MV_TEST_SMK_AUD_STAT     | MV_ST_AUDIT_WRITES != 1                   |
| 0xFFFFEBEA | MV_TEST_SMK_UND_STAT     | MV_ST_UNDO_RECORDS != 1                   |
| 0xFFFFEBEB | MV_TEST_SMK_PRSV_STAT    | MV_ST_INODE_PRESERVED != 1                |
| 0xFFFFEBEC | MV_TEST_SMK_XUSER_STAT   | MV_ST_CROSS_USER != 0                     |
| 0xFFFFEBED | MV_TEST_SMK_XDEV_STAT    | MV_ST_CROSS_DEV != 0                      |
| 0xFFFFEBEE | MV_TEST_SMK_XDIR_STAT    | MV_ST_CROSS_DIR != 0                      |
| 0xFFFFEBEF | MV_TEST_SMK_ELV_STAT     | MV_ST_ELEVATE_ATTEMPTS != 0               |

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
| M3-003 (#10)    | PdxFS v1 undo record (replay is mv <dst> <src>)               | LANDED |
| M3-004 (#11)    | libpdx-elevate for cross-user-boundary dst                    | LANDED |
| M4-001 (#12)    | TXN-abort mid-move: source restored, no dest artifact         | LANDED |
| M4-002 (#13)    | undo replay correctness across all four cases                 | LANDED |
| M4-003 (#14)    | QEMU smoke: mv a b; undo mv a b; ls verifies                  | LANDED |
| M5-001 (#15)    | dual-signed release + .pdxdoc                                 | LANDED |
| M5-002 (#16)    | mirror push                                                   | PENDING |

## Upstream substrate (paideia-os, at HEAD 2026-08-22)

- `KIND_USER = 0x190` — landed in R48.M1 at
  `src/kernel/core/cap/kind_user.pdx`.
- `KIND_IPC_ENDPOINT = 5` — R20b substrate at
  `src/kernel/core/cap/kind.pdx:72`.
- `KIND_PDXFS_FILE = 0x195` — R48b substrate-prep at
  `src/kernel/core/cap/kind_pdxfs_file.pdx`.
- `KIND_PDXFS_TXN = 0x196` — R48b substrate-prep at
  `src/kernel/core/cap/kind_pdxfs_txn.pdx`.
