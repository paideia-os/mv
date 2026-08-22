# RELEASE.md — mv signing + release runbook

**Wave:** R50 (Wave 2)
**Milestone:** M5 (signed release)
**Upstream design:** `design/tooling/plan.md` §6.3 (repository model)
+ §9.3 (signing pipeline) + D4 (dual-signed release) in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo.

This document describes the SHAPE-PENDING → LIVE handoff for
`manifest.pdxsig`. mv v1.0.0 ships the manifest with authoritative
headers (name, version, deps, caps digest, artifact digests) and
RESERVED signature slots because the two pieces of live signing
infrastructure named in `design/tooling/plan.md` §10 do not yet
exist at HEAD:

- **T-INFRA-001** — dual-sign package repository infrastructure
  (`pkgs.paideia-os`).
- **T-INFRA-002** — signing bot host + policy for
  `paideia_root_pk`.

Once both stand up, the flow below regenerates `manifest.pdxsig`
with real ML-DSA-65 bytes, re-tags the repo as `v1.0.0-signed`,
and completes the mirror push (see `MIRROR-PUSH.md`).

## 1. Release flow (per §9.3, adapted for shape-pending state)

**Landed at v1.0.0 (this release, 2026-08-22):**

1. Author tags the repo — `git tag -a v1.0.0 -m "mv 1.0.0"` (done
   as part of `mv.M5-002`).
2. `manifest.pdxsig` at repo root carries authoritative headers +
   the SHAPE-PENDING signature slots. Every `sig_bytes` field is
   the literal `SHAPE-PENDING`; every `key_fp`, `digest`, and
   per-file digest is `pending:<label>` so a downstream tool cannot
   mistake the shape file for a valid one.
3. `pkg install --from-source mv` from `github.com/paideia-os/mv`
   at tag `v1.0.0` works today (bypasses `[sig.author]` per §9.3,
   still verifies the source-tree root against the tag).

**Deferred until T-INFRA-002 standup:**

4. `paideia-as release --sign` (needs the release-sign feature
   parity in the paideia-as toolchain) computes the author
   signature over SHA3-512 of `header_bytes || pkg_tar_bytes` per
   the manifest's `[sig.author]` stanza. Output: same file with
   real 3309-byte ML-DSA-65 signature substituted for the
   `SHAPE-PENDING` marker.
5. Author pushes the signed tuple `{pkg.tar, manifest.pdxsig}` to
   `pkgs.paideia-os` staging (see `MIRROR-PUSH.md`).
6. Paideia signing bot re-signs with `paideia_root_pk` after a
   human-in-the-loop review confirms the source-tree at `v1.0.0`
   matches the author's tag. Output: the `[sig.root]` stanza now
   holds the real 3309-byte ML-DSA-65 signature.
7. Bot moves the tuple to `pkgs.paideia-os/main/mv/1.0.0/`.
8. Author re-tags the repo `v1.0.0-signed` and appends a
   CHANGELOG note; `pkg upgrade` on user machines picks the
   release up on next check.

## 2. Regeneration recipe (for when T-INFRA-002 is live)

```
# On a machine holding the author key:
git clone https://github.com/paideia-os/mv
cd mv
git checkout v1.0.0
paideia-as release --sign \
  --author-key ~/.paideia/keys/author-sk.pdxkey \
  --input manifest.pdxsig.template \
  --output manifest.pdxsig \
  --pkg-tar dist/mv-1.0.0.tar

# Verify the SHAPE-PENDING → LIVE transition:
grep -c 'SHAPE-PENDING' manifest.pdxsig   # must print 1
                                          # ([sig.root] still pending
                                          #  until the bot re-signs)
grep -c 'pending:'      manifest.pdxsig   # must print 1
                                          # (paideia-root-pk-fingerprint
                                          #  filled in by the bot)

# Push the signed tuple to staging:
pkg push --staging --pkg dist/mv-1.0.0.tar --sig manifest.pdxsig

# Retag:
git tag -a v1.0.0-signed -m "mv 1.0.0 (author-signed)"
git push origin v1.0.0-signed
```

The bot side re-runs the same regeneration recipe with the root
key against the same source-tree commit; the `sig_alg` fields in
each stanza make the input bytes each signature covers explicit,
so two independent implementations converge on the same signature
input.

## 3. Verification recipe (for a user's `pkg install`)

```
pkg install mv                  # fails today (no signed tuple in
                                #   pkgs.paideia-os); user sees
                                #   exit 5 with a diagnostic
                                #   naming T-INFRA-001 + T-INFRA-002.

pkg install --from-source mv    # works today; clones the repo at
                                #   v1.0.0, builds locally, skips
                                #   [sig.author] check per §9.3.
                                #   Still verifies source-tree
                                #   root matches the tag unless
                                #   --trust-my-own-hash is passed.
```

## 4. Files touched by this release

| File                | M5 issue | Purpose                                    |
|---------------------|----------|--------------------------------------------|
| `manifest.pdxsig`   | #15      | Dual-signed release manifest (SHAPE-PENDING). |
| `deps.list`         | #15      | Shared-library dependencies per §6.3.      |
| `doc/mv.pdxdoc`     | #15      | Man-equivalent read by `doc mv`.           |
| `CHANGELOG.md`      | #15      | v1.0.0 entry + M1-M5 rollup.               |
| `RELEASE.md`        | #15      | THIS FILE (signing runbook).               |
| `STATUS.md`         | #15/#16  | M5 landing sections + Current milestone bump. |
| `README.md`         | #15      | Install command + release-status note.     |
| `MIRROR-PUSH.md`    | #16      | `pkgs.paideia-os` mirror push runbook.     |

## 5. Reserved for future revisions

- **1.0.1** — when R42 PdxFS v1 substrate lands, the Pdxfs::*
  trampolines flip from M2 STUBs to real syscall dispatch and the
  QEMU acceptance smoke (`tests/README.md` §"QEMU smoke harness")
  runs end-to-end. Release regenerates every per-file digest in
  the manifest.
- **1.0.0-signed** — the identical source tree with real ML-DSA-65
  signature bytes replacing the two SHAPE-PENDING slots. No source
  change, no CHANGELOG bullet beyond the signing note.
