# mv — architecture

**Wave:** R50 (Wave 2)
**Repo:** github.com/paideia-os/mv
**Upstream design:** `design/tooling/r49-r50-plan.md` §5.7 in
[paideia-os](https://github.com/paideia-os/paideia-os).

This document describes the internal shape of the mv binary. It does
not repeat the wave-level rationale from the paideia-os plan doc; read
that first for D2 (semantic pipes), D3 (audit-first), D4 (signed
manifests), I5 (destructive-op undo), and I6 (capability handoff
visible + refusable). mv is the tool that moves one path onto another
by opening a PdxFS v1 transaction, unlinking the source dirent,
linking the destination dirent, and committing atomically. Every
successful mv is journaled to `/system/audit/user-events/` at M3 with
a PdxFS undo record whose replay is `mv <dst> <src>` (I5 uplift).

## 1. Public surface

mv is not a library — it is a binary. Its "surface" from a
programmatic point of view is a small set of module entry points the
`_start` frame calls in order:

- `Mv` (`src/mv.pdx`) — top-level orchestration, constants shared
  across the mv modules (KIND ordinal mirrors, MV_* return-code band),
  session-level stats (bounded stats table, reset/note/stat), and the
  M1 scaffold.
- `MvArgv` (`src/argv.pdx`) — the argv surface. Wraps libpdx-argv's
  `parse_argv` and stores the parsed flags (`-v`, `-i`, `--dry-run`)
  + two positionals (`<src>`, `<dst>`) into mv-owned .bss slots for
  the rename dispatch. Rejects unknown flags with `MV_ARG_UNKNOWN_FLAG`
  and wrong positional counts with `MV_ARG_TOO_FEW_POS`.
- `Rename` (`src/rename.pdx`) — the same-directory rename path.
  `rename_same_dir(src_ptr, dst_ptr) → u64` opens a PdxFS TXN, unlinks
  the source dirent, links the destination dirent, and commits. M1
  ships the SKELETON: argument gating + counter dispatch returns
  `MV_RN_STUB`; the actual sys_pdxfs_txn_open / sys_pdxfs_unlink /
  sys_pdxfs_link / sys_pdxfs_txn_commit calls land at M2 once the
  userspace syscall wrappers exist alongside the R42 PdxFS v1
  substrate.

The mv `_start` frame (not part of M1 — the loader's entry convention
lands with the R14b paideia-os bootstrap) calls the three modules in
this order:

```
1. Mv::mv_reset()                         // clear stats
2. let rc = MvArgv::mv_argv_parse(argc, argv)
   if rc != 0 { exit rc }                 // MV_ARG_* rejects
3. let rc = Rename::rename_same_dir(mv_argv_src, mv_argv_dst)
4. exit rc                                // 0 on success, MV_RN_* on error
```

At M1 `mv_argv_parse` runs libpdx-argv against the real argv (so the
flag / positional plumbing is testable end-to-end without a live
PdxFS), then hands its two positional pointers to
`rename_same_dir`, which returns `MV_RN_STUB` after validating +
recording the intent to open a TXN. Every M1 entry point returns a
documented sentinel so the M2 tests can diff a live run against the
M1 skeleton with a mechanical rule (`if rc == MV_RN_STUB: still on
M1; if rc == 0: M2 live and the TXN committed`).

## 2. `Mv` module (src/mv.pdx)

### 2.1 Constants

The Mv module owns:

- **KIND ordinal mirrors.** The kernel's KIND ordinals mv talks about
  (`MV_KIND_USER = 0x190`, `MV_KIND_IPC_ENDPOINT = 5`,
  `MV_KIND_PDXFS_FILE = 0x195`, `MV_KIND_PDXFS_TXN = 0x196`).
  Redeclared here for the same reason libpdx-elevate mirrors ELV_*
  and shell mirrors SH_* — the mv repo does not link paideia-os's
  kernel .o graph at build time. Drift is caught by the M4 smoke
  matrix; the `MV_` prefix documents the mirror invariant.
- **Return-code band `0xFFFFEBxx`.** mv's own error-code family,
  disjoint from the underlying kernel's syscall errno family
  (`-EFAULT = 0xFFFFFFFFFFFFFFF2` etc.), from libpdx-cap's band
  (`0xFFFFFFxx`), from libpdx-elevate's bands (`0xFFFFEAxx` /
  `0xFFFFE5Exx`), and from shell's band (`0xFFFFECxx`). See §5 for
  the full table.
- **Session-level `.bss` singleton.** An 8-slot stats counter table
  (`_mv_stats`), cache-line aligned, mirrors the shape of
  `_shell_stats` (shell repo) and `_elevate_broker_stats` (paideia-os
  `src/kernel/core/ipc/elevate_broker.pdx:70`).

### 2.2 `mv_reset()`

Clears the eight-word `_mv_stats` counter table. Called by `_start`
before the run and by tests before each fixture. Leaf function; `r10`
as base + `rcx` as loop index. Same shape as `shell_reset`.

### 2.3 `mv_note(which)` + `mv_stat(which)`

Bounded increment + bounded read for the counter table. `which >=
MV_ST_MAX` is a no-op (increment) or returns 0 (read). Same shape as
`shell_note` / `shell_stat`.

## 3. `MvArgv` module (src/argv.pdx)

### 3.1 Contract

```
mv_argv_parse(argc: u64, argv: u64) -> u64
```

- Reset libpdx-argv's ParsedArgs, then dispatch its `parse_argv`.
- Validate every flag against the mv-accepted set (`v`, `i`, `dry-run`).
- Validate positional count == 2. `<src>` and `<dst>` are stored to
  `mv_argv_src` / `mv_argv_dst` in mv's own .bss.
- Store `-v` / `-i` / `--dry-run` presence as 0/1 booleans in
  `mv_argv_verbose` / `mv_argv_interactive` / `mv_argv_dry_run`.
- Returns 0 on success, or one of the `MV_ARG_*` codes on error.

### 3.2 Storage

Singleton .bss slots (same pattern as libpdx-argv's ParsedArgs, one
layer up so mv's rename dispatch does not have to re-parse the flag
array):

```
mv_argv_src         : u64   (positional 0 pointer)
mv_argv_dst         : u64   (positional 1 pointer)
mv_argv_verbose     : u64   (1 iff -v seen)
mv_argv_interactive : u64   (1 iff -i seen)
mv_argv_dry_run     : u64   (1 iff --dry-run seen)
```

`mv_argv_reset` zeroes them so a subsequent parse starts empty.

### 3.3 Flag whitelist

M1 accepts exactly three flags: `v`, `i`, `dry-run`. Anything else
returns `MV_ARG_UNKNOWN_FLAG` with the offending index recorded to
`mv_argv_error_flag_index` for the M2 stderr diagnostic renderer.
The whitelist is a byte-by-byte compare against the three canonical
names (same idiom libpdx-argv uses for `pdx-schema`); no runtime
table lookup is needed at M1.

## 4. `Rename` module (src/rename.pdx)

### 4.1 Contract

```
rename_same_dir(src: u64, dst: u64) -> u64
```

- Open a PdxFS TXN with `PXT_MODE_MODIFY` (see paideia-os
  `src/kernel/core/cap/kind_pdxfs_txn.pdx`).
- Unlink the dirent named by `src` from its parent directory.
- Link a fresh dirent named by `dst` at the same parent, pointing at
  the same inode.
- Commit the TXN. Either both writes land or neither does.
- Returns 0 on success, `MV_RN_STUB` on the M1 happy path (validated,
  no live TXN), or one of the `MV_RN_*` codes on error.

### 4.2 M1 skeleton

M1 gates `src != 0 && dst != 0` and returns `MV_RN_STUB` (0xFFFFEB20)
— the "we validated the args, we would open a TXN and thread an
unlink + link through it if the userspace sys_pdxfs_txn_open wrapper
existed, but it doesn't yet" signal. This mirrors libpdx-elevate's
`ELVC_STUB` idiom and shell's `LR_STUB` / `EX_STUB`:

- PdxFS v1 is scheduled at R42 in the paideia-os release table, and
  `KIND_PDXFS_FILE` / `KIND_PDXFS_TXN` (0x195 / 0x196) have landed as
  substrate rows (paideia-os `src/kernel/core/cap/kind_pdxfs_file.pdx`,
  `kind_pdxfs_txn.pdx`), but the userspace-side sys_pdxfs_txn_open /
  sys_pdxfs_link / sys_pdxfs_unlink / sys_pdxfs_txn_commit wrappers
  are not yet linked into a mv-facing library. Faking a rename at M1
  would either mutate the caller's argv (defeating the mv-is-a-tool
  invariant) or manufacture an outcome mv did not actually observe
  (defeating D3 audit-first at M3 when the audit hook goes live).

- A skeleton that returns `MV_RN_STUB` validates every part of the
  call graph that is not the substrate boundary: the argv gate, the
  counter dispatch (bumps `MV_ST_INVOCATIONS` on entry,
  `MV_ST_TXN_OPENS` on the would-open path, `MV_ST_ERRORS` on
  reject), the return-code plumbing. M2 flips the sys_pdxfs_* calls
  in one place; every consumer already spells the entry point
  correctly.

`rename_same_dir` bumps `MV_ST_INVOCATIONS` on every entry,
`MV_ST_TXN_OPENS` on the would-open happy path, and `MV_ST_ERRORS`
on every reject path so mv's own stats table records the failure
without needing the caller to touch a journal.

### 4.3 M2 call graph (documented, not yet built)

At M2, `rename_same_dir` will:

1. Resolve `src` against the caller's `KIND_PDXFS_FILE(write,
   <src-parent>)` cap set to find the parent directory inode.
2. Resolve `dst` similarly against the `<dst-parent>` cap; for the
   M1-shape same-dir case the two are the same inode and only one
   descriptor is opened.
3. `sys_pdxfs_txn_open(mode=PXT_MODE_MODIFY, snap_gen=<current>)`
   returns a fresh KIND_PDXFS_TXN slot in the caller's cap_table.
   Refusal maps to `MV_RN_TXN_OPEN_FAIL`.
4. `sys_pdxfs_unlink(txn, src_parent, src_basename)` removes the
   source dirent inside the TXN (not durable until commit).
   Refusal maps to `MV_RN_UNLINK_FAIL`.
5. `sys_pdxfs_link(txn, dst_parent, dst_basename, src_inode)` adds
   the destination dirent inside the SAME TXN, pointing at the
   inode the unlink just released from its old name.
   Refusal maps to `MV_RN_LINK_FAIL`.
6. `sys_pdxfs_txn_commit(txn)` drains the WAL group and makes both
   writes durable atomically. Refusal maps to `MV_RN_COMMIT_FAIL`
   and the TXN moves to `PXT_STATE_ABORTED` — the caller sees no
   visible change (source keeps its old name; destination never
   appeared).
7. On `-v`, print `"mv <src> <dst>"` to KIND_TTY(write). On
   `--dry-run`, skip steps 3-6 entirely and print the same line.
8. Return 0.

M3 wires the MoveRecord[] semantic-pipe emission + libpdx-audit
journal write BEFORE step 6 so the audit record is durable
regardless of the commit outcome (I5 uplift: undo replay is
`mv <dst> <src>` even if the primary op failed mid-stream on a
device write).

### 4.4 Cross-directory + cross-device evolution (M2-M3)

M2-001 lands cross-directory same-device move: the txn threads the
unlink through `src_parent` and the link through `dst_parent`
(distinct dir inodes, same underlying device — one TXN still).
M2-002 lands the cross-device degrade path: `sys_pdxfs_link` refuses
across devices, so mv falls back to internal cp+rm (still under ONE
TXN scope so a mid-copy failure leaves the source intact). M3-004
lands the cross-user boundary case via libpdx-elevate.

## 5. Return-code band `0xFFFFEBxx`

```
0xFFFFEB00  MV_OK                  general success sentinel (unused at M1)
0xFFFFEB10  MV_ARG_STUB            (reserved; unused at M1)
0xFFFFEB11  MV_ARG_BAD_ARGC        argc == 0
0xFFFFEB12  MV_ARG_BAD_ARGV        argv == 0
0xFFFFEB13  MV_ARG_TOO_FEW_POS     positional count != 2 (need <src> <dst>)
0xFFFFEB14  MV_ARG_UNKNOWN_FLAG    flag not in {v, i, dry-run}
0xFFFFEB15  MV_ARG_PARSE_ERR       libpdx-argv reported an ERR_* code
0xFFFFEB20  MV_RN_STUB             Rename.M1: validated, no live TXN yet
0xFFFFEB21  MV_RN_BAD_SRC          src == 0
0xFFFFEB22  MV_RN_BAD_DST          dst == 0
0xFFFFEB23  MV_RN_TXN_OPEN_FAIL    M2+: sys_pdxfs_txn_open refused
0xFFFFEB24  MV_RN_UNLINK_FAIL      M2+: sys_pdxfs_unlink refused inside TXN
0xFFFFEB25  MV_RN_LINK_FAIL        M2+: sys_pdxfs_link refused inside TXN
0xFFFFEB26  MV_RN_COMMIT_FAIL      M2+: sys_pdxfs_txn_commit refused
```

The band sits below shell's `0xFFFFECxx` and above libpdx-elevate's
`0xFFFFEA00..0xFFFFEA0F` so a downstream consumer can tell which
layer refused the operation from the high two bytes of the return
alone.

## 6. Stats slot map

```
0  MV_ST_INVOCATIONS   mv_argv_parse / rename_same_dir entries
1  MV_ST_RENAMES       successful same-dir renames (M2+)
2  MV_ST_TXN_OPENS     sys_pdxfs_txn_open calls (M2+); would-open at M1
3  MV_ST_TXN_COMMITS   sys_pdxfs_txn_commit successes (M2+)
4  MV_ST_TXN_ABORTS    sys_pdxfs_txn_commit failures (M2+)
5  MV_ST_ERRORS        any error / reject path from any mv module
6..7 reserved          M2 (cross-dir move, cross-device degrade)
```

## 7. paideia-as conformance

Every function in mv src/ obeys the constraints in
`design/kernel/paideia-as-conformance.md` (paideia-os):

- Module names PascalCase basename (`Mv`, `MvArgv`, `Rename`); no
  directory prefix.
- No `test` mnemonic; every zero-check uses `cmp reg, 0`.
- Every `cmp reg, imm` uses `imm ≤ 0x7FFFFFFF`. The M1 modules compare
  against small immediates only (`cmp rdi, 0`, `cmp rax, 8`); the
  KIND ordinals (`0x190`, `0x195`, `0x196`) are `mov rax, imm32`
  emissions, not compares.
- Byte reads use `xor rax, rax; mov_b rax, [ptr]` (#1248 mitigation).
  M1's MvArgv flag-name compare uses this pattern; the substrate
  bodies at M2 pick it up for dirent-name reads.
- SysV push/pop parity preserved. All M1 skeleton functions are LEAF
  functions (Mv::mv_reset, mv_note, mv_stat) or match matched pushes
  with pops around the nested `mv_note` calls (MvArgv::mv_argv_parse
  saves r12/r13 for argc/argv; Rename::rename_same_dir saves r12/r13
  for src/dst). One `sub rsp, 8` bracket where a call is nested,
  same idiom as `elevate_client_lookup_broker`.

## 8. Testing

Tests land at M4 (per §5.7 M4 in the plan doc). M1 ships
`tests/README.md` as a placeholder describing the fixture matrix M4
will populate:

- Same-dir rename (O(1) path): open TXN, unlink+link, commit; verify
  source gone + destination present.
- Cross-dir same-device move (M2): link-unlink atomicity across two
  parents inside one TXN.
- Cross-device degrade path (M2): internal cp+rm fallback; verify
  --verbose emits the `was_cross_device` diagnostic.
- TXN-abort mid-move (M4): source restored, no dest artifact.
- Undo replay correctness (M4): `mv <dst> <src>` recovers the
  starting state across all four cases (same-dir, cross-dir,
  cross-dev, cross-user).
- Cross-user elevate flow (M4): mv onto a path whose parent belongs
  to another user runs through the libpdx-elevate hop.
