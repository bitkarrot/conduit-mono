# Core library rebuild plan (v3)

How to get from today's repository to the
[core library blueprint](core-library-blueprint.md), with two people, without
a risky all-at-once rewrite.

Written 2026-11-12. Measured against `main` at `6a67eb6e`. This document replaces the ordering in the
[core library plan (v2)](core-library-plan.md); v2 stays as the reference for
the verified cycle cuts and for the configuration, identity and publishing
details that this plan reuses.

## Summary

1. Build the blueprint's packages new and clean, in this repository, next to
   the existing code.
2. Replace the existing code one domain at a time ("slice"). Each slice is
   written, compared against the old behaviour, switched on behind a flag,
   then the old code is deleted.
3. Rewrite the parts that are pure rules. Port, without redesign, the parts
   that move money or define stored and transmitted formats.
4. Use the unit tests deleted on 6 November as the comparison oracle. They
   still pass against today's code and caught about two thirds of deliberate
   breaks. They are a source to mine, not a suite to bring back, and they do
   not cover everything.
5. New code gets a new, smaller, layered test suite, with time and size
   budgets enforced from the first commit.
6. Three milestones: a usable buyer library (M1), a usable merchant library
   (M2), and the full blueprint (M3). The pilot slice measures the pace.

## Assumptions to confirm

| Assumption                                                                                               | If false                                                                                            |
| -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Eric is not active in code for the near term, and his nine open drafts are closed.                       | File moves will conflict with large open PRs; the working agreement below needs a merge window.     |
| Real users have orders, wallet data or funds in the live apps.                                           | If not, the compatibility rules below can be relaxed and more of the money code could be rewritten. |
| Feature work pauses, or is limited to small fixes, in a domain while its slice is open.                  | Every feature landed mid-slice is written twice.                                                    |
| An AI coding assistant writes much of the code; a person reads every line that touches money or secrets. | Review, not writing, is what sets the pace.                                                         |

## Approach

| Kind of code                                                              | Approach                                                                                       |
| ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Pure rules: events, money, catalog, cart, shipping, pricing               | Rewrite clean in `protocol/`. Prove equal by running old and new on the same inputs.           |
| Workflows in `apps/*/src/lib` (about 52,000 lines)                        | Rebuild as state machines in `engine/`, reusing the existing decision logic.                   |
| Sequencing inside route components (`checkout.tsx`, the `orders` screens) | Replace the state logic with the new machines. Keep the markup, copy and components.           |
| Settlement, payment recovery, wallet code                                 | Port. Same behaviour, same record formats, restructured only as far as the blueprint requires. |
| Encryption of secrets at rest (5 files)                                   | Port byte for byte, with known-answer test vectors.                                            |
| Database schema and its 31 migrations                                     | Keep unchanged until the old code is gone.                                                     |
| `packages/ui` components                                                  | Keep. Re-point the 27 that import core to the new React hooks.                                 |

### Why not a fresh-repository rewrite

- Proving payment, retry and recovery behaviour is the slow part, and a
  rewrite does not shorten it.
- Stored data and published orders already exist in today's formats.
- There is no written specification in the repository (no README, no protocol
  document, seven notes in `docs/knowledge`). The old code and the deleted
  tests are the specification, and both are only usable next to the new code.

### Why not refactor purely in place

- 82 of the 230 files in `protocol/` mix rules with I/O and total about 81,000
  lines; each needs splitting by hand.
- Extracting a state machine out of a 5,715-line component is harder than
  writing one with the component open as reference.
- New packages can enforce every blueprint rule from their first commit. Old
  code can only be ratcheted.

## The legacy bridge

While both worlds exist, the new packages reach the old code through one
temporary package.

```
apps  ->  sdk  ->  engine  ->  ports (interfaces)
                                  ^
            platform-web, wallets |   real implementations, added slice by slice
            legacy-bridge  -------+   temporary: implements ports by calling @conduit/core
```

- `packages/legacy-bridge` is the only new package allowed to import
  `@conduit/core`. A boundary check enforces it.
- On day one every port (storage, relay, signer, wallet, rates) is implemented
  by the bridge. Each slice replaces part of the bridge with a real
  implementation.
- Old code may import new packages (they sit below it). New packages never
  import old code.
- The bridge and `@conduit/core` are deleted together at the end.

## Rules while old and new code coexist

These are the conditions under which the approach is safe. Each one came out
of the cross-check below.

1. **One database, one schema.** New code never opens its own database. Its
   repositories receive the existing `db` instance through the bridge. No
   schema version is added until the old code is gone, so any release can be
   rolled back.
2. **Same lock names.** 16 files coordinate browser tabs with
   `navigator.locks`. A workflow moved to new code keeps the old lock name
   until the old holder is deleted, so an old tab and a new tab still exclude
   each other.
3. **Same web-storage keys and formats.** 40 files use `localStorage`, 8 use
   `sessionStorage`, with about 37 key constants.
4. **One relay pool, one wallet instance, one signer session per page.** New
   code uses the old ones through the bridge until the underlay slice.
5. **One owner per behaviour.** A flag selects old or new for a whole domain.
   Nothing is ever published or paid by both.
6. **Wire formats are frozen.** Events, encrypted messages and recovery
   bundles produced by new code are byte-compatible with the old, checked by
   golden files.
7. **Mixed versions must work.** There is no service worker, so a tab left
   open runs the old bundle beside a new tab. Rules 1 to 3 therefore hold
   across releases, not only within one page.
8. **Fixes to old code carry a test.** Until a domain is switched, bug fixes
   go into the old code together with a test, and the same test is applied to
   the new module.

## Tests

### The deleted suite

- Pull request #792 (6 November) removed 676 files, about 344,000 lines of
  unit tests. #779 (5 November) removed the Playwright suite, about 100 files.
- Restored on top of today's `main` in a scratch copy: 9,386 tests in 662
  files; 9,311 pass, 74 fail, 28 errors; 8 minutes 46 seconds.
  The failures sit in about 50 files and come from apps and scripts removed
  since, and from the 27 commits merged after the deletion.
- They are behaviour tests: none mock modules, 98 run the real database code
  on a fake IndexedDB, and 241 exercise the apps' `lib` folders, which have no
  tests today.
- Deliberate-break check: 66 single-operator changes (for example `===` to
  `!==`) were made one at a time across 11 source files, and the tests that
  import each file were run. 42 were caught (64%). Each source file was run
  against at most ten of its test files, so this is a lower bound.
- Coverage is uneven. All 6 breaks were caught in `productStock.ts` and in
  `order-phase.ts`; 5 of 6 in `checkout-spark-settled-router.ts` and
  `shipping-policy.ts`; only 2 of 6 in `order-payment-recovery.ts`, which has a
  single test file; none of 6 in `products.ts` under the ten-file limit.
- Not restored as a suite: larger than the code it tested, one flat folder,
  four files over 5,000 lines, and slower than the CI test job's five-minute
  limit.

How it is used:

1. Tag the commit before the deletion (`b102f221^`) as `tests-archive`.
2. At the start of each slice, copy that domain's old tests into a temporary
   `legacy-tests/<domain>/` folder and get them passing against the old code.
3. Break the old code on purpose in a few places. Keep the tests that notice;
   drop those that do not.
4. From the kept tests, extract fixtures and expected values into golden
   files.
5. Point the kept tests at the new module (for ported code this is mostly an
   import-path change; for rewritten code it needs a thin adapter).
6. Delete `legacy-tests/<domain>/` when the slice's old code is deleted.

The Playwright suite stays deleted. Its hermetic fakes
(`e2e/helpers/hermetic-*`: Spark, NWC, LNURL, network) are salvaged into
`packages/testing`.

### The new suite

| Layer                     | What is tested                                                                                    | Budget                         |
| ------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------ |
| `protocol`                | Rules with plain inputs and outputs; golden files for every event and message format              | Seconds per package            |
| `engine` machines         | State transition tables: state plus event gives state                                             | Seconds per package            |
| `engine` workflows        | Whole workflows against fakes. Money workflows also run with a failure injected at every step     | Under a minute per workflow    |
| `platform-web`, `wallets` | Contract suites: the fake and the real implementation pass the same tests                         | Under two minutes              |
| Scenario                  | A buyer client and a merchant client complete a purchase through an in-memory relay, with reloads | Under two minutes              |
| Browser                   | About ten Playwright tests on the critical paths, on the salvaged hermetic fakes                  | Under five minutes, own CI job |

Rules for new tests:

- Tests live beside the code they test and obey the same 500-line file cap.
- Each test names the behaviour it protects. A test that cannot fail when the
  behaviour breaks is removed.
- CI runs only the packages a pull request touches, plus the scenario test.
- No custom reporters, evidence files or source-text assertions.
- Money workflows are also checked by hand on regtest before each switch-on;
  an assistant cannot verify a real payment.

## Phases

Phase 6 overlaps the others from milestone M1.

### Phase 0: Prepare

1. Agree this plan and settle the decisions listed at the end.
2. Add Bitkarrot to `.github/CODEOWNERS`. Today every path lists Eric and
   James only, so with Eric away nobody can give James's pull requests a
   code-owner review.
3. Tag `tests-archive`. Close the stale drafts.
4. Sequence James's open drafts: #760 and #797 land or are parked before the
   pilot; #741, #742 and #743 before the slices that touch products, event
   markets and checkout.
5. Decide the Dependabot queue: #800 (TypeScript 6 to 7, a major version) and
   #803 (Spark SDK, pinned exactly and tied to the reviewed-version capability
   gate) are money-path or tooling changes, not routine bumps.
6. Land the three commits on `refactor/import-cycle-guard`. Finish the
   commerce cycle cut already in progress. Do not start the other three cuts;
   they happen inside their slices (see cross-check).
7. Write the compatibility inventory: every table, web-storage key, lock
   name, event kind and tag, encrypted payload and recovery-bundle format,
   with the old file that owns it.

Done when: the inventory is merged and both people can approve each other's
pull requests.

### Phase 1: Foundations

1. Create empty `packages/protocol`, `engine`, `platform-web`, `wallets`,
   `sdk`, `react`, `testing` and `legacy-bridge`, each with its own build,
   strict compiler settings and test command.
2. Add the checks to `tools/` and make them required: package boundaries, no
   runtime or type-level cycles, 500-line file cap, pull-request size cap, no
   environment reads, lint, public API snapshot for `sdk` and `react`.
3. Define the ports in `engine/ports`, starting from the store interfaces
   that already exist, plus relay, signer, wallet, rates, media and clock.
4. Implement every port in `legacy-bridge`.
5. Define `ConduitConfig` and `createConduitClient(config)`. Each app gains
   one `env.ts` that builds the config. App identity is a config field from
   the start; no new package knows the names "market" or "merchant".
6. Add the flag mechanism (one config field per domain) and the install test:
   pack the packages, install them into an empty project outside the
   workspace, import under plain Node, build a ten-line app.
7. Walking skeleton: one read-only screen value (for example a product title)
   travels app, `react`, `sdk`, `engine`, port, bridge, old code, and ships to
   production behind a flag.

Done when: the skeleton is live, every check is required, and the install
test passes in CI.

### Phase 2: Pilot slice

Kernel (events, signing and verification, kinds, money, ids, typed errors),
then catalog and cart: about 26,000 lines of old code across `protocol/` and
the two `lib` folders, with large existing tests (`cart-model`, 1,558 lines).

Purpose: prove the slice loop end to end on code that cannot lose money, and
measure the real pace.

**Checkpoint.** Use the pilot's measured pace to schedule the later phases.
If the slice loop proves slower or riskier than changing the old code
directly, stop and reconsider the in-place route in plan v2.

### Phase 3: Buyer path

| Slice                  | Old code (rough)                          | Notes                                                                                                                         |
| ---------------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Shipping and pricing   | 6,000 lines                               | Also re-points `apps/checkout-pricing`, which imports four core paths. It runs on Cloudflare Workers, which proves purity.    |
| Buyer orders, messages | part of 33,500 lines                      | Order lifecycle, order message formats, tracking. First use of the single outbox runner.                                      |
| Wallets                | 12,000 lines                              | Port into `packages/wallets`. Also re-points `apps/address-gateway`.                                                          |
| Buyer checkout         | part of 56,000 lines, plus `checkout.tsx` | The largest slice. Port the settled-payment code using v2's verified cut (step 1d). Machine replaces the 25 `useState` calls. |

**Milestone M1:** `templates/shop` builds against the packed packages and
completes a purchase on regtest. The official Shop app uses only the public
packages for browse, cart, checkout and order status.

### Phase 4: Merchant path

| Slice                         | Old code (rough)     | Notes                                                                                         |
| ----------------------------- | -------------------- | --------------------------------------------------------------------------------------------- |
| Merchant products             | 9,400 lines of `lib` | Editor, publishing, stock, variations. Listing and deletion queues move to the outbox runner. |
| Merchant orders, fulfilment   | rest of orders       | Invoices, order phases, shipping settings.                                                    |
| Settlement, recovery, payouts | rest of checkout     | Port. Both people review every pull request.                                                  |

**Milestone M2:** `templates/sell` publishes a product, receives an order and
marks it shipped on regtest.

### Phase 5: Underlay and removal

1. Identity and network (28,000 lines): relay lists, routing, inbox
   declarations, profiles, sign-in. Uses v2's verified cut for the
   account-network cycle (step 1b).
2. Replace the bridge's relay, signer and storage adapters with real
   `platform-web` implementations. `storage-dexie` takes ownership of the
   schema file; row types move into each module's `model.ts`.
3. Event markets (21,000 lines), unless deferred (see decisions).
4. Delete `packages/core`, `legacy-bridge` and both `lib` folders.
5. Cosmetic moves last: `apps/checkout-pricing` and `apps/address-gateway` to
   `services/`, `scripts/ci` to `tools/`. Renaming `apps/market` and
   `apps/merchant` changes Cloudflare Pages settings and is optional.

**Milestone M3:** the blueprint's layout and rules hold with no exceptions
list.

### Phase 6: Public face (overlapping from M1)

As plan v2, Phase D: small documented SDK surface, React package, templates,
recipes, `AGENTS.md` and `llms.txt` inside each package, the
assistant-builds-a-shop acceptance test, then publishing with provenance.

## The slice loop

Every slice goes through the same steps, and is done only when all are true.

1. **Inventory.** List the old files, the old tests, and the compatibility
   inventory entries the slice touches.
2. **Oracle.** Restore and filter the old tests (see Tests). Capture golden
   files from the old code.
3. **Spec note.** One page in `docs/` stating the formats and rules, written
   from the old code. This becomes public documentation later.
4. **Build.** `protocol` module (`model`, `rules`, `codec`), then `engine`
   workflow (`machine`, `effects`), then the real port implementations.
5. **Parity.** The kept old tests and the golden files pass against the new
   module. For pure rules, old and new also run over a corpus of public
   events and must agree.
6. **Switch.** The app uses the new path behind its flag. One browser test
   covers the path. Money slices get a manual regtest pass.
7. **Soak.** On in production for two weeks with the flag still present.
8. **Delete.** Old code, flag, bridge adapter and `legacy-tests/<domain>/`
   are removed in one pull request.

## Working agreement

- Each slice has one author and one reviewer; they swap each slice. Money and
  secret-handling pull requests are read by both.
- No self-merge in the new packages. Branch protection applies to
  administrators too.
- Pull requests stay under the size cap. A move and a behaviour change never
  share one.
- Assistant-written code is welcome. The description of every pull request
  says what was verified and how. Nothing merges unread.
- New packages are owned in `CODEOWNERS` by the two people doing this work.
- One short decision record per choice that is expensive to reverse.

## Cross-check

Before writing this plan, the statements made while it was being worked out
were tested again.

### Confirmed

| Statement                         | How it was checked                                                   | Result                                                           |
| --------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------- |
| The deleted unit tests still pass | Restored into a scratch copy of `main` and run                       | 9,311 of 9,386 pass                                              |
| They are behaviour tests          | Searched for module mocks, source-text reads, fake IndexedDB use     | 0 mock modules; 45 files read source text                        |
| They notice when behaviour breaks | 66 deliberate breaks in 11 source files                              | 42 caught; uneven, so step 3 of the oracle procedure is required |
| Services depend on core           | Listed imports of `apps/checkout-pricing` and `apps/address-gateway` | 6 distinct deep import paths                                     |

### Corrected

| Earlier statement                                                         | Correction                                                                                                                                                |
| ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Three of the four cycle cuts become optional."                           | They are deferred, not optional. The cycle files are ported, so the cuts in plan v2 (1b, 1c, 1d) are made when each slice ports them.                     |
| "Rewrite the screens on top of the new state machines."                   | Too broad. With one browser test, redoing 34,000 lines of screens invites visible regressions. Replace the state logic; keep the markup.                  |
| "256 of the old tests cover the apps' `lib` folders."                     | 241. The earlier figure counted files importing both apps twice.                                                                                          |
| "The database is kept as is, wrapped behind the new interfaces."          | True only with one shared instance. `db/index.ts` also imports row types from 19 protocol and wallet files, so the schema file cannot move until Phase 5. |
| "The code is the only specification" then "the tests are a specification" | Both are needed, and neither is written down. Each slice produces a spec note.                                                                            |

### Missed until now

1. **Cross-tab coordination.** Locks, one `BroadcastChannel` and six
   `storage` event listeners are part of the contract (rules 2 and 7).
2. **Web storage.** State lives outside IndexedDB too (rule 3).
3. **Code owners.** Bitkarrot is not listed, which blocks two-person review.
4. **The two services are consumers.** They must be re-pointed in the pricing
   and wallet slices.
5. **Dependency upgrades are not neutral.** The Spark SDK is pinned exactly
   and gated by a reviewed-version check; TypeScript 7 changes the compiler
   the guard script uses.
6. **No production error signal.** Analytics and error telemetry were retired
   (`docs/knowledge/telemetry-retirement.md`). A regression after a switch is
   seen only through user reports, so the soak step and the manual regtest
   pass carry more weight. Decide whether any privacy-respecting signal is
   acceptable.
7. **Old dependents of switched code.** Old checkout reads the cart. After
   the cart slice, old checkout is re-pointed to the new cart rather than the
   old cart code being kept alive.
8. **Bundle size during the overlap.** Old and new code ship together until
   the old code is deleted. Watch the build output; heavy wallet SDKs stay lazily loaded.
9. **The CI test job has a five-minute limit.** Per-package test runs are
   needed from Phase 1.
10. **Folder renames touch deployment.** Cloudflare Pages and the gateway
    deploy workflow point at today's paths.
11. **Package scope.** Check who owns the `@conduit` scope on npm before any
    name is final.
12. **Nostr library.** Core depends on both NDK and `nostr-tools`. The new
    `protocol` package should use `nostr-tools` and the `noble` libraries
    only; NDK, if kept, belongs in `platform-web`.
13. **Secrets at rest.** Five files encrypt stored data. They are ported with
    test vectors and never redesigned inside a slice.

## Milestones and checkpoints

| Milestone | Reached at the end of | What exists                                              |
| --------- | --------------------- | -------------------------------------------------------- |
| Pilot     | Phase 2               | The slice loop proven on kernel, catalog and cart        |
| M1        | Phase 3               | An outside builder can make a shop                       |
| M2        | Phase 4               | An outside builder can make a merchant app               |
| M3        | Phase 5               | The blueprint's layout and rules hold with no exceptions |

- The pace is set by how fast money code can be read and reviewed, not by
  how fast it can be written.
- No schedule is fixed here. The pilot slice measures the pace, and the later
  phases are scheduled from that.
- The work can stop after any milestone with a working repository.

## Risks

- **Parity gaps.** Behaviour that no old test covers and nobody remembers.
  About a third of deliberate breaks went unnoticed, including four of six in
  the payment-recovery file. Mitigation: new tests written for every gap the
  deliberate-break step finds, golden files, the event corpus, the soak
  period.
- **Two code paths during the migration.** Mitigation: one flag per domain, short
  soaks, deleting old code promptly.
- **The oracle decays.** Each merge to `main` breaks a few more old tests.
  Restore the money-path tests early, in Phase 0 if possible.
- **Assistant-written volume.** The new code can grow as fast as the old did.
  Mitigation: the file cap, the pull-request cap, and no unread merges.
- **Two people.** Illness or other work stalls a slice. Mitigation: slices
  are small and independently shippable.

## Decisions needed

From James and Bitkarrot:

1. Go ahead with this route, or stay with plan v2's in-place route.
2. Feature policy while slices are open.
3. Whether event markets are in scope before M3 or deferred.
4. Whether any production error signal is acceptable.
5. Branch protection: apply to administrators, and who owns the new packages.
6. What happens to the fixes in the closed drafts (for example #730 and
   #639): redo by hand, or drop.

From the project owners (unchanged from plan v2): the Conduit fee for
third-party apps, which Conduit-run services outside apps may use, client-tag
identity, responsibility for lost funds, support ownership, package names.

## First steps

1. Review this document together; settle decisions 1 to 6.
2. `CODEOWNERS` change; tag `tests-archive`; close stale drafts.
3. Land the guard branch (three commits) after rebasing onto `main`.
4. Write the compatibility inventory.
5. Restore the payment and recovery tests into `legacy-tests/` so the oracle
   stops decaying.
6. Create the package skeletons and the required checks.
7. Start the walking skeleton.
