# MIRROR-PUSH.md — mv mirror push runbook

**Wave:** R50 (Wave 2)
**Milestone:** M5 (signed 1.0 release)
**Upstream design:** `design/tooling/plan.md` §6.3 (repository
model) + §9.3 (signing pipeline) + §10 (infrastructure issues
T-INFRA-001 through T-INFRA-005) in the [paideia-os](https://github.com/paideia-os/paideia-os)
repo.

This document is the runbook for pushing mv v1.0.0 to the
`pkgs.paideia-os/main/` mirror. Two of the three legs of the
mirror push are gated on infrastructure that does not yet exist
at HEAD; the third (the git tag on the mv repo) IS the leg landed
by this milestone (M5-002) and is the interim distribution point
`pkg install --from-source mv` reads today.

## 1. What "mirror push" means

Per `design/tooling/plan.md` §6.3, package repositories are
static file trees served over HTTP(S). The default repo tree
under `https://pkgs.paideia-os/main/` looks like:

```
pkgs.paideia-os/
  main/
    index.pdxsig                     # signed manifest of {name, version, hash} tuples
    mv/
      1.0.0/
        pkg.tar                      # the package archive
        manifest.pdxsig              # the dual-signed manifest
    <other tool>/
      <version>/
        pkg.tar
        manifest.pdxsig
```

Mirror push = pushing `{pkg.tar, manifest.pdxsig}` into
`pkgs.paideia-os/main/mv/1.0.0/` after both signatures have been
computed (author + Paideia signing bot per §9.3).

## 2. Landed at M5-002 (2026-08-22)

**The `v1.0.0` git tag on `github.com/paideia-os/mv`.** This is
the interim distribution point that `pkg install --from-source mv`
reads today. It carries:

- `manifest.pdxsig` — dual-signed release manifest with
  authoritative headers + SHAPE-PENDING signature slots (see
  `RELEASE.md`).
- `doc/mv.pdxdoc` — man-equivalent for `doc mv`.
- `deps.list` — shared-library dependencies.
- `CHANGELOG.md` — v1.0.0 rollup.
- `caps.decl` — six required caps.
- `src/*.pdx` — the mv sources (source-complete + audit-complete
  + undo-complete for all four move shapes).
- `tests/*.pdx` + `tests/README.md` — M4 fixture matrix + QEMU
  smoke live-harness runbook.
- `design/architecture.md` — internal module shape.

Tag command (executed by mv.M5-002):

```
git tag -a v1.0.0 -m "mv 1.0.0 — dual-signed release (SHAPE-PENDING)"
git push origin main v1.0.0
```

## 3. Deferred until infrastructure standup

### Leg A — signing-bot re-sign (T-INFRA-002)

The `manifest.pdxsig` in the current tag carries SHAPE-PENDING
signature slots. Regenerating with real ML-DSA-65 bytes requires:

- `paideia-as release --sign` (toolchain-side feature; scheduled
  with T-INFRA-002),
- Human-in-the-loop review at the signing bot (policy per §9.3
  step 4).

After regeneration, retag the repo as `v1.0.0-signed` (same tree,
manifest.pdxsig replaced) and follow Leg B.

### Leg B — `pkgs.paideia-os/main/` mirror push (T-INFRA-001)

Once the signing bot has produced the fully-signed manifest, push
the `{pkg.tar, manifest.pdxsig}` tuple:

```
pkg push \
  --dest pkgs.paideia-os/main/mv/1.0.0/ \
  --pkg  dist/mv-1.0.0.tar \
  --sig  manifest.pdxsig
```

`pkg push` (the mirror-side of `pkg install`) is itself part of
T-INFRA-001 and lands with the pkgs.paideia-os hosting substrate.
The push updates the top-level `index.pdxsig` to include the new
tuple; `pkg upgrade` on user machines picks it up on next check.

### Leg C — announce (informational)

Once the tuple is live in `pkgs.paideia-os/main/mv/1.0.0/`, append
a `mv-1.0.0-signed` CHANGELOG note (no source change) and update
`STATUS.md`'s Current-milestone line from "M5 (signed 1.0
release) — mirror push landed (repo tag; pkgs.paideia-os pending
T-INFRA)" to "M5 (signed 1.0 release) — mirror live at
pkgs.paideia-os/main/mv/1.0.0/".

## 4. Verification recipes

### From the mv side (release engineer)

```
# Confirm the tag exists and points at the M5-002 commit:
git tag -l v1.0.0                     # prints "v1.0.0"
git rev-parse v1.0.0^{commit}         # prints the M5-002 SHA

# Confirm the release artifacts are present in the tag tree:
git show v1.0.0 --stat -- manifest.pdxsig deps.list \
    doc/mv.pdxdoc CHANGELOG.md RELEASE.md
```

### From the pkgs.paideia-os side (once T-INFRA-001 live)

```
# Confirm the tuple is present at the mirror:
curl -fsSL https://pkgs.paideia-os/main/mv/1.0.0/manifest.pdxsig
curl -fsSL https://pkgs.paideia-os/main/mv/1.0.0/pkg.tar

# Confirm the top-level index has the new entry:
curl -fsSL https://pkgs.paideia-os/main/index.pdxsig | \
    grep -E '^mv\s+1\.0\.0\s+'
```

### From the user side

```
# Today (from-source):
pkg install --from-source mv          # clones v1.0.0, builds, installs
mv --help                             # via pdx-help + doc/mv.pdxdoc
doc mv                                # renders doc/mv.pdxdoc

# When Leg A + Leg B land:
pkg install mv                        # dual-sig verify + install
pkg verify mv                         # re-verify both signatures
```

## 5. File attribution for M5-002

| File                | M5 issue | Purpose                                    |
|---------------------|----------|--------------------------------------------|
| `MIRROR-PUSH.md`    | #16      | THIS FILE (mirror push runbook).           |
| `STATUS.md`         | #16      | M5-002 landing note + rollup update.       |
| `README.md`         | #16      | Add MIRROR-PUSH.md to Documents list.      |
| `v1.0.0` git tag    | #16      | Interim distribution point (until T-INFRA-002 + T-INFRA-001 stand up). |
