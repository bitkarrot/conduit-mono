# Core library plan (v2)

What has to happen so that someone outside this repository can build a buyer
(Shop) or merchant (Sell) app on Conduit without re-implementing the commerce,
wallet and shipping logic.

Version 2, written 2026-11-12 after a review of version 1. Measured against
`main` at `6a67eb6e` plus the local branch `refactor/import-cycle-guard`.
Numbers come from static analysis (import graph and type-checker traces), not
from performing the moves. Effort figures are estimates.

The shape this plan steers toward is described in the
[core library blueprint](core-library-blueprint.md).

The order of work here is superseded by the
[core library rebuild plan (v3)](core-library-rebuild-plan.md), which builds
the blueprint's packages beside the existing code. This document remains the
reference for the verified cycle cuts (Step 1) and for the configuration,
identity and publishing steps.

## Goal

The target user is a builder who leans on an AI coding assistant and does not
understand how orders, payments, wallets or shipping work underneath.

The plan is finished when all of these are true:

1. A new Vite + React project can `npm install` the public packages and, in
   roughly a page of code, list products, fill a cart, check out, and show order
   status. A second small project can publish a product, receive an order and
   mark it shipped.
2. That project never imports anything outside the documented public surface.
3. There is no public call that marks an order as paid, moves money, or reveals
   a secret without the library's own checks running first.
4. An AI assistant given only the published docs builds a working shop.

## What the review of version 1 found

Version 1 planned a low-level `@conduit/protocol` package that the app supplies
with storage and configuration. That is the wrong shape for this audience, and
two of its steps did not work as written.

### It aimed at the wrong layer

| Finding                                                                                                                                                                                                                                                                                                      | Consequence                                                                                           |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| About 52,000 lines of commerce logic live in the apps, not in `packages/core`: `apps/market/src/lib` (116 files, 28,050 lines) and `apps/merchant/src/lib` (72 files, 24,157 lines). 184 of those 188 files do not use React. 46 Market files are named `checkout-*`.                                        | A library cut from `packages/core` alone leaves the outside builder to rewrite checkout.              |
| The payment tests protect app code. 11 of the 20 root test files import from `apps/`, including `order-payment-service`, `order-payment-recovery` and `checkout-spark-settled-*` in `apps/market/src/lib`.                                                                                                   | The logic the project considers most critical is not in the shared package.                           |
| The checkout sequence itself lives in one React component. `apps/market/src/routes/checkout.tsx` is 5,715 lines; `CheckoutPage` uses 25 `useState`, 15 `useEffect`, 13 `useRef` and imports 35 `lib` files. Version 1 listed splitting it as out of scope.                                                   | There is no headless "run a checkout" call to expose.                                                 |
| Version 1 asked the consumer to supply storage (26 store interfaces, some needing multi-table transactions) and an environment object.                                                                                                                                                                       | The opposite of what an inexperienced builder can do. The library must ship working defaults.         |
| Wallets and React hooks were left out of the library.                                                                                                                                                                                                                                                        | Both are things the target user needs handled.                                                        |
| The public surface is 2,771 names. Even pruned to the 919 in use it is not learnable.                                                                                                                                                                                                                        | A small task-level surface is needed on top.                                                          |
| Reliability duties are hand-wired in each app's `main.tsx`: resuming pending order deliveries on focus, online and a 60-second timer, pruning caches, clearing query caches when the account changes.                                                                                                        | A new app that omits them loses orders silently.                                                      |
| App identity is closed: `ConduitAppId = "market" \| "merchant"` (36 files), and published events are stamped with Conduit's NIP-89 client tag. `TRADEMARKS.md` forbids third parties presenting as Conduit.                                                                                                  | A third-party app would publish as "Conduit Market" unless identity becomes configurable.             |
| Core reads 43 `VITE_*` variables, among them a Breez API key, the pricing service URL and keys, treasury addresses and the relay list. The Spark router checkout path also builds in a Conduit fee (`CONDUIT_FEE_NUMERATOR = 21n` over `1_000n`, fixed recipient) in `checkout-spark-router-obligations.ts`. | Which Conduit-run services and which fee apply to third-party apps is undecided. See Owner decisions. |
| Nothing covered documentation, starter templates, AI-assistant context, safety guarantees or support policy.                                                                                                                                                                                                 | For this audience those are most of the product.                                                      |

### Steps that did not work as written

- **Step 1b (account-network cycle).** Moving `filterEligibleAccountRelayTargets`
  drags 26 declarations, 717 of the file's 1,172 lines, including
  `currentOwnerGrantSelection`, which holds the three `await import()` calls
  that form the cycle. Moving it moves the cycle. Corrected below.
- **Step 1c (shipping cycle).** The proposed leaf needs `parseProductEvent`
  from `products.ts`, which the step itself forbade: `shippingPolicyQuoteSchema`
  drags 49 declarations, 74% of `shipping-policy.ts`. Corrected below.
- **Step 1d (settlement cycle).** Works, but it is about 1,150 lines across four
  money-path files, not a small leaf of primitives.
- **Rule 3 ("the old file re-exports the moved function").** False whenever a
  function moves upward to break a cycle (Steps 1a and 1d): the re-export
  recreates the cycle.
- **Wave 1 of the package move** listed `schemas/` as already clean.
  `schemas/index.ts` reaches 52 files at runtime, including the database, config
  and NDK.
- **Undeclared edges.** `protocol/nwc.ts` imports `@getalby/sdk`;
  `relay-executor.ts` and `relay-publish.ts` import
  `wallets/portable-wallet-address.ts`; one protocol file imports `app-links.ts`,
  which reaches the database.
- **Steps 3 and 5 contradicted each other.** One made protocol functions take
  storage as a parameter; the other had core re-export them unchanged for the
  apps. The plan never chose how dependencies are supplied.
- **One wrong fact.** Importing `db/index.ts` constructs the database object but
  does not open IndexedDB; it opens on first query. Importing the whole core
  barrel without a browser already works (checked under Bun).

### What held up

Step 0 (the guard), Step 1a, the characterization-test rule, publishing from
this repository, and deciding late whether to publish at all.

## Starting point

| Fact                                                       | Value                                      |
| ---------------------------------------------------------- | ------------------------------------------ |
| `packages/core/src/protocol` (non-test)                    | 230 files, about 136,000 lines             |
| `packages/core/src/wallets`                                | 39 files, about 11,900 lines               |
| React-free logic in `apps/market/src/lib`                  | 113 of 116 files, about 27,500 lines       |
| React-free logic in `apps/merchant/src/lib`                | 71 of 72 files, about 24,100 lines         |
| Route components                                           | Market 19,557 lines, Merchant 14,185 lines |
| Runtime import cycles in core                              | 4 (8, 8, 5 and 3 files)                    |
| Runtime import cycles in the apps                          | 0                                          |
| Names exported from the `@conduit/core` barrel             | 2,771 (1,624 values, 1,147 types)          |
| Of those, imported by an app, `ui`, or root tests          | 919                                        |
| Barrel names used only by Market / only by Merchant / both | 365 / 321 / 188                            |
| `VITE_*` variables read inside core                        | 43, across 8 files                         |
| Core files that import the database at runtime             | 37                                         |
| Core files that import `config` at runtime                 | 31                                         |
| React hooks in core                                        | 18, plus 2 context providers               |
| React hooks in the Market app                              | 19                                         |
| Unit test files                                            | 29 (9 in `packages/core`, 20 in `tests`)   |
| Test files inside `apps/`                                  | 0                                          |
| Browser (Playwright) test files                            | 1                                          |

## Target layout

```
packages/
  sdk/     @conduit/sdk     PUBLIC. Headless. One client object with task-level
                            calls. Ships default storage, relays, wallets and
                            background jobs.
  react/   @conduit/react   PUBLIC. Provider and hooks over the SDK.
  ui/      @conduit/ui      Optional, public later. Ready-made components.
  core/    @conduit/core    INTERNAL engine. Bundled into the SDK build; no
                            stability promise to outsiders.
templates/
  shop/    Starter buyer app using only the public packages.
  sell/    Starter merchant app using only the public packages.
apps/
  market/, merchant/        Become the first consumers of sdk and react.
```

Illustrative shape of the public API (names are not final):

```ts
const conduit = createConduitClient({ app: { name: "My Shop" } })
await conduit.start()

const products = await conduit.shop.catalog.list({ merchant })
conduit.shop.cart.add(products[0], 1)

const checkout = conduit.shop.checkout.begin({ shippingAddress })
checkout.subscribe((state) => render(state)) // needs_wallet, awaiting_payment,
// paid, failed_recoverable, ...
```

Design choices this plan makes:

- **Keep one client per page, configured once.** Core is built on module-level
  singletons (`db`, `config`, the account signer). Replacing them with injected
  dependencies across 37 and 31 files is the most expensive step in version 1
  and this audience does not need it. Instead add one explicit
  `configureConduit(config)` call, made by `createConduitClient`.
- **Ship the Dexie database inside the library.** Storage independence is
  deferred (see Deferred track).
- **Buyer and merchant code are separate directories** that may not import each
  other, enforced by the guard.

## Rules for every step

1. One concern per pull request. A move and a behaviour change never share one.
2. Characterization test first: a test pinning current behaviour lands before
   the code moves, as PRs 806 to 814 do.
3. When code moves **down** (into a file the old one may import), the old file
   re-exports it so importers do not change. When code moves **up** (out of a
   file to break a cycle), the old file must not re-export it; re-point the
   direct importers. The `@conduit/core` barrel keeps the public name either way.
4. `bun run imports:check`, `bun run typecheck` and `bun test` pass on every PR.
5. Record each new boundary in `docs/knowledge`.

---

## Phase A: Foundations

Estimated at two to three weeks. Worth doing whatever else is decided.

### Step 0: Land the guard (done locally, not committed)

Three PRs from branch `refactor/import-cycle-guard`:

1. `scripts/ci/check_import_cycles.ts`, the `imports:check` script, and one CI
   step. Fails on a self-import, a new runtime cycle, a known cycle growing or
   merging, or a stale allowlist entry.
2. 12 core files (10 in `wallets/`, 2 in `hooks/`) import by relative path
   instead of `from "@conduit/core"`.
3. 6 components in `packages/ui` import `./Button` and so on instead of
   `../index`.

Also rebase PR 760 (it is conflicting), and close or rebase the October drafts.

Done when: `imports:check` runs in CI on `main` with four known cycles.

### Step 1: Break the four runtime cycles

Every cut below was checked two ways: a type-checker trace of exactly which
declarations must move together, and a simulation on the import graph showing
the cycle disappears. Delete each `KNOWN_CYCLES` entry as its cycle goes.

#### 1a. Commerce facade (3 files), through the commerce read series

- Move `fetchProfile`, `fetchProfileContext` and the `ProfileFetchOptions` type
  (22 lines) out of `profiles.ts` into a file above the facade, for example
  `profile-fetch.ts`. `profiles.ts` must not re-export them; re-point the
  direct importers (13 references outside `profiles.ts`).
- `commerce-inbox.ts` line 152 does `await import("./commerce")` to call
  `planLegacyDirectMessageRead`. `transportReadPlans` already accepts an
  optional `legacyRelayPlan`; have `commerce.ts` always supply it and delete
  the dynamic import.

#### 1b. Account network (8 files), corrected

The real loop: `relay-reader` must filter targets by eligibility; eligibility
reads stored evidence; the files that own stored evidence also fetch from
relays through `relay-reader`. The fix is to separate "read what is stored"
from "fetch from the network".

1. Move `getEventSourceRelayUrls` and its one helper (4 lines) from
   `relay-reader.ts` into a leaf, for example `event-source-relays.ts`.
   `account-network-evidence.ts` imports it from there.
2. Split `owner-relay-list-evidence.ts`: move `readRetainedOwnerRelayList` and
   what it needs (39 declarations, 793 of 1,415 lines, including
   `dexieOwnerRelayListEvidenceRepository`, `getOwnerRelayListEvidence` and
   `applyOwnerRelayListEvidenceReconciliation`) into a new file, for example
   `owner-relay-list-retained.ts`. It needs nothing from `relay-reader`. The old
   file keeps the network fetching and re-exports the moved names.
3. Split `private-message-routing.ts`: move
   `readRetainedInboxDeclarationEvidence` and what it needs (12 declarations,
   182 of 1,990 lines) into a new file, for example
   `inbox-declaration-retained.ts`. The old file re-exports.
4. In `account-network-local-state.ts`, replace the three `await import()`
   calls in `currentOwnerGrantSelection` with static imports of the two new
   files and `inbox-declaration-evidence.ts`.

#### 1c. Shipping and schemas (5 files), corrected, after PR 760

The true order is: product schemas, then product parsing, then the shipping
quote schema, then the order schema. `schemas/index.ts` holds both the bottom
and the top of that chain, so it has to be split.

1. New `schemas/product.ts`: `productSchema`,
   `productShippingAdjustmentsSchema`, `productSupplierAllocationSchema` and
   what they need (10 declarations, 302 of 1,752 lines). `schemas/index.ts`
   re-exports it. `products.ts` and `product-supplier-allocation.ts` import
   from `../schemas/product`, never from the index.
2. New leaf `shipping-policy-primitives.ts` holding, from `shipping-policy.ts`:
   - `MERCHANT_SHIPPING_POLICY_D_TAG`, `SHIPPING_POLICY_EXTENSION_TAG`,
     `shippingMoneyToMinorUnits`, `shippingMinorUnitsToAmount`,
     `convertShippingMinor`, `normalizeShippingPolicyRegion`,
     `normalizeShippingPolicySubdivision` (12 declarations, 126 lines);
   - `parsePrivateOrderShippingPolicyTags` and what it needs (17 declarations,
     269 lines).

   It imports only `zod`, `../pricing`, `./address-validation`, `./kinds` and
   `./nip89`.

3. `schemas/product.ts` and `shipping.ts` import the leaf instead of
   `shipping-policy.ts`. `shipping-policy.ts` re-exports the moved names.
4. `shippingPolicyQuoteSchema` stays in `shipping-policy.ts`.

#### 1d. Spark settlement (8 files), last

1. New file, for example `checkout-spark-settled-plan.ts`, receiving:
   - from `checkout-spark-settled-router.ts`:
     `isCheckoutSparkNativeTreasuryPlan`,
     `getCheckoutSparkSettledLegGeneration`,
     `getCheckoutSparkSettledClosedGeneration`,
     `deriveCheckoutSparkSettledTransferId`,
     `deriveCheckoutSparkSettledRenewalTransferId`, and what they need: 35
     declarations, 664 of 1,763 lines, including
     `restoreCheckoutSparkSettledPlan` and the plan types;
   - from `checkout-spark-treasury-finalization.ts`:
     `createCheckoutSparkNativeTreasuryFinalization`,
     `restoreCheckoutSparkNativeTreasuryFinalization`,
     `restoreCheckoutSparkNativeTreasuryPlan` and what they need: 15
     declarations, 279 lines;
   - from `checkout-spark-settled-returned.ts`:
     `restoreCheckoutSparkSettledReturnClosure` (7 declarations, 83 lines);
   - from `checkout-spark-settled-outgoing.ts`:
     `classifyCheckoutSparkSettledPayment` (8 declarations, 125 lines).

   The router and treasury pieces call each other, so they go in the same file.
   The four old files re-export what moved.

2. Move `deriveCheckoutSparkNativeTreasuryBudget` (6 declarations, 114 lines)
   from `checkout-spark-treasury-finalization.ts` into
   `checkout-spark-merchant-settlement.ts`. The treasury file must not
   re-export it.
3. Split this into at least three PRs (plan file, budget move, cleanup). The
   existing `tests/checkout-spark-settled-*.test.ts` must pass unchanged.

Done when: `KNOWN_CYCLES` is empty.

---

## Phase B: Make it installable outside this repository

Estimated at two to four weeks.

### Step 2: One configuration object instead of 43 environment variables

- Define a public `ConduitConfig` type covering everything core reads from the
  environment: network, relay lists, media server, pricing service, wallet
  provider keys, feature flags, app identity.
- Add `configureConduit(config)`. `config.ts` stops reading `import.meta.env`
  and holds whatever was configured, with a safe default preset so a new app
  works with no arguments.
- Move every `import.meta.env` read out of core into each app, in one file per
  app (for example `apps/market/src/env.ts`) that builds a `ConduitConfig` and
  calls `configureConduit` as the first statement of `main.tsx`. The eight
  files to clear: `config.ts`, `build-info.ts`, `pricing/rates.ts`,
  `context/AuthContext.tsx`, `protocol/checkout-spark-treasury-policy.ts`,
  `wallets/spark-breez-address-access.ts`,
  `wallets/checkout-spark-local-router-canary.ts`,
  `wallets/account-address-request.ts`.
- Confirm no module computes a constant from `config` at import time (a first
  search found none), since those would be fixed before configuration runs.
- Add a guard rule: no `import.meta.env` under `packages/`.

Done when: the guard rule passes, and `bun run build:preview` and
`bun run build:production` still pass their artifact verification.

### Step 3: App identity and Conduit-operated services

- Split the two meanings of `ConduitAppId` across its 36 files. Where it means
  "buyer or seller behaviour", replace it with a `role` of `"shop"` or
  `"merchant"`. Where it means "the official Conduit app", read identity from
  `ConduitConfig`.
- `appendConduitClientTag` stamps the configured identity, and stamps nothing
  when none is configured. Only the official apps' `env.ts` supplies Conduit's.
- For each Conduit-run service, apply the owners' decision (see Owner
  decisions) as a config default: `wss://relay.conduit.market`, the media
  server, the pricing signer and its public keys, the address gateway, the
  treasury addresses, and the Conduit fee recipient.
- Make the database name part of `ConduitConfig`. The constructor already
  accepts one; the default is `"conduit"`.

Done when: a build with no Conduit identity configured publishes events with no
Conduit client tag, and a test asserts it.

### Step 4: A real build and an install test

- Give the SDK package a build that emits JavaScript and `.d.ts` files into
  `dist/`, with `exports` pointing at `dist` and a source condition so the
  monorepo apps keep fast resolution.
- Add a CI job that packs the package, installs the tarball into an empty
  project outside the workspace, and:
  - imports the entry point under plain Node with no browser globals (this
    already works under Bun today; the test keeps it true for server rendering);
  - builds a ten-line Vite app against it.
- The headless entry point must not pull in React. Today the core barrel does,
  through `pricing/rates.ts`.

Done when: that CI job passes.

---

## Phase C: Move the workflows out of the apps

Estimated at two to four months. This is the bulk of the work and the part
that makes the library useful to the target user.

### Step 5: Lift the React-free logic into the shared package

1. Classify all 188 `lib` files as workflow (moves), view-model (stays) or
   app glue (stays). Six file names exist in both apps (`auth.ts`,
   `checkout-spark-invoice-recipient.ts`,
   `checkout-spark-settled-continuation.ts`,
   `checkout-spark-settled-leg-preparation.ts`, `event-actor-identity.ts`,
   `productVariations.ts`); compare and merge those first.
2. Destination: `packages/core/src/shop/` and `packages/core/src/merchant/`.
   Guard rule: neither imports the other; shared code goes in `protocol/` or
   `wallets/`.
3. Move in vertical slices, each with its tests, in this order:
   - cart (`cart-model`, `cart-repository` and related);
   - order payment and recovery (`order-payment-service`,
     `order-payment-recovery`, `order-payment-retry`,
     `checkout-order-attempt`): the root tests already cover these and only
     need their import paths changed;
   - checkout (`checkout-payment` and the `checkout-spark-settled-*` files);
   - order publishing;
   - merchant products (`product-publishing`, `productStock`,
     `productVariations`, `productDraft`);
   - merchant orders, invoices and settlement recovery (`order-phase`,
     `merchant-invoice`, `checkout-spark-settled-recovery`,
     `checkout-spark-organizer-routing`).
4. Few files resist moving: 3 read `import.meta.env` (use Step 2's config), 7
   import the router or `@conduit/ui`, 3 import app hooks or components, and 23
   touch `window`, `document` or web storage directly.

Done when: each app's `lib` holds only view-model and glue files, and no root
test imports payment logic from `apps/`.

### Step 6: Headless controllers out of the route components

- Extract the sequencing in `CheckoutPage` into a controller with explicit
  states, actions and a `subscribe` method, in `packages/core/src/shop/`. The
  component becomes a renderer of controller state.
- Because there is one browser test file in the repository, first write
  browser tests for the checkout paths (pay, cancel, reload mid-payment,
  wallet missing, invoice expiry). They are the only safety net.
- Then repeat for Market `orders.tsx` (24 `useState`, 14 `useEffect`),
  Merchant `orders.tsx` (35 `useState`) and Merchant `products.tsx` (21
  `useState`, 9 `useEffect`).
- Do checkout alone and ship it before starting the others.

Done when: the official apps' route files call controllers and hold no payment
or order sequencing of their own.

### Step 7: One background runtime

- `conduit.start()` owns what `main.tsx` wires by hand today: resuming pending
  order relay deliveries on start, focus, online, visibility change and a
  timer; pruning commerce caches, guest order data and trust snapshots; and
  the timers found in `order-payment-service`, `cart-repository`,
  `product-listing-delivery` and `product-deletion-delivery`.
- `conduit.stop()` tears them down.

Done when: both official `main.tsx` files contain no timers or event
listeners for these duties.

---

## Phase D: The public face

Estimated at one to two months.

### Step 8: The SDK surface

- `packages/sdk` exports `createConduitClient` and a small set of namespaces:
  `account`, `wallet`, `shop.catalog`, `shop.cart`, `shop.checkout`,
  `shop.orders`, `shop.messages`, `merchant.products`, `merchant.shipping`,
  `merchant.orders`, `merchant.receiving`. Target: tens of calls.
- Rules for everything public:
  - No call returns or accepts a private key, wallet seed or wallet connection
    string, except explicit connect and backup flows.
  - Whether an order is paid is decided only inside the library from payment
    proof. There is no public setter.
  - Every failure is a typed error with a stable code and a plain message.
  - Every public name has a doc comment with an example.
- A snapshot test lists the public names and fails when the list changes
  without being updated.

Done when: the snapshot test is in CI and both templates compile against the
SDK alone.

### Step 9: The React package

- `packages/react` exports `ConduitProvider` (wrapping `AuthProvider`,
  `ConduitSessionProvider`, and the account-change cache reset now in
  `main.tsx`) and hooks: the 18 in core, plus hooks over the Step 6
  controllers (`useCart`, `useCheckout`, `useOrders`, `useProducts` and the
  merchant equivalents), drawn from the Market app's 19.
- Move the hooks and both contexts out of core. Guard rule: no `react` import
  under `packages/core`.

Done when: core has no React dependency and the official apps use
`@conduit/react`.

### Step 10: Templates, documentation and AI context

- `templates/shop` and `templates/sell`: minimal apps on the public packages
  only, built in CI on every PR.
- Documentation: a quickstart, one recipe per task, a configuration reference,
  and a safety page stating what the library guarantees and what a builder
  must not do.
- Ship an `AGENTS.md` (and `llms.txt`) inside each published package
  describing the supported calls and the things not to do. Coding assistants
  read `node_modules`; this is how they find the right path.
- `@conduit/ui` is optional. To publish it: ship compiled CSS, remove Conduit
  logos and fonts from the defaults, and depend on `sdk` and `react` only (27
  component files import core at runtime today).

### Step 11: Prove it

1. Both official apps use only the public surface for cart, checkout and
   orders.
2. A fresh directory outside the workspace installs the packed tarballs and
   builds each template.
3. Assistant test: give an AI coding assistant only the published docs and ask
   for a small shop. Record every place it reaches for internals or mishandles
   money, fix the library or docs, and repeat.

Done when: the assistant-built shop completes a purchase on regtest without
importing internals.

### Step 12: Publish and support

- Versioning with Changesets, starting at `0.x`, published from this
  repository with npm provenance and two-factor publishing, since the package
  handles wallet credentials.
- Extend `SECURITY.md` to cover library consumers.
- State the database promise: once outside apps ship, the Dexie schema (31
  versions today) lives in their users' browsers. Migrations become forward
  only and are part of the public contract.

---

## Deferred track: a storage-independent protocol package

Version 1's main subject. Only needed if Conduit must run where there is no
IndexedDB (a server, React Native). Not required for the goal above. If it is
taken up later, version 1's outline applies with these corrections:

- Decide first how dependencies are supplied (parameters, or a client object),
  because that fixes the public signatures.
- Split `schemas/index.ts` further before moving it; it reaches 52 files.
- `db/index.ts` imports row types from 19 protocol and wallet files, and about
  54 protocol files import from `../db`. That two-way link holds a 110-file
  type knot together; removing the database's outgoing imports leaves clusters
  of 36 and 32.
- 34 protocol files use the database at runtime; 15 use multi-table
  transactions. The `Store` interface plus `Dexie…Repository` class pattern
  already exists 26 times and is the one to extend.
- Account for `protocol/nwc.ts` (`@getalby/sdk`), the two imports of
  `wallets/portable-wallet-address.ts`, and the import of `app-links.ts`.

## Owner decisions

Engineering cannot settle these, and Steps 3, 8 and 12 depend on them.

1. **Fee.** Does the Conduit fee in the Spark router checkout path apply to
   third-party apps? Keep and disclose it, make it configurable, or leave that
   path out of the public SDK.
2. **Services.** Which Conduit-run services may third-party apps use by
   default: the relay, media server, pricing signer, address gateway? Each app
   will need its own Breez API key regardless.
3. **Identity.** Confirm third-party apps publish without Conduit's client
   tag.
4. **Responsibility.** What Conduit promises, and does not, when an app built
   on the SDK loses a user's funds.
5. **Support.** Once published, API and schema changes cost more. Who owns
   that.
6. **Names and scope.** Package names, and whether `@conduit/core` is bundled
   into the SDK (recommended) or published separately.

## Not in this plan

- Splitting `commerce.ts` beyond Step 1a; the commerce read series owns that.
- Changing the Dexie schema.
- Running more than one client on a page.

## Risks

- **Thin tests.** 29 unit test files, none inside `apps/`, and one browser
  test. Steps 5 and 6 move code that has little or no coverage; writing the
  characterization tests is most of their cost.
- **Step 6.** Pulling state out of a 5,715-line component is where a
  behaviour change is most likely to slip in. One flow at a time.
- **Merge traffic.** File moves conflict with open branches. Announce each
  slice and keep slices small.
- **Money paths.** Steps 1d and 5 move settlement and payment code. Review as
  such.
- **Publishing wallet code.** A package that handles keys is a supply-chain
  target and a liability question (Owner decision 4).

## Decision points

- After Phase A: everything so far is worthwhile with or without a library.
- After Phase B: an experienced developer could already build on the package.
  Phase C is what serves the inexperienced one, and is most of the cost. Commit
  to it only with the Owner decisions made.
