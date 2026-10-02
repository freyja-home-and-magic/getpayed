# getpayed - Development Documentation

## Overview

getpayed is a Planet Nine native invoicing app: a freelancer writes an invoice, shares it (via the system share sheet or a copyable link), and the recipient pays it online through a real Stripe-backed payment page. There is no separate "save" step — publishing the invoice (creating its BDO record) *is* what saves it.

**Location**: `/getpayed/app/`
**Identifier**: `club.home.getpayed` (product name `GetPayed`)
**Stack**: Tauri 2 (Rust core in `src-tauri/`, vanilla JS/HTML/CSS frontend in `src/`), targeting iOS.
**Status**: Actively developed / TestFlight builds.

**It used to be called Gelder**, and the rename is incomplete on purpose: `BDO_HASH` is still `"gelder-invoice"` and the `gelder://` URL scheme is still registered, because both are baked into records and Stripe return URLs already out in the world. Changing either orphans published invoices. Expect to keep seeing the old name in identifiers, and don't "tidy" it.

## Architecture

### Frontend (`app/src/`)

Three files, no framework or bundler:
- `index.html` — five views as sibling `<section>`s toggled via a `hidden` attribute: `list-view`, `create-view`, `detail-view`, `payouts-view`, `profile-view`.
- `main.js` — all view logic, calling into the Rust backend via `core.invoke(...)`.
- `style.css` — dark/green/purple visual language shared across the Planet Nine app family.

### Backend (`app/src-tauri/src/lib.rs`)

Tauri commands, invoked from `main.js`:

| Command | Purpose |
|---|---|
| `load_invoices` / `create_invoice` | Local invoice store (`invoices.json` in the app data dir) + publish to BDO |
| `set_invoice_status` | Status moves, including the manual "I got paid another way" override |
| `convert_estimate_to_invoice` | Turns an accepted estimate into an invoice |
| `check_payment_status` | Polls eumachia; returns a `PaymentCheck` (paid **and** payout state), not a bool |
| `retry_payout` | Re-runs a failed payout through eumachia's `POST /pay/:uuid/payout` |
| `get_payout_status` / `refresh_payout_status` | Whether this install has a usable Stripe payout destination |
| `start_stripe_onboarding` | Opens Stripe Connect onboarding **inside the app** via the Stripe iOS SDK |
| `publish_stripe_connected` | Writes `stripeConnected` + `payout` onto the shared Canonical Profile |
| `load_canonical_profile` / `save_canonical_profile` | Shared cross-app profile (App Group) |

No server of its own — getpayed is a thin client over three allyabase services. Which base it talks to is not hardcoded: `src-tauri/bases.json` holds one entry per base (`prod.8as.world`, `dev.8as.world`), and `build.rs` bakes exactly one into the binary. Release builds are prod; `npm run dev:ios` is dev. See "Which base a build talks to" below. Anything published before the switch (build 27 and earlier) lives on dev, and stays there:
- **BDO** (`GATEWAY_BDO_URL`) — publishes each invoice as its own public record (own sessionless keypair per invoice, `BDO_HASH = "gelder-invoice"`), and the shared canonical-profile-adjacent data.
- **Addie** (`GATEWAY_ADDIE_URL`) — mints a Stripe Express connected account for payouts.
- **eumachia** (`EUMACHIA_PAY_URL`, now its own repo at `~/Work/eumachia`) — the *only* piece of this stack that can render a real, interactive "Pay Now" page (savage strips all JavaScript from what it serves, so it can't host a Stripe Elements checkout). Eumachia owns the payment-status record and the actual PaymentIntent/payout flow.

`GATEWAY_ENV` (derived from the same `bases.json` host, e.g. `prod-8as-world`) namespaces every cached uuid (invoice's BDO uuid, Addie identity) so a build against one base never collides with or silently loses what's already published under another.

### Invoice lifecycle

1. **Create** (`create_invoice`): mints a per-invoice BDO keypair, calls `create_user` to get a uuid, computes two permanent links — `share_url` (a savage-rendered SVG view) and `pay_url` (eumachia's interactive pay page, pre-signed with the same read hash/timestamp/signature scheme used for savage share links) — then republishes the full record with both links baked in. `creator_addie_pub_key` is snapshotted onto the invoice *only* if Stripe is already connected at creation time; if absent, eumachia has nowhere to route a payout split.
2. **Share**: the OS share sheet gets `share_url`.
3. **Pay**: the recipient opens `pay_url`, served by eumachia (see `~/Work/eumachia/src/server/node/eumachia.js`) — a self-contained HTML page mounting a Stripe Payment Element, with a sticky "Confirm Payment" footer, that POSTs to eumachia's own `/pay/:uuid/intent` and `/pay/:uuid/complete` routes. Handles both immediate confirmation and the redirect-return path (3D Secure etc.) via Stripe's `redirect_status` query param.
4. **Reconcile**: getpayed doesn't get pushed a "paid" event — the user taps **Check Payment** in the detail view, which polls eumachia's `/pay/:uuid/status` and, if paid, republishes the invoice's own BDO record with `paid: true` (`republish_invoice`, shared with the manual "Mark as Paid" path).

### Stripe connection gating (added September 2026)

Invoice creation is gated on having a connected Stripe account — an invoice created without one never gets a payout split (see `creator_addie_pub_key` above and `payOutCreator` in eumachia's `payments.js`), so a payer's money would have nowhere to go. The list view shows an explanatory banner + "Connect Stripe Account" CTA in place of "+ New Invoice" until `get_payout_status` reports connected; `openCreateForm()` in `main.js` also guards this server-side-truth-independent of button state, redirecting to the Payouts view if called while disconnected.

### Stripe Connect onboarding: in-app, via the iOS SDK (September 2026)

Onboarding used to open Stripe's hosted page in the system browser and wait for a `gelder://stripe-return` deep link. It now happens **inside the app** through Stripe's own iOS SDK, wrapped in a local Tauri plugin, `tauri-plugin-stripe-connect`. There is no deep-link handling left; the scheme stays registered only so old return URLs still open the app instead of failing.

How it fits together, because none of it is obvious:

- **Account Session, not Account Link.** The embedded components need a session token from `accountSessions.create`, which is a different Stripe call from the hosted-onboarding link. Addie exposes it at `POST /user/:uuid/processor/stripe/account-session`, with `GET /user/:uuid/processor/stripe/account` for status; both answer **503 when Stripe keys are unset**, which is how an environment without keys is meant to present itself.
- **The Stripe SDK links into the app target, not the plugin.** Tauri's iOS plugins build as static libraries, which drops SwiftPM resource bundles — and StripeConnect ships its UI resources that way, so linking it inside the plugin produces a crash at presentation rather than a link error. The SDK is therefore a dependency of the *app* target (pinned `StripeConnect` 26.11.0 via `stripe-ios-spm`), and the plugin talks to it through an ObjC-named bridge: `app-target/StripeConnectBridge.swift`, `@objc(TauriStripeConnectBridge)`, selector `presentWithRequest:`, callbacks as `@convention(block)` closures. `src/lib.rs` declares no Rust commands at all — it is `ios_plugin_binding!` and nothing else.
- **The privacy manifest declares what onboarding collects.** StripeConnect 26.11.0 ships no `PrivacyInfo.xcprivacy` of its own, and Apple counts a third-party SDK's collection as the app's, so `ios-native/PrivacyInfo.xcprivacy` lists the business and identity data Stripe asks for (name, contact details, address, payout bank account, tax ID, ID photo, DOB/SSN) alongside what the invoices publish. The App Store Connect privacy answers have to match it. Recheck both when bumping the SDK.
- **`NSCameraUsageDescription` is mandatory.** The onboarding component asserts on a missing usage description when it reaches identity verification, rather than degrading.
- **Connected accounts need `card_payments` alongside `transfers`.** Stripe will not perform transfers on a `transfers`-only account without special approval. Both of Addie's account-creation paths request both.

Addie still marks an account "connected" the instant it is created, before onboarding is actually finished, so `refresh_payout_status` re-reads the real status and the UI trusts that over the creation-time signal.

### Payout reporting: "paid" and "I got the money" are different facts

For most of this app's life only the first was knowable. A payout could fail — unfinished onboarding, no destination, an unreachable eumachia — and the invoice would still read **Paid** while the creator's money sat on the platform account. That is the gap this closes.

eumachia now records the outcome of the payout leg on the payment record (`summarizePayout`, `classifyPayoutError`, `writePayout` in its `payments.js`) and exposes `POST /pay/:uuid/payout` to retry one. getpayed reads that into a `PayoutRecord`:

| Field | Meaning |
|---|---|
| `state` | `sent` \| `failed` \| `none`; anything else is treated as unknown |
| `reason` | eumachia's classification: `onboarding_incomplete`, `no_payout_account`, `already_paid_out`, `platform_funds`, `payment_not_succeeded`, `unreachable`, `unknown` |
| `amount` | cents actually transferred — **less than the invoice total, the platform keeps 9%** |
| `transfer_id`, `error` | Stripe's ids and wording, shown as supporting detail only |

`PayoutRecord::is_retryable()` is `state == "failed"` and not `already_paid_out`, since Stripe refusing a duplicate means the money *did* go out. That predicate also stops the poller: `check_payment_status` returns a `PaymentCheck { changed, paid, payout, payout_identity_stale }` instead of a bool, and without it the app re-polled `none` and `already_paid_out` forever.

The frontend's `describePayout(doc, {identityStale})` maps each `reason` to a `{tone, headline, detail, action}` so the copy lives in one place rather than being spread across branches, and `renderPayoutPanel` draws Paid / payout-pending / payout-failed with a Retry where retrying can help. **The app branches on `reason`, never on Stripe's message string.**

`payout_identity_is_stale` covers the case where the invoice was created under a different payout identity than the install now has — retrying would pay the wrong account, so it asks rather than guessing.

### Canonical Profile (shared across apps)

A separate record from invoices entirely, synced via the **`group.club.home.front`** iOS App Group — the same group BizBuz, Linkitylink and idothis read/write. Read/written through `tauri-plugin-app-group`'s `read_value_sync`/`write_value_sync` under the key `canonical.profile`.

**getpayed is the only app that writes the payout fields**, and every app overwrites the whole record on save, so all four have to carry forward what they don't own:

| Field | Owned by | getpayed's job |
|---|---|---|
| `photo`, `fields` | shared — each app has an editor | edit |
| `address` | Gettit* | carry forward |
| `idothisCategories`, `serviceZip`, `idothisRateCents` | idothis | carry forward |
| `stripeConnected`, `payout` | **getpayed** | write (`publish_stripe_connected`) |

\*Gettit and Letemcook are named in these docs as family members but are
**not checked out at `~/Work`**, so nothing here was verified against them. Treat their ownership of `address` as inherited documentation, and assume the four apps in this workspace are the ones that actually have to agree.

`payout` is `{pubKey, addieURL}` and travels as a unit: addie is distributed, so a pubkey alone is ambiguous — it only means something resolved against the addie instance holding that account. Addie's own payee shape carries the same two fields, so this can be lifted straight into a payout request. BizBuz and LinkityLink read it for their referral cards, which assert where money goes.

Adding a field to this record means adding it to **four** Rust structs, or the next app to save deletes it. `canonical_profile_round_trips_payout_key` in each app's `lib.rs` pins the contract; getpayed's is the copy that matters most, since it is the writer.

### Which base a build talks to (September 2026)

`GATEWAY_BDO_URL`, `SAVAGE_URL`, `GATEWAY_ADDIE_URL`, `EUMACHIA_PAY_URL` and `GATEWAY_ENV` are no longer literals in `lib.rs`. They live per-base in **`src-tauri/bases.json`**, and `build.rs` picks one entry and bakes it in via `env!()`, so the binary carries exactly one base. Everything is derived from a single `host`, because the 8as.world droplets route by path: nginx terminates TLS on 443 and proxies `/<service>/` to that service's local port.

**Releases are always prod.** Everything `build:ios` exports for App Store Connect is a prod build; there is no dev build on TestFlight. Dev testing runs against the local Tauri dev server instead:

    npm run dev:ios      # device or simulator, pointed at dev.8as.world
    npm run dev          # desktop webview, fastest loop for frontend work

An odd-dev/even-prod build-number scheme was tried on 2026-09-28 and taken back on 2026-09-30. It fails on local storage: both halves share one bundle id and therefore one app container, so a dev build's `cards.json`, `base.json`, `bdo_keys.json` and addie identity survive into the next prod install of the same app. That state names a base, and reading a record against the wrong base does not error — BDO answers `{"bdo": null}`, which parses as empty — so the app comes up looking fine and quietly finds nothing.

`scripts/dev-ios.cjs` exists because `tauri ios dev` does one thing less than the release build: it leaves the **App Group entitlement** as an empty dict. Without it the Canonical Profile, the "Import from ..." handoffs between sibling apps, and GetPayed's `stripeConnected` signal to idothis are all dead. The dev script writes that entitlement before handing over. On desktop (`npm run dev`) the app-group plugin resolves to its no-op stub, so the UI works but nothing crosses apps.

Three things enforce it, because nothing at runtime can:

- `scripts/set-build-env.cjs` translates the choice into `src-tauri/.build-env`. It has to be a file, not an environment variable — an iOS build compiles the Rust crate inside an Xcode Run Script phase that inherits nothing. idothis shipped three consecutive IPAs pointed at prod while reporting dev before that was understood.
- `every_service_url_names_the_same_base` in `lib.rs` asserts every base-derived const names the same host as the compiled build env.
- `scripts/build-ios.cjs` reads the evidence back out of the **finished IPA** and fails the build on a mismatch. A test IPA on the wrong base is worse than no IPA, because it looks like it works.

getpayed was hardcoded to **dev** until this change, so prod is the newly reachable half here, not dev. Build 27 and everything before it is a dev build.

## Build & Deploy

`app/package.json` scripts:
- `npm run dev` — `tauri dev`
- `npm run build` — desktop `tauri build`
- `npm run build:ios` — `scripts/build-ios.cjs`, the real distribution path (produces a signed IPA)
- `npm run ios:dev` — `tauri ios dev`

### `scripts/build-ios.cjs`

1. Bumps `.build-number` (App Store Connect rejects re-uploading the same `CFBundleVersion`).
2. Wipes and regenerates `src-tauri/gen/apple/` via `tauri ios init` — a clean slate every time, which means...
3. ...several things get patched back in immediately after, since they don't survive that regeneration: the `ios-native/` source path (carries `PrivacyInfo.xcprivacy`), `TARGETED_DEVICE_FAMILY` restricted to iPhone-only, `ITSAppUsesNonExemptEncryption: false` (skips the App Store Connect encryption questionnaire), the real app icon (`tauri ios init` reverts to Tauri's stock icon), flattened alpha channels on all icon sizes (App Store rejects an alpha channel on the 1024×1024 marketing icon), and the App Group entitlement (`group.club.home.front`).
4. `tauri ios build --export-method app-store-connect`, with a manual `xcodebuild -exportArchive` fallback for a known Xcode 26 export-plist quirk.
5. Copies the resulting IPA to `builds/v{version}/{ProductName}-{buildNumber}.ipa`.

Upload to App Store Connect is a deliberately separate, manual step (Transporter.app or `xcrun altool`) — the script never uploads anything itself.

There's also a top-level `~/Work/planet-nine/builds/` folder holding one "latest" IPA per sibling app (getpayed, BizBuz, IDothis, Linkitylink) — this is a manually-maintained convenience copy, not written by any build script. After a real build, copy the new IPA there too if it should replace the one currently sitting in that folder.

## Known Limitations (documented, not oversights)

- **No payment webhook verification**: `/pay/:uuid/complete` (in eumachia) trusts the payer's browser telling it Stripe's `confirmPayment()` succeeded — Addie exposes no server-side webhook/verification route to check against. Acceptable for a personal invoicing tool, not a high-stakes ledger.
- **Payment status is pull, not push**: getpayed never gets notified when an invoice is paid online; the user must open the invoice and tap "Check Payment."
- **Invoices carry `creator_addie_pub_key` with no `addieURL`.** Same distributed-addie gap the Canonical Profile's `payout` pair exists to close, still open here: a key alone only resolves against the right addie instance. Fix it by snapshotting the pair, as the profile does.
- **No dispute or webhook handling anywhere** in this stack — not in getpayed, eumachia or addie.
- **Read-modify-write race on eumachia's payment record**: concurrent payments could theoretically race on eumachia's own BDO payments map. Acceptable at this scale.

## Related Documentation

- **How these services fit together** — the seams, the silent failure modes,
  and the five stacked bugs between a card being charged and a creator being
  paid: `allyabase/CLAUDE.md` → "How the apps and services fit together".
  Read that before touching the money path.
- **eumachia** is now its own repo at `~/Work/eumachia` (extracted from
  `allyabase/deployment/eumachia/`): `src/server/node/eumachia.js`,
  `src/payments/payments.js`. Its `test/README.md` is the manual-testing
  guide — 31 cases, each naming the Stripe test card or token that triggers
  it, including the ones where the payer is charged and the creator is paid
  nothing. `addie-js` **0.0.8 is the floor** there; earlier versions silently
  drop the `merchant` argument and the money stays on the platform account.
- Sibling apps sharing the same Tauri scaffolding and App Group: BizBuz,
  Linkitylink, idothis (see their own `CLAUDE.md`)

## Last Updated
September 30, 2026 — Took back the odd-dev/even-prod build-number scheme
adopted two days earlier. It cannot work: both halves share one bundle id and
therefore one app container, so a dev build's local state survives into the
next prod install and points it at the wrong base, silently. Everything
exported for App Store Connect is now prod, and dev testing runs against the
local Tauri dev server via the new `npm run dev:ios` (`scripts/dev-ios.cjs`),
which patches in the App Group entitlement that `tauri ios dev` leaves empty.
`build-ios-pair.cjs` and `build:ios:pair` are gone. The per-base switch
(`bases.json` + `build.rs` + `set-build-env.cjs`) stays exactly as it was —
only what drives it changed.

September 28, 2026 — Added the build-time base switch
(`src-tauri/bases.json` + `build.rs`) and the family-wide parity scheme: odd
build numbers are built against dev, even against prod, with
`npm run build:ios:pair` producing one of each. This app had been hardcoded to
dev — build 27 and everything before it is a dev build — so prod is the newly
reachable half here. `GATEWAY_ENV` is now derived from the same host as the
service URLs instead of being hand-maintained beside them.

September 24, 2026 — Renamed throughout from Gelder to getpayed (with the
identifiers that deliberately keep the old name); documented in-app Stripe
Connect onboarding through the iOS SDK and why the SDK links into the app
target rather than the plugin; documented payout reporting end to end
(`PayoutRecord`, the `reason` taxonomy, retry, stale payout identity), the
Canonical Profile field ownership, and the two gaps still open
(`creator_addie_pub_key` without an `addieURL`, no dispute/webhook handling).
Still pointed at **dev** — every URL in `lib.rs` is `dev.8as.world`, unlike
BizBuz and LinkityLink, which are on prod.

September 2, 2026 — Documented current architecture (this file didn't exist before); added the Stripe-connection invoice-creation gate and the eumachia payment-page fixes (duplicate click-handler, input lock during confirmation, sticky footer, redirect-return completion) from this session.
