# tests/ — M4 fixture matrix + QEMU smoke runbook

**Wave:** R50 (Wave 2) **Milestone:** M4 (tests + smoke)
**Upstream design:** `design/tooling/r49-r50-plan.md` §5.7 in paideia-os.

M4 lands the correctness matrix mv M1-M3 promised. Three fixture
files, one per M4 issue:

| Fixture file            | Issue | Entry point                              | Purpose                                                           |
|-------------------------|-------|------------------------------------------|-------------------------------------------------------------------|
| `test_txn_abort.pdx`    | #12   | `TestTxnAbort::test_txn_abort_run`       | mv.M4-001 TXN-abort mid-move: source restored, no dest artifact.  |
| `test_undo_replay.pdx`  | #13   | `TestUndoReplay::test_undo_replay_run`   | mv.M4-002 undo replay correctness across all four move cases.     |
| `test_smoke.pdx`        | #14   | `TestSmoke::test_smoke_run`              | mv.M4-003 QEMU smoke -- shape-smoke + live-harness runbook below. |

Each entry point returns `0` on all-pass or a distinct `MV_TEST_*`
code (0xFFFFEBCx / 0xFFFFEBDx / 0xFFFFEBEx bands) on the first
failing assertion. Failure codes are documented in the corresponding
file header. A caller-driver (the eventual `mv-test-runner` binary
that will ship alongside `mv` when the R51 `pdx-testctl` harness
lands) reports the exact code so a debugger identifies the failing
vector without dumping stats manually.

## Return-code bands used by the M4 fixtures

The three fixture files reserve disjoint sub-bands within the mv
error space (0xFFFFEBxx):

| Band       | Owner              | Notes                                                       |
|------------|--------------------|-------------------------------------------------------------|
| 0xFFFFEBCx | `test_txn_abort.pdx`   | 11 codes (MV_TEST_ABT_OK..MV_TEST_ABT_ERR_LEAK).            |
| 0xFFFFEBDx | `test_undo_replay.pdx` | 8 codes (MV_TEST_UND_OK..MV_TEST_UND_STAT_FAIL).            |
| 0xFFFFEBEx | `test_smoke.pdx`       | 16 codes (MV_TEST_SMK_OK..MV_TEST_SMK_ELV_STAT).            |

The all-pass sentinel is aliased to `0` in every fixture (via
`xor rax, rax; ret`) so a POSIX-conforming caller sees exit code 0
on success. Named `MV_TEST_*_OK` constants exist for internal
spelling consistency but are never actually returned as a non-zero
value.

## Substrate state at M4 landing (2026-08-22)

Every `Pdxfs::*` trampoline is an M2 STUB (returns 0 or 1 without
issuing a syscall). This means:

- **Reachable failure vectors:** argv-gate rejects in `Rename::
  rename_same_dir` and `MvArgv::mv_argv_parse` (null pointer, unknown
  flag, wrong positional count). These bump `MV_ST_ERRORS` and
  return distinct `MV_ARG_*` / `MV_RN_*` codes without touching the
  TXN layer.
- **Unreachable failure vectors (deferred to R42 substrate):**
  every mid-TXN failure (unlink refused, link refused, commit
  refused, cross-device fallback failed). Live fault injection
  requires `sys_pdxfs_fault_inject` (proposed sysno 527) which
  ships with the R42 substrate landing. When R42 lands,
  `test_txn_abort.pdx` gains four new entry points -- one per stage
  (UNLINK / LINK / COMMIT / AUDIT) -- that arm the fault, invoke
  `Move::move_dispatch`, verify the specific `MV_MV_*` return code,
  and verify `MV_ST_TXN_ABORTS` bumped without `MV_ST_MOVES`
  bumping.

The three M4 fixtures compile + link today; the fault-injection
extension is a Round-2 amendment to `test_txn_abort.pdx` that will
follow the R42 substrate landing without touching M1-M3 code.

## QEMU smoke harness (deferred to R42 + shell.M5 + pdx-undo)

The live end-to-end smoke fixture -- the acceptance gate M4-003
promises -- has three cross-repo dependencies:

1. **R42 PdxFS v1 substrate** (paideia-os kernel): the eight
   `sys_pdxfs_*` dispatch entries at reserved band 512-527 must be
   wired so the M2 stub bodies flip to real syscall trampolines.
2. **shell.M5 dual-signed release**: the shell process that
   exec's `mv` and `pdx-undo` with the correct InitCap sidecar.
3. **pdx-undo tool** (R51): the WAL-replayer that reads the undo
   record `Undo::mv_undo_write` threaded into the TXN and invokes
   `mv <replay_src> <replay_dst>`.

Once all three land, the harness runs the following command
sequence inside QEMU:

```
$ paideia-os-image  # boot into a shell prompt
> touch a
> ls
a
> mv a b
> ls
b
> pdx-undo mv a b
> ls
a
```

Each intermediate `ls` is a semantic-pipe consumer that emits
`PdxFsDirEntry[]` records; the harness compares each record's
`name` field against the expected set. A drift (extra `b`, missing
`a`, silent shred) fails the harness with a specific
`SMK_LIVE_*` code the R51 harness ships in its own band.

The `test_smoke.pdx` fixture in this directory is the shape-smoke
that runs **today** -- it exercises the full mv pipeline against
the current stubs and verifies every subsystem (Move + Schema +
Audit + Undo + Inode + Elevate) fires exactly once. If the shape
smoke passes, main can be confident that the module wiring is
intact; a substrate-driven divergence at the live QEMU smoke is
then a substrate bug, not a mv wiring bug.

## Fixture inputs (why the tests use fixed pointer values)

All three fixture files pass small non-zero pointer values (`0x1000`,
`0x1111`, `0x2222`, `0xDEAD`, etc.) as `src` / `dst` arguments to
`Rename::rename_same_dir`, `Move::move_dispatch`, `Undo::
mv_undo_populate`, etc. The mv module functions do NOT dereference
these pointers themselves (they hand them to `Pdxfs::*` trampolines
which are M2 stubs), so the fixture values are opaque bit-patterns
the tests use as identity markers.

When R42 lands, the fixture pointers must be swapped for real path
strings pointing at PdxFS-hosted fixture files. The swap is a
mechanical edit at the top of each `test_*_run` entry point --
the assertion structure downstream does not change.

## Running the fixtures (once the runner exists)

Today the fixture entry points are not driven by a runner -- they
are entry-point declarations that ship with the mv binary. The
runner they will be driven by is:

- `mv-test-runner` -- a small paideia-as binary that links the mv
  module + the three test modules + calls each entry point in
  sequence, printing per-test pass/fail with the returned
  `MV_TEST_*` code. Filed under `paideia-os/mv-test-runner` at
  R51 alongside the `pdx-testctl` harness.

Until the runner lands, the fixture entry points serve as:

1. **Compile-time contracts:** the tests reference the mv module
   symbols by exact name; a rename or a signature change in mv
   breaks the tests' link step and is caught at build time.
2. **Documentation of the M4 acceptance matrix:** the file
   headers describe each fixture's substrate-state assumptions
   and the invariants the assertions check.
3. **Ready-to-run harness input:** once `mv-test-runner` lands,
   the fixtures run without further edits.
