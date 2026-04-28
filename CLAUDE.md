# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Chrome MV3 extension that lets a user sign in to farcaster.xyz from desktop using a browser EVM wallet (MetaMask, ClearWallet, etc.) instead of the mobile QR flow. Built with Svelte 5 + Vite + `@crxjs/vite-plugin`, packaged with Bun. Continuation of the deprecated `warp-easy-login-browser-extension`.

## Commands

Package manager is **Bun** (not npm/pnpm). `bun.lock` is the lockfile.

- `bun run dev` — Vite dev server (popup only; the extension itself needs a build to be loaded into Chrome).
- `bun run build` — full extension build. This is a 4-step pipeline and the order matters:
  1. `compile-inject` — `bun build src/client/inject.ts` to IIFE in `src/client/inject.js`.
  2. `content` — `bun build src/client/content.ts` to IIFE in `src/client/content.js`.
  3. `vite build` — bundles popup + manifest into `dist/`.
  4. `post-build` (`scripts/post-build.ts`) — rewrites `dist/manifest.json` so `content_scripts[0].js[0]` points at the IIFE output, copies `src/client/content.js` into `dist/`, and deletes Vite's `dist/assets/content*` chunks. Without this step the content script will fail to load — never skip it.
- `bun run check` — `svelte-check` against `tsconfig.json` (TypeScript + Svelte typecheck; there is no separate test suite).
- `bun run pretty` — Prettier on the repo.
- `bun run pub` / `bun run only-pub` — full release flow (build → bump → tag → `gh release create`). Requires being on `main` and the `gh` CLI; see `scripts/version-release.ts` and `scripts/create-release.ts`. Don't run these casually.

To load locally: `bun run build`, then load `dist/` as an unpacked extension in `chrome://extensions`.

## Architecture

The extension has **three execution contexts** and the auth flow crosses all of them. Read all three files together when changing the login flow — a change in one usually requires changes in another.

### 1. Popup (`src/action-page/`, `src/pages/`, `src/components/`, `src/lib/`)
- Svelte 5 (runes mode: `$state`, `$props`, `$bindable`). Entry is `src/action-page/index.ts` which mounts `pages/main.svelte`.
- The "Sign in" button in `pages/main-tab.svelte` does **not** sign anything itself. It uses `chrome.scripting.executeScript` to inject a small function into the active farcaster.xyz tab that calls `window.postMessage({type:'farcaster-login'}, '*')`. This is the trigger that kicks off the in-page flow.
- `src/lib/utils.ts` wraps `chrome.storage.sync` for popup options (theme, last tab); `src/lib/extension-runtime.ts` is a thin wrapper for `chrome.tabs.create`.

### 2. Content script (`src/client/content.ts`)
- Declared in `src/manifest.json` for `https://farcaster.xyz/*`, `all_frames: true`, `run_at: document_start`.
- Two jobs:
  - Inject `src/client/inject.js` into the page's main world via a `<script src=chrome.runtime.getURL(...)>` tag (the file is listed under `web_accessible_resources` so the page can load it).
  - Bridge: listen for `window.postMessage` events whose `type` is one of the four `messges.ts` constants and forward them to the service worker via `chrome.runtime.sendMessage`.
- The content script must be IIFE-bundled (not ESM), which is why `compile-inject` and `content` use `bun build --format iife` and `post-build` swaps the manifest path.

### 3. Injected page script (`src/client/inject.ts`)
- Runs in the page's main world so it can access `window.ethereum`. **This is the only place that touches the wallet or the Farcaster API.**
- Listens for `{type:'farcaster-login'}` postMessages from the same window, then:
  1. `eth_requestAccounts` → custody address.
  2. Builds a deterministic `generateToken` payload via `serialize()` (custom canonical JSON: keys sorted, no whitespace, rejects `NaN`/`Infinity`). The Farcaster API verifies the signature against this exact byte sequence — **do not change `serialize()` without matching the server's expectation**.
  3. `personal_sign` the serialized payload with the custody address.
  4. POST to `https://client.farcaster.xyz/v2/auth` with `Authorization: Bearer eip191:<base64(sig)>`.
  5. On success, write `{secret, expiresAt}` into IndexedDB at `localforage / keyvaluepairs / auth-token` (this is the schema farcaster.xyz reads to consider the user logged in), then `window.location.reload()`.
- Note: `expiresAt` is computed as `Date.now() + TOKEN_TTL_MS` (1 year). The same value goes into the signed payload and the IndexedDB record — they must match. If the Farcaster API ever rejects a 1-year TTL, lower `TOKEN_TTL_MS`. (Earlier versions hardcoded `1777046287381` and broke on 2026-04-24.)

### Service worker (`src/service-worker.ts`)
- Pure notification dispatcher. Listens via `chrome.runtime.onMessage` and shows a desktop notification per status (`AUTH_SUCCESS` / `NO_WALLET` / `SIG_DENIED` / `NO_AUTH_TOKEN`). No network, no key handling.

### Shared message constants
`src/client/messges.ts` (note the typo in the filename — kept intentionally; importers depend on it). Both `content.ts` and `service-worker.ts` import from it. The injected script declares its own copies as local consts because it's bundled standalone as IIFE.

## Conventions and gotchas

- **Path alias `@/*` and `src/*`** both resolve to `./src/*` (see `tsconfig.json` and `vite.config.ts`). `@/...` is preferred in Svelte/popup code.
- **i18n:** `manifest.json` uses `__MSG_appName__` / `__MSG_appDesc__`. Strings live in `_locales/en/messages.json`. Add new locales under `_locales/<code>/`.
- **Manifest permissions** (`storage`, `notifications`, `tabs`, `activeTab`, `scripting`) are deliberately minimal. The extension only runs on `https://farcaster.xyz/*` (content script + web-accessible resource match). Don't broaden these without a real reason — keeping the surface small is part of the project's value proposition.
- **Don't add transaction-signing or typed-data signing.** The whole flow uses only `personal_sign` so a custody-key signature can never move funds. The commented-out `ethTypeSign` block in `inject.ts` is intentionally dead — leave it dead unless you have a strong reason.
- **`.prettierrc`** sets the project style; run `bun run pretty` before committing non-trivial changes.
- **Releases** are GitHub Releases of a zip containing `dist/` + `LICENSE` + `README.md` + `PRIVACY_POLICY.md` (see `scripts/constants.ts`). The Chrome Web Store listing is uploaded manually from that zip.
