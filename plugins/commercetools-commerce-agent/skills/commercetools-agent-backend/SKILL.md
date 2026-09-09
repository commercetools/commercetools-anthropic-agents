---
name: commercetools-agent-backend
description: Mapping a StorefrontBackend or MerchantBackend (from anthropics/commerce-agents) onto commercetools — price selection, id shaping, transport choice, locale fallback, and the platform behaviors that break a naive implementation. Use when writing or reviewing code in a CommercetoolsStorefront/CommercetoolsMerchant subclass, ct_common/, or anything else translating commerce-agents backend calls into commercetools API calls. Not for the runtime agent's own conversational skills (ct_shopping/skills, ct_merchant/skills) — this is for the developer writing the backend, not the agent talking to a shopper or operator.
---

# commercetools agent backend

Rules for mapping commerce-agents' `StorefrontBackend`/`MerchantBackend` methods onto
commercetools, and the platform behaviors that make the obvious implementation wrong.

## Price selection

- Pass `priceCurrency`, `priceCountry`, and `localeProjection` on every search or listing read.
  Omitting any one of the three does not error — the result comes back with no name, no image,
  and no price, which reads as an empty or broken catalogue rather than a missing parameter.
- Read `price.discounted.value` over `price.value` wherever Product Discounts might be active.
  A catalogue with even a modest fraction of products discounted misquotes every one of them if
  list price is read instead.
- A write that targets a price must target the `id` the platform's own price selection resolved,
  not a heuristic guess. A variant can carry several prices in the same currency scoped to
  different countries or channels; commercetools' resolution can pick the *unscoped* price for a
  currency/country pair even when a scoped price for that country also exists. Read the resolved
  `price.id` from a selected read and write against that id — never reconstruct which price
  "should" apply.
- A merchant-facing read and a shopper-facing read must agree on which price they're reporting
  (list vs. discounted) if the two are ever compared — mixing them makes a price change look
  larger or smaller than it is.

## Ids

- A family is the product id. A purchasable record is `{productId}#{variantId}`, because a
  commercetools variant has no global id of its own. A commercetools id is never a compound
  string containing `#`, so this composite id cannot collide with a real platform id.
- A predicate comparing `id` to a value that is not a UUID is rejected outright by the platform
  as a malformed parameter — it does not simply match nothing. Only put `id` in a predicate when
  the value is a real UUID.

## Catalogue shape

- Don't derive "is this attribute an option" from `attributeConstraint` alone. A Product Type can
  have `CombinationUnique` unset (`None`) even where its variants encode a real customer-facing
  choice. Derive options from which attributes actually differ across a family's variants.
- A family whose variants are identical on every attribute is duplicates, not choices — collapse
  it to a plain product under the master variant. When duplicate variants differ only in stock,
  collapse to the cheapest in-stock one.
- A Product Search filter on an attribute marked `isSearchable: false` returns zero matches with
  no error. Check which attributes a shopper is likely to name (color, finish, material) are
  actually searchable before relying on a platform-side filter for them; apply an unsearchable
  attribute as a filter in the backend, after the platform's own text match, instead.
- Read product names from `nameAllLocales`, not `name(locale:)`. Asking for one locale returns
  null — not a fallback — when the product has no value for exactly that key. Fall back
  client-side across the locales you support, or a product authored under the wrong locale key
  renders with no title at all.
- commercetools' text search is loose: a query can match items that share only a word, not a
  category. Don't hand a platform-side `sorts` (e.g. `price asc`) to the whole match set — it
  will surface the cheapest loosely-matching item, not the cheapest relevant one. Fetch a small
  window over the relevance-ranked results and sort locally within it. A price-range filter is
  the exception: push it down to the platform, since it should apply across the whole catalogue,
  not just the fetched window.

## Stock and inventory

- A missing `availability` on a variant means no InventoryEntry exists for it — inventory is not
  tracked for that variant, not that it's out of stock. Treating absence as zero can empty a
  catalogue where most variants have no InventoryEntry.
- A sku can carry one InventoryEntry per supply channel. If the storefront creates carts with no
  supply channel, the channel-less entry is the sellable figure — judging any other single
  channel, or the sum across channels, as "the" stock will misjudge availability. Note stock
  sitting in other channels separately rather than silently rolling it into an availability
  figure, since restocking when the units are a channel away is the wrong call.
- Report negative stock as it stands. A negative InventoryEntry quantity means the item is
  legitimately oversold — don't floor it at zero.

## Transport

- REST Product Search responses are substantially larger than the same query through GraphQL
  with explicit field selection, and a REST cart read on a wide Product Type can be an order of
  magnitude larger than the equivalent GraphQL query. Default to GraphQL with explicit field
  selection for anything a model might see the raw size of (cart, order); REST is fine for a
  catalog search whose result is reshaped before it reaches the model.
- Cap any commercetools response text that reaches the model at a fixed character limit before
  it's included in a tool result or context — an uncapped raw response, especially over REST, can
  consume an outsized share of the context window on its own.
- Fetch reference data (Product Types, policy Custom Objects, shipping methods) once per process
  and cache it. Re-fetching reference data on every request is a common and costly mistake against
  this platform.

## Shipping

- `/shipping-methods` and `/shipping-methods/matching-cart` both answer with a bare JSON array,
  not a paged `{results}` envelope — a client written for the paged shape will silently misread
  these endpoints.
- `matching-cart` requires a shipping address already set on the cart and fails without one. If
  the agent never collects an address before checkout, the country-level `/shipping-methods` list
  (not `matching-cart`) is the answer available before checkout.

## Identity

- Bind the principal at session start and read it off the session record afterward. No request
  route and no tool argument should carry a customer, merchant, or operator id once a session
  exists.
- A guest maps to a Cart's `anonymousId`. A Cart's `customerId` field only accepts a registered
  Customer — setting a guest's identifier there is silently accepted as *neither* field being set,
  which is a confusing failure to debug after the fact.
- Store-scoping a commercetools API client requires `:{storeKey}` explicitly in the client's
  scope. A project-scoped client (without that suffix) can still call `/in-store/...` endpoints
  for any store, unrestricted — scope it explicitly if store isolation matters.
- An API client's scopes cannot be edited after creation. If checkout is in scope, provision
  `manage_sessions` (a separate grant from `manage_project`, served from a different host —
  `session.{region}.commercetools.com`) at client-creation time, not later.

## Merchant writes

- Run the staged-change status check and the write guardrails *before* the platform call, not
  after. Checking after means an already-applied change can still reach the platform and write a
  second time before the check raises — a price move or a restock applied twice.
- With host-approved writes, the approval mark must be set immediately before the write executes
  and cleared immediately after, regardless of outcome, so a later unrelated turn can't spend a
  leftover approval.
- What the platform cannot supply (traffic, conversion, unit cost, margin, campaign performance)
  should come back as `None` with an explanatory note, never a zero — a zero reads as a real
  measured value, not as data the platform doesn't have. commercetools has no campaign object at
  all; don't approximate one.

## Extensions and Tax Categories

- An API Extension on Carts with an unguarded `custom(fields(...))` trigger condition breaks
  *every* cart write in the project for every client, not just this agent — such a predicate
  cannot be evaluated against a cart carrying no custom object. Check for this before the first
  real cart write against a shared or unfamiliar project.
- Every sellable product needs a Tax Category. Its absence blocks cart creation, and the failure
  surfaces at cart-write time, not at product-save time, so it's easy to miss until a shopper
  actually tries to buy the item.

## Miscellany

- Order totals are `taxedPrice.totalGross`. A cart subtotal should be summed from line items —
  `cart.totalPrice` silently includes shipping once a shipping method is set on the cart.
- Never pipe a commercetools JSON response through `echo ... | jq` when debugging from a shell —
  control characters in the response can corrupt the parse. Write the response to a file and run
  `jq` against the file instead.
