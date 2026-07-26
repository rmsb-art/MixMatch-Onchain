# CLAUDE.md

Context for AI sessions working in this repo. Read this first.

## Repo identity (important — avoid a known footgun)

- This repo is **`rmsb-art/MixMatch-Onchain`**. Always confirm `git remote -v`
  points at `https://github.com/rmsb-art/MixMatch-Onchain.git` before pushing
  or opening a PR.
- There is an unrelated repo, `rmsb-art/Sidewalk`, with **no shared git
  history** with this one. A previous session's `origin` got misconfigured to
  point at Sidewalk, which caused pushes/PRs to silently target the wrong
  repo and produce confusing "no changes" / "no common ancestor" errors. If
  you ever see a PR fail with "No common ancestor" or a diff full of
  unrelated files, check the remote URL first.
- Default branch is `main`. There is no `dev` branch on the real remote —
  ignore any local `origin/dev` ref if one is cached; it's stale.
- Before pushing a long-lived feature branch, rebase onto latest `origin/main`
  — this repo merges very frequently (many small `fix/issue-###-...` PRs), so
  a branch can fall 50+ commits behind within a day. Rebasing keeps the PR
  diff limited to your own changes instead of showing spurious deletions of
  everything `main` gained since your branch point.

## Monorepo layout

- Turborepo + pnpm workspaces. `apps/*` (api, web, mobile) and `packages/*`
  (shared, stellar).
- Every package: `tsc` for build (no bundler/tsup anywhere), Vitest for tests
  (`apps/mobile` uses Jest instead — it's Expo/RN), flat `eslint.config.mjs`
  re-exporting `../../eslint.config.base.mjs`, `tsconfig.json` extending
  `../../tsconfig.base.json`.
- `apps/api` env config: `apps/api/src/shared/config/env.ts`, fail-fast
  `requireEnv()` pattern. See `docs/ENVIRONMENT.md` and
  `apps/docs/env-integration.md` for the variable → consumer mapping — keep
  both updated when adding a new env var.

## `packages/stellar` (`@mixmatch/stellar`)

Was a placeholder scaffold; now has real functionality:

- `config.ts` — `loadStellarConfig()`, reads `STELLAR_NETWORK`
  (`testnet`|`public`), optional `RPC_URL`/`HORIZON_URL` overrides.
- `client.ts` — `createStellarClient()` / `DefaultStellarClient`, wraps
  `Horizon.Server` + Soroban `rpc.Server`.
- `wallet.ts` — `Wallet` interface + `KeypairWallet`. Custody model: this
  package never persists/encrypts a secret key itself, only holds it in
  memory; see the doc comment at the top of the file before changing it.
- `account.ts` — `generateStellarAccount()`, `loadStellarAccount()`.
- `friendbot.ts` — `fundTestnetAccount()` (testnet-only).
- `payment.ts` / `payment-errors.ts` / `idempotency.ts` —
  `StellarPaymentService.submitNativePayment()`: builds/signs/submits native
  XLM payments, classifies Horizon failures into a stable
  `StellarPaymentError.kind`, and dedupes submissions sharing an
  `idempotencyKey` via an injectable `IdempotencyStore` (in-memory by
  default — durable/cross-process idempotency is an app-level concern, not
  yet built).

### Testing convention for this package

- `pnpm --filter @mixmatch/stellar test` runs fast, network-free unit tests
  only (mocked Horizon).
- `src/testnet.integration.test.ts` makes **real** calls to Stellar testnet
  (Friendbot + Horizon + actual payments). It's skipped by default
  (`describe.skipIf`) and only runs with
  `RUN_STELLAR_INTEGRATION_TESTS=true pnpm --filter @mixmatch/stellar test`.
  `.github/workflows/shared.yml` runs the default (non-integration) suite on
  every PR touching `packages/**`.

## Current status (as of last session)

- PR open: **https://github.com/rmsb-art/MixMatch-Onchain/pull/1** — "Build
  Core Stellar Payment Service", branch `feat/stellar-payment-service` →
  `main`. Contains both the account/wallet foundation and the payment
  service work. All unit tests passing, integration tests verified live
  against testnet before pushing.
- Next likely work: whatever payment/Soroban issue follows in the tracker
  (nothing queued as of this note — check the issue tracker/PR comments for
  what's next).
