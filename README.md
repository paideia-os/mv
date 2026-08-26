# mv

Move (or rename) one path onto another inside a single PdxFS v1 transaction,
with an audit-journal record written before commit and a WAL undo record
whose replay is `mv <dst> <src>`.

## Synopsis

```
mv [-v] [-i] [--dry-run] <src> <dst>
```

Exactly two positionals are required. `MvArgv::mv_argv_parse` rejects any
other positional count with `MV_ARG_TOO_FEW_POS` and any flag outside the
whitelist `{v, i, dry-run}` with `MV_ARG_UNKNOWN_FLAG` (`src/argv.pdx`).
The multi-source form `mv f1 f2 … dir/` is not accepted at 1.0.0.

## Description

`mv` moves a dirent from one parent directory to another. `Move::move_dispatch`
(`src/move.pdx`) is the single entry point for all four move shapes, and every
one of them runs inside one PdxFS transaction opened in `PXT_MODE_MODIFY`:

- **same-dir** — `src_parent == dst_parent`; the unlink and the link touch the
  same parent inode.
- **cross-dir** — different parents on the same device; `pdxfs_unlink` at
  `src_parent` and `pdxfs_link` at `dst_parent` are both threaded through the
  one TXN, so commit publishes both dirent writes or neither.
- **cross-device** — `pdxfs_link` returns `PDXFS_ERR_EXDEV` (`-18`), and
  `Move::move_cross_dev_body` byte-copies the file through a 4 KiB scratch
  buffer (`mv_move_xdev_buf`) with `sys_open`/`read`/`write`/`close`, then
  unlinks the source dirent *inside the same outer TXN*. The operation is no
  longer O(1); `-v` says so on stderr.
- **cross-user** — `Inode::mv_inode_preserve` compares the source inode's owner
  key against the destination parent's. On a mismatch,
  `Elevate::mv_elevate_cross_user` asks `svc.elevate-broker` to lend
  `KIND_USER(sign, dst_owner_key)` for a 60-second window so the inode can be
  re-signed. A denied elevate is not fatal: the inode keeps its old owner tail
  and the degrade is reported under `-v`.

The pipeline inside `move_dispatch` is fixed:
`resolve_parent(src)` → `resolve_parent(dst)` → `inode_of(src)` →
`txn_open` → `unlink` → `link` (or the EXDEV fallback) → `inode_preserve` →
**MoveRecord emit** → **audit write** → **undo write** → `txn_commit`.
The audit write is a hard gate — under the D3 audit-first discipline, an `mv`
that cannot log itself is refused, and `mv_md_audit_fail` aborts the TXN so the
source dirent survives and no destination artifact appears. The undo write is
best-effort by design: a failed `pdxfs_txn_write_undo` leaves the move
committed and audited, it only means `pdx-undo` cannot replay it automatically.
The undo record itself is simply the two paths swapped
(`replay_src = dst`, `replay_dst = src`), so replaying it re-invokes `mv` in
reverse.

`mv` is a Category-A coreutil in wave R50 of paideia-os, released as `v1.0.0`
(2026-08-22, dual-signed with `manifest.pdxsig` in the SHAPE-PENDING state
described in `RELEASE.md`). It links five shared libraries per `deps.list`:
`libpdx-cap`, `libpdx-argv`, `libpdx-semantic-pipe`, `libpdx-audit`,
`libpdx-elevate`.

**Substrate caveat.** Every `Pdxfs::*` trampoline in `src/pdxfs.pdx` is still an
M2 stub that returns `0` (or `1` for handle-valued calls) without issuing a
syscall; live TXN dispatch lands with the R42 PdxFS v1 substrate in the
paideia-os kernel. The IPC path used by the audit and elevate hops
(`sys_ipc_send` = 42, `sys_svc_lookup` = 43) is real R20b substrate and is live.
There is also no `_start` frame in this tree — `mv_argv_parse` and
`move_dispatch` are both present and tested, but nothing yet reads
`mv_argv_src`/`mv_argv_dst` into the dispatcher.

```
# When T-INFRA-001 (pkgs.paideia-os) is live:
pkg install mv

# Today, from source (skips the [sig.author] check per §9.3;
# still verifies the source-tree root matches the tag):
pkg install --from-source mv
```

## Options

Flag names are matched byte-for-byte against the whitelist in
`MvArgv::mv_argv_parse`; all three are boolean (no value operand), and all
three default to `0`.

| Short | Long        | Argument | Default | Description |
|-------|-------------|----------|---------|-------------|
| `-v`  | —           | none     | `0`     | Emit the cross-device / cross-user advisories on stderr (fd 2) after a successful commit. Sets `mv_argv_verbose`. |
| `-i`  | —           | none     | `0`     | Interactive. Parsed into `mv_argv_interactive`; **no consumer at 1.0.0** — the prompt lands with the shell.M4 line reader reachable through `KIND_TTY`. |
| —     | `--dry-run` | none     | `0`     | Parse-and-validate only. Parsed into `mv_argv_dry_run`; **no consumer at 1.0.0** — `move_dispatch` does not yet gate on it. |

Two caveats that the source contradicts the `.pdxdoc` on, and which the source
wins:

- The long spellings `--verbose` and `--interactive` documented in
  `doc/mv.pdxdoc` are **not** accepted. The whitelist compares against the
  literal names `"v"`, `"i"`, `"dry-run"`; anything else returns
  `MV_ARG_UNKNOWN_FLAG` with the offending index left in
  `mv_argv_error_flag_index`.
- `-v` does not print a generic `mv <src> <dst>` line. `Verbose::mv_verbose_diag`
  emits only the two advisories, and only when the corresponding flag fired:

  ```
  mv: crossed device (cp+rm fallback, not O(1))
  mv: crossed user (inode kept old owner)
  ```

  Both can fire in one operation. Emission is best-effort — the function always
  returns `0`, even if the fd-2 write is refused.

## Semantic pipe output

Every successful move emits exactly one `MoveRecord` — 80 bytes, ten `u64`
fields — via `Schema::mv_schema_emit`. At 1.0.0 that is a raw
`pdxfs_write(fd = 1, …)` to stdout; when `libpdx-semantic-pipe` lands the same
call site switches to a schema-tagged endpoint without changing any caller.
A reader validates `magic == 0xFFFFEB5000000001` (high half = the `0xFFFFEB50`
Schema band, low half = layout version 1) before decoding.

The emit is best-effort from `move_dispatch`'s perspective: a short write
(`MV_SCH_SHORT_WRITE`) or a negative errno (`MV_SCH_EMIT_FAIL`) bumps
`MV_ST_ERRORS` but does not gate the commit. Success bumps
`MV_ST_RECORDS_EMITTED` (stats slot 11).

## Exit codes

`mv` follows the shared I4 process-exit convention:

| Exit | Meaning |
|------|---------|
| `0` | Success — MoveRecord emitted, TXN committed, undo record threaded. |
| `1` | Reserved; `mv` has no "query returned nothing" outcome. |
| `2` | User error — the `MV_ARG_*` band (`0xFFFFEB1x`). |
| `3` | System error — the `MV_RN_*` / `MV_MV_*` bands (`0xFFFFEB2x` / `0xFFFFEB3x`). |
| `4` | Capability denied — a cap from `caps.decl` is missing from the caller's InitCap sidecar. |
| `5` | Signing / verification failure — reserved; `manifest.pdxsig` is checked by `pkg`, not by `mv`. |

Internally, every function returns a `u64` in the `0xFFFFEBxx` band whose high
two bytes identify the mv layer to a downstream consumer. The sub-bands are
declared in `src/mv.pdx`; the full per-code table is in `STATUS.md`.

| Sub-band | Module | Covers |
|----------|--------|--------|
| `0xFFFFEB00` | `Mv` | `MV_OK` |
| `0xFFFFEB1x` | `MvArgv` | bad argc/argv, wrong positional count, unknown flag, libpdx-argv parse error |
| `0xFFFFEB2x` | `Rename` | M1 same-dir skeleton: bad src/dst, TXN open/unlink/link/commit refusals |
| `0xFFFFEB3x` | `Move` | resolve/inode/txn-open/unlink/link/commit failures, plus the six cross-device fallback codes `0xFFFFEB38`–`0xFFFFEB3D` |
| `0xFFFFEB4x` | `Inode` | reserved — `mv_inode_preserve` returns `0` for every outcome |
| `0xFFFFEB5x` | `Schema` | `MV_SCH_EMIT_FAIL`, `MV_SCH_SHORT_WRITE` |
| `0xFFFFEB6x` | `Audit` | `MV_AUD_LOOKUP_FAIL`, `MV_AUD_SEND_FAIL` |
| `0xFFFFEB7x` | `Undo` | `MV_UND_WRITE_FAIL` |
| `0xFFFFEB8x` | `Elevate` | `MV_ELV_LOOKUP_FAIL`, `MV_ELV_SEND_FAIL`, `MV_ELV_DENIED` |

## Capabilities

`caps.decl` declares six required capabilities, each refusable at exec via
`--no-cap:<name>`:

```
requires:
  - KIND_USER (self)
  - KIND_IPC_ENDPOINT (invoke)
  - KIND_PDXFS_FILE (write, <src-parent>)
  - KIND_PDXFS_FILE (write, <dst-parent>)
  - KIND_PDXFS_TXN (invoke, mint)
  - KIND_ELEVATE_CHANNEL (invoke, svc.elevate-broker)

declares_output_schemas: (none)
```

The effect and capability rows on the entry points, verbatim from source:

```
Move::move_dispatch          : (u64, u64) -> u64 !{mem, sysreg} @{cap, sched}
Move::move_cross_dev_body    : (u64, u64) -> u64 !{mem, sysreg} @{cap, sched}
MvArgv::mv_argv_parse        : (u64, u64) -> u64 !{mem}         @{}
Audit::mv_audit_write_move   : ()         -> u64 !{mem, sysreg} @{cap}
Undo::mv_undo_write          : (u64)      -> u64 !{mem, sysreg} @{}
Schema::mv_schema_emit       : ()         -> u64 !{mem, sysreg} @{}
Elevate::mv_elevate_cross_user : (u64)    -> u64 !{mem, sysreg} @{cap}
Inode::mv_inode_preserve     : (u64, u64) -> u64 !{mem, sysreg} @{cap}
Verbose::mv_verbose_diag     : ()         -> u64 !{mem, sysreg} @{}
```

`@{cap}` appears wherever a capability is minted or consumed — `pdxfs_txn_open`
mints a fresh `KIND_PDXFS_TXN` (`0x196`), `svc_lookup` mints a
`KIND_IPC_ENDPOINT` (`5`), and `ipc_send` consumes it. `@{sched}` marks the
paths that may block on the WAL drain or on device I/O.

## Examples

Rename in place. Same parent, one TXN, one dirent rewrite:

```
mv old.txt new.txt
```

Move into a subdirectory. `src_parent != dst_parent`, still one TXN;
`MV_ST_CROSS_DIR` bumps alongside `MV_ST_MOVES`. If `sub/` is not writable, the
exec-time `cap_manifest_verify` fails on the `KIND_PDXFS_FILE (write,
<dst-parent>)` requirement and `mv` exits `4` before any TXN opens:

```
mv file.txt sub/file.txt
```

Cross-volume move with the advisory shown. `pdxfs_link` returns `EXDEV`, the
cp+rm fallback runs inside the outer TXN, and stderr gets one line:

```
$ mv -v report.pdx /mnt/archive/report.pdx
mv: crossed device (cp+rm fallback, not O(1))
```

A rejected flag. `--force` is not in the whitelist, so parsing stops before any
filesystem work and `mv_argv_parse` returns `MV_ARG_UNKNOWN_FLAG`
(`0xFFFFEB14`), exit `2`:

```
mv --force a b
```

An audit-journal outage. `svc.audit-journal` is unreachable, `ipc_send` returns
`-104`, `mv_audit_write_move` returns `MV_AUD_SEND_FAIL` (`0xFFFFEB62`), and
`move_dispatch` aborts the TXN — `a` is still `a` and `b` never appeared:

```
mv a b
```

## Audit records

Three fixed-layout records are built in `.bss` and shipped off the process.
All fields are `u64`; all offsets are literal in the emitting assembly.

**MoveRecord** — 80 bytes, `src/schema.pdx`. Populated by
`mv_schema_populate(src_ptr, dst_ptr)`, emitted on the semantic pipe, and sent
verbatim as the `sys_ipc_send` payload to `svc.audit-journal` by
`Audit::mv_audit_write_move` before the commit.

| Offset | Field | Notes |
|--------|-------|-------|
| 0 | `magic` | `MV_MOVE_RECORD_MAGIC` = `0xFFFFEB5000000001` |
| 8 | `was_rename` | `1` iff `src_parent == dst_parent` |
| 16 | `was_cross_device` | mirrors `Move::mv_move_was_cross_device` |
| 24 | `was_cross_user` | mirrors `Move::mv_move_was_cross_user` |
| 32 | `src_ptr` | pointer to the NUL-terminated source path |
| 40 | `dst_ptr` | pointer to the NUL-terminated destination path |
| 48 | `src_inode` | the moved inode |
| 56 | `src_parent` | source parent inode |
| 64 | `dst_parent` | destination parent inode |
| 72 | `txn_handle` | lets a consumer correlate the record to PdxFS TXN state |

The path fields are opaque pointers into the caller's argv memory — `mv` does
not copy the strings into the record. Both M3 consumers write synchronously, so
the pointers are live for the duration; an inline path trailer is deferred to
the R42 substrate landing.

**UndoRecord** — 32 bytes, `src/undo.pdx`. Written into the TXN's WAL group by
`mv_undo_write(txn_handle)` via `pdxfs_txn_write_undo` (sysno 522), so it is
durable exactly when the commit succeeds and discarded with the rest of the WAL
group on abort.

| Offset | Field | Notes |
|--------|-------|-------|
| 0 | `magic` | `MV_UNDO_MAGIC` = `0xFFFFEB7000000001` |
| 8 | `replay_op` | `MV_UNDO_OP_MV` = `0x6D76`, `"mv"` packed little-endian |
| 16 | `replay_src_ptr` | the **original dst** — source of the reverse move |
| 24 | `replay_dst_ptr` | the **original src** — target of the reverse move |

**ElevateRequest** — 32 bytes, `src/elevate.pdx`. Sent to `svc.elevate-broker`
by `mv_elevate_cross_user(dst_owner_key)` on the cross-user path.

| Offset | Field | Notes |
|--------|-------|-------|
| 0 | `magic` | `MV_ELV_REQUEST_MAGIC` = `0xFFFFEB8000000001` |
| 8 | `requested_kind` | `ELV_KIND_USER` = `0x190` |
| 16 | `requested_key` | the destination owner's key to sign under |
| 24 | `duration_secs` | `MV_ELV_WINDOW_SECS` = `60` |

Note that `doc/mv.pdxdoc`, `CHANGELOG.md`, and the `manifest.pdxsig` stanzas
describe the MoveRecord as 64 bytes under schema id `0x4D56` and the undo magic
as `0x4D566E52`. Those predate the M3 implementation; the layouts above are what
`src/schema.pdx` and `src/undo.pdx` actually build and are authoritative.

## See also

- [libpdx-argv](https://github.com/paideia-os/libpdx-argv) — the flag/positional
  parser `MvArgv` wraps.
- [libpdx-audit](https://github.com/paideia-os/libpdx-audit) — user-events
  journal client.
- [cp](https://github.com/paideia-os/cp) — peer coreutil; shares the
  cross-device byte-copy discipline.
- [rm](https://github.com/paideia-os/rm) — peer coreutil; shares the audit and
  elevate integration.

In-tree: `doc/mv.pdxdoc` (read via `doc mv`), `design/architecture.md` for the
module shape, `STATUS.md` for the per-milestone rollup and the complete
return-code table, `CHANGELOG.md`, `RELEASE.md` for the signing runbook, and
`MIRROR-PUSH.md` for the `pkgs.paideia-os` mirror push.

MIT — see `LICENSE`.
