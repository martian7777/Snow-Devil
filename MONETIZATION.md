# Snow Devil — Monetization Design

> Status: **design / not yet implemented.** This document describes the planned
> Free/Pro model and licensing system. No billing code exists in the app yet.

## 1. Model at a glance

| Decision | Choice |
| --- | --- |
| Business model | **Freemium** — a genuinely useful Free tier plus a paid **Pro** unlock |
| Audience | **Solo developers / maintainers (B2C)** — single-user, local-first; no teams/seats for v1 |
| Platforms | Windows, macOS, Linux (all from the existing release pipeline) |
| Payment provider | **Lemon Squeezy** (Merchant of Record + native License Key API) |
| Pro billing | **Subscription** (yearly primary, monthly option). One-time perpetual is also possible — see §7 |
| Enforcement | **Client-side** entitlement, license-key activated, cached in the OS keychain, re-validated online with an offline grace period |

Snow Devil is local-first with **no backend of its own**. The design keeps it
that way: licensing talks directly to Lemon Squeezy's License API from the Rust
side, so shipping paid tiers does **not** require standing up a server.

## 2. Free vs Pro

The Free tier must be independently useful (it drives the funnel); Pro unlocks
depth, automation, and scale.

### Free
- Full **Demo Mode**.
- Connect **one** GitHub account (the bundled OAuth one-click flow).
- **Home**, **Flow** (current view), **Notifications** inbox.
- **Repository Explorer** and **Pull Request Diff**.
- Basic analytics with limits:
  - history capped to roughly the **last 90 days**,
  - up to **N tracked repositories** (e.g. 5).

### Pro (license required)
- Full **Flow Analytics**, **CI Health**, **Personal Focus**, **Inventory** — the advanced analytics with metric lineage.
- **Unlimited** history depth and **unlimited** tracked repositories.
- **Data export & reports** — CSV / JSON / Markdown exports and date-range delivery reports.
- **Native desktop notifications + background sync** — OS notifications for review requests, failing checks, and aging work.
- **Evidence Graph** deeper traversal.

### Deferred to a later release (design for, don't build yet)
- **Multi-account** (personal + work GitHub) — strong Pro differentiator.
- Team / shared / cloud features — out of scope for the B2C v1.

### Gating principle
Gates should be **capability and scale limits**, never data hostage-taking. A
user who lets Pro lapse keeps their local data; they simply return to Free
limits (history window, repo count, no export/notifications). Nothing already
synced is deleted.

## 3. How enforcement works (desktop reality)

There is no server to check entitlement against on every request, so gating is
**client-side**. This is friction for the honest majority, not DRM — a
determined user can bypass any local paywall, and we deliberately **do not**
over-invest in obfuscation.

The entitlement lifecycle:

1. **Activate** — user pastes a license key (or is redirected post-checkout);
   the app calls Lemon Squeezy `activate`, receives an `instance_id`, and stores
   `{ licenseKey, instanceId, tier, validUntil }` in the **OS keychain** (the
   same secure store already used for the GitHub token — see
   [`src-tauri/src/auth/secure_store.rs`](src-tauri/src/auth/secure_store.rs)).
2. **Validate** — on launch and periodically (e.g. every 24 h), the app calls
   Lemon Squeezy `validate` with `licenseKey` + `instanceId`. The result
   refreshes the cached entitlement.
3. **Offline grace** — if validation can't reach the network, the cached
   entitlement remains valid for a **grace window (7–14 days)**. After the
   window with no successful validation, the app falls back to **Free**.
4. **Lapse / cancel** — when a subscription ends, Lemon Squeezy marks the
   license `expired`/`disabled`; the next `validate` returns invalid and the app
   downgrades to Free.
5. **Deactivate** — on sign-out or moving machines, the app calls `deactivate`
   so the seat frees up (respecting the license's activation limit).

Product-integrity check: the client hard-codes the Lemon Squeezy
`store_id` / `product_id` / `variant_id` and confirms an activated key belongs to
this product before honoring it.

## 4. Why Lemon Squeezy (and not Paddle / Stripe)

| Capability | Lemon Squeezy | Paddle Billing | Stripe |
| --- | --- | --- | --- |
| Merchant of Record (handles global VAT/sales tax) | ✅ | ✅ | ❌ (you handle tax) |
| **Native license-key API** (activate/validate/deactivate + activation limits) | ✅ built for desktop apps | ❌ none (build your own / add Keygen) | ❌ |
| Hosted checkout openable in the system browser | ✅ | ✅ | ✅ |
| License keys work on subscription products | ✅ | n/a | n/a |
| Backend required for MVP | ❌ (License API is enough) | likely | likely |

For a **local-first desktop app with no server**, Lemon Squeezy's built-in
license API is the deciding factor — it is exactly the primitive this product
needs, and it keeps us serverless.

**Constraints to accept:**
- MoR fee (~5% + payment processing) taken from each sale.
- Payouts on the provider's schedule; limited invoice/tax branding control.
- A machine must reach the network **at least periodically** to validate.
- Lemon Squeezy is Stripe-owned (2024 acquisition) — so we **abstract licensing
  behind an interface** (see §6) to make a future swap (e.g. Keygen) cheap.

## 5. Purchase & activation flow

```
                    ┌─────────────────────────────────────────────┐
  Upgrade modal ──▶ │ Open Lemon Squeezy checkout in system browser│
   (in-app)         └─────────────────────────────────────────────┘
                                     │  purchase
                                     ▼
                    ┌─────────────────────────────────────────────┐
                    │ Lemon Squeezy issues a license key (email +  │
                    │ post-checkout screen)                        │
                    └─────────────────────────────────────────────┘
                                     │  user pastes key
                                     ▼
   Rust: POST /v1/licenses/activate (license_key, instance_name = machine)
                                     │  { instance.id, meta }
                                     ▼
        Store { licenseKey, instanceId, tier, validUntil } in OS keychain
                                     │
                    ┌────────────────┴─────────────────┐
                    ▼                                   ▼
   On launch + every ~24h:                  On sign-out / move machine:
   POST /v1/licenses/validate               POST /v1/licenses/deactivate
   → refresh entitlement / grace            → free the activation seat
```

## 6. Technical architecture (planned)

Mirrors the existing command + store patterns already in the codebase.

**Rust (backend)**
- `src-tauri/src/licensing/mod.rs` — Lemon Squeezy License API client
  (`reqwest`, reusing the retry helper in
  [`src-tauri/src/github/http.rs`](src-tauri/src/github/http.rs)) plus keychain
  read/write via the existing `secure_store` pattern.
- `src-tauri/src/commands/licensing.rs` — Tauri commands:
  `activate_license`, `validate_license`, `deactivate_license`, `get_entitlement`.
  Register in [`src-tauri/src/lib.rs`](src-tauri/src/lib.rs).
- Entitlement cache: keychain-only (preferred) or an optional `license_state`
  row in [`src-tauri/src/db/migrations.rs`](src-tauri/src/db/migrations.rs).

**Provider abstraction** — define a `LicenseProvider` trait
(`activate/validate/deactivate`) so Lemon Squeezy is one implementation and a
future provider (Keygen, etc.) can drop in without touching the UI or gating.

**Frontend**
- `src/services/licensing-api.ts` — invoke wrappers for the commands.
- `src/stores/license-store.ts` (Zustand) — holds `{ tier, status, validUntil }`;
  hydrated on startup, refreshed after validate.
- `src/lib/entitlements.ts` — single source of truth mapping features → required
  tier, plus a `useEntitlement(feature)` hook. **All** gates read from here so
  the policy lives in one place.
- `src/components/licensing/UpgradeModal.tsx` (upsell) and
  `LicenseActivation.tsx` (paste/activate/deactivate a key).

**Gate points** — advanced analytics pages, export actions, notification/background
sync, and the repo/history limit checks call `useEntitlement(...)` and either
render the feature or an inline upgrade prompt.

### Feature → tier map (illustrative)
```ts
// src/lib/entitlements.ts
export const FEATURE_TIERS = {
  advancedAnalytics:  'pro',
  unlimitedHistory:   'pro',
  unlimitedRepos:     'pro',
  dataExport:         'pro',
  desktopNotifications:'pro',
  evidenceGraphDeep:  'pro',
  // everything else defaults to 'free'
} as const;
```

## 7. Pricing (starting point — validate with the market)

- **Pro Yearly** — primary, discounted (e.g. ~2 months free vs monthly).
- **Pro Monthly** — lower commitment, higher churn.
- Optional **one-time perpetual** with 1 year of updates, if recurring billing
  proves a hard sell to the dev audience.
- A **14-day Pro trial** (no card) can be layered on later to lift conversion.

Exact numbers are a business decision; the licensing plumbing is price-agnostic.

## 8. Edge cases & rules

| Situation | Behavior |
| --- | --- |
| Offline at launch | Use cached entitlement within the grace window; downgrade to Free only after it expires with no successful validate |
| Subscription lapses/cancelled | Next `validate` returns invalid → downgrade to Free; local data preserved |
| User moves to a new machine | `deactivate` old instance (or rely on the activation limit); `activate` new one |
| Activation limit reached | Surface a clear message with a "manage devices"/deactivate path |
| Refund / chargeback | Lemon Squeezy disables the license → downgrades on next validate |
| Clock tampering to extend grace | Accept as low-value risk; validation timestamps are server-issued, grace is short |
| Demo Mode | Always fully available regardless of tier |

## 9. Privacy & trust

Consistent with the app's local-first stance:
- Only the license key and activation metadata leave the machine, and only to
  Lemon Squeezy for activation/validation.
- No usage telemetry is introduced by monetization.
- The license key and instance id live in the OS keychain, never in plaintext
  files or logs.

## 10. Rollout sequence

1. **Entitlements scaffolding** — `entitlements.ts` + `useEntitlement` + store,
   defaulting everyone to Free. Wire gate points behind it (no billing yet).
2. **Pro feature gating** — apply limits (history/repo caps) and gate advanced
   analytics/export/notifications on the `pro` tier.
3. **Licensing backend** — Rust Lemon Squeezy client + activate/validate/
   deactivate commands + keychain cache + offline grace.
4. **Purchase UX** — Upgrade modal (opens checkout) + License activation screen.
5. **Go live** — create the Lemon Squeezy store/product/variants, hard-code the
   ids, test end-to-end with a test-mode license, then switch to live.

## 11. Open decisions (confirm before building §3–5)

- Pro billing cadence: subscription (recommended) vs one-time perpetual.
- Free-tier limits: exact repo count and history window.
- Trial: include a no-card Pro trial at launch, or add later.
- Price points for monthly/yearly.

## Verification plan (when implemented)

- Unit-test `entitlements.ts` across free / pro / expired / offline-grace states.
- End-to-end with a Lemon Squeezy **test-mode** license: activate → validate →
  confirm Pro unlocks; expire/deactivate → confirm graceful downgrade; pull the
  network → confirm grace, then lockout after the window.
