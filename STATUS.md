# mv — status

**Wave:** R50 (Wave 2)
**Current milestone:** M1 (design + skeleton) — in progress

See `design/tooling/r49-r50-plan.md` §5.7 in paideia-os for the full
breakdown.

## M1 — design + skeleton

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
- `src/rename.pdx` (issue #3): PENDING.

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

## Milestone rollup

| ID              | Title                                                         | State  |
|-----------------|---------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + caps.decl (src-parent + dst-parent write + TXN)    | LANDED |
| M1-002 (#2)     | argv surface via libpdx-argv (mv [-v|-i|--dry-run])           | LANDED |
| M1-003 (#3)     | first runnable: same-dir rename in single TXN                 | OPEN   |

## Upstream substrate (paideia-os, at HEAD 2026-08-21)

- `KIND_USER = 0x190` — landed in R48.M1 at
  `src/kernel/core/cap/kind_user.pdx`.
- `KIND_IPC_ENDPOINT = 5` — R20b substrate at
  `src/kernel/core/cap/kind.pdx:72`.
- `KIND_PDXFS_FILE = 0x195` — R48b substrate-prep at
  `src/kernel/core/cap/kind_pdxfs_file.pdx`.
- `KIND_PDXFS_TXN = 0x196` — R48b substrate-prep at
  `src/kernel/core/cap/kind_pdxfs_txn.pdx`.
