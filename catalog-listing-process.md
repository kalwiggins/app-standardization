# Catalog Listing Process

> **Scope.** The standard process for building an ecommerce catalog from a supplier feed: competitor research to decide what to list, enrichment, title and copy rules, verification, and the Foundry build. Reference implementation is `catalog-rvgearpro` (RVGearPro and OffRoad Gear Pro); any new catalog or storefront repo should follow the same stages and rules, swapping in its own suppliers, competitors and taxonomy. Script names below refer to that repo.

The end-to-end process for taking a supplier SKU and turning it into a live, priced, imaged, original-copy listing on RVGearPro (RVG) or OffRoad Gear Pro (OGP). Written 2026-09-13 from the scripts in this repo and the batch notes under `data/`.

The one-line version: **competitors decide what belongs on the store, the supplier feed decides what we can source, Amazon and the manufacturer supply the facts, we write the words, Foundry holds the record.**

---

## 1. The principles behind every decision

1. **SKU-level, never brand-level.** A brand tells you nothing. Dometic USA is 202 RV SKUs; Dometic Marine is 3,317 boat SKUs under the same name. We decide product by product.
2. **A competitor listing is the relevance signal.** An RV-specialist retailer stocking a part is proof it belongs on an RV store. We never guess relevance from a title.
3. **Identity comes from the supplier, facts from the best source, words from us.** The supplier record (Meyer channel-SKU or Keystone VCPN) is the identity and the cost. Amazon or the manufacturer supplies specs, fitment and images. Every sentence of copy is written fresh.
4. **Never invent.** No spec, warranty, certification, fitment year or brand history that is not in the seed. When a fact conflicts across sources, resolve it or drop it. A missing fit costs a sale; a wrong fit costs a return.
5. **Price is a policy, not a negotiation.** MAP when the brand has one, otherwise 30% gross margin, and only ever adjusted by a documented reprice script with a manifest.
6. **Everything is idempotent and leaves a manifest.** Every build, reprice and image pass can be rerun safely and can be reversed from its backup file.

---

## 2. The stack

| Piece | What it is |
|---|---|
| **Foundry IMS** | Catalog of record. Products, variants, brands, categories, custom fields, channels. REST at `api.foundryims.com/api/v1`, key in `.env`. |
| **Meyer** | Primary supplier feed. ~800k channel-SKUs, 1,627 brands, inbound dropship channel. Carries title, MPN, UPC, cost, MAP (spotty), weight and dims. No images, no category. |
| **NTP / Keystone** | Second supplier. CSV feed keyed by VCPN with JobberPrice, Cost, UPC, freight columns. No MAP field. Loaded as its own supplier channel. |
| **RVGearPro channel** | Headless storefront channel for rvgearpro.com. SKU prefix `RVG`. Taxonomy "RV X" (7 departments, 39 mids, 147 leaves). |
| **OffRoad Gear Pro channel** | Second storefront for ATV, UTV and overlanding. SKU prefix `OGP`. Taxonomy "ATV & UTV X" plus "Overlanding X". Has its own custom navigation. |
| **competitors.db** | SQLite in `data/`. Crawled competitor listings, Meyer SKUs, Meyer pricing, Amazon enrichment. Every stage reads and writes here. |
| **easyparser** | Amazon data API. UPC to ASIN lookup, product detail, keyword search. Roughly 2 credits per item. Needs a browser User-Agent. |
| **Compose subagents** | Claude subagents that write the copy, one per slice of 15 to 20 seeds, reading a spec file and writing JSONL. |

---

## 3. Stage 0. Deciding what to list

This is the competitor research. It runs before any money is spent on enrichment.

### 3.1 Crawl the competitors

We crawl full product catalogs from the RV retailers that have already done the curation work.

| Site | Access | What we capture |
|---|---|---|
| RVUpgradeStore (Volusion) | Sitemap, ~13k products | Richest. Brand, UPC, MPN, price, store SKU. RVupgrades.com redirects here, so it is not crawled separately. |
| RV Parts Country (Shift4Shop) | Sitemap, ~8k products | MPN, price. Brand is guessed from the title. UPC rarely present. |
| Camping World | Algolia JSON search API, sharded by brand facet | Brand, MPN, price, variants. No UPC or description. Heavy marine noise. |
| etrailer | Sitemap, needs `curl_cffi` impersonating Chrome to pass Akamai | Crawler exists but has never been run at catalog scale (850k products). Best descriptions. Run it targeted by brand. Its part numbers are brand-prefixed (CAM40155), so only the numeric suffix is kept as MPN and matches rely on brand agreement. |

Every crawler writes the same row: competitor, brand, store SKU, MPN, UPC, title, price, URL, description (2,000 chars), has_variants, scraped_at.

Rules: real desktop Chrome User-Agent (bot tokens get 403s), 0.4 to 1 second between requests, exponential backoff on 403, 429 and 503, checkpoint every 200 rows and skip already-stored URLs on resume. Competitor descriptions are seed material only and are never copied.

Scripts: `crawl.py <competitor>`, `scraper/*.py`. Data lands in the `competitor_skus` table (keyed by competitor plus URL, upserted) with an append-only `price_history` table so re-runs track price moves and new SKUs.

### 3.2 Join to the supplier feed

`build_catalog.py` matches every competitor listing to a Meyer channel-SKU. The join order is:

1. **UPC exact.** Accepted with no brand check, because a barcode is globally unique. Mostly RVUpgradeStore.
2. **Exact MPN plus brand agreement.** MPN is normalized by strip and uppercase only; dashes are kept because they can be significant. Every MPN hit must also pass a brand test: the first meaningful token of the Meyer brand (3+ characters, not a stop word like "inc", "products", "rv") has to appear somewhere in the competitor's brand, title, description or URL. Exactly one surviving candidate is a match. More than one is ambiguous and dropped. MPN alone collides across brands (Camco 22813 versus Auto Meter 22813; Garmin and NGK both have a 72877).

There is no fuzzy or scored matching at this stage. It is binary on purpose. Never use Foundry's `search=` parameter for matching either; it does substring matching and returns wrong parts.

Sub-brand mapping (competitor "Camco" covering Meyer "Eaz Lift", Meyer splitting "Dometic USA" from "Dometic Marine") is a small hand-maintained map, still an open refinement. Pack size is also unresolved: a `pack_mismatch_warn` flag is set when Meyer's pack size is above 1, but margins are not divided by it, so competitor multipacks can inflate the top of the margin sort.

Output: `data/catalog_matches.csv` (every competitor to Meyer match) and `data/catalog_unmatched.csv` (competitor products Meyer does not carry, which is the sourcing gap list behind `supplier-gap-brands.md`).

### 3.3 Consolidate into tiers

`consolidate.py` collapses matches to one row per Meyer SKU in `data/items_to_sell.csv`, with:

- **Tier A**: listed by an RV specialist (RVUpgradeStore or RV Parts Country). Trusted.
- **Tier B**: Camping World or etrailer only. Verify before listing, because Camping World owns Overton's and half of its Meyer overlap is boat parts.
- **Margin** against the lowest sane competitor price (anything over 20x cost is a data error and dropped).
- **below_cost** flag when a competitor already sells under Meyer's wholesale cost. Those are not listed from Meyer.
- Source list, family and variant flags. Sorted by tier, then source count, then margin.

Regeneration order for the whole stage: crawl each site, `pull_meyer.py`, `build_catalog.py`, `consolidate.py`, `relevance_filter.py`, `map_categories.py` (adds category path and id), then `pull_meyer_pricing.py` for cost, MAP, weight and dims.

The first full run produced 13,561 distinct items, 8,440 Tier A, of which about 5,400 were profitable. That profitable Tier A set was the first build.

### 3.4 Relevance filter

`relevance_filter.py` quarantines obvious non-RV noise with a brand blocklist and word-boundary keyword rules. Naive keyword filters false-positive badly ("fifth WHEEL", "4 GAUGE wire", anode "ROD", awning "DUNE"), so the filter is deliberately light and Tier A came out about 97% clean. Decisions locked with the user:

- Keep tow-vehicle and truck items (Bilstein, heavy-duty wipers).
- Drop marine crossover (Taylor Made, Navico, TH Marine, Attwood, Johnson Pump, Garmin).
- Never drop grey-area items before enrichment. Enrich first, then decide from clean Amazon data.

Quarantined items go to a CSV, never silently deleted.

### 3.5 Other candidate sources

- **Meyer's printed catalogs.** The 2026 RV, Powersports and Overlanding flipbooks expose their full text layer in a public `book_config.js`. Part numbers in that text are Meyer's own curation of what is RV-worthy or powersports-worthy. `flipbook_candidates.py`, `mine_powersports.py` and `mine_overlanding.py` pull part-number-shaped tokens per page, accept a feed match only when the brand also appears on that page (kills MPN collisions), and drop anything already in a `listings_*.jsonl`. Candidates are ranked by competitor source count, RV-specialist presence, margin, brand tier, and page prominence (a spotlight page beats a dense parts list). The shortlist is the triple intersection: flipbook-featured, competitor-validated, not yet carried.
- **Second supplier brand scan.** For NTP/Keystone, scope by brand first. Compare Cost to JobberPrice across the feed. A brand where cost equals jobber has no margin at any price and is excluded before any other work (1,421 of 2,065 Husky Liners SKUs, for example). The punchlist lives in `~/Downloads/supplier-price-compare/PUNCHLIST.csv`.

### 3.6 Price reality check before composing

Do this before spending compose tokens on a batch. Pair the supplier's cost and jobber with the Amazon street price (`amz_price` from enrichment) and, for any brand that sells direct, the brand's own website price. Then identify the pricing regime:

1. **MAP as a constant fraction of jobber.** The Amazon-to-jobber ratio clusters tightly on one number (Husky Liners 0.90, Covercraft and Method 1.00). Price at that number.
2. **Jobber is a fiction.** Ratio is low and scattered (Cattleman median 0.55). Price to the observed street price, or to the median street-to-cost ratio of the product type.
3. **No discount.** Cost equals jobber. Exclude.
4. **Manufacturer direct price is the ceiling.** If the brand runs its own store, nobody buys from us above that price. TrailFX killed two marquee items this way.

Drop any SKU where Amazon street sits below our cost (32 of 45 TrailFX LED lights). Freight-heavy items (LTL) only work as stocked inventory, never single-unit dropship; see `ntp_freight_cost.py`.

---

## 4. Stage 1. Enrichment

Meyer titles are often just the brand name and the feed has no images, so every candidate is enriched from Amazon.

**Per item** (`enrich_scaled.py` for Meyer, `enrich_ntp.py` for NTP, `pilot_enrich_price.py` for OGP batches):

1. `PRODUCT_LOOKUP` by UPC. A barcode match is identity.
2. If no UPC hit, `SEARCH` on "brand mpn".
3. Verify before spending the next credit: the Amazon brand must agree with ours, or the Amazon model number must agree with our MPN (allowing brand-prefixed MPNs like CMC02103 versus 02103). Failures are recorded as MISMATCH and never fetched further.
4. `DETAIL` on the ASIN: title, feature bullets, specifications, description, images, sibling ASINs (`variant_asins_flat`), weight, street price.

Results go to the `amazon_enrichment` table, keyed by supplier SKU, with INSERT OR REPLACE so a killed run resumes by re-running. Six workers, roughly one item per second.

**Match verification** (`rescore_matches.py`, `verify_ntp_matches.py`): UPC-resolved is `confirmed`. Search-resolved needs model agreement, otherwise `review`. A product-type agreement check compares the supplier description to the Amazon title and flags `MISMATCH:type`. The strongest single identity check is **MPN present in the Amazon title**. It caught a UPC that returned the adjacent color and a floor liner whose UPC returned a bed liner.

**Confidence score** (`data_confidence` custom field), computed in `assemble_worklist.py`: 2 points if resolved by UPC else 1, plus competitor source count up to 3, plus 1 if an RV specialist carries it, plus 1 if the Amazon price is within 25% of our sell price. HIGH is 5 or more, MEDIUM 3 or more, else LOW. LOW items and search-resolved ASIN collisions are held, not built. Later batches use a simpler vocabulary on the same field: MEDIUM (Amazon match with images), NO_AMAZON, and RESEARCHED (images and fitment came from the manufacturer).

**When Amazon is wrong or empty, go to the manufacturer.** Amazon is unreliable for fitment-critical parts. Known good sources:

- TrailFX and Cattleman: Shopify `/products/<handle>.json` plus the YMM fitment API (`/apps/ymm/app/verify`).
- Covercraft: OCC API at `p1api.covercraft.com` with `fields=FULL`. Validate the response against `finalPartNumber` and UPC, because a wrong SKU returns a different product with a 200. The fitment list caps at 100 rows and truncates year ranges silently. Corroborate and only ever widen.
- Husky Liners: huskyliners.com product pages for fitment and the one part-specific photo.
- RSI SmartCap and Stowe: manufacturer fitment tables, images via `attach_researched_images.py`.

Take the printed design or color from the feed and vehicle fitment from the manufacturer when they disagree.

---

## 5. Stage 2. Assemble the worklist

`assemble_worklist.py` (Meyer), `assemble_ntp.py` (NTP, batch-parameterized by `NTP_BATCH=<name>`), `assemble_ogp.py` (OGP) join enrichment, pricing, relevance and family detection into one worklist CSV plus compose seed JSON. Each row carries supplier SKU, brand, MPN, UPC, ASIN, cost, MAP, sell price, price source, images, bullets, specs, confidence, family id and size label.

**Families.** Only size-type variation (length, BTU, gallons, inches, trailer length) becomes a Foundry FAMILY product with one child variant per size. Color and orientation stay separate SIMPLE products. Family signal comes from Amazon sibling ASINs plus brand and base-MPN with the size suffix stripped; Meyer's own group key is empty. Freeze the seed file before dispatching composers, because slices are dispatched by seed index and re-assembling mid-compose shifts them.

**Categories.** Every listing must land on a leaf. The allowed leaf list is `data/leaf_paths.txt` (or a per-batch `leaf_paths.txt`). A batch may pin every SKU to a path in `category_map.json`, which the merge step enforces byte for byte. Assign the bottom-most leaf that makes sense; create a leaf when a real gap appears (Tow Bars, Holding Tanks, Overlanding Floor Liners & Mats were all added this way).

**Specs are pre-curated deterministically** before any LLM sees them. Drop Best Sellers Rank, ASIN, Date First Available, Customer Reviews, Automotive Fit Type and duplicates. This removes the invented-spec risk that showed up in the first full composition run.

**Price** is computed here (see section 9) and written into the worklist with its source label.

---

## 6. Stage 3. Compose the listing

Copy is written by general-purpose Claude subagents, each handed a slice of seeds, the compose spec, and the batch notes. Output is one JSON object per listing in `data/<batch>/compose/*.jsonl`.

### 6.1 The spec

The canonical spec is `data/ntp/COMPOSE_SPEC.md`. Its opening lines set the voice:

> You write product copy for RVGearPro, an RV / travel-trailer accessories store. VOICE: helpful & practical, RVer-to-RVer. Knowledgeable, straightforward, benefit-focused. No hype.

OGP batches carry a voice override in their notes: written for people whose truck is a tool, trailheads, job sites, mud and gear, still practical and no hype.

### 6.2 How a title is built

The format is fixed:

```
<Brand> <product name> | <short spec or fitment tail>
```

- **Brand first**, using the customer-facing brand name in the title even when the Foundry brand record is spelled differently (title says "Husky Liners", brand field is "Husky Liner"; title says "Covercraft", brand field is "Covercraft Industries LLC").
- **Product name** is the real product line and model, decoded from the supplier abbreviation and cross-checked against the Amazon or manufacturer title. Supplier titles are shorthand ("SFS TRLR COVER 15'1'-18'", "DHS021B", "TUB TUNDRA CC 5'6 UR") and are never echoed.
- **Pipe separator** ` | ` between name and tail. Never an em dash.
- **Tail** is the two or three facts a buyer scans for. For general RV parts: capacity, size, voltage, material. For fitment parts: years, make, model, cab or body qualifier, then color. For bolt-pattern parts: the pattern and finish, never a vehicle.
- **Never a bare brand name.** Never a size in a FAMILY parent's name; sizes live in the option labels, and the merge step rejects a family name containing one.
- **No MPN in the title.** The deterministic cleaner strips a trailing "(MPN)" or "| MPN". The part number lives on the variant and in specs.
- **Length.** The deterministic path caps names at 90 characters, cut on a word boundary. Composed names should stay in that range. The SEO meta title is derived separately: everything before the first pipe plus " | RVGearPro", capped at 60.
- **Keep fitment exact.** Never widen a year range in a title. If the fitment source is capped, the name uses the corrected overall span and the description says to confirm the year.

Real examples from the catalog:

| Kind | Title |
|---|---|
| RV part | PullRite SuperGlide 2700 5th-Wheel Hitch \| 15,000 lb Capacity, Short-Bed, Stainless Steel |
| RV appliance | Suburban SF-35VHQ Ducted RV Furnace \| 35,000 BTU, LP Gas, Low-Profile |
| Fitment part | Husky Liners WeatherBeater Front & 2nd Row Floor Liners \| 2015-2020 Ford F-150 SuperCrew, Black |
| Seat cover | Covercraft Carhartt Duck Weave Custom Seat Covers \| Front Row, 2019-2024 Chevrolet Silverado 1500, Brown |
| Bolt-pattern part | TrailFX Wheel Adapter \| 5x114.3 to 5x127, Silver |
| Family parent | ADCO SFS AquaShed Travel Trailer Cover |

When many SKUs in a batch differ only by a qualifier (seat configuration, console type, cab), derive the disambiguating tail by diffing the qualifier clauses within each duplicate group. A fixed vocabulary misses what actually separates them; 19 Covercraft listings collided into 9 names before this rule.

### 6.3 The rest of the listing

| Field | Rule |
|---|---|
| `short` | One or two original sentences. Becomes the `short_description` custom field. |
| `description` | HTML. One or two `<p>`, then a `<ul>` of 4 to 6 `<li>`, then a closing `<p>`. For families the closing paragraph names the sizes or fits offered. |
| `features` | 4 to 6 short strings, stored as `feature_1` to `feature_6` so the storefront controls each bullet. Truncated to 6 at merge. |
| `specs` | Up to 8 `{name, value}` pairs from the seed only. Omit dimensions that vary by size within a family. For Meyer batches specs are attached from the pre-curated worklist at build time, not written by the composer. |
| `category_path` | Exactly one allowed leaf path, verbatim. |
| `labels` (families) | One clean customer-facing option label per variant MPN, unique within the family, abbreviations expanded ("DT PENGUIN I II" becomes "Dometic Penguin I & II"). |

Hard rules in every spec:

- **Original wording.** No Amazon sentences or bullet phrasing. An 8-gram overlap check against the Amazon description runs at merge and copiers are held for rewrite.
- **No invented facts.** Only what is in the seed or the batch notes. No weights, thread counts, ratings, warranty terms or care instructions that are not present.
- **Never mention price**, deals, value or savings.
- Seeds with `has_images=false` are written from the supplier title and the notes only, kept generic, and flagged as an issue naming the SKU.
- **Held routing.** A composer that cannot confirm identity sets `hold` on a single or `review` on a family. Merge and expand divert those to `held_singles.json` and `held_families.json` instead of the build list. Known wrong-product Amazon matches, bundle mismatches and brand hijacks are held the same way.

### 6.4 Batch notes

Each batch directory has a `COMPOSE_NOTES.md` that decodes that supplier's abbreviations, states the voice, pins the name format for that product type, points at the fitment file when Amazon is not to be trusted, and lists the category rule. Reading the existing ones (`ogp_covercraft`, `ogp_husky`, `ogp_trailfx`, `rvg_tail`) is the fastest way to write a new one.

### 6.5 Running composers

- Slice size 15 to 20 seeds, no more than about 6 agents in flight. The first full run at higher concurrency hit rate limits and stalled.
- Namespace scratch filenames per agent; a sibling overwrote another's file once.
- Composers are expected to push back when the data under them changes. When they do, re-diff any slice whose modification time is newer than the merge.
- Merge with `merge_singles_listings.py` or `merge_ntp_compose.py`. Merge validates structure, enforces leaf and pinned categories, maps family labels from MPN to VCPN, and keys by SKU so duplicate slices collapse. Gaps must be filled by hand.

---

## 7. Stage 4. Verify before and after build

Two kinds of verifier, both scripted per batch.

**Pre-build, on the composed JSONL** (`verify_compose_fitment.py`, `verify_batch2.py`): re-read the current fitment and seed files and re-check every composed name, category and hedge against them. Agents' self-reports cannot catch data that changed under them. Check for duplicate names across slices, non-leaf categories, Amazon-copy overlap, missing fields, and any year span in a title that exceeds the source.

**Post-build, against live Foundry** (`verify_ntp.py`, `verify_ogp_batch.py`, `verify_final.py`): for every SKU in the worklist, confirm the supplier channel-SKU is linked to a variant, the variant price equals the worklist price to the cent, cost is present, the product type is FAMILY or SIMPLE as expected, family variants carry an option value, a category is set, a description exists, UPC is present, any MEDIUM-confidence item has at least one image, no image URL points at Amazon (everything must be Foundry-hosted), and inventory aggregated. The OGP verifier also checks the storefront channel-SKU is listed and its own price matches. Writes `verify.csv` and prints the issue list. Zero issues is the bar before storefront listing.

Recurring catches worth re-running on every batch:

- MPN in Amazon title (identity).
- Row and color agreement between supplier description and Amazon title (read only the last pipe segment of the Amazon title, check "cargo" before "2nd/3rd").
- Word-boundary anchoring on short model tokens (`TITAN` matched "Titanium", `E-?350` matched "GLE350").
- Right-hand-drive or export parts labelled correctly but still dropped, because a US buyer will return them.
- Images that are byte-identical across parts are configuration-level line shots, not product photos. Do not attach them.

---

## 8. Stage 5. Build in Foundry

`build_full.py` is the engine; `build_ntp.py` and `build_ogp.py` wrap it with the supplier channel id, SKU prefix and batch paths. Run it with the image venv (`~/Code/catalog-mepselect/.venv-img/bin/python`) so images get normalized. Zero LLM tokens; REST only.

**SKU convention.** `RVG-<brand initials>-<MPN>` (OGP uses `OGP-`). Initials are the first letter of up to three brand words, or the first three letters of a single-word brand. Go Power! 75013 becomes `RVG-GP-75013`; Camco 39080 becomes `RVG-CAM-39080`.

**Per SIMPLE product, in order:**

1. Look up the supplier channel-SKU. If it already has a `variantId`, skip (idempotency key).
2. `POST /products` with name, brandId, categoryId, HTML description, SKU, `type: SIMPLE`. Returns the product with its default variant. Brands are created first, single-threaded, if missing.
3. `PUT /channel-skus/{id} {variantId}` to link the supplier SKU to our variant. This is what makes it sourceable and syncs cost, weight and dims from the feed. It was the step missed in the first pilot.
4. `PUT /variants/{id}` with price, cost, upc, mpn, weight (and `map` where the batch pins one).
5. Images: download each candidate, OCR it, keep the first image as the protected hero, fill to 6 with the least text-heavy images, and drop anything with 6 or more confident words (callout graphics). Each kept image gets background removal, trim, scale so the subject fills 90% of the canvas, centered on a 1200x1200 white square, saved as optimized JPEG, then byte-uploaded through the presigned URL so Foundry hosts its own copy. If the image venv is not active the script silently falls back to raw Amazon URL attach, which the verifier then flags as hotlinks. Always run with the venv.
6. `PUT /products/{id} {customFields}` with `amazon_asin`, `short_description`, `feature_1..6`, `specifications` (JSON array), `data_confidence`, and `image_status`. The custom-fields object is replaced whole, so send every key every time. Product PUT is otherwise partial and preserves everything not sent.

**FAMILY products** add: create the product with `type: FAMILY`, `POST /products/{pid}/options {name: "Size"}`, then per child (sorted by the numeric size; the first child reuses the auto-created default variant, the rest come from `POST /variants` with a temporary `tmp-<sku>`), `PUT /variants/{vid}` with the real SKU, price, cost, UPC, MPN and weight, then `PUT /variants/{vid}/options {options:[{productOptionId, value}]}` to bind the label. That last call is the only way option values persist. Custom fields are written once from the parent member. A leftover `tmp-` SKU means a build died mid-family; `repair_ntp_families.py` finishes it.

**After the batch:**

- `POST /variants/aggregate-inventory` so linked variants show supplier stock.
- `backfill_ids.py` scan and apply, which reconciles any UPC or MPN a transient write failure dropped.
- **List on the storefront.** Creating a product lists it nowhere. `list_ntp_storefront.py` (or `build_ogp.list_on_ogp`) does `POST /channel-skus {channelId, channelSku: foundrySku, variantId, price, cost, upc, mpn, isListed: true}` on the storefront channel. Product pages appear in the sitemap within minutes.
- If the MPN already exists from the other supplier, do not rebuild. Link the new supplier channel-SKU to the existing variant as a second source (`link_meyer_second_source.py`).
- Products with no usable image are built and then archived (`archive_no_image.py`) into an image queue rather than published bare.

**Build gotchas that are now handled in code but worth knowing:** 409 DUPLICATE_FOUNDRY_SKU and dropped connections are transient and retried 4 times; control characters in image URLs are stripped; SKUs containing `/` need a safe filename; Foundry rate-limits at the CloudFront edge around 5k calls per 10 minutes, so builds run with a small pause or 3 to 8 workers; background shell jobs get killed, so run builds foreground in chunks and rerun, the build is idempotent.

---

## 9. Pricing policy

The rule since 2026-09-08 (the older 20% figure in `SCALEUP-PLAN.md` is superseded; every assemble script uses 0.70):

```
price = MAP                              if the supplier states one (or the batch pins one)
      = floor(cost / 0.70) + 0.99        otherwise (30% gross margin, 42.86% markup)
```

Refinements by supplier:

- **NTP/Keystone** has no MAP field. Jobber is the natural ceiling, so `assemble_ntp.py` uses `min(formula, floor(jobber) - 0.01)`. On brands where MAP equals jobber (Covercraft, Method) pin `map = JobberPrice` in the batch CSV so we sit at MAP, not a cent under it.
- **MAP inference.** If a brand shows MAP on Meyer, treat it as MAP-controlled on Keystone too. If the Amazon-to-jobber ratio clusters on one constant, that constant is the MAP ratio (Husky Liners is exactly 0.90).
- **Market-aware overrides.** A `price_override` column (source "market") sets a specific price; `reprice_to_market.py` raises to just under street when we are far below it; `reprice_to_map.py` moves in either direction to MAP and stamps the variant `map` field. All dry-run by default, all write a manifest.
- **Amazon street is intel, not a cap** on MAP brands. Being above Amazon on a MAP item is expected.

**Storefront price is separate from variant price.** Each storefront channel-SKU carries its own price, copied only at list time. Any reprice must PUT both `/variants/{id}` and `/channel-skus/{id}` on the storefront channel. Never use Foundry's "List all products" button to refresh prices; it re-lists intentionally unlisted items.

Re-importing a landed-cost feed changes cost but not price, so margins erode silently until a reprice pass.

---

## 10. Per-batch checklist

1. Scope the brand: cost-to-jobber ratio, pricing regime, manufacturer direct price, MAP evidence. Exclude no-margin and below-street SKUs now.
2. Build `data/<batch>/families.csv` (13-column schema), `brands.json`, and `exclude.json` if needed.
3. Enrich, then verify matches. Decide whether Amazon or the manufacturer is the fitment source.
4. Assemble. Freeze the seed file. Write `leaf_paths.txt` or `category_map.json` and `COMPOSE_NOTES.md`.
5. Dispatch composers in waves. Merge. Run the pre-build verifier. Fix duplicate names and gaps by hand.
6. Build with the image venv. Rerun until the skip count equals the batch size.
7. Aggregate inventory, backfill ids, run the post-build verifier to zero issues.
8. List on the storefront channel. Link any second sources. Archive no-image items into the queue.
9. Spot-check a handful of live pages: name, category, price, images, custom fields.
10. Record the batch in memory notes: what was excluded and why, any new category, any pricing regime discovered.

---

## 11. Script index

| Stage | Scripts |
|---|---|
| Crawl and match | `crawl.py`, `scraper/`, `build_catalog.py`, `consolidate.py`, `relevance_filter.py`, `build_priority.py` |
| Candidate mining | `flipbook_candidates.py`, `mine_powersports.py`, `mine_overlanding.py`, `filter_powersports.py` |
| Supplier feed | `meyer.py`, `pull_meyer.py`, `pull_meyer_pricing.py`, `ntp_batch.py`, `ntp_freight_cost.py` |
| Enrich and verify match | `enrich_scaled.py`, `enrich_ntp.py`, `pilot_enrich_price.py`, `rescore_matches.py`, `verify_ntp_matches.py`, `classify_husky.py` |
| Assemble | `assemble_worklist.py`, `assemble_ntp.py`, `assemble_ogp.py`, `group_families.py`, `family_build.py`, `map_categories.py`, `prep_compose_v2.py` |
| Compose and merge | `compose_v2.workflow.js`, `data/ntp/COMPOSE_SPEC.md`, `merge_singles_listings.py`, `merge_ntp_compose.py` |
| Verify | `verify_compose_fitment.py`, `verify_batch2.py`, `verify_ntp.py`, `verify_ogp_batch.py`, `verify_final.py` |
| Build | `build_full.py`, `build_ntp.py`, `build_ogp.py`, `foundry_client.py`, `normalize_image.py`, `img_select.py`, `attach_researched_images.py` |
| Post-build | `backfill_ids.py`, `list_ntp_storefront.py`, `link_sources.py`, `link_meyer_second_source.py`, `archive_no_image.py`, `kill_listings.py` |
| Pricing | `reprice_to_map.py`, `reprice_to_market.py`, `trailfx_direct_price.py` |
| Categories and SEO | `create_categories.py`, `build_ogp_nav.py`, `write_category_content.py`, `write_brand_content.py`, `product_meta.py` |
