# Core library blueprint

How this repository would be organised if it were designed from scratch. It is
the destination the [core library plan](core-library-plan.md) steers toward,
not a proposal to rewrite: the plan moves the existing code toward this shape
one step at a time.

Written 2026-11-12. Figures about today's code are measured against `main` at
`6a67eb6e`.

## Principles

1. **Dependencies point one way**, and a CI check enforces it.
2. **Business rules are pure.** Anything that decides "is this order paid?" or
   "what does shipping cost?" makes no network, storage or wallet calls, so it
   can be tested with plain inputs and outputs.
3. **The outside world sits behind interfaces** (storage, relays, signer,
   wallet, clock), each with a real implementation and a fake for tests.
4. **Workflows are headless state machines.** "Run a checkout" is an object
   with states and actions, not logic inside a screen.
5. **Buyer and merchant code never import each other.**
6. **Apps are thin:** screens, navigation, branding and one config file.

## Layers

```
apps (shop, sell)            screens, routes, branding
   |
ui  ---------->  react       components; provider and hooks
                   |
                  sdk        one client object: the public API
                   |
        +----------+----------+
     engine                platform-web, wallets
  (workflows + interfaces)   (implementations of the interfaces)
        |                     |
        +------> protocol <---+
              (pure rules, no I/O)
```

A package may import only from packages below it in this diagram.

## Directory layout

```
conduit/
  packages/
    protocol/            PURE. No I/O, no browser, no React.
      kernel/            events, signing and verification, kinds, money, ids,
                         errors
      catalog/           products, variations, stock rules
      shipping/          policies, zones, quotes
      pricing/           currencies, conversion, signed-rate verification
      orders/            order lifecycle, order message formats
      checkout/          plans, allocation, obligations, settlement rules
      identity/          profiles, verification, trust
      network/           relay lists, routing policy, inbox declarations
      event-markets/     markets, rosters, schedules

    engine/              Workflows. Depends on protocol only.
      ports/             interfaces: Storage, RelayTransport, Signer,
                         WalletProvider, RateSource, MediaStore, Clock
      shop/              browse, cart, checkout-session, order-tracking,
                         messages
      merchant/          product-editor, publishing, inventory, fulfilment,
                         shipping-settings, payouts, settlement-recovery
      account/           sign-in, profile, network-settings
      wallet/            connect, balance, backup
      runtime/           outbox, retries, pruning, background jobs

    platform-web/        Browser implementations of the ports.
      storage-dexie/     schema, migrations, one repository per record type
      relay/             connection pool, health, read and publish
      signers/           browser extension, remote signer, local key
      rates/             client for the pricing service
      media/             media server client

    wallets/             Wallet implementations (heavy SDKs, loaded on demand).
      spark/
      nwc/
      lightning-address/

    sdk/                 PUBLIC. createConduitClient(config): wires the above
                         together; presets; the only supported API.
    react/               PUBLIC. ConduitProvider and hooks over the SDK.
    ui/                  Components only. No business logic.
    testing/             Fakes for every port, signed-event builders, an
                         in-memory relay, a buyer-plus-merchant scenario
                         harness.

  apps/
    shop/                routes/, screens/, branding/, env.ts
    sell/                same shape
  services/
    pricing-signer/      today: apps/checkout-pricing
    address-gateway/
  templates/
    shop/                starter on the public packages only
    sell/                starter on the public packages only
  tools/                 CI checks: boundaries, cycles, size ratchet, API
                         snapshot
  docs/                  architecture, decision records, recipes
```

## Shape inside each module

Every folder in `protocol/` has the same files, so a person or a coding
assistant always knows where something goes.

| File       | Holds                                                 |
| ---------- | ----------------------------------------------------- |
| `model.ts` | types and validation schemas                          |
| `rules.ts` | pure decisions and calculations                       |
| `codec.ts` | converting to and from signed network events          |
| `index.ts` | the folder's public names; nothing else is importable |

Every workflow in `engine/` has:

| File         | Holds                                             |
| ------------ | ------------------------------------------------- |
| `machine.ts` | pure state transitions: state plus event to state |
| `effects.ts` | the calls to ports that a transition asks for     |
| `index.ts`   | the workflow's public names                       |

When a file outgrows the size limit it splits by topic inside its folder (for
example `rules/quotes.ts`, `rules/zones.ts`); the folder's `index.ts` stays the
only way in.

## Rules the tooling enforces

- **Boundaries.** `protocol` imports nothing internal. `engine` imports
  `protocol` only. `shop/` and `merchant/` never import each other. Apps import
  `sdk`, `react` and `ui` only.
- **No import cycles**, runtime or type-level.
- **Size limits.** A file cap of around 500 lines and a pull-request size cap,
  each with an owner override.
- **Public API snapshot** for `sdk` and `react`, so every change to the surface
  is a visible diff.
- **No environment reads below `apps/`.** Configuration is one typed object
  passed to the client.
- **Owners per package**, with `protocol/checkout` and `wallets/` needing the
  strictest review.

## Design decisions

- **One client object.** `createConduitClient(config)` is the composition
  root: it builds the storage, relay, signer and wallet implementations and
  hands them to the workflows. Nothing below it creates its own dependencies,
  and there are no module-level singletons.
- **Storage is one repository per record type**, and each repository exposes
  one method per atomic operation, so a multi-table transaction is never split
  across callers.
- **Publishing goes through one outbox.** A workflow records its intent
  durably, then the runtime delivers and retries it. Today's separate listing,
  deletion, shipping and order outboxes become one mechanism.
- **Money paths are append-only and idempotent.** Settlement records are never
  edited in place, and every send carries a key that makes a retry safe.
- **Payment status is derived, never set.** No public call marks an order paid;
  the library decides it from proof.
- **Failures are typed.** Every public call returns either a value or an error
  with a stable code and a plain message.
- **App identity is configuration.** The library never assumes it is running
  inside an official Conduit app.

## Testing

| Layer                     | Test type                                                              |
| ------------------------- | ---------------------------------------------------------------------- |
| `protocol`                | fast unit tests with plain inputs and outputs                          |
| `engine`                  | workflow tests against fakes, including failure and recovery paths     |
| `platform-web`, `wallets` | contract tests: every implementation passes the same suite as its fake |
| whole system              | scenario tests: buyer and merchant clients against an in-memory relay  |
| `apps`                    | a few browser tests per critical path                                  |

## What to keep from today

- The signed-event admission checks.
- The store-interface-plus-Dexie-class pattern (26 store interfaces exist).
- The outbox tables for publishing.
- The characterization-test habit and the `docs/knowledge` notes.
- The policy scripts in CI.

## How today's code maps onto it

| Today                                                         | Blueprint                                                                          |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `packages/core/src/protocol` (230 files in one flat folder)   | split between `protocol/*` (rules) and `platform-web/relay`, `storage-dexie` (I/O) |
| `apps/*/src/lib` (about 52,000 lines)                         | `engine/shop` and `engine/merchant`                                                |
| Sequencing inside `checkout.tsx` and the `orders.tsx` screens | `engine/*/machine.ts`                                                              |
| `packages/core/src/wallets`                                   | `wallets/`                                                                         |
| `packages/core/src/hooks` and `context`                       | `react/`                                                                           |
| `packages/core/src/db`                                        | `platform-web/storage-dexie`                                                       |
| `packages/core/src/schemas/index.ts` (one 1,751-line file)    | each module's `model.ts`                                                           |
| `apps/checkout-pricing`, `apps/address-gateway`               | `services/`                                                                        |
| `scripts/ci`                                                  | `tools/`                                                                           |

## Difference from the plan

The plan reaches roughly this shape with two deliberate shortcuts:

- It keeps `protocol`, `engine` and `platform-web` together inside one internal
  package (`@conduit/core`), separated by folders and guard rules, and defers
  the pure `protocol` split to its deferred track.
- It keeps one configured client per page built on the existing module-level
  singletons, where this blueprint has none.

Both are cheaper routes to the same public packages (`sdk`, `react`). Neither
blocks moving to the full blueprint later.
