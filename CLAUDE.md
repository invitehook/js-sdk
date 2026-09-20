# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Use **pnpm**, not npm. The repo is a pnpm workspace (`pnpm-workspace.yaml`); there is no `package-lock.json`.

```bash
pnpm install                          # tsc comes from the root devDependency
pnpm -r build                         # tsc -p . in every package (src/ -> dist/)
pnpm -r test                          # node --test in every package

pnpm --filter invitehook build        # single package
pnpm --filter invitehook test
cd packages/invitehook && node --test test/index.test.js   # single test file

pnpm publish:check                    # dry-run both registries (bypasses the clean-tree checks)
pnpm publish:npm                      # pnpm -r publish -> both npm packages
pnpm publish:jsr                      # pnpm dlx jsr publish -> @invitehook/js-sdk
pnpm publish:all                      # npm, then jsr
```

The real publish targets deliberately do not pass `--no-git-checks` / `--allow-dirty`, so both tools apply their own clean-tree checks; only `publish:check` bypasses them. `pnpm -r publish` skips the private root and no-ops on a version already on the registry, so it is safe to re-run.

Tests are plain JS and import from `../dist/index.js`, so **the build must run before the tests**. A stale or missing `dist/` produces import errors, not test failures.

## Architecture

The same library ships to three registries, all at version `0.0.0` — these are placeholder publishes that reserve the `invitehook` name and the `@invitehook` scope, not releases.

npm, via two workspace packages:

- `packages/invitehook` → `invitehook` (unscoped)
- `packages/scoped` → `@invitehook/invitehook` (scoped, `publishConfig.access: public`)

Their `src/`, `test/`, and `tsconfig.json` are identical. **Any source or test change to one package must be mirrored in the other** — nothing enforces this automatically.

JSR, via `jsr.json` at the repo root → `@invitehook/js-sdk`. It exports `./mod.ts`, which re-exports `./packages/invitehook/src/index.ts`, so JSR ships the TypeScript source of the unscoped package directly (no build step) and picks up new exports automatically. `publish.include` keeps the tarball to those files plus `README.md` and `assets/` — without it JSR would sweep in the whole monorepo, since `jsr.json` sits at the root. `assets/` is in the list because the README references the banner by a relative path, which only resolves on JSR if the file ships in the tarball.

Both tsconfigs extend `tsconfig.base.json` (ES2020, NodeNext modules, `strict`, `declaration`). npm packages are ESM (`"type": "module"`) and publish only `dist/`.

Note that `mod.ts` uses a `.ts` import specifier, which JSR requires and tsc rejects under NodeNext. It is deliberately outside both tsconfig `include` globs; validate it with the jsr dry-run, not with `tsc`.

## Current state

Both `src/index.ts` files are empty while the tests expect an exported `hello(name?)` returning `"Hello, <name|world>!"`, so `pnpm -r test` currently fails until the implementation lands. The published artifacts are intentionally empty modules. The README is a placeholder holding the `assets/coming-soon.svg` banner; the intended public API is `useInviteHook()`, unimplemented.
