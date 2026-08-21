# This is a fork, and this branch is the one hermes runs

`dimakrest/gbrain`, forked from `garrytan/gbrain` (MIT). Two branches, two
installs, and they are deliberately not the same version:

| Branch | Base | Runs on | Install |
|---|---|---|---|
| `mine` | `v0.46.24.0` | the laptop | source, `bun link` |
| `hermes` | `v0.45.9.0` | the hermes VM | source, pinned commit + `bun install --frozen-lockfile` |

They diverge because the two installs already did, before any fork existed.
The honest end state is one branch; it is reached by bumping hermes on purpose,
not by pretending now.

## This branch carries no patches

`hermes` is `v0.45.9.0` plus this file. That is the point: the swap from
upstream's published binary to this tree is meant to change nothing that runs,
so that when the first real patch lands it is the only variable.

Upstream's published `gbrain-linux-x64` for `v0.45.9.0` was verified against
this source tree on 2026-08-21: built with the same Bun (1.3.13, the version
`.github/workflows/release.yml` pins) from a path of the same length as CI's,
the two binaries differ in **203 bytes out of 180,611,392** — seven clusters,
every one of them the embedded build directory string. Not one byte of code.
So upstream's artifact genuinely corresponds to this tag, and this tree is that
program.

## Rules

1. **Never allocate a schema migration number.** Migration numbers are a single
   shared integer namespace with upstream. It is the one irreversible mistake.
2. **Hold the version.** Bumping hermes past 0.45.9.0 runs new migrations on
   ordinary startup with no deploy gate, and 0.46 is where the self-upgrade
   channel stops being dormant (`autopilot.ts`, `core/minions/worker.ts` both
   reach it). Both are reasons to bump deliberately, never incidentally.
3. **Keep `self_upgrade.mode = off` on the host.** The binary self-updater is
   hardcoded to `api.github.com/repos/garrytan/gbrain/releases/latest`. On a
   fork that is a channel for replacing our code with somebody else's.
4. **Keep the `upstream` remote.** `detectInstallMethod()` classifies a source
   install by grepping `.git/config` for `garrytan/gbrain`
   (`src/commands/upgrade.ts`, `detectBunLink()`). With only an `origin` of
   `dimakrest/gbrain` it misclassifies. The remote is load-bearing, not decorative.
5. **Rebase onto tags, keep patches few, small and separable.**

## Where the install lives

`infra/prod/hermes-startup.sh.tftpl` in `company-brain-2`, gated on
`var.gbrain_enabled`, pinned by `var.gbrain_commit`. The plan and the evidence
are in `infra/docs/gbrain-fork-hermes.html`.
