---
description: Interview the user about their commercetools project and scaffold a shopping agent, a merchant agent, or both on the anthropics/commerce-agents packages, using this repo's ct_common/, backend classes, and CLAUDE.md decision record as the reference. Use when the agent being built is backed by commercetools specifically. For a backend on another platform, or no platform decided yet, use commerce-builder's /scaffold-commerce-agent instead.
argument-hint: "[what is being built: the role, and the commercetools project key if known]"
---

Scaffold a commerce agent on the `anthropics/commerce-agents` packages, backed by commercetools.
The shopping agent serves a customer over a `StorefrontBackend`; the merchant agent serves an
operator over a `MerchantBackend`. This repo (`commercetools-anthropic-agents`) already did this
mapping once, against a live commercetools project — its `ct_common/` is commercetools
infrastructure with nothing project-specific in it, and `CLAUDE.md` is the decision record for
every non-obvious choice. The user said:

$ARGUMENTS

Ask for the role first when that leaves it open: it selects which backend class and skills
directory Step 3 copies.

## Step 1: Locate the reference and read it

1. The reference is the current repo when `ct_common/client.py` exists; otherwise clone it — this
   checkout, or, once public, `https://github.com/commercetools/commercetools-anthropic-agents`.
   It in turn pins `anthropics/commerce-agents` at a reviewed commit in its own `requirements.txt`;
   note that ref for Step 3.
2. Read, in this order:
   - `CLAUDE.md` in full. It is the primary input for this command — every commercetools-specific
     claim below traces back to a line in it, and several decisions there look arbitrary and are
     not (price selection, which price a write targets, why REST is used for one transport and
     GraphQL for another).
   - `ct_common/client.py` (the authenticated client: token cache, REST and GraphQL, error
     mapping), `ct_common/graphql.py` (the GraphQL documents and shape normalizers),
     `ct_common/mapping.py` (projection to agent record — this is where the traps live),
     `ct_common/cache.py` (reference-data TTL cache), `ct_common/errors.py`, and, for the shopping
     agent, `ct_common/checkout.py`.
   - The role's `backend.py` (`CommercetoolsStorefront` in `ct_shopping/backend.py`, or
     `CommercetoolsMerchant` in `ct_merchant/backend.py`), `config.py`, `executor.py`, and its
     `skills/*/SKILL.md`; shopping agent also `session.py` for the `customerId`/`anonymousId`
     split.
   - `service/main.py` for how both agents are actually constructed: `skills_dir=`, `config=`,
     `executor_class=`, and which store backs sessions.

## Step 2: Interview

Ask everything in one message, prefilled from $ARGUMENTS and a look at the user's project where
credentials are already available (project key, region, which scopes an existing API client
carries). A skipped question takes the default in parentheses, listed as an assumption in Step 2b.

1. **Role**: shopping agent, merchant agent, or both. Both means two sibling modules, each with
   its own backend, config, skills directory, and sessions, mounted in one process the way
   `service/main.py` mounts the merchant router beside the storefront so an approved change shows
   in the shop at once. (Default: shopping agent.)
2. **Project and region**: the commercetools project key, and which region it is hosted in — the
   Sessions API that commercetools Checkout runs on lives on its own host
   (`session.{region}.commercetools.com`), a separate service from the Composable Commerce API.
3. **API client scopes**: does the client already exist, and does it carry both
   `manage_project` and, if checkout is in scope, `manage_sessions`? These are separate grants,
   and a commercetools API client's scopes cannot be edited after creation — this repo runs on its
   own client created with both up front rather than risk that. If store-scoping matters (the
   client should only reach one Store), ask whether `:{storeKey}` is explicit in the scope: a
   project-scoped client can still call `/in-store/...` unrestricted without it. (Default: no
   client yet; note in the plan that it must be created with every scope this agent will ever
   need.)
4. **Transport**: REST or GraphQL for catalog reads is a real tradeoff here, not a style choice —
   this repo measured REST Product Search at 144,902 bytes for five products against 34,786 bytes
   for the same five over GraphQL, and a REST cart on a wide Product Type at 103 KB against 2.6 KB
   over GraphQL with explicit field selection. Confirm the user wants GraphQL with explicit
   selection for cart/order/anything the model might see the raw size of, and ask what the fenced
   result cap should be (this repo caps at 12,000 characters). (Default: GraphQL for cart and
   order reads, REST Product Search for catalog reads, 12,000-character cap.)
5. **Catalogue shape** — each of these changes what a backend method actually returns:
   - Do most product families use variants only for near-identical duplicates (same price, same
     stock, no attribute actually differs), or do variants encode a real customer-facing choice
     (size, color, tier)? A family with only duplicate variants should collapse to a plain product
     under the master variant, cheapest in-stock variant winning ties.
   - Is `attributeConstraint: CombinationUnique` set on the Product Types that do offer real
     variant choices? Don't assume it's set everywhere it should be — check, since a catalogue can
     have it on some Product Types and `None` on the ones that actually vary by color/size/finish.
     Derive "is this attribute an option" from which attributes actually differ across a family's
     variants, not from the constraint alone.
   - Which searchable-looking attributes are actually marked `isSearchable: true`? A Product
     Search filter on an attribute with `isSearchable: false` returns zero matches with no error —
     it will look like "we don't carry that" rather than a config problem. Check the two or three
     attributes a shopper is most likely to name first.
   - Are Product Discounts in active use? If so, reads must prefer `price.discounted.value` over
     `price.value` — on this project's catalogue, over a third of products carried an active
     discount, and reading list price on those misquotes the customer.
   - Are any products named under a locale other than the one the storefront defaults to (e.g.
     authored under `en` when the storefront runs `en-US`)? `name(locale:)` returns null, not a
     fallback, when a product has no value for exactly the requested key — read `nameAllLocales`
     and fall back client-side, or products will render with no title.
6. **Identity**: `client_credentials` against one commercetools API client held by the process,
   with the principal bound at session start and read off the session record afterward (no route,
   no tool argument carries a customer id — this is asserted by a test in the reference repo and
   should be asserted here too). Confirm a guest maps to a Cart's `anonymousId`, never
   `customerId` — a Cart's `customerId` only accepts a registered Customer, and setting the wrong
   one silently leaves both fields unset.
7. **(Merchant agent) Staged-change ledger**: where do staged changes live so they survive a
   restart and are shared across workers — this repo's own ledger is process-wide and in-memory,
   which is explicitly listed as unfinished (`CLAUDE.md`, "What is left"). Ask whether commercetools
   Custom Objects, a real database, or something else backs the ledger for this project.
8. **(Merchant agent) Approval surface and enforcement**: a portal button, a console prompt, or
   none yet. With `require_host_approval=True`, `apply_change` only succeeds for a change id the
   host itself marked approved immediately before the executor runs — confirm whether that host
   integration exists yet or is a TODO. Order matters here: both the staged-status check and the
   guardrails must run *before* the platform write, not after — running them after means applying
   an already-applied change writes to commercetools a second time before raising.
9. **(Merchant agent) Campaigns and analysis**: commercetools has no campaign object at all, so
   `enable_campaigns` should stay off and any campaign flow parked under `skills/_staged/` unless
   the user has their own campaign system to back it. Ask whether order-event analytics reach a
   queryable warehouse (e.g. via Subscriptions → Pub/Sub → BigQuery); without one, leave the
   analysis delegate off and traffic/conversion/margin figures return `None` with a note, never a
   zero.
10. **(Shopping agent, if checkout is in scope) Checkout ownership**: does this project already run
    commercetools Checkout with a Payment Integration, or does it need to be set up? If setting up:
    the Checkout Application must be `PaymentOnly` (or the mode this deployment needs), and the
    storefront's exact origin must be added to its `allowedOrigins`, or the widget will not load.

Do not ask about scale, cost, or model tier; the config defaults in the reference packages hold
until evals say otherwise.

## Step 2b: Plan back and record

Play the plan back in one message and get a yes before writing code:

- Role(s), project key and region, and the API client's scopes (never the credential itself).
- Transport per read path, and the fenced-result cap.
- Catalogue shape: how options are derived, which attributes are searchable, whether discounted
  price wins, which locale names are read.
- Identity binding, and the guest/`anonymousId` mapping.
- Merchant agent: ledger backing, approval surface, `require_host_approval` value, campaign and
  analysis posture.
- Shopping agent: checkout ownership and the Checkout Application's `allowedOrigins`.
- Assumptions taken for skipped questions, and the reference repo's path and pinned commit.

On yes, write the plan into the project's `CLAUDE.md` under `## Commerce agent decision record`
(a subsection per agent when both), in the same shape as this repo's own — decision, then why it
matters, then the evidence. `/add-commerce-flow` and `/author-commerce-evals` (from `commerce-builder`)
read this section; update it when a decision changes.

## Step 3: Scaffold

**Copy `ct_common/` wholesale.** `client.py`, `graphql.py`, `mapping.py`, `cache.py`, `errors.py`,
and, for the shopping agent, `checkout.py`, are commercetools infrastructure — none of it is
specific to this project's catalogue or policies. Copying it (or, once this repo is public,
depending on it directly) is faster and safer than re-deriving the traps `mapping.py` already
encodes: price selection, id shaping, locale fallback.

**Write the backend subclass following this repo's as a template**, porting only what actually
differs — attribute names, the policy-content container name, the search notes:

| Method | commercetools call | Template (shopping) | Template (merchant) |
|---|---|---|---|
| Search / listings | `productProjectionSearch` (GraphQL) or `/product-projections/search` (REST) with mandatory price selection (`priceCurrency`, `priceCountry`, `localeProjection`) — omit any of the three and a result carries no name, image, or price | `search_products` | `search_listings` |
| Product / listing detail | `product` (GraphQL, masterData) | `get_product_details` | `get_listing` |
| Cart | Carts API, found by `anonymousId`/`customerId` predicate, never stored client-side | `get_cart`, `add_to_cart`, `update_cart_item`, `remove_from_cart` | — |
| Orders | GraphQL `orders(where:)`, scoped to the principal; totals from `taxedPrice.totalGross`, never `cart.totalPrice` once a shipping method is set | `get_orders` | `get_business_snapshot`, `query_metrics` |
| Policies | Custom Objects in a policy container (`policy-content` here) — commercetools has no policy system | `search_policies` | — |
| Shipping | `/shipping-methods` or `/shipping-methods/matching-cart` — both answer with a bare JSON array, not `{results}`, and `matching-cart` needs an address already on the cart | `get_fulfillment_options` | — |
| Inventory | Inventory Entries, one per supply channel — judge the channel-less entry as the sellable figure if the storefront creates carts with no supply channel, and name other channels' units in guardrail notes rather than treating them as available | — | `get_inventory_alerts` |
| Pricing | `/product-projections/search` with price selection; the resolved `price.id` is what a write must target — a heuristic (e.g. "prefer the price scoped to the shopper's country") can disagree with what the platform actually resolves | — | `get_pricing_context`, `stage_price_update` |
| Writes | `apply_change` maps a staged change to update actions (`changeName`/`setDescription`, `changePrice`, `addQuantity`, `publish`/`unpublish`); the staged-status check and guardrails run before the call | — | `apply_change` |

**Pin the reference.** `requirements.txt` here pins `anthropics/commerce-agents` as PEP 508 direct
references (`name @ git+url@<commit>#subdirectory=<pkg>`) rather than bare `git+` lines, because a
bare git URL has no distribution name for `uv`'s generated `pyproject.toml` to key on. Follow the
same shape; never an editable path.

**Ids.** A family is the product id; a purchasable record is `{productId}#{variantId}`, because a
commercetools variant carries no global id of its own — a commercetools id never contains `#`, so
the two namespaces cannot collide. Keep this even for a catalogue with no real variant options: a
plain product still needs a stable purchasable id past the master variant.

## Step 4: Verify and hand off

1. Run the project's linter and type checker on the generated backend.
2. Construct the agent with the stub or copied backend and `FakeClient([text_message("hello")])`
   from `commerce_common.testing`; send one message and assert a `text_delta` event comes back.
3. With real commercetools and Anthropic credentials, run one live turn and show the reply; then
   grep the shell to confirm no route and no tool argument carries a customer, merchant, or
   operator id after session start — only the session id.
4. Before the first real cart write, check for an API Extension with an unguarded
   `custom(fields(...))` trigger condition on Carts — one such condition cannot evaluate against a
   cart with no custom object and fails *every* cart write in the project with
   `ExtensionPredicateEvaluationFailed`, for this agent and everything else pointed at the same
   project. `scripts/check_extensions.py` in this repo is the pattern to port.
5. Confirm every sellable product carries a Tax Category — its absence blocks cart creation, and
   the failure surfaces at cart time, not at product-save time, so it is easy to miss until a
   shopper hits it.
6. Confirm the mandatory price-selection fields (`priceCurrency`, `priceCountry`,
   `localeProjection`) are passed on every search and listing read; a call missing one still
   succeeds and just returns products with no price.
7. Print what to do next:
   - commit the scaffold with the decision record;
   - wire any backend method left as a stub, and the checkout Application's `allowedOrigins` if
     checkout is in scope;
   - if a merchant agent, confirm the ledger and approval surface from Step 2 are actually wired,
     not just planned;
   - continue with `commerce-builder`'s `/add-commerce-flow` and `/author-commerce-evals` for
     flow-by-flow wiring and eval coverage — those commands are backend-agnostic once the
     commercetools backend exists.

Both agents: repeat 1 to 6 per agent.
