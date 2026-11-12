# Trust-based product feeds plan

How to add custom product feeds driven by each user's web of trust, once the
repository has the shape described in the
[core library blueprint](core-library-blueprint.md).

Written 2026-11-12. Facts about existing code are measured against `main` at
`6a67eb6e`. Type and function names in the sketches are illustrative.

## Goal

A signed-in buyer can choose a feed of products ordered by how close each
merchant is to them in their follow graph, see why each product is shown, and
define feeds of their own. An outside builder can offer the same in their app
by passing a feed definition, without touching ranking code.

"Web of trust" here means: merchants the user follows, merchants followed by
people the user follows, minus anyone the user has muted, with reports from
followed accounts counting against a merchant.

## Assumption

The rebuild in the [rebuild plan](core-library-rebuild-plan.md) is complete:
`protocol`, `engine`, `platform-web`, `sdk`, `react` and `ui` exist as separate
packages, with the boundary, cycle and size checks enforced.

## What exists today

| Existing code                                    | What it gives this feature                                                                                              |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `protocol/follows.ts` (1,446 lines)              | Reads and verifies follow lists, and reports coverage as `complete`, `limited` or `unavailable`.                        |
| `apps/market/src/lib/productCatalogRead.ts`      | Feed sources already exist: `"following"`, `"conduit"` and `"combined"` (the default).                                  |
| `apps/market/src/lib/productFeedDiversity.ts`    | Reorders a feed so one merchant does not dominate it.                                                                   |
| `protocol/social-visibility.ts`                  | A pure visibility decision that already names the relationships `self`, `direct_follow`, `second_hop`, `outside_trust`. |
| `apps/market/src/lib/brainstorm-verification.ts` | Calls a third-party trust service for the verified badge.                                                               |
| `protocol/shopper-trust.ts` (1,886 lines)        | Merchant-side evidence about a shopper. Related, but not used by feeds.                                                 |

## What is missing

- **Nothing computes the second hop.** `second_hop` appears only as a type;
  no code derives it.
- **Follow-list reads are small.** One read is capped at 8 authors and 4
  relays per author. A two-hop graph needs hundreds of lists.
- **No stored graph.** Only the user's own follow list is retained, and the
  database's trust projection states that raw follow graphs must never be
  stored in it.
- **No score.** A merchant is either followed or not.
- **Feeds are a fixed set of three,** chosen in app code.

## Two existing rules this feature must respect

1. `social-visibility.ts` states that its result "must never authorize product
   discovery, checkout, payment, or settlement". Trust feeds therefore select
   and order what is shown in one view. They never decide whether a product is
   valid or reachable: every product stays available through the unfiltered
   feed, search and direct links.
2. The stored trust projection forbids raw follow graphs. This plan persists
   only derived results (see Storage).

## Design

### Principles

- **A feed is data, not code.** One `FeedDefinition` object describes sources,
  trust depth, filters and ranking weights.
- **Scoring and ranking are pure functions** that also return their reasons.
- **Score and coverage are separate.** A feed always knows, and can say, when
  the graph behind it is incomplete.
- **Local by default.** Scores come from the viewer's own graph. External
  scoring services are optional and sit behind an interface.
- **Trust changes what is shown, never what is accepted.**

### Where the code goes

```
packages/
  protocol/
    trust/
      model.ts     TrustScore, TrustCoverage, relationship types
      rules.ts     score(viewer, followLists, mutes, reports) -> scores
      codec.ts     follow lists, mute lists, reports: parse and verify
    feeds/
      model.ts     FeedDefinition and its validation schema; presets
      rules.ts     rank(products, scores, definition, now) -> ranked items
  engine/
    ports/         TrustSource: interface for an external score provider
    account/
      trust-graph/ machine.ts, effects.ts: fetch lists in batches, score, store
    shop/
      feed/        machine.ts, effects.ts: candidates, scores, rank, paginate
    runtime/       background refresh and pruning of trust data
  platform-web/
    trust/         provider clients (for example the existing Brainstorm call)
    storage-dexie/ repositories for trust scores and saved feed definitions
  sdk/             conduit.shop.feeds, conduit.account.trust
  react/           useProductFeed, useTrustStatus
  ui/              feed picker, reason label, coverage notice
  testing/         follow-graph builders, fake TrustSource, golden rankings
apps/
  shop/            the feed screen and the feed editor screen
```

Dependency direction inside `protocol`: `feeds` imports `catalog` and `trust`;
`trust` imports `identity` and `kernel`. `checkout` and `orders` may import
neither `trust` nor `feeds`, enforced by the boundary check.

### The feed definition

```ts
type FeedDefinition = {
  id: string
  name: string
  sources: FeedSource[]
  trust: { maxHops: 0 | 1 | 2; minScore: number }
  filters: { tags?: string[]; shipsTo?: string }
  ranking: {
    trustWeight: number
    recencyWeight: number
    diversity: "merchant" | "none"
  }
  whenNoTrustData: "use_default_source" | "show_empty"
}

type FeedSource =
  | { kind: "follow_graph" } // the viewer's web of trust
  | { kind: "curated_list"; address: string } // a published list of accounts
  | { kind: "authors"; pubkeys: string[] } // a fixed set
  | { kind: "default" } // the app's configured source
```

Presets ship with the library: `following` (one hop), `trusted-network` (two
hops), `everything`. The three source modes that exist today become presets
with identical results.

### Scores

```ts
type TrustScore = {
  pubkey: string
  hops: 0 | 1 | 2 | null // null: outside the graph
  score: number // 0 to 1
  endorsers: string[] // up to three followed accounts who follow this one
}

type TrustCoverage = {
  state: "complete" | "limited" | "unavailable"
  listsRead: number
  listsWanted: number
  asOf: number
}
```

Starting rule, chosen to be simple and explainable:

- The viewer and accounts they follow score 1.
- A second-hop account scores `0.5 × (1 − 0.5^n)`, where `n` is how many of
  the viewer's follows also follow it. One endorser gives 0.25, three give
  about 0.44, and it never reaches the score of a direct follow.
- A muted account is excluded before ranking.
- A report from a followed account lowers the score; a report from an
  unfollowed account has no effect.
- Raw follower counts are never used. They are cheap to fake.

### Ranking

`rank` combines trust and recency by the definition's weights, applies the
filters, then the existing merchant-diversity pass, and breaks ties by event
id so the order is stable. The clock is passed in. Each item carries its
reasons:

```ts
type RankedProduct = {
  product: Product
  score: number
  reasons: Array<
    | { kind: "you_follow" }
    | { kind: "followed_by"; pubkeys: string[]; more: number }
    | { kind: "curated_list"; name: string }
    | { kind: "default_source" }
  >
}
```

### How data flows

1. `trust-graph` (background): read the viewer's follow list, then their
   follows' lists in batches within a fixed budget, verifying each event.
2. Compute scores with `protocol/trust`. Keep edges in memory only.
3. Store the derived scores and the coverage record.
4. `feed` (on demand): take the definition, resolve its sources to a set of
   merchant accounts, read their products through the existing catalog read,
   look up scores, call `rank`, and return pages.
5. When the follow list, mute list or relay settings change, the runtime marks
   scores stale and refreshes them.

### Storage

- **Stored:** per viewer and account, the hop count, score and up to three
  endorsers; one coverage record per viewer; saved feed definitions.
- **Not stored:** follow-list events or graph edges for other people.
- All of it is a cache that can be deleted and rebuilt. It is cleared when the
  account changes, like the other per-account caches.

### Public surface

```ts
const feed = conduit.shop.feeds.open("trusted-network") // or a FeedDefinition
feed.subscribe((state) => render(state)) // loading, partial, ready, limited

conduit.shop.feeds.list() // presets plus saved definitions
conduit.shop.feeds.save(definition)
conduit.account.trust.status() // coverage, last refresh
```

An outside builder can also set the available presets and the default feed in
`ConduitConfig`.

### Privacy

- Follow lists, mute lists and reports are public events; reading them reveals
  to relays which accounts are being looked up, as today.
- An external score provider learns which accounts a user asks about. It is
  off unless the app configures a `TrustSource`, and the app discloses it.
- Saved feed definitions stay on the device unless the user publishes one.

## Steps

Each step is its own small series of pull requests and follows the rules in
the rebuild plan: tests first, one concern per pull request.

1. **Decide and write down.** Settle the decisions below. Write a one-page
   spec note with the scoring rule and the feed definition format.
2. **`protocol/trust`.** Carry over follow-list parsing and verification from
   today's `follows.ts`. Add mute-list and report parsing. Implement scoring.
3. **`protocol/feeds`, presets first.** Define and validate `FeedDefinition`.
   Implement `rank`. Express today's three source modes as presets and prove,
   with golden files, that they give the same order as today.
4. **Trust-graph workflow.** Batched reads with a budget, storage of derived
   scores, coverage reporting, background refresh and pruning.
5. **Feed workflow.** Source resolution, scoring lookup, ranking, pagination,
   and the fallback when there is no trust data.
6. **SDK and React surface.** Add the calls above to the public API snapshot.
7. **Screens.** Feed picker, reason label on each product, a notice when
   coverage is limited. The default feed stays as it is today.
8. **Custom feeds.** Editor screen, saving, validation errors in plain words.
9. **Optional.** A `TrustSource` implementation for an external provider;
   publishing and importing feed definitions.

Release behind a flag, with the new feeds offered as a choice before any
default changes.

## Tests

| Layer                | What is tested                                                                                              |
| -------------------- | ----------------------------------------------------------------------------------------------------------- |
| `protocol/trust`     | Scoring tables built from small hand-written graphs; mutes and reports; malformed and future-dated lists    |
| `protocol/feeds`     | Golden rankings; stable order; every reason kind; invalid definitions rejected; presets equal today's modes |
| Trust-graph workflow | Budget respected; partial reads give `limited` coverage; refresh after a follow or mute; account switch     |
| Feed workflow        | Empty graph falls back; signed-out visitor; pagination does not repeat or skip items                        |
| Boundary check       | `checkout` and `orders` cannot import `trust` or `feeds`                                                    |
| Browser              | Switch feed, see reasons, follow a merchant and see the feed change                                         |

`packages/testing` gains a graph builder, so a test reads as
`graph().follows("me", "alice").follows("alice", "shop")`.

## Risks

- **Gaming.** Fake accounts following each other. Mitigation: scores come only
  from the viewer's own follows; counts from outside the graph are ignored.
- **Empty or narrow feeds.** A new user follows nobody. Mitigation: the
  fallback source, and the unfiltered feed always one tap away.
- **Stale or partial graphs presented as complete.** Mitigation: coverage is
  part of the feed state and is shown.
- **Load on relays and the device.** Hundreds of follow-list reads.
  Mitigation: a fixed budget, batching, background scheduling, a two-hop cap.
- **Trust leaking into money decisions.** Mitigation: the boundary check and
  the rule that trust is presentational.
- **Third-party dependence.** Mitigation: providers are optional and the local
  score works without them.

## Decisions needed

1. Scoring: local only, an external provider, or both combined.
2. The scoring rule and its constants. The one above is a starting point.
3. Whether reports count, and from whom.
4. Whether curated lists published by others count as a feed source.
5. Whether saved feeds can be published and shared.
6. What is stored: derived scores with up to three endorsers, as proposed, or
   scores alone.
7. Whether a trust-based feed ever becomes the default.
