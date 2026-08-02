# Local Native Addon Findings

This note records the native-addon mismatch found in the local OMP fork on macOS arm64.

## Finding

The active launcher is the fork checkout:

```text
~/.bun/bin/omp
  -> ~/omp-forked/packages/coding-agent/scripts/omp
```

The checkout is:

```text
repository: git@github.com:jaydgoss/oh-my-pi.git
branch:     fix/openrouter-byok-zero-cost
OMP source: 17.2.2
```

The fork's TypeScript source imports native diff functions from `@oh-my-pi/pi-natives`:

- `packages/coding-agent/src/edit/diff.ts` imports `diffLines`.
- `packages/coding-agent/src/edit/modes/components/diff.ts` imports `diffWords`.
- `packages/natives/native/index.js` exports `diffLines`, `diffWords`, and `astGrep` from the native binding.

However, the local native addon is a symlink to an older globally installed package:

```text
packages/natives/native/pi_natives.darwin-arm64.node
  -> ~/.bun/install/global/node_modules/@oh-my-pi/pi-natives-darwin-arm64/pi_natives.darwin-arm64.node

leaf package version: 17.0.7
fork source version:  17.2.2
```

Loading the fork confirms the mismatch:

```text
typeof astGrep   === "function"
typeof diffLines === "undefined"
typeof diffWords === "undefined"
```

This produces the TUI errors:

```text
diffWords is not a function
diffLines is not a function
```

AST-grep is not the direct cause. The older native addon exports `astGrep`, but does not export the newer native diff functions. The `astGrep.enabled: true` setting only makes the AST functionality available; it does not supply or remove the diff exports.

The OMP log records these as `Tool renderer failed` errors. The edit itself can still apply; the failure is in rendering the edit diff afterward.

## Related upstream work

- [PR #1825 — sync pi-natives on OMP update](https://github.com/can1357/oh-my-pi/pull/1825) describes the same class of coding-agent/native-addon version drift.
- [Issue #4385 — stale native addon staging/cache](https://github.com/can1357/oh-my-pi/issues/4385) is related to old native binaries being reused.
- No exact upstream or fork issue matching `diffWords is not a function` was found.

## Suggested solution

Build the native addon from this fork so the Rust binary matches the TypeScript source and package version.

### Install prerequisites once

```bash
brew install bazelisk
rustup toolchain install nightly-2026-07-28 \\
  --component rustfmt clippy rust-analyzer
```

The checkout pins the Rust toolchain in `rust-toolchain.toml` and Bazel in `.bazelversion`. If Homebrew's Rustup shim is needed, use the path documented in `README.local.md`.

### Build from the fork

```bash
cd ~/omp-forked
PATH="$(brew --prefix rustup)/bin:/opt/homebrew/bin:$PATH" \\
  bun run build:native
```

The build script writes the host addon into `packages/natives/native/`. The generated `.node` file is ignored and must not be committed.

### Verify the loaded exports

```bash
cd ~/omp-forked
bun -e 'import * as n from "@oh-my-pi/pi-natives"; console.log({ diffLines: typeof n.diffLines, diffWords: typeof n.diffWords, astGrep: typeof n.astGrep })'
```

Expected result:

```text
{ diffLines: "function", diffWords: "function", astGrep: "function" }
```

Also verify that the addon is no longer resolving to the old global 17.0.7 leaf package:

```bash
readlink packages/natives/native/pi_natives.darwin-arm64.node
stat -f '%z bytes %Sm' packages/natives/native/pi_natives.darwin-arm64.node
```

### Relaunch OMP

After the build, restart the OMP process so the native module is loaded again:

```bash
omp --version
```

Then perform a harmless edit and confirm that the edit result renders without a `diffWords` or `diffLines` error.

## Maintenance rule

After rebasing or updating the fork:

1. Run `bun install --frozen-lockfile`.
2. Re-run `bun run build:native` whenever `packages/natives` or the Rust crates change.
3. Verify all three exports before using the edit tool.
4. Keep the launcher linked to the fork with `sh scripts/link-omp.sh`.
5. Do not commit generated `.node` binaries or rely on a globally installed native package from another OMP version.
