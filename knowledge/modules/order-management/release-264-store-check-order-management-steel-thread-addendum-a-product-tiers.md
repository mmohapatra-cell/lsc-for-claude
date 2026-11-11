# Addendum A — Add a product with list price and volume tiers (after the core run)

> **Status: PROPOSAL for review (not merged).** Drafted against steel thread v6 on branch `docs/order-management-store-check-steel-thread` (commits `5c04252`, `89e88fa`). It reuses v6 steps by number and never redefines them. Items marked *(unverified — fallback: …)* follow the v6 convention and need a test-org run before merge. Open review questions are at the end.

**Purpose.** Let the admin add one product to an org that already passed the core thread (Acts 0–15, or at least Acts 0–10), give it a list price and volume or tier discounts from their own answers, and have it price correctly on web and on the iPad. The core 0-to-1 run is not changed: this file runs only after it, and every write goes through a v6 step that is already idempotent.
**Estimated duration:** ~20 agent minutes + 5 human minutes (A.in) + 10 human minutes (iPad check, A.12). Estimate, not yet measured. **Depends on:** Act 7 `PASSED` (procedure active, both schedules active and dated), Act 10 (a cache exists to regenerate), Act 11 (the rep's iPad is onboarded). Act 9 only if `ADD_TO_ASSORTMENT = yes`.

## Story

The core thread builds a fixed demo shape: three starter products and exactly two tier schedules, one `Range` and one `Slab`, whose Ids are embedded as constants in the pricing procedure (7.10 → 7.13). A real admin's first request after go-live is usually "add this product at this price with these volume breaks". Re-running Act 7 for that is wrong: `PRICING_TIERS` accepts exactly two entries, and changing a schedule means a new procedure version.

The addendum avoids a new procedure version. Each `PriceAdjustmentTier` row carries its own `Product2Id`, so a new product's bands are added as new tier rows on the **existing** schedule of the chosen method. The procedure, recipe, STT and selector are untouched. What has to happen afterwards is the same as for any pricing change in v6: dates (7.10a), decision-table refresh (7.12), a simulation gate, one cache regeneration (10.8), one sync and an app relaunch.

## User Thread

| User Thread | Needed (features, settings, services) | Known Gaps |
| :---- | :---- | :---- |
| 🧑‍💼 Admin: "Add Femivanta Vitamin at €10 with volume breaks at 101, 201 and 301." 🤖 Agent: asks only what is missing, shows a worked price example for Range vs Slab, normalises the bands and shows the change before writing *(P1, P2)*. | A.in validation; v6 7.3–7.8 | The intake rejects bands with gaps, overlaps, a 0 lower bound or no top bound, and shows the corrected set to accept. |
| 🧑‍💼 Admin: "Don't break pricing for my other products." 🤖 Agent: adds tier rows to the existing schedule, re-simulates one existing product as a regression check, and stops before any change that would switch volume pricing off org-wide *(P4)*. | `PriceAdjustmentTier` Collections insert; 7.12 refresh; 7.14-style simulation on the active version | Inserting tiers into an **active** schedule is unverified (A.7). |
| 🧑‍💼 Admin: "Can my rep order it on the iPad today?" 🤖 Agent: regenerates the cache once, asks the rep to sync and relaunch, and gives the quantities and prices to check *(P3)*. | 10.8 regeneration; 11.3 sync | iPad pricing is proven only by the human check (A.12). |

## Runtime input schema (addendum keys)

Same rules as the v6 [Runtime input schema](./release-264-store-check-order-management-steel-thread-v6.md#runtime-input-schema): every question has a recommended answer and its source, nothing has a silent default, and answers go to the run's frozen copy (`<WORK_DIR>/runs/inputs/run-inputs.json`) and the manifest as `prefilled`, `answered` or `accepted recommendation`. Addendum keys are prefixed `ADD_` so they never collide with core keys, and a second addendum run in the same folder writes a new `addendum-a-NN` section of the manifest.

| Key | Meaning | Required | Example (illustration only) | Asked at | Recommended answer (source) |
| :-- | :-- | :-- | :-- | :-- | :-- |
| `ADD_PRODUCT` | Name, code, description, category, unit of measure, image (optional) of the one product to add | yes | Femivanta Vitamin | A.in | Name: none (customer-supplied). Code: `<PFX>P` + the next free 3-digit number after the highest `<PFX>P###` in `Product2` (source `detected`). Category: the first `CATEGORY_n` of `<CATALOG>` (source `manifest`); a new category name runs 7.2 for it. UoM: the UoM used by the core products (source `manifest`) |
| `ADD_LIST_PRICE` | Price book(s) and list price per currency | yes | `<PFX>Price_Book`, EUR 10.00 | A.in | The first `PRICEBOOK_n`, in its currency (source `manifest`). **Validated:** the currency must equal the price book's currency, the store currency of every `STORE_n` the rep serves, and the rep's `DefaultCurrencyIsoCode` (G12, FP-4.8); any mismatch → ⏸ STOP before A.1 |
| `ADD_TIER_METHOD` | `Range` (the whole line quantity gets the rate of the band it falls in), `Slab` (each band of the quantity gets its own rate), or `none` | yes | Range | A.in | `Range`. Shown with a worked example on the customer's own numbers (Step details A.in) before they answer. Cumulative is rejected (FP-7.7) |
| `ADD_TIER_BANDS` | Bands: lower bound, upper bound, discount % | yes unless `ADD_TIER_METHOD = none` | 1–100 0 %, 101–200 5 %, 201–300 10 %, 301–999999 15 % | A.in | None (customer-supplied), then normalised by the A.in rules and shown back for acceptance |
| `ADD_TO_ASSORTMENT` / `ADD_PRIORITY` | Add the product to `<ASSORTMENT>` (Store Assortment view and Store Check), and flag it Priority (★) | yes | yes / no | A.in | `yes` / `no`: Store Assortment is the main quoting entry on the iPad (8.10). With `no`, the rep finds the product through Browse Catalogs only |

**Derived, not asked** (recorded as `accepted recommendation`):

| Placeholder | How it is derived | First used at |
| :-- | :-- | :-- |
| `<ADD_SCHED_ID>` | The Id of `<PFX>EssentialsVolumeSchedule` (`Range`) or `<PFX>DisplayTierSchedule` (`Slab`), read by `Name` at A.0 | A.7 |
| `<ADD_TIER_FROM>` / `<ADD_TIER_TO>` | That schedule's own `EffectiveFrom` / `EffectiveTo`, read at A.0 (tier dates must sit inside the parent's range, 7.10) | A.7 |
| `<ADD_TIER_BASELINE>` | Tier count per thread schedule before A.7, read at A.0 | A.7, A.13 |
| `<ID:add-product-id>` | The `Product2` Id from A.1 | A.2 onwards |

## Build steps

Every write step runs the v6 step it names, with the addendum values in place of the core ones and limited to the one new product. "Run 7.x" means that row's mechanism, postcondition and idempotency rule as written in v6, unchanged.

| # | Step | API / metadata name | Mechanism | Postcondition (query/check) | Idempotency (detect → reuse, else create) | Playbook |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| A.0 | Preconditions (read-only) | Manifest; `ExpressionSetVersion`; `PriceAdjustmentSchedule`; `PriceAdjustmentTier`; `User` | Read the run manifest: Act 7 `PASSED` or `PASSED_WITH_NOTES`, Act 10 and 11 done. Read back: exactly one active version under `<PRICING_PROCEDURE>` (as 7.15); both thread schedules `IsActive=true` with both dates set (as 7.10a, read-only); `<ADD_TIER_BASELINE>`; the rep's `DefaultCurrencyIsoCode` and the `STORE_n` currencies (as G12) | All true, else ⏸ STOP in the P5 format naming the missing precondition and the core step that sets it. Nothing is written in A.0 | Read-only | — |
| A.in | Ask the addendum inputs: `ADD_PRODUCT`, `ADD_LIST_PRICE`, `ADD_TIER_METHOD`, `ADD_TIER_BANDS`, `ADD_TO_ASSORTMENT`, `ADD_PRIORITY` | n/a | 👤 HUMAN — as 7.in: list each key still missing or `PENDING`, with its recommendation and source; the customer accepts, edits or answers. Validate with the rules in Step details A.in, show the normalised bands and the expected price table, and write only after the customer accepts | Every key filled; bands pass every A.in rule; currency check passed | Keys already filled are not asked again | FP-A.1, FP-A.2 |
| A.1 | Create the product | `Product2` | Run 7.3 for `ADD_PRODUCT` (and 7.2 first if the category is new) | One `Product2` with `ProductCode = <ADD_PRODUCT.code>`, `IsActive=true` | 7.3 rule: detect by `ProductCode`; reuse | — |
| A.2 | Selling-model option | `ProductSellingModelOption` | Run 7.4, option only: reuse the shared `<PFX>One_Time` PSM (`<ID:psm-id>`), add one PSMO for `<ID:add-product-id>` | One PSMO for the product on the shared PSM | 7.4 rule | — |
| A.3 | Category membership | `ProductCategoryProduct` | Run 7.5 for the one pairing | One row | 7.5 rule | — |
| A.4 | Price book entries | `PricebookEntry` (Standard, then custom) | Run 7.6 for one product × the `ADD_LIST_PRICE` price book(s) × currency: Standard PBE first, then the custom entry, both with `ProductSellingModelId`. Never PATCH an existing `UnitPrice`; an existing entry with a different price → ⏸ STOP showing both values | Standard and custom PBE active, `UnitPrice = <ADD_LIST_PRICE>` | 7.6 rule | FP-7.5 |
| A.5 | Territory availability for Quotes | `ProductTerritoryAvailability` (`Purpose='Quote'`) | Run 7.7 for `<ID:add-product-id>` × `<TERRITORY>`; poll as 7.7 | PTA `Active`, ≥ 1 generated detail row | 7.7 rule | — |
| A.6 | Shares | `ProductTerritoryAvailabilityShare`, `ProductTerrDtlAvailabilityShare` | Run 7.8 for the new PTA and detail rows only, `Private` objects only. The catalog share already exists (reused) | 7.8 postcondition for the new rows | 7.8 rule | — |
| A.7 | Add the bands as tiers on the existing schedule | `PriceAdjustmentTier` on `<ADD_SCHED_ID>` | Skip if `ADD_TIER_METHOD = none`. Collections insert, `allOrNone=true`, one tier per band (body in Step details A.7), dates `<ADD_TIER_FROM>`/`<ADD_TIER_TO>`. The schedule stays active *(unverified — fallback: see "Active-schedule fallback" in Step details A.7; on a `customer` org it is a live ⏸ STOP)* | Tier count on `<ADD_SCHED_ID>` = `<ADD_TIER_BASELINE>` + number of bands; every new tier has `Product2Id = <ID:add-product-id>` and both dates; the other schedule's count is unchanged | 7.10 tier rule: detect by schedule + `Product2Id` + `LowerBound`; insert only missing bands; a band that exists with a different `UpperBound` or `TierValue` → ⏸ STOP showing both (never PATCH silently) | FP-A.3, FP-7.7 |
| A.7a | Verify dates | `PriceAdjustmentSchedule`, `PriceAdjustmentTier` | Run 7.10a (read-back of the two thread schedules; repairs only empty dates) | 7.10a postcondition (hard gate) | 7.10a rule | FP-7.14 |
| A.8 | Refresh decision tables | `refreshDecisionTable` | Run 7.12 (all three tables, bounded poll, `LastModifiedDate` after the POST) | 7.12 postcondition | 7.12 rule | FP-7.8 |
| A.9 | Simulate on the active version (hard gate for A.10–A.12) | — | The 7.14 call shape against the **active** version (as 7.20), with the price book and currency of `ADD_LIST_PRICE`. Cases in Step details A.9 | Every case equals its expected `NetUnitPrice`, including the regression case on an existing tiered product. On any mismatch ⏸ STOP in the P5 format; do not regenerate the cache. The new tiers stay; offer to delete only the tiers A.7 created, on the customer's yes | Re-run safely (no persistence) | FP-7.4, FP-7.6, FP-A.4 |
| A.10 | (Only if `ADD_TO_ASSORTMENT = yes`) Add to the Store Check assortment | `AssortmentProduct`; `ProductTerritoryAvailability` (`Purpose='Visit'`) + shares | Run 9.12 for the one product on `<ASSORTMENT>` (`IsRequired = ADD_PRIORITY`), then 9.22 for it (Visit-purpose PTA, generated detail row, shares as 9.22 and the recorded 9.20a decision). Store Check targets for the product are not set by this addendum (record `not applied`) | `AssortmentProduct` row for the product with the right `IsRequired`; 9.22 postcondition for it | 9.12 / 9.22 rules | FP-A.5 |
| A.11 | Regenerate the mobile metadata once | `LifeSciMobileMetadataRecord` | Run 10.8: 10.3 (new parent), 10.4–10.7. One regeneration for all addendum changes | 10.8 postcondition: newest parent `Active`, its ZIP generated after A.8 | 10.8 rule | FP-A.4 |
| A.12 | iPad check | — | 👤 HUMAN — the rep syncs (11.3), fully closes and reopens the app, opens a new Draft quote for `<STORE_1>`, adds the product and sets the quantities from the A.9 table (at least the first and last unit of the first discounted band, and the first unit of the top band). The agent compares the shown Net Unit Price with the A.9 results. With `ADD_TO_ASSORTMENT = yes`, also check that the product is in the Store Assortment view (★ if Priority) | Every checked price equals A.9; product visible in the views chosen | Read-only | FP-A.4, FP-A.5 |
| A.13 | Record the new baseline | Manifest | Write the new tier counts per thread schedule and the addendum product, so later core checks (7.10 re-run, 13.0a) compare against them (see *Changes outside this file*) | Manifest updated | Overwrite the `addendum-a-NN` section only | FP-A.6 |

### Step details

**A.in Validation and normalisation**

Apply these in order. Show every change to the customer as "you said → I will use", and write only after they accept.

1. **Integers, ascending, no gaps, no overlaps:** the lower bound of each band = the previous band's upper bound + 1. v6 writes contiguous inclusive bands (starter 1–9, 10–999). The enablement deck's Slab example only works with inclusive bounds (100–199 and 200–299; 250 units at list 9.00 = 2,159.10; [release-264-pricing.md](./release-264-pricing.md), lines 33–35). A.9 tests both edges of every band to prove it on the org.
2. **The first lower bound is ≥ 1.** "0 to 100" becomes 1–100. Whether `LowerBound = 0` is accepted is undocumented.
3. **The top band needs an explicit upper bound.** v6 always sends one, and whether an empty `UpperBound` is accepted is undocumented. The recommendation is `999999`: a quantity above the top bound matches no tier and gets list price, so the bound must be above any real order.
4. **"Above N" means N + 1 and up.** For example ">301" next to "201–300" becomes 301–999999, not 302–999999. Ask if the intent is unclear.
5. **A 0 % band is optional.** Quantities below the first band already get list price (same deck example: 1–99 at 9.00). Keep the 0 % band if the customer gave it. It is explicit, and A.9 proves it either way.
6. **Each % is between 0 and 100. The method is `Range` or `Slab` only.** Cumulative is rejected (FP-7.7).
7. **Currency:** the `ADD_LIST_PRICE` currency = the price book currency = every served store's currency = the rep's `DefaultCurrencyIsoCode`. Otherwise ⏸ STOP (FP-A.2). This is the same rule as G12 and FP-4.8.

**Worked example shown at A.in (Femivanta Vitamin, list €10.00).** Show it on the customer's numbers before they pick the method:

| Quantity | Range: net unit price | Range: line total | Slab: line total | Slab: average unit |
| :-- | :-- | :-- | :-- | :-- |
| 100 | 10.00 | 1,000.00 | 1,000.00 | 10.00 |
| 250 | 9.00 | 2,250.00 | 2,400.00 (100 × 10.00 + 100 × 9.50 + 50 × 9.00) | 9.60 |

**A.7 Tier insert (fast path)**

```json
POST /services/data/v<API_VER>/composite/sobjects
{
  "allOrNone": true,
  "records": [
    {"attributes":{"type":"PriceAdjustmentTier"},"PriceAdjustmentScheduleId":"<ADD_SCHED_ID>","Product2Id":"<ID:add-product-id>","ProductSellingModelId":"<ID:psm-id>","LowerBound":"<TIER_LOWER>","UpperBound":"<TIER_UPPER>","TierType":"AdjustmentPercentage","TierValue":"<TIER_PCT>","EffectiveFrom":"<ADD_TIER_FROM>","EffectiveTo":"<ADD_TIER_TO>"}
  ]
}
```
- There is one record per band, in band order. The field names and value shapes are those of the 7.10 body. Only the parent changes: it is the existing schedule Id instead of `@{sched1.id}`.
- No `PriceAdjustmentSchedule` write and no procedure change: the 7.13 constants still point at the same two schedules.
- **Active-schedule fallback.** v6 (7.10, "activating with a partial tier set is the documented failure mode") and the deck (release-264-pricing.md:33, "leave Active unchecked while loading tiers… then reactivate") both load tiers with the schedule inactive. Inserting into an active schedule has not been tried.
  - If the insert is rejected, or A.9 shows the new tiers are not applied after the A.8 refresh, the fallback is: deactivate `<ADD_SCHED_ID>` → insert → 7.10a → reactivate (7.11) → 7.12 → A.9.
  - **While the schedule is inactive, no product on that schedule gets its volume discount on server pricing.** That includes the core products.
  - So on a `customer` org the fallback is a live ⏸ STOP (new consent item CA1: "Volume pricing on `<schedule name>` is off for every product until reactivation, about N minutes"). On a `test` org it follows `IRREVERSIBLE_CONSENT`.
  - Test on a test org first and record which path worked here, so this note can drop its *unverified* tag.

**A.9 Simulation cases**

The body is the 7.14 body. Use `configurationVersionId` = the active version's `9QM` Id (from the manifest, as 7.20), `PriceBooks` = the `ADD_LIST_PRICE` price book, and `CurrencyIsoCode` = its currency. Send one line per case in one call:

- For each band: one case at the lower bound and one at the upper bound. The top band uses its lower bound only.
- **Regression:** the core `Range` (or `Slab`) product of `PRICING_TIERS`, at its first band's lower bound, expected as recorded at 7.14.
- **No tier:** the new product at quantity 1 when the first band starts above 1.

Expected values for the Femivanta example (Range, list 10.00):

| Qty | 1 | 100 | 101 | 200 | 201 | 300 | 301 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Expected `NetUnitPrice` | 10.00 | 10.00 | 9.50 | 9.50 | 9.00 | 9.00 | 8.50 |

- If qty 100 returns 9.50 or qty 200 returns 9.00, the upper bound is exclusive on this org. Stop, report it, and adjust rule 1 for the run.
- If qty 101 returns 10.00, the new tiers are not being read: see the A.7 fallback and FP-A.4.

### End State

| Area | End state | Where to see it | Verified by |
| :---- | :---- | :---- | :---- |
| Product | `ADD_PRODUCT` active, PSMO on the shared PSM, in its category, Standard and custom PBE at `ADD_LIST_PRICE` | `Product2`, `PricebookEntry` | A.1–A.4 |
| Availability | Quote PTA (and Visit PTA if in the assortment) active for `<TERRITORY>`, shared to the rep as 7.8 / 9.22 | `ProductTerritoryAvailability` | A.5, A.6, A.10 |
| Tiers | One tier per band on the existing schedule of `ADD_TIER_METHOD`, all dated; decision tables refreshed after the insert | `PriceAdjustmentTier` | A.7–A.8 |
| Pricing | The active procedure version is unchanged; the simulation matches every band edge, and the existing tiered product still prices as at 7.14 | Simulation response | A.9 |
| iPad | New cache active; the rep sees the product and the tiered price | Device | A.11, A.12 |

### Checkpoint A

Report: status `PASSED`, or `PASSED_WITH_NOTES` only with `KNOWN_LIMITATION` (UoM not applied; Store Check targets not set for the product).
- **Changed:** 1 product, 1 PSMO, 1 category membership (+1 category if new), Standard and custom PBE, 1 Quote PTA and its shares, N tiers on `<schedule name>`, 1 decision-table refresh and 1 cache regeneration. With `ADD_TO_ASSORTMENT = yes`, also 1 `AssortmentProduct` and 1 Visit PTA.
- **Reused:** catalog, PSM, price book, schedules, procedure, recipe, STT and selector.
- **State which A.7 path ran:** active insert, or the CA1 fallback.

**Addendum A is complete.**

## Troubleshooting rows (to add to the index)

| ID | Error signature | Root cause | Fix | Step |
| :-- | :-- | :-- | :-- | :-- |
| FP-A.1 | A.in rejects the bands | Gap, overlap, lower bound 0, no top bound, or "above N" read as N + 2 | Accept the normalised bands A.in shows | A.in |
| FP-A.2 | The new product does not show in Add Products, or shows with no price | Its price book currency differs from the rep's `DefaultCurrencyIsoCode` or the store currency (same defect as FP-4.8) | Price the product in the rep's and store's currency; don't change existing quotes | A.in, A.4 |
| FP-A.3 | The tier insert on an active schedule is rejected (error text not yet observed) | Unverified platform rule on adding tiers to an active schedule | A.7 active-schedule fallback, after CA1 consent | A.7 |
| FP-A.4 | Web simulation gives the tiered price, the iPad charges list price for the new product | Cache not regenerated after A.8, tier dates empty, or app not relaunched after sync | 7.10a → 7.12 → 10.8 → 11.3 sync → relaunch | A.9, A.11, A.12 |
| FP-A.5 | The product is missing from the Store Assortment view on the iPad | `ADD_TO_ASSORTMENT = no`, or the Visit PTA or its share is missing | Run A.10, then A.11–A.12 | A.10 |
| FP-A.6 | A later 13.0a or 7.10 re-run reports an unexpected tier count | Those checks count `PRICING_TIERS` bands only | Compare with the A.13 manifest baseline | A.13 |
| FP-A.7 | Quantities above the top band get list price | The top band's upper bound is below the order quantity | Raise the bound with a new tier row only after the customer's yes; never PATCH a live tier silently | A.in, A.7 |

## Proposed `known-causes.yaml` entries

```yaml
  - id: om-addendum-tier-bands-not-contiguous
    title: Volume tier bands leave a quantity uncovered
    module: order-management
    symptoms:
      - "a quantity gets list price although it is inside the discount range the admin described"
    keywords: [PriceAdjustmentTier, LowerBound, UpperBound, Range, Slab, volume tiers]
    cause: |
      The bands have a gap (for example 201-300 followed by 302+), start at 0,
      or the top band's upper bound is lower than the order quantity.
    fix:
      - "Bands are inclusive integers: each lower bound is the previous upper bound + 1, the first starts at 1."
      - "Give the top band an upper bound above any real order quantity."

  - id: om-addendum-new-product-list-price-on-ipad
    title: New product prices at list price on the iPad only
    module: order-management
    symptoms:
      - "simulation shows the tiered price but the iPad shows list price for a product added after setup"
    keywords: [PriceAdjustmentTier, EffectiveFrom, EffectiveTo, refreshDecisionTable, LifeSciMobileMetadataRecord, sync]
    cause: |
      The tiers were added after the last mobile metadata generation, the tier
      dates are empty, or the app was not relaunched after the sync.
    fix:
      - "Check tier dates (schedules first, then tiers), refresh the decision tables, regenerate once, sync, fully close and reopen the app."
```

## Changes outside this file

The core thread stays as it is, with one exception (item 4).

1. **Run guide.** Add one line under the start prompt: "To add a product with its own list price and volume tiers after setup, run [Addendum A](./release-264-store-check-order-management-steel-thread-addendum-a-product-tiers.md)."
2. **Module `_index.md`.** Link the addendum next to v6.
3. **Troubleshooting index.** Add the FP-A rows above. Add the two `known-causes.yaml` entries (40 → 42).
4. **v6, two postconditions. This is the only core edit, and it is needed for re-runs after an addendum.**
   - **13.0a** says "2 schedules and 4 tiers (`<PRICING_TIERS>`: 2 bands × 2)". It should read "2 schedules and the tier count in the manifest (`PRICING_TIERS` bands plus any Addendum A bands)".
   - **The 7.10 postcondition** ("each with one tier per band of its `<PRICING_TIERS>` entry") needs the same wording.
   - Without this change, a core re-run on an org that ran Addendum A reports a false mismatch (FP-A.6).

## Out of scope (candidate Addendum B)

- **A separate schedule for the new product**, for example different effective dates or a price-book-specific schedule. The procedure embeds one constant and one tier step per schedule (7.13), so a third schedule needs a new procedure version: rerun 7.13–7.15 at the next rank, activating the new version before deactivating the old one. The v6 graph does not document how a third tier step is added, so that path is not proposed here.
- **Changing an existing product's price or bands.** v6 never PATCHes a live `UnitPrice` or tier. That needs its own consent and rollback design.
- Store Check targets for the new product (9.19), and amount or override tier types.

## Review questions for Harsha and Manoj

1. **Tier inserts into an active schedule.** Have you inserted a `PriceAdjustmentTier` into an active schedule in a run? Was it accepted, and did it price after a 7.12 refresh alone (A.7)?
2. **Bound semantics.** Are bounds inclusive on 264? Does an empty `UpperBound` or a `LowerBound = 0` work? Rules 2 and 3 assume neither works.
3. **Separate file or appended section.** Is a separate file under `knowledge/modules/order-management/` acceptable, or would you rather append this to v6 after Act 15?
4. **The item 4 edit.** Are you OK with changing the two postconditions (13.0a, 7.10) to read the manifest baseline?
5. **Regeneration.** Does 10.8 apply to tier and price book inserts alone? It lists decision tables and tier dates as triggers, so I assumed yes.
