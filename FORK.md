# This is a fork

`dimakrest/gbrain`, forked from `garrytan/gbrain` (MIT). Branch **`mine`** is the
line that is actually installed; it is one upstream release tag plus whatever
commits sit on top.

    master          untouched mirror of upstream. Never commit here.
    mine            v<tag> + our commits. This is what `gbrain` runs.
    upstream/*      fetched, read-only. Rebase targets.

Forked at **v0.46.24.0** on 2026-08-21, with zero patches — the swap was a
byte-for-byte no-op, verified with `diff -rq` against the previously installed
copy.

## How it is installed

Source mode (`bun-link`), which is the only fork-safe method:

    ~/.bun/bin/gbrain                          -> ~/development/gbrain/src/cli.ts
    ~/.bun/install/global/node_modules/gbrain  -> ~/development/gbrain

Nothing else changed. Same Postgres (`127.0.0.1:5470`), same 655 pages, same
6,569 chunks — the data never moved and nothing re-embedded.

**The `upstream` remote is load-bearing.** `detectInstallMethod()` returns
`bun-link` only because `detectBunLink()` walks up to `.git/config` and finds
the substring `garrytan/gbrain` in it. Our `origin` is `dimakrest/gbrain`, so
that substring comes from the `upstream` remote alone. Delete that remote and
gbrain stops recognising this as a source clone.

Verify after any install change:

    bun -e "process.argv[1]='$HOME/development/gbrain/src/cli.ts';
            console.log((await import('./src/commands/upgrade.ts')).detectInstallMethod())"
    # must print: bun-link

## Rules

1. **Never allocate a schema migration number.** The only irreversible mistake
   here. Migration numbers are one shared integer namespace; if we ship `v133`
   and upstream also ships `v133`, this database can never take an upstream
   upgrade again. Patches change behaviour, never storage shape. If it is truly
   unavoidable, allocate from 9000+ and record it in this file.
2. **Never let the install become the `binary` method.** `src/core/binary-self-update.ts`
   fetches `api.github.com/repos/garrytan/gbrain/releases/latest` — hardcoded,
   no override — and atomically replaces the live binary. On a fork that is a
   silent revert of everything here.
3. **Never `npm i -g gbrain` / `bun add -g gbrain`.** The npm name is an
   unrelated squatter package (upstream issue #658). The global
   `package.json` dependency was deliberately emptied so `bun update -g` has
   nothing to reinstall over the link.
4. **Rebase onto tags, never `master`.** ~19 commits/day on master vs ~24
   releases/month. And `latest-stable` is a *moving* tag — always
   `git fetch upstream --tags --force`, or you rebase onto a stale copy.
5. **Keep patches few, small and separable** — one commit per concern. That is
   the whole difference between a 5-minute rebase and an afternoon. Past ~5
   commits across ~5 subsystems, upstream them or fork for real.

## Taking an upstream release

    git fetch upstream --tags --force
    git log --oneline v<old>..v<new> -- src/core/migrate.ts   # schema moved? read it first
    git log --oneline v<old>..v<new> -- src/core/ai/          # what moved under us
    git rebase v<new>
    bun install && bun run test && bun run verify
    gbrain doctor
    git push --force-with-lease origin mine

Skipping releases is free. Skipping *schema* versions is not.

## Reading the test suite

`bun run test` sizes its parallel shards to free memory. On this 24 GB Mac it
collapses to `1x1`, PGLite's WASM engine OOMs, and the headline `fail=` count is
mostly noise. **Read `.context/test-summary.txt`, never the headline number.**
The runner already re-runs suspects serially and in an oom-rescue pass; those
lines are the real signal.

Stop the jobs supervisor before a full run — several host-integration tests
(launchd, worker registry) write into the real `~/.gbrain` and fail against a
live supervisor.

    gbrain jobs supervisor stop && bun run test ; gbrain jobs supervisor start --detach

### Baseline at v0.46.24.0, zero patches

Run 2026-08-21 on this Mac, supervisor stopped, `DATABASE_URL` unset:

    [unit-parallel] elapsed=855s | pass=19022 fail=4 skip=21
    shard 1/1: pass=19022 fail=0 skip=21 rc=0      <- parallel set fully green
    serial:    rc=1 fail=4

The parallel shard passes clean. All 4 failures are serial host-integration
tests that reach for the real `~/.gbrain`, launchd, and a spawned CLI:

| Test | |
|---|---|
| `agent-scheduler-contract.serial` | shell-chain contract |
| `worker-registry.serial` | `registerWorker` writes under gbrainPath / `readWorkers` round trip |
| `autopilot-launchd-lifecycle.serial` | shimmed (all platforms) — `seedBrain init --migrate-only` exits -1 |
| `autopilot-launchd-lifecycle.serial` | REAL launchd (darwin) |

Note this is a *better* result than the same suite gives with the supervisor
running: that run reported `fail=231`, almost all of it a PGLite WASM OOM
cascade after the runner collapsed to a single shard. Stopping the supervisor
and letting the PGLite snapshot fixture load is what makes the number
meaningful. `bun run test` still exits 1 — that is expected at baseline.

After a rebase, "did I break something" means diffing against this known-red
set, not expecting green.

## Rollback

    rm ~/.bun/bin/gbrain ~/.bun/install/global/node_modules/gbrain
    bun install -g github:garrytan/gbrain#latest-stable

Backups from the migration are in `~/backups/` — a `pg_dump -Fc` of the whole
brain, a `gbrain export` of all pages, and a facts export (`gbrain export` does
**not** include facts; that gap is why the separate exporter exists).
