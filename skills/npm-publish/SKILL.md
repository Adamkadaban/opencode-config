---
name: npm-publish
description: End-to-end release workflow for publishing JavaScript/TypeScript packages to the npm registry. Use when cutting a release, bumping versions, configuring `package.json` for distribution, setting up trusted publishing via GitHub Actions OIDC (with provenance), tagging dist-tags (latest/next/beta), recovering from a botched publish (deprecate/unpublish), or troubleshooting auth/2FA/OTP issues. Covers npm, pnpm, yarn, and bun equivalents.
---

# npm Publish — Release Workflow

Ship a package to https://registry.npmjs.org safely and reproducibly. Default path is **automated release via GitHub Actions using npm Trusted Publishers (OIDC)** with build provenance, falling back to local publishing with a granular access token + 2FA when CI isn't available.

## When to Use

- Publishing a new package for the first time
- Cutting a versioned release of an existing package
- Setting up CI to publish on tag push
- Adding/changing `dist-tag`s (`latest`, `next`, `beta`, `canary`)
- Auditing a `package.json` before first publish (files, exports, bin, types)
- Diagnosing 403/E402/EOTP/ENEEDAUTH errors
- Deprecating or unpublishing a bad release

## Pre-flight checklist

Run **all** of these before `npm publish`. Stop at the first failure.

1. **Clean working tree** — `git status` must be empty; you publish from `HEAD`.
2. **On the release branch** — usually `main`; never publish from a feature branch unless intentional.
3. **Pull latest** — `git pull --ff-only`.
4. **Install fresh** — `npm ci` (or `pnpm i --frozen-lockfile`, `yarn install --immutable`, `bun install --frozen-lockfile`).
5. **Lint + typecheck + test** — whatever the project uses; CI-equivalent locally.
6. **Build** — produce `dist/` (or whatever `files` points at). Confirm output exists.
7. **Inspect the tarball** — `npm pack --dry-run` (see [Tarball inspection](#tarball-inspection)). This is the single most important sanity check.
8. **Check current published version** — `npm view <pkg> version` to avoid version collisions.
9. **Check you're logged in** (local publish only) — `npm whoami`.

## package.json essentials

A correct `package.json` prevents 90% of "I published broken stuff" incidents.

```json
{
  "name": "@scope/pkg",
  "version": "1.2.3",
  "description": "...",
  "license": "MIT",
  "repository": { "type": "git", "url": "git+https://github.com/owner/repo.git" },
  "homepage": "https://github.com/owner/repo#readme",
  "bugs": { "url": "https://github.com/owner/repo/issues" },
  "keywords": ["..."],
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./package.json": "./package.json"
  },
  "bin": { "mycli": "./dist/cli.js" },
  "files": ["dist", "README.md", "LICENSE"],
  "engines": { "node": ">=20" },
  "publishConfig": {
    "access": "public",
    "provenance": true,
    "registry": "https://registry.npmjs.org"
  },
  "scripts": {
    "build": "...",
    "prepublishOnly": "npm run build && npm test"
  },
  "sideEffects": false
}
```

Key points:

- **`files`** is an allowlist. Combined with `.npmignore` it determines tarball contents. Prefer `files` only — it's explicit.
- **`exports`** wins over `main`/`module` on modern Node. Always declare `types` *first* in each conditions object (TypeScript resolution is order-sensitive).
- **`publishConfig.access: "public"`** is REQUIRED for scoped packages (`@scope/x`), otherwise `npm publish` fails with E402.
- **`publishConfig.provenance: true`** turns on Sigstore provenance attestations when publishing from a supported CI (GitHub Actions OIDC). Free, verifiable supply-chain signal.
- **`prepublishOnly`** runs on `npm publish` but NOT on `npm install` — safe place for build+test gating.
- **`sideEffects: false`** enables tree-shaking. Only set if your code truly has no top-level side effects.

For CLI packages, ensure the bin file has `#!/usr/bin/env node` and is executable (`chmod +x`).

## Versioning

Follow [SemVer](https://semver.org). Use the package manager to bump — never hand-edit `package.json` for releases (it skips the git tag).

| Tool  | Patch              | Minor              | Major              | Pre-release         |
| ----- | ------------------ | ------------------ | ------------------ | ------------------- |
| npm   | `npm version patch`| `npm version minor`| `npm version major`| `npm version prerelease --preid=beta` |
| pnpm  | `pnpm version patch` | …                | …                  | `pnpm version prerelease --preid=beta` |
| yarn  | `yarn version --patch` | `--minor`      | `--major`          | `--prerelease`      |
| bun   | `bun pm version patch` | …              | …                  | `bun pm version prerelease --preid=beta` |

These commands: bump `version`, create a commit (`chore(release): vX.Y.Z`), and create an annotated git tag (`vX.Y.Z`). Push with `git push --follow-tags`.

For projects using **Changesets**, **semantic-release**, or **release-please**, defer to those; this skill is for hand-driven or simple CI-driven releases.

## Tarball inspection

Always do this once, every release:

```bash
npm pack --dry-run --json | jq '.[0] | {name, version, size, unpackedSize, files: [.files[].path]}'
```

Look for:

- **No source maps unless intended** (`*.map`)
- **No tests, fixtures, `.github/`, `.vscode/`, `tsconfig.json`** unless you want them
- **No `.env`, `.npmrc`, secrets** — if you see one, **abort**. Add to `files`/`.npmignore` and re-check.
- **README.md, LICENSE present**
- **Reasonable size** — if it jumped 10× from last release, find out why.

Run `npm pack` (no `--dry-run`) to write the tarball, then `tar -tzf <pkg>-<version>.tgz` to inspect on disk.

## Authentication

### Trusted Publishing (recommended) — GitHub Actions OIDC

No long-lived token. npm verifies your workflow's OIDC identity. Provenance is automatic.

1. **Configure on npmjs.com**: package settings → "Publishing access" → "Trusted Publisher" → add GitHub repo `owner/repo` and workflow filename (e.g. `release.yml`).
2. **Disable "Require 2FA for publishing"** *for that publisher* (otherwise CI can't publish), or use the "automation" 2FA mode. Keep 2FA enabled on your account.
3. **Workflow** — see [release.yml example](#github-actions-workflow-trusted-publisher).

### Local publish — granular access token + OTP

Used for first-time publish (before trusted publisher exists), or when not using CI.

1. **Account 2FA**: enable on npmjs.com → Account → Two-Factor Authentication. Choose "Authorization and writes" (will prompt for OTP on publish).
2. **Granular token** (preferred over Classic): npmjs.com → Access Tokens → Generate New Token → Granular. Scope to specific package(s), set short expiry, "Read and write" permission.
3. **Login**: `npm login` (browser flow) or write `//registry.npmjs.org/:_authToken=NPM_TOKEN` to `~/.npmrc`. **Never commit `.npmrc`.**
4. **Publish**: `npm publish --otp=123456` (or omit `--otp` and npm will prompt).

Verify auth: `npm whoami`. List token scope: `npm token list`.

## Publish commands

```bash
# Standard publish to "latest" tag
npm publish

# Scoped package, first publish — must be public
npm publish --access public

# Pre-release: do NOT pollute "latest"
npm publish --tag next

# With provenance (auto in CI w/ OIDC; explicit flag works locally w/ token + repo)
npm publish --provenance

# Publish to a tarball you've already built and inspected
npm publish ./pkg-1.2.3.tgz

# Dry run: prints what WOULD happen, no upload
npm publish --dry-run
```

Equivalents:

| Action        | npm                       | pnpm                | yarn (berry)            | bun                |
| ------------- | ------------------------- | ------------------- | ----------------------- | ------------------ |
| Publish       | `npm publish`             | `pnpm publish`      | `yarn npm publish`      | `bun publish`      |
| With tag      | `npm publish --tag next`  | `pnpm publish --tag next` | `yarn npm publish --tag next` | `bun publish --tag next` |
| Pack          | `npm pack`                | `pnpm pack`         | `yarn pack`             | `bun pm pack`      |
| Whoami        | `npm whoami`              | `pnpm whoami`       | `yarn npm whoami`       | —                  |

`pnpm publish` runs `pnpm build` automatically if defined and additionally validates the workspace. Yarn berry requires `yarn npm publish`, NOT `yarn publish` (that's the v1 command).

## GitHub Actions workflow (trusted publisher)

`.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags: ['v*.*.*']

permissions:
  contents: read
  id-token: write   # REQUIRED for OIDC + provenance

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org
      - run: npm ci
      - run: npm test
      - run: npm run build
      - run: npm publish --provenance --access public
```

Trigger by pushing a tag: `npm version patch && git push --follow-tags`. The tag push fires the workflow; `npm publish` authenticates via OIDC against your configured trusted publisher.

If not using trusted publishing, replace the last step's auth with:

```yaml
      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

`setup-node` with `registry-url` writes the `.npmrc` reading `NODE_AUTH_TOKEN`.

## dist-tags

Tags label versions. Default is `latest`. Users get `latest` from `npm install <pkg>`.

```bash
npm dist-tag ls <pkg>                  # list tags
npm dist-tag add <pkg>@1.2.3 latest    # promote a version to latest
npm dist-tag add <pkg>@2.0.0-rc.1 next # mark a pre-release
npm dist-tag rm <pkg> beta             # remove a tag
```

Common pattern: publish RCs as `--tag next`, then `npm dist-tag add pkg@2.0.0 latest` once stable.

## Recovery

### Bad release just shipped

- **Within 72 hours, no dependents**: `npm unpublish <pkg>@<version>` removes it. After 72h or with dependents, npm refuses.
- **Otherwise**: publish a fixed patch version immediately, then `npm deprecate <pkg>@<bad-version> "Broken; use >=X.Y.Z"`. Deprecation shows a warning on install but doesn't break existing users.
- **For security issues**: also file an advisory at https://github.com/<owner>/<repo>/security/advisories.

### Wrong dist-tag

```bash
npm dist-tag add <pkg>@<good-version> latest   # demote bad release from latest
```

### Yanked the wrong file list

Bump version (`npm version patch`), fix `files`/`exports`, re-publish. Never try to re-publish the same version — npm refuses.

## Common errors

| Error                          | Cause / Fix                                                                            |
| ------------------------------ | -------------------------------------------------------------------------------------- |
| `E402 Payment Required`        | Scoped pkg without `--access public` or `publishConfig.access`. Add it.                |
| `E403 Forbidden`               | Name taken, or token lacks write scope, or 2FA-for-publish enabled and OTP missing.    |
| `EPUBLISHCONFLICT`             | Version already published. Bump version.                                               |
| `EOTP`                         | OTP required: `npm publish --otp=123456`.                                              |
| `ENEEDAUTH`                    | Not logged in: `npm login` or set `NODE_AUTH_TOKEN`.                                   |
| `ERR_PNPM_PUBLISH_NO_VERSIONS` | `package.json` missing version, or `private: true` set.                                |
| `provenance` errors in CI      | Missing `permissions: id-token: write`, or workflow not registered as trusted publisher. |
| `404 Not Found` on first publish | Scoped pkg, scope not yet linked, or org doesn't exist. Create org first on npmjs.com.|

## Verifying after publish

```bash
npm view <pkg>                        # metadata
npm view <pkg> versions --json        # all versions
npm view <pkg> dist-tags              # current tag mapping
npm view <pkg>@<version>              # specific version's manifest
```

Then in a clean dir:

```bash
mkdir /tmp/verify && cd /tmp/verify
npm init -y
npm i <pkg>@<version>
node -e "console.log(require('<pkg>'))"
```

For provenance-enabled releases, the npmjs.com page shows a "Provenance" badge linking to the source commit and workflow run.

## Quick checklist (paste into release PR)

- [ ] Tests, lint, typecheck pass locally
- [ ] `CHANGELOG.md` updated
- [ ] `npm pack --dry-run` reviewed; no secrets, reasonable size
- [ ] Version bumped via `npm version <type>`
- [ ] `git push --follow-tags`
- [ ] CI release workflow green
- [ ] `npm view <pkg> version` matches
- [ ] Smoke install in clean dir works
- [ ] GitHub Release notes published
