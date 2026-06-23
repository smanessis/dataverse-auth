# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
npm run build          # tsc compile → dist/

# Test locally (requires a real Dataverse org URL)
npm run start org.crm.dynamics.com
npm run start list
npm run start org.crm.dynamics.com test-connection

# Bump version, build, and push tags (runs prepublishOnly → version → postversion)
npm version patch
```

> **Note:** This project has no linter, formatter, or test runner configured. ESLint, Jest, and Prettier were all removed. `npm run build` (tsc) is the only verification step.

## Architecture

The CLI has a **two-process design** for interactive authentication:

1. **Main process** (`src/dataverse-auth.ts`) — the CLI entry point. Parses args via `DataverseAuthArgs`, handles all non-interactive commands (`list`, `remove`, `test-connection`, `device-code`) directly, and for interactive auth spawns a **child Electron process**.

2. **Electron child process** (`src/index.ts`) — launched by the main process via `child_process.spawn`. Opens a `BrowserWindow` (`src/MsalAuth/InteractiveAuthenticate.ts`) that navigates to the Azure AD login page, intercepts the OAuth redirect to capture the auth code, then writes the result as JSON to stdout. The parent process reads that JSON on close.

3. **MSAL token exchange** (`src/MsalAuth/MsalNodeAuth.ts`) — after the parent receives the auth code from the child, it calls `acquireTokenByCodeMsal` to exchange it for a token via `PublicClientApplication`. Silent token refresh uses `acquireTokenSilent`.

### Token storage

- **Token cache** (`src/MsalAuth/MsalCachePlugin.ts`): MSAL tokens are serialized, AES-encrypted with `cryptr` (key = OS username), and stored in `~/dataverse-auth-cache`.
- **Environment→username lookup** (`src/MsalAuth/MsalNodeAuth.ts`): A separate JSON file at `~/dataverse-auth-users` maps environment URLs to MSAL account usernames, used to resolve the right cached account on `acquireTokenSilent`.

### Key constraints

- **The package is ESM** (`"type": "module"`, `module`/`moduleResolution: "nodenext"`). Consequences when editing source:
  - **Relative imports must carry a `.js` extension** (e.g. `import { x } from "./MsalAuth/SimpleLogger.js"`) even though the source files are `.ts`. `nodenext` errors otherwise.
  - **No `require`, `__dirname`, or `__filename`.** `src/dataverse-auth.ts` recreates them: `createRequire(import.meta.url)` to resolve the Electron binary path, and `dirname(fileURLToPath(import.meta.url))` for the directory.
  - **CommonJS deps with named exports may need a default-import + destructure.** `enquirer` is imported as `import enquirer from "enquirer"; const { prompt } = enquirer;` because its named exports aren't statically detectable under ESM. Watch for this pattern if adding CJS deps.
- The Electron main process (`dist/index.js`) runs as **ESM** — supported in Electron 28+. If a future Electron version breaks ESM main, the fallback is to make only that entry CommonJS (rename `index.ts` → `index.cts`, point `main` at `dist/index.cjs`).
- `node-fetch` was removed in favor of Node's built-in global `fetch` (Node 18+).
- `skipLibCheck: true` in tsconfig is required for compatibility with `@azure/msal-common` v16 type definitions.
- `types: ["node"]` in tsconfig is required in TypeScript 6, which no longer auto-injects `@types/*` globals. `@types/node` is an explicit devDependency (it used to be pulled in transitively via `@types/node-fetch`).
