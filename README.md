# mv

paideia-os move with PdxFS undo record + destructive-op audit.

## Status

**Release:** v1.0.0 (2026-08-22) — dual-signed release
(SHAPE-PENDING; see `RELEASE.md` for the signing-bot handoff).

See `STATUS.md` for the per-milestone rollup and
`design/tooling/r49-r50-plan.md` §5.7 in the [paideia-os](https://github.com/paideia-os/paideia-os)
repo for the wave-level design.

## Install

```
# When T-INFRA-001 (pkgs.paideia-os) is live:
pkg install mv

# Today, from source (skips the [sig.author] check per §9.3;
# still verifies the source-tree root matches the tag):
pkg install --from-source mv
```

## Documents

- `doc/mv.pdxdoc` — read via `doc mv` (I7 §2 man-equivalent).
- `caps.decl` — six required caps (I6, refusable via `--no-cap:<name>`).
- `deps.list` — five shared-library dependencies (§7).
- `design/architecture.md` — internal module shape.
- `CHANGELOG.md` — release history.
- `RELEASE.md` — signing runbook.
- `MIRROR-PUSH.md` — `pkgs.paideia-os` mirror push runbook.

## License

MIT — see `LICENSE`.
