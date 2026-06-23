# dataverse-auth — Modernization Changelog (final)

## Overview

Modernized the `dataverse-auth` npm package (v2.0.8): upgraded all runtime
dependencies, migrated the project from CommonJS to **ES Modules**, replaced
`node-fetch` with Node's native `fetch`, and removed the ESLint / Jest / Prettier
toolchain entirely.

**Final state:** `npm outdated` is empty, `npm audit` reports **0 vulnerabilities**,
and `npm run build` (tsc) compiles cleanly. `npm run build` is now the only
verification step — there is no linter, formatter, or test runner.

> This document describes the **net result** (original → final). A few
> intermediate steps taken during the work were later superseded and are **not**
> reflected here — most notably Jest/ESLint/Prettier were briefly upgraded before
> being removed, and a temporary `js-yaml` npm `override` and several transitional
> `tsconfig` settings (`node10`, `ignoreDeprecations`, jest type globals) were
> dropped. See the relevant sections for details.

---

## Security Vulnerabilities Fixed

| Advisory | Severity | Package | Resolution |
|---|---|---|---|
| GHSA-w5hq-g745-h8pq | Moderate | `uuid` (via `@azure/msal-node <=5.1.4`) | Fixed by upgrading `@azure/msal-node` to v5.2.5 |
| GHSA-h67p-54hq-rp68 | Moderate | `js-yaml <=3.x` (via jest → `babel-plugin-istanbul` → `@istanbuljs/load-nyc-config`) | **Eliminated by removing Jest entirely.** The vulnerable copy only existed inside the Jest toolchain. (A temporary `overrides` block forced `js-yaml ^4.2.0` while Jest was still present; it was removed along with Jest, since the vulnerable dependency tree no longer exists.) |

---

## Final Dependency State

### Runtime Dependencies

| Package | Before | After | Notes |
|---|---|---|---|
| `@azure/msal-node` | ^1.11.0 | ^5.2.5 | Fixes `uuid` CVE; major upgrade |
| `@azure/msal-common` | ^7.1.0 | ^16.0.0 | Required peer dep for msal-node v5 |
| `electron` | ^33.2.1 | ^42.0.0 | Major version bump |
| `chalk` | ^4.1.0 | ^5.6.2 | v5 is ESM-only — enabled by the ESM migration |
| `cryptr` | ^6.0.3 | ^6.0.3 | Unchanged |
| `enquirer` | ^2.3.6 | ^2.3.6 | Unchanged |
| `minimist` | ^1.2.6 | ^1.2.6 | Unchanged |
| `node-fetch` | ^2.x | _removed_ | Replaced by Node's global `fetch` (Node 18+) |

### Dev Dependencies

| Package | Before | After | Notes |
|---|---|---|---|
| `typescript` | ^4.7.3 | ^6.0.3 | Required for msal-node v5 type defs; see TypeScript 6 section |
| `@types/node` | _(transitive)_ | ^26.0.0 | Now an explicit devDependency (see node-fetch section) |
| `@types/cryptr` | ^4.0.1 | ^4.0.1 | Unchanged |
| `@types/minimist` | ^1.2.2 | ^1.2.2 | Unchanged |
| `genversion` | ^3.1.1 | ^3.1.1 | Unchanged |

### Dependencies Removed Entirely

- **Testing:** `jest`, `ts-jest`, `@types/jest`
- **Linting:** `eslint`, `@eslint/js`, `@typescript-eslint/eslint-plugin`, `@typescript-eslint/parser`, `typescript-eslint`, `eslint-config-prettier`, `eslint-plugin-prettier`, `eslint-plugin-sonarjs`
- **Formatting:** `prettier`
- **HTTP:** `node-fetch`, `@types/node-fetch`

---

## ESLint, Jest, and Prettier Removal

The linting, testing, and formatting toolchains were removed from the project
entirely. The project now has no linter, formatter, or test runner — `npm run build`
(tsc) is the only verification step.

### Removed files

- `eslint.config.js`, `.eslintrc.json`
- `jest.config.js`
- `.prettierrc.json`
- `src/__tests__/` (the test directory and its single test file)

### Related changes

- **`package.json`:** removed the `lint`, `lint:fix`, and `test` scripts, and the
  temporary `"overrides": { "js-yaml": ... }` block (its sole purpose was patching
  a CVE in the now-removed Jest toolchain).
- **`tsconfig.json`:** `types` reduced to `["node"]` (the transitional `"jest"`
  entry was removed along with Jest).
- **Source files:** removed now-dead `eslint-disable` comments from
  `dataverse-auth.ts` and `MsalNodeAuth.ts`.

Removing ESLint and Jest pruned 361 transitive packages; removing Prettier pruned 1 more.

---

## node-fetch → native fetch

`node-fetch` was removed in favor of Node's built-in global `fetch` (available
since Node 18; runtime is Node 26).

- **`src/MsalAuth/MsalNodeAuth.ts`:** removed `import fetch from "node-fetch"`. The
  existing `await fetch(...)` call in `whoAmI` now uses the global `fetch`;
  `.status` / `.statusText` / `.json()` are identical on the standard `Response`.
- **`package.json`:** removed `node-fetch` and `@types/node-fetch`. Added
  `@types/node` (`^26.0.0`) as an explicit devDependency — it was previously only
  pulled in transitively via `@types/node-fetch`, so removing node-fetch broke the
  `types: ["node"]` resolution until added back properly.
- Pruned 27 transitive packages.

---

## ESM Migration (CommonJS → ES Modules)

The package was migrated to **ES Modules** so `chalk` could move to v5 (ESM-only)
and to modernize to the current module standard.

### `package.json`

- Added `"type": "module"`.
- `main`, `types`, `bin`, and scripts unchanged. (`genversion --es6` already emits
  `export const version = ...`, which is valid ESM.)

### `tsconfig.json` (final values)

| Setting | Before | After |
|---|---|---|
| `module` | `commonjs` | `nodenext` |
| `moduleResolution` | `node` | `nodenext` |
| `target` | `es6` | `es2022` |
| `lib` | `["es2015","dom"]` | `["es2022","dom"]` |
| `types` | _(absent)_ | `["node"]` |
| `skipLibCheck` | _(absent)_ | `true` |
| `sourceMap` | _(absent)_ | `true` |

### Source changes

- **`.js` extensions on all 11 relative imports** across `index.ts`,
  `dataverse-auth.ts`, `InteractiveAuthenticate.ts`, and `MsalNodeAuth.ts` —
  `nodenext` requires explicit extensions (source stays `.ts`, imports point at the
  emitted `.js`).
- **`src/dataverse-auth.ts`** — replaced CommonJS-only constructs:
  - `require("electron/")` → `createRequire(import.meta.url)` then
    `require("electron/")` (preserves the binary-path-string return value used as
    the spawn command).
  - `const currentDir = __dirname` → `dirname(fileURLToPath(import.meta.url))`.
  - `import { prompt } from "enquirer"` →
    `import enquirer from "enquirer"; const { prompt } = enquirer;` — `enquirer` is
    CommonJS and its named exports aren't statically detectable under ESM (this
    failed at runtime until fixed).
- **`src/index.ts`** — the Electron main process and its
  `import { app, BrowserWindow } from "electron"` remain ESM imports, which work in
  Electron's ESM main process (v28+). This is a *different* electron import from the
  CLI's `createRequire` path resolution: the CLI runs in plain Node (where
  `require("electron")` yields the binary path), while `index.js` runs inside the
  Electron runtime (where the real API is available).

---

## TypeScript 6 Upgrade

Upgraded to **TypeScript 6.0.3**. The final compiler settings are covered by the
ESM `tsconfig.json` table above; two TS6-specific points:

- **`types: ["node"]` is mandatory.** TS6 no longer auto-injects `@types/*` globals,
  so `@types/node` must be declared explicitly or Node globals become invisible to
  the compiler.
- **`skipLibCheck: true` is required** to suppress type errors in third-party
  `.d.ts` files — specifically `@azure/msal-common` v16, whose type definitions
  otherwise fail to check under this configuration.

> Transitional note: during the upgrade, before the ESM migration, the project
> briefly used `moduleResolution: "node10"` with `ignoreDeprecations: "6.0"` (the
> renamed-and-deprecated successor to the removed `"node"` resolution). Moving to
> `nodenext` for the ESM migration removed the need for both.

### `src/index.ts` — `unknown` error parameter

TypeScript now types the `setUncaughtExceptionCaptureCallback` error parameter as
`unknown` instead of `any`. Added an explicit cast:

```ts
// Before
process.setUncaughtExceptionCaptureCallback((error) => {
  logger.Log(LogLevel.Error, error.message);

// After
process.setUncaughtExceptionCaptureCallback((error) => {
  logger.Log(LogLevel.Error, (error as Error).message);
```

---

## Verification

- `npm outdated` — empty
- `npm audit` — **0 vulnerabilities**
- `npm run build` — compiles cleanly
- `node dist/dataverse-auth.js list` — runs correctly (non-interactive path)
- The interactive Electron login flow requires live testing against a real
  Dataverse environment.

### Fallback (if a future Electron breaks ESM main)

Keep everything ESM except the Electron main entry: rename `index.ts` → `index.cts`
(emits `index.cjs`) and point root `main` at `dist/index.cjs`.
