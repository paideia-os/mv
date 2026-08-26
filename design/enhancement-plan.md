# mv — Enhancement Plan (v1.x)

Status: authored 2026-08-25, against `main` @ `a65fd74` (tag `v1.0.0`).
Scope: source-verified audit of `paideia-os/mv` at HEAD, the gap between what
the tree claims and what it does, and the concrete issue plan for the
`Enhancement v1.x — mv` milestone.

Every claim below was checked against real source in this tree. Where a
document and the source disagree, the source is treated as authoritative and
the disagreement is itself recorded as a defect.

---

## 1. Current state

`mv` is a Category-A coreutil from wave R50, tagged `v1.0.0` on 2026-08-22.
Sixteen issues across five milestones (M1–M5) are closed; **zero issues are
open**. The tree is eleven `src/*.pdx` modules, three `tests/*.pdx` fixtures,
and a per-repo `tools/build.sh` that compiles each `.pdx` to an ELF64 object.

What genuinely works at HEAD:

- `MvArgv::mv_argv_parse` (`src/argv.pdx`) — a real byte-by-byte flag
  whitelist over `{"v", "i", "dry-run"}` with positional-count gating and a
  distinct error code per failure.
- `Move::move_dispatch` (`src/move.pdx`) — the four move shapes (same-dir,
  cross-dir, cross-device, cross-user) threaded through one TXN scope, with a
  distinct `MV_MV_*` code per stage and an explicit abort on every post-open
  failure.
- `Move::move_cross_dev_body` — a real 4 KiB byte-copy loop over live
  `sys_open`/`read`/`write`/`close` (these are R14b substrate, not stubs).
- `Audit::mv_audit_write_move` — a real `sys_svc_lookup` + `sys_ipc_send` pair
  (R20b substrate, live) gating the commit under the D3 audit-first rule.
- `Verbose::mv_verbose_diag` — correct: both advisory literals and both length
  constants were hand-verified (46 and 40 bytes, matching the string bodies).

### 1.1 The substrate boundary

Ten of the eighteen `Pdxfs::*` trampolines in `src/pdxfs.pdx` are M2 stubs that
return `0` (or `1` for handle-valued calls) **without issuing a syscall**:
`pdxfs_txn_open`, `txn_commit`, `txn_abort`, `unlink`, `link`,
`resolve_parent`, `inode_of`, `device_of`, `inode_owner`,
`inode_rebind_owner`. Live TXN dispatch waits on the R42 PdxFS v1 substrate.

This boundary is the single most important fact about the tree, because it
means **the entire failure half of `move_dispatch` is currently unreachable**.
Every stub returns success, so no test in `tests/` can enter an error branch
that is gated on a stub's return value. Defect 2.1 lives in exactly that blind
spot.

---

## 2. Defects found

### 2.1 HIGH — stack-frame corruption in `move_dispatch` (`src/move.pdx:741`)

**This is a correctness bug with arbitrary-control-flow consequences, not a
leak.**

`Move::move_dispatch` uses the odd-count five-push prologue idiom
(`src/move.pdx:431-435`):

```
push rbx; push r12; push r13; push r14; push r15;
```

Entry `rsp % 16 == 8`; five pushes land `rsp % 16 == 0`. There is **no**
`sub rsp, N`. Verified mechanically: `move.pdx` contains exactly ten `push`
instructions (five in each of its two framed functions) and **zero** `sub rsp`
instructions anywhere in the file. The matching epilogue is therefore exactly
five pops.

Sixteen of the seventeen framed exit paths in `move.pdx` do exactly that.
The seventeenth, `mv_md_commit_fail`, does this:

```
mov rax, 0xFFFFEB37;                 // MV_MV_COMMIT_FAIL
add rsp, 8;                          // <-- line 741, UNMATCHED
pop r15; pop r14; pop r13; pop r12; pop rbx;
ret;
```

**Mechanism.** After the prologue the frame reads, upward from `rsp`:

| Offset | Holds |
|--------|-------|
| `[rsp+0]`  | saved `r15` |
| `[rsp+8]`  | saved `r14` |
| `[rsp+16]` | saved `r13` |
| `[rsp+24]` | saved `r12` |
| `[rsp+32]` | saved `rbx` |
| `[rsp+40]` | **return address** |

`add rsp, 8` slides `rsp` up one slot before the pops, so every pop is
off-by-one-slot and the sequence walks one word past the saved registers:

- `pop r15` ← saved `r14`
- `pop r14` ← saved `r13`
- `pop r13` ← saved `r12`
- `pop r12` ← saved `rbx`
- `pop rbx` ← **the return address**
- `ret`     ← pops the caller-frame word *above* the return address and jumps to it

The consequences are, in order of severity:

1. **`ret` transfers control to an unintended address.** The jump target is
   whatever `u64` sits immediately above `move_dispatch`'s return address in
   the caller's frame — a caller-controlled stack word, not a code pointer.
   This is a stack-frame corruption of the CWE-787-adjacent class: in the
   benign case it faults, in the general case it is a control-flow transfer to
   an address an attacker who controls caller stack contents can influence.
2. **`rbx` is clobbered with a code address**, and `r12`–`r15` are restored
   from the wrong slots — a four-register SysV callee-save violation that
   silently corrupts the caller's state even if the `ret` somehow lands.
3. `MV_MV_COMMIT_FAIL` (`0xFFFFEB37`) is never actually observed by any
   caller, because control never returns to the caller.

**Root cause.** A copy-paste of the *even*-count alignment idiom. Its sibling
`Rename::rename_same_dir` (`src/rename.pdx`) legitimately uses two pushes plus
`sub rsp, 8`, and therefore legitimately carries `add rsp, 8` in its epilogue
(lines 163, 172, 181). `move_dispatch` uses the odd-count idiom and needs no
such adjustment. The file's own header comment at `move.pdx:79-83` even
mis-describes the prologue as "5 pushes + `sub rsp, 8`", which is not what the
code at 431-435 does — the stale comment is the fingerprint of the same
confusion that produced line 741.

**Why it shipped green.** `pdxfs_txn_commit` is an M2 stub
(`xor rax, rax; ret`), so it always returns `0`, so `cmp rax, 0; jne
mv_md_commit_fail` at `move.pdx:601-602` is never taken. The branch is
**unreachable today** and becomes reachable the instant the R42 substrate swaps
that body to `mov rax, 513; syscall; ret` — at which point any real commit
failure (ENOSPC, WAL-group full, device I/O error, cap revoked mid-TXN) enters
a corrupting epilogue. The M4 fixture matrix cannot catch it either:
`tests/test_txn_abort.pdx` Block C asserts the *happy* path and can never reach
this label while the stub returns success.

This is the archetypal latent defect — ships green, detonates on substrate
landing, in the error path that exists precisely to keep a failed move safe.

**Fix.** Delete line 741. One line. Also correct the stale prologue comment at
`move.pdx:79-83`. Scoped as **mv.ENH-001**, filed HIGH.

### 2.2 HIGH — no `_start` frame: `mv` is not an executable

`src/mv.pdx:37-41` states that M1 does not ship the `_start` entry frame and
that "the mv repo's `_start` lands at M2". M2, M3, M4 and M5 all closed; it
never landed. Verified: no `_start` symbol is defined anywhere in `src/`.

Consequently nothing reads `mv_argv_src` / `mv_argv_dst` into `move_dispatch`,
nothing maps the `0xFFFFEBxx` return bands onto the I4 process exit codes
`{0,2,3,4}` that `README.md` documents, and no `sys_exit` is ever issued.
`mv` v1.0.0 is a library of well-formed modules with no way to run any of them.
The README is honest about this; the tag is not. Scoped as **mv.ENH-002**.

### 2.3 HIGH — no cwd-relative path resolution

Verified by exhaustive grep over `src/`, `doc/`, `design/`, `README.md` and
`STATUS.md`: there is **no** occurrence of `getcwd`, `chdir`, `cwd`, or any
leading-`/` or path-prefix test anywhere in the repository. Argv path pointers
pass verbatim from `mv_argv_parse` into `pdxfs_resolve_parent` as opaque `u64`.

`mv` therefore has no notion of a current working directory. Relative operands
(`mv a b`, `mv f.txt sub/`) resolve correctly only if the kernel's
`sys_pdxfs_resolve_parent` (sysno 517) itself applies the caller's cwd — and
that syscall is a stub that does not exist yet. R86 landed `sys_chdir` /
`sys_getcwd` *after* mv shipped v1.0.0, so mv has never been reconciled against
a real cwd. Every relative-path invocation from a shell with a non-root cwd
will resolve against the wrong parent.

This is shared with `rm` and `cp` and wants one convention, not three.
Scoped as **mv.ENH-003**.

### 2.4 MEDIUM — silent destination clobber, uncovered by the undo record

`move_dispatch` calls `pdxfs_link` at the destination with no prior existence
check. If `dst` already names a file, the move overwrites it with no prompt, no
advisory, and no record of what was destroyed.

The tree's audit-first design does **not** cover this case, and it is worth
being precise about why. The `UndoRecord` (`src/undo.pdx`) is 32 bytes holding
only `{magic, replay_op, replay_src_ptr, replay_dst_ptr}` — the two paths
swapped. Replaying it runs `mv <dst> <src>`, which restores the *moved* file to
its original name. It says nothing about the inode that used to live at `dst`;
that inode is unlinked and unrecoverable, and no undo record was ever written
for it. The `MoveRecord` likewise carries no `dst_existed` or
`clobbered_inode` field.

So `-i` is **not** redundant against the audit trail — the audit trail is
post-hoc recovery for the source, and it structurally cannot recover a
clobbered destination. The two are complementary, and the clobber case is the
one gap the audit design leaves genuinely open. See §3 for the decision.
Scoped as **mv.ENH-004** (guard) and **mv.ENH-005** (prompt).

### 2.5 MEDIUM — `deps.list` and `README.md` overstate library linkage

`deps.list` names five libraries and `README.md:56-58` repeats the claim
verbatim ("It links five shared libraries per `deps.list`"). Resolved every
external call target against local definitions:

| Library | Claimed consumer | Actually called? |
|---------|------------------|------------------|
| `libpdx-argv` | `src/argv.pdx` | **Yes.** `parse_argv` and `reset` are undefined locally — genuine external linkage. |
| `libpdx-audit` | `src/audit.pdx` | **No.** `audit.pdx` defines its own `pub let ipc_send` and `pub let svc_lookup` syscall trampolines. |
| `libpdx-elevate` | `src/elevate.pdx` | **No.** `elevate.pdx` reuses `audit.pdx`'s local `ipc_send`/`svc_lookup`; no `elevate_client_*` symbol is referenced. |
| `libpdx-semantic-pipe` | `src/schema.pdx` | **No.** `schema.pdx` calls the local `pdxfs_write` (raw `sys_write`). The README already concedes this in its own §Semantic pipe. |
| `libpdx-cap` | `src/mv.pdx`, `src/inode.pdx` | **No.** `sys_user_self` and `pdxfs_inode_rebind_owner` are both local trampolines in `src/pdxfs.pdx`. No `libpdx-cap` symbol appears. |

One of five is real. Because `tools/build.sh` compiles each `.pdx` to a
separate `.o` and never links, the unresolved-external check that would have
caught this never runs. `pkg install mv` enforces `deps.list` semver
constraints at install time, so four spurious entries are four spurious
install-time failure modes. Scoped as **mv.ENH-008**.

### 2.6 LOW — stale record layouts in `.pdxdoc`, `CHANGELOG.md`, `manifest.pdxsig`

`README.md:279-282` already documents this: those three files describe the
`MoveRecord` as 64 bytes under schema id `0x4D56` and the undo magic as
`0x4D566E52`, whereas `src/schema.pdx` builds 80 bytes under
`0xFFFFEB5000000001` and `src/undo.pdx` uses `0xFFFFEB7000000001`. The README
is correct and the other three are pre-M3 fossils. A signed manifest that
describes the wrong record layout is a supply-chain-relevant inaccuracy, not
just a typo. Scoped as **mv.ENH-009**.

### 2.7 LOW — `--dry-run` parsed, never consumed

`mv_argv_dry_run` is set by the parser and read by nothing; `move_dispatch`
does not gate on it. `mv --dry-run a b` performs a real, committed move. This
is worse than an unimplemented flag — it is a flag whose documented purpose is
"do not open the TXN" (`doc/mv.pdxdoc:62-64`) and whose actual behavior is a
full destructive move. Scoped as **mv.ENH-007**.

---

## 3. The `--verbose` / `--interactive` decision

### 3.1 What the gap actually is

`doc/mv.pdxdoc:55-60` documents `-v, --verbose` and `-i, --interactive`. The
parser whitelist (`src/argv.pdx:212-243`) accepts only the literal names
`"v"`, `"i"`, `"dry-run"`. So the *long spellings* are rejected with
`MV_ARG_UNKNOWN_FLAG`. Separately, the `.pdxdoc` mis-describes what `-v` does:
it claims `-v` prints `mv <src> <dst>` to stderr on success, whereas
`Verbose::mv_verbose_diag` emits only the two advisory lines and only when the
corresponding flag fired.

So there are three distinct sub-gaps, and they do not get the same answer.

### 3.2 Decision: implement `--verbose`, implement `-i` in two stages, strip the false claims

**`--verbose` long spelling — IMPLEMENT.** The short `-v` already works and its
output is correct (both length constants verified against their literals). The
only gap is the long spelling, and the parser already proves it handles
multi-character flag names, because `--dry-run` is a working long-only flag
matched byte-by-byte over seven characters. Adding `"verbose"` alongside `"v"`
is roughly eight lines in the same idiom. Stripping the documented long form
instead would be the wrong trade: it is cheap to honor, the doc has promised it
since M5, and the org's own flag convention pairs short with long.

One caveat worth recording for the coordinating pass: sibling `rm` reportedly
has a verbose-output defect. mv's verbose path is **not** the same code and is
not defective — mv should not adopt rm's implementation. The right shared
convention is a common advisory-emitter helper, which is an org-level item
noted in §6, not something mv should fork into.

**`-i` interactive — IMPLEMENT, split across two issues.** §2.4 establishes
that this is not duplicative of the audit trail: the undo record cannot recover
a clobbered destination, so the clobber is a genuine, uncovered data-loss
vector and pre-hoc prevention is the only thing that closes it. But the prompt
itself needs a TTY line reader (`KIND_TTY`, shell.M4) that does not exist in
this tree, and mv has no `_start` and therefore no stdin plumbing at all
(§2.2). Blocking the whole safety improvement on that is the wrong call, so it
splits:

- **mv.ENH-004** — the half that needs no TTY: detect that `dst` exists before
  linking, add `MV_MV_DST_EXISTS` (`0xFFFFEB3E`), record `dst_existed` in the
  `MoveRecord`, and refuse by default. This is the safety-critical half and it
  is implementable now against the existing `pdxfs_inode_of` trampoline.
- **mv.ENH-005** — the prompt itself, gated on `KIND_TTY`, depending on
  ENH-004. `-i` stays parsed until then.

**Strip from `doc/mv.pdxdoc` — the false claims, not the flags.** Two
statements are simply untrue of the source and should be removed rather than
implemented: the claim that `-v` prints `mv <src> <dst>` (it does not, and the
two-advisory behavior is the better design — a per-move echo line belongs to a
shell, not to a tool that emits a structured `MoveRecord`), and the claim at
`doc/mv.pdxdoc:219-221` that POSIX `-f` and `-n` are "covered by `-i`", which
is false twice over: `-i` has no consumer at all, and no-clobber is ENH-004's
job. Scoped as **mv.ENH-006**.

---

## 4. Gap versus what real paideia-os users need at HEAD

Ranked by what actually blocks a user typing `mv a b` at a shell prompt today:

1. **It cannot run.** No `_start` (§2.2). This is the whole ballgame.
2. **It would move the wrong files if it did run.** No cwd resolution (§2.3).
3. **It can destroy data silently.** Clobber uncovered by undo (§2.4).
4. **Its error paths are unsafe the moment they become live.** §2.1 — and R42
   is what makes both the tool functional *and* this branch reachable, so the
   fix must land before or with that substrate, never after.
5. `--dry-run` actively lies (§2.7) — a user reaching for the safe-preview flag
   gets a destructive move.
6. Multi-source `mv f1 f2 dir/` is unimplemented (README is explicit). This is
   the single most-used real-world `mv` form and it is absent. **mv.ENH-011.**

Items 1–3 are the difference between a tagged artifact and a usable tool.

---

## 5. Issue plan

Milestone: **Enhancement v1.x — mv** (milestone #6), issues #17-#27.

| ID | Issue | Title | Effort | Deps | Priority |
|----|-------|-------|--------|------|----------|
| ENH-001 | #17 | fix stack-frame corruption in `move_dispatch` commit-fail epilogue | XS | none | **HIGH** |
| ENH-002 | #18 | ship the `_start` entry frame and I4 exit-code mapping | L | none | HIGH |
| ENH-003 | #19 | resolve relative operands against the kernel cwd (R86) | M | #18 | HIGH |
| ENH-004 | #20 | destination-clobber guard + `dst_existed` in MoveRecord | M | none | MED |
| ENH-005 | #21 | `-i` interactive confirm via `KIND_TTY` | M | #18, #20 | MED |
| ENH-006 | #22 | accept `--verbose`; strip the false `-v`/`-i` claims from `.pdxdoc` | S | none | MED |
| ENH-007 | #23 | make `--dry-run` actually gate the TXN | S | none | MED |
| ENH-008 | #24 | correct `deps.list` / README to the one library actually linked | S | none | MED |
| ENH-009 | #25 | reconcile stale record layouts in `.pdxdoc` / CHANGELOG / manifest | S | #20 | LOW |
| ENH-010 | #26 | stack-discipline regression fixture over every framed epilogue | M | #17 | MED |
| ENH-011 | #27 | multi-source form `mv f1 f2 ... dir/` | L | #18, #20 | MED |

ENH-010 is deliberately paired with ENH-001: a one-line fix to a branch no test
can reach is not a durable fix. The fixture should assert push/pop parity per
framed exit path so the next copy-paste of the wrong alignment idiom fails the
build rather than shipping green for three milestones.

---

## 6. Companion paideia-os monorepo work (flagged, not filed here)

Noted for the coordinating pass; **not** filed as issues in this repo.

- **R42 PdxFS v1 substrate** — sysnos 512–522. Ten of mv's trampolines are
  stubs until this lands. ENH-001 must land before or with it.
- **`sys_pdxfs_resolve_parent` cwd semantics** — ENH-003 needs a decision on
  whether cwd resolution happens kernel-side in sysno 517 or userspace-side in
  each tool. One answer, applied to `mv`/`rm`/`cp` together.
- **`sys_pdxfs_fault_inject` (sysno 527)** — `STATUS.md:97` defers live
  fault-injection to it. Without it, no test can reach `mv_md_commit_fail`,
  which is exactly how §2.1 survived to a tag.
- **`bin_seeds` entry for `mv`** — needed once ENH-002 produces an executable.
- **`KIND_TTY` / shell.M4 line reader** — blocks ENH-005.
- **Shared advisory-emitter convention** — mv, rm and cp each hand-roll stderr
  advisories; rm's is reportedly defective. One helper, one convention.

---

## 7. Verdict on the `v1.0.0` tag

**Not defensible as shipped — but the honest fix is a re-tag, not a walk-back.**

The stack-frame corruption alone would not justify revoking a tag; it is a
one-line defect in a branch that is unreachable in the shipped configuration,
and that is a normal thing to find and patch. Taken by itself it argues for
`v1.0.1`.

The tag fails on §2.2. `mv` v1.0.0 has no `_start`, which means the artifact
tagged `1.0.0` cannot be executed at all — it is a set of object files, not a
program. A `1.0.0` on a coreutil carries exactly one implicit promise, that the
named command runs, and this one cannot. Compounding it, `deps.list` declares
four library dependencies the source never calls (§2.5), so the manifest that
`pkg` verifies at install time does not describe the artifact.

The right disposition is narrow and specific:

1. Do **not** revoke `v1.0.0`. It is a real, signed, honestly-documented
   milestone of module-level work, and the README already states the two
   central limitations plainly. The tag's defect is that it over-claims
   *maturity*, not that its contents are fraudulent.
2. Amend `STATUS.md` and `CHANGELOG.md` to state at the top that v1.0.0 is a
   **module-complete, not-yet-executable** release — matching what `README.md`
   already says and what the tag currently implies otherwise.
3. Treat ENH-001 + ENH-002 + ENH-003 as the gate for a `v1.1.0` that is the
   first genuinely runnable `mv`.

The README, notably, is already the most honest document in the tree: it
volunteers the missing `_start`, the stub substrate, the doc-vs-source flag
contradictions, and the stale record layouts. The gap being corrected here is
between that README and the tag it sits under.
