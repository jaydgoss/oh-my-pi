# Local Forked OMP Setup

This directory is the local checkout used to run the patched fork of OMP.
It is intentionally separate from ordinary workspaces.

## Current checkout

- Fork: https://github.com/jaydgoss/oh-my-pi
- Branch: `fix/openrouter-byok-zero-cost`
- Canonical remote: `upstream` → `https://github.com/can1357/oh-my-pi.git`
- Personal remote: `origin` → `git@github.com:jaydgoss/oh-my-pi.git`
- Current patch commit: `35d938e`

The launcher is linked as:

```text
~/.bun/bin/omp
  → ~/omp-forked/packages/coding-agent/scripts/omp
```

The wrapper runs the checkout's TypeScript source, so OMP uses this branch rather than the globally published package or a hand-edited generated bundle.

## Patch behavior

`packages/ai/src/providers/openai-shared.ts` preserves the catalog cost estimate only when OpenRouter reports:

```text
usage.cost === 0
is_byok === true
```

A zero cost from a non-BYOK response remains authoritative. Positive reported costs continue to replace the catalog estimate through the existing scaling logic.

Regression coverage is in `packages/ai/test/openai-responses-openrouter.test.ts` for both Chat Completions and Responses usage shapes, including the non-BYOK zero-cost case.

## Relinking OMP

Run this after moving the checkout or reinstalling dependencies:

```sh
cd ~/omp-forked
bun install --frozen-lockfile
bun --cwd=packages/coding-agent link
sh scripts/link-omp.sh
omp --version
```

The repository's `scripts/link-omp.sh` safely recreates the global launcher link.

## Updating from upstream

Keep the patch branch rebased on the canonical repository:

```sh
cd ~/omp-forked
git fetch upstream
git switch fix/openrouter-byok-zero-cost
git rebase upstream/main
```

Resolve conflicts if needed, then run the relevant checks:

```sh
bun install --frozen-lockfile
bun --cwd packages/ai check
bun --cwd packages/ai test
```

If native sources or the OMP version changed, rebuild the native addon when Rust/Bazel is available:

```sh
bun run build:native
```

On this machine, the matching prebuilt platform addon is installed as `@oh-my-pi/pi-natives-darwin-arm64@17.2.2` and linked into `packages/natives/native/`. That local binary link is ignored through `.git/info/exclude`; it must not be committed.

After a rebase, update the fork branch with the lease-protected force push required by rewritten history:

```sh
git push --force-with-lease origin fix/openrouter-byok-zero-cost
```

Do not pull from `origin` to update the patch. `origin` is the personal fork; `upstream` is the source of new OMP updates.

## Returning to the published OMP package

The dev link can be replaced with a published package if needed:

```sh
rm -f ~/.bun/bin/omp
bun install -g @oh-my-pi/pi-coding-agent@<version>
omp --version
```

Re-run the relinking commands above to return to the forked checkout.