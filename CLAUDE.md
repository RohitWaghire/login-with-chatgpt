# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An SDK that lets an app's end users "Login with ChatGPT" and then call OpenAI models **on the user's own ChatGPT subscription** — the app supplies no OpenAI API key. It works by driving OpenAI's Codex OAuth device-code flow (`auth.openai.com`, Codex `client_id` in `packages/core/src/constants.ts`) and proxying the Codex Responses API. Access/refresh/id tokens are server secrets: encrypted at rest, never sent to the browser. The browser holds only an HttpOnly session cookie plus public user claims (account id, email, name, plan).

## Commands

Bun-first monorepo (Bun workspaces). Run from the repo root:

- `bun install` — install all workspaces
- `bun test` — run the full test suite (`bun test packages`)
- `bun test packages/core` — test one package; `bun test packages/core/test/oauth.test.ts` for one file; `bun test -t "pattern"` for one test by name
- `bun run typecheck` — `tsc --build` across all project refs (the real "does it compile" gate)
- `bun run build` — emit `dist/` for the four published packages (ordered: core → server → react → ai)
- `bun run demo` — run `examples/demo` (`bun --hot`), a full working app
- `bun run docs` (alias `dev`) — Fumadocs/Next.js docs site on `PORT` (default 3001)

There is no separate lint step in the packages; `typecheck` + `test` are the gates. (The `docs/` subpackage has its own Biome config and `bun.lock` — treat it as a separate project.)

## Architecture

Four published packages in `packages/`, layered so `core` has zero runtime deps and everything else builds on it:

- **`core`** (`@opencoredev/loginwithchatgpt-core`) — framework-agnostic engine, only Web-standard `fetch`/`crypto`. Owns the OAuth mechanics and has **no session/cookie/HTTP-server concept**. Key modules: `device.ts` (device-code request/poll/exchange), `oauth.ts` + `pkce.ts` (loopback PKCE flow), `tokens.ts` (`ensureFreshTokens` refresh-on-expiry), `jwt.ts` (decode id-token → `ChatGPTUser`, `deriveAccountId`), `codex-transport.ts` (`createCodexFetch`, `listCodexModels`, request normalization/filtering for the Codex Responses endpoint), `store.ts` (`KeyValueStore` interface + `MemoryStore`).
- **`server`** (`@opencoredev/loginwithchatgpt-server`) — `createChatGPTHandler()` returns **one** Web-standard `(Request) => Response` handler mounted at a base path (default `/api/chatgpt`). This is the heart of the repo (`handler.ts`, ~680 lines). Routes: `POST /login`, `GET /status`, `GET /session`, `POST /logout`, and (when the responses proxy is enabled, default on) `POST /responses` + `GET /models`. `session.ts` is the state machine + storage; `crypto.ts` does cookie signing (`sign`/`unsign`) and token encryption (`encryptJson`/`decryptJson`) keyed by `LWC_SECRET`; `cookies.ts` handles the HttpOnly session cookie.
- **`react`** (`@opencoredev/loginwithchatgpt-react`) — `useLoginWithChatGPT()` hook (drives login/polling against the backend handler) and the styled `<LoginWithChatGPT />` button. Talks only to your backend routes, never to OpenAI directly.
- **`ai`** (`@opencoredev/loginwithchatgpt-ai`) — Vercel AI SDK provider (`createChatGPT`, `createChatGPTProxyProvider`) so client/server code calls `streamText()`/`generateText()` through the proxy like any other provider. `ai` and `@ai-sdk/openai` are **peer** deps.

### Load-bearing invariants

- **Token boundary**: `accessToken`/`refreshToken`/`idToken` never appear in any route response. `dangerouslyGetTokens()` / `dangerouslyAllowTokenExport` exist but are off by default and named to warn — don't route around this to "make it work." Only `ChatGPTUser` is browser-visible.
- **Serverless-safe polling**: the server runs no background loop. Each `GET /status` advances **at most one** upstream poll, rate-limited per session via `lastPolledAt` at the interval OpenAI requested. All flow state lives in the session store.
- **Session state machine** (server session and React hook share it): `unauthenticated → pending → authenticated`, or `→ expired` (device codes expire ~15 min). The hook adds client-only `loading` and `connecting` states.
- **`LWC_SECRET`** signs the cookie and encrypts tokens. Without it the SDK generates an ephemeral secret and logs everyone out on restart — set it (`openssl rand -hex 32`) for anything real.
- **`/responses` proxy guardrails** live in `ResponsesProxyPolicy` (handler options): `allowedModels`, max body size (default 8 MiB), per-session rate limit (default 30/window), default injected model.

## Conventions

- **Bun for everything** — `bun <file>` not `node`; `bun test` not jest/vitest; `bun install`; `bunx` not `npx`. Bun auto-loads `.env` (no `dotenv`). Prefer Bun built-ins (`Bun.serve`, `bun:sqlite`, `Bun.file`, `Bun.$`) over Express/node:fs/execa. The demo server uses `Bun.serve()` with HTML imports.
- **ESM + explicit `.ts` extensions** in imports (see any `index.ts`). Targets Node 18+ / Bun / edge.
- Packages are published to npm under the `@opencoredev/loginwithchatgpt-*` scope; `bun run release` builds then `npm publish`es all four.

## Android port (`android/`)

An on-device Kotlin port of the SDK lives in `android/` — a separate Gradle project, not part of the Bun workspace. Three modules: `lwc-core` (pure Kotlin/JVM engine mirroring `packages/core`, JUnit tests ported from the TS oracles), `lwc-android` (Keystore token store + Custom Tab + facade), `sample` (Compose demo). Unlike the web SDK there is no server: tokens live on-device in Keystore-backed encrypted storage.

- Build with the **committed wrapper** (`.\gradlew.bat` / `./gradlew`, Gradle 8.11.1) — a newer system Gradle is incompatible with the pinned AGP.
- Gates: `./gradlew :lwc-core:test` (unit) and `./gradlew :lwc-core:run --args="..."` (live spike; needs an interactive ChatGPT device login, so only a human can run it).
- Wire-format rules the live run proved (encoded in `CodexTransport.kt`): `/responses` needs `stream: true` for SSE, and `input` must be a list of message items — a bare string is rejected with `400 {"detail":"Input must be a list"}`. Keep the Kotlin transport behavior in lockstep with `packages/core/src/codex-transport.ts` when either changes.

## Working with OpenAI's endpoints

This rides on OpenAI's Codex OAuth client and `chatgpt.com/backend-api/codex`, not an official third-party-login product. When touching `core`'s transport/oauth/device code, don't invent API-key auth paths, and don't assume one model slug works for every account — model availability is per-account and discovered via `/models` (`listCodexModels`). The `skills/login-with-chatgpt/SKILL.md` agent skill encodes these same rules for coding agents.
