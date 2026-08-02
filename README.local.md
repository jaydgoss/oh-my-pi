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

Resolve conflicts if needed, then install dependencies, rebuild native bindings, and run the relevant checks:

```sh
bun install --frozen-lockfile
PATH="$(brew --prefix rustup)/bin:/opt/homebrew/bin:$PATH" bun run build:native
bun --cwd packages/ai check
bun --cwd packages/ai test
```

Verify native addon exports match the current fork source:

```sh
bun -e 'import * as n from "@oh-my-pi/pi-natives"; console.log({ diffLines: typeof n.diffLines, diffWords: typeof n.diffWords, astGrep: typeof n.astGrep })'
```

## Self-contained native toolchain

The checkout pins Rust in `rust-toolchain.toml` (`nightly-2026-07-28`) and Bazel in `.bazelversion` (`9.2.0`). Install the build tools once:

```sh
brew install bazelisk
rustup toolchain install nightly-2026-07-28 --component rustfmt clippy rust-analyzer
export PATH="$(brew --prefix rustup)/bin:/opt/homebrew/bin:$PATH"
```

Build the native addon from this fork's source:

```sh
cd ~/omp-forked
PATH="$(brew --prefix rustup)/bin:/opt/homebrew/bin:$PATH" bun run build:native
```

The build uses Bazelisk and the repository's Bazel version pin, then installs the host addon into `packages/natives/native/`. The generated `.node` file is ignored by the repository's `.gitignore` and must not be committed.

For details on native addon version drift and troubleshooting missing exports (e.g. `diffWords is not a function`), see [README.local-native-addon.md](./README.local-native-addon.md).

After this setup, the fork does not rely on any globally installed `@oh-my-pi/pi-coding-agent` or `@oh-my-pi/pi-natives` package. The CLI source and native addon both come from this checkout; ordinary third-party npm dependencies remain managed by the workspace lockfile.

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