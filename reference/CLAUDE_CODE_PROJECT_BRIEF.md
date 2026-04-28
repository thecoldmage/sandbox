# Scaffold Express — Competitor Price Comparison: Product Matching Project

## Objective

Build a comprehensive competitor price comparison dataset by matching Scaffold Express's product catalog against 6 competitors. The output is a single spreadsheet where each row is a Scaffold Express product, with columns for each competitor's matched product name, price, and URL.

---

## Source Data

**File:** `Combined_Competitor_Price_List__2_.xlsx`

This Excel file contains 8 tabs. Each competitor tab was scraped from their website.

### Tab: ScaffoldExpress (our catalog)
- **479 active products** (filter `Status (Product)` == `ACTIVE`; ignore DRAFT/ARCHIVED)
- Columns: `Sku`, `Title (Product)`, `Price`, `Online Store Url (Product)`, `Status (Product)`
- SKU format: `PSV-XXX` (e.g., PSV-100, PSV-CL-8105, PSV-RL-9475, PSV-KR-ST21)

### Competitor Tabs

| Tab | Products | Title Column | Price Column | URL Column | Notes |
|-----|----------|-------------|-------------|-----------|-------|
| ScaffoldMart | 378 | ProductTitle | Price + Sale Price | sitemap-href | Has separate sale price column |
| ScaffoldStore | 312 | ProductTitle | Price | sitemap-href | Prices formatted as text: "Our Price: $49.80" |
| USAScaffolding | 485 | ProductTitle | Price | sitemap-href | Some products sold as multi-packs (e.g., "4-Pack") |
| ScaffoldSupply | 337 | ProductTitle | Price | sitemap-href | Has a junk row "volusiontest" — skip it |
| BadgerLadder | 158 | ProductTitle | Price | sitemap-href | Sells mostly kits/packages; fewer individual components; price ranges present |
| Granite | 272 | ProductTitle | Price | sitemap | NOTE: URL column is `sitemap`, not `sitemap-href`; sells branded product lines (Amplify, BlackTop, AXIS); many bundles |

### Tab: TrackedProducts
- 130 products previously matched manually
- Can be used as validation/ground truth to test match quality

---

## The Core Challenge: Product Matching

These companies all sell the same types of scaffold equipment, but use completely different naming conventions. There are no shared SKUs or universal product IDs. Matching must be done by understanding what the product actually is.

### Why Text Similarity Fails

A naive text-matching approach was attempted and produced ~50% accuracy on non-obvious matches. The problems:

1. **Shared vocabulary swamps signal.** "Scaffold," "frame," "tube," "cross brace" appear in hundreds of products. The distinguishing details — a dimension, a style code, a stem diameter — are a small fraction of the title.

2. **Naming conventions differ radically across sites.** Examples:
   - SE: `7" X 1-3/8" W-Style Coupling Pin w/ 1/8" Collar - PSV-100`
   - ScaffoldMart: `Stack Pin – W-Style, Collared`
   - USAScaffolding: `7″ x 1-3/8″ Coupling Pin | w/ 1/8″ Collar`
   - ScaffoldSupply: `W-Style Coupling Pin with Collar`

3. **Dimensions are critical but inconsistently specified.** A 5'x5' frame is NOT a 5'x6'4" frame, but if a competitor just says "5' Scaffold Frame," you can't tell which one without reading the product page.

4. **Style codes matter.** S-Style, W-Style, BJ-Style, and V-Style are different products with different tube diameters and locking mechanisms. Some competitors omit style entirely.

5. **Competitors sell bundles/kits.** SE sells individual components; some competitors (especially Granite, BadgerLadder) sell multi-packs or kits. A "5'x5' Scaffold Frame – 8 pack" is not a direct comparison to a single frame unless you divide the price by 8.

6. **Cup Lock vs Ring Lock.** SE sells both Cup Lock and Ring Lock system scaffold lines. These are similar but distinct product lines. ScaffoldStore labels theirs "Ring-Lock" even when the equivalent SE product is "Cup Lock." They may actually be the same item — needs verification by checking product pages.

---

## Recommended Approach

### Stage 1: Categorize All Products

Classify every product (SE + all competitors) into categories. This prevents cross-category false matches (a "5' guard rail" matching to a "5' cross brace").

Suggested categories based on the SE catalog:

| Category | Count | Key Differentiators |
|----------|-------|-------------------|
| Coupling Pins | 6 | Length, stem diameter, style (W/BJ/S), collar (yes/no, size) |
| Span Pins | 2 | Round vs square |
| Toggle Pins | 2 | Length (long vs short) |
| Cross Braces (tube) | ~20 | Height x width dimensions, style |
| Cross Braces (angle iron) | ~12 | Height x width, heavy-duty vs standard |
| Guard Rails | 14 | Length |
| Guard Rail Posts | 6 | Style (W/S/BJ), male vs female, flip lock |
| Guard Rail Panels | 4 | Length, end vs side |
| Ladder Frames (single) | ~12 | Width x height, style |
| Ladder Frames (double) | ~10 | Width x height, style |
| Ladder Frames (triple) | ~3 | Width x height, style |
| Walk-Thru Frames | ~19 | Width x height, style, heavy-duty |
| Mason Frames | 4 | Width x height, style |
| Shoring Frames | ~12 | Width x height |
| Screw Jacks | 9 | Length, stem diameter, fixed vs swivel base |
| Socket Jacks | 2 | Stem diameter |
| Base Plates | 9 | Size, stem diameter, type (tube-lock, extension, etc.) |
| Casters | 8 | Diameter, stem type (square/round), stem diameter |
| Decks/Walkboards | 13 | Length, width, material (aluminum, alum/plywood), hatch vs standard |
| Steel Planks | 9 | Length, hook type (raised vs flat) |
| Clamps | 28 | Type (swivel, right angle, beam, half, stud, A/C), bolt type (I vs T) |
| Side Brackets | 8 | Width, type (tube/saddle/angle iron), board count |
| Outriggers | 3 | Length |
| Post Shores | 16 | Range, capacity (10K/25K), galvanized vs black |
| Scaffold Towers | 18 | Height, material, rolling vs non-rolling |
| Stair Components | 24 | Type (unit, tower, stringer, starter bar), height |
| Tubes w/ End Fittings | 6 | Length |
| System Scaffold (Ring Lock) | ~45 | Component type (vertical, ledger, double ledger, bay brace, etc.), length |
| System Scaffold (Cup Lock) | ~42 | Same as Ring Lock but Cup Lock line |
| Other | ~100+ | Miscellaneous (pins, locks, connectors, mortar stands, etc.) |

### Stage 2: Match Within Categories Using Product Pages

For each category, take the SE products and the competitor products in that same category, then:

1. **Attempt text-based matching first** for obvious matches (e.g., both say "7' x 3' cross brace").
2. **For ambiguous matches, fetch the product page** using the URL from the spreadsheet. Read the full product description, specifications, and any additional details. Match based on the actual specifications.
3. **For products where the competitor's page reveals specs** (like stem diameter, exact dimensions, material), use those specs to confirm or reject the match.

### Stage 3: Build Output

The final output spreadsheet should have:

**Columns:**
- `SKU` — Scaffold Express SKU
- `SE Description` — Scaffold Express product title
- `SE Price` — Scaffold Express price
- `SE URL` — Scaffold Express product URL
- Then for each competitor (ScaffoldMart, ScaffoldStore, USAScaffolding, ScaffoldSupply, BadgerLadder, Granite):
  - `[Competitor] Product` — Matched product title
  - `[Competitor] Price` — Cleaned numeric price
  - `[Competitor] URL` — Product URL
  - `[Competitor] Confidence` — high / medium / low
  - `[Competitor] Notes` — Any relevant notes (e.g., "price is per 4-pack, divide by 4", "different style but same dimensions")

**Rules:**
- Only include SE products that have at least one competitor match
- Leave cells blank where no match exists at that competitor
- Color code: white = high confidence, yellow = medium (needs review), no color = no match
- Include a summary/legend sheet

---

## Key Matching Rules

1. **Same product = same type + same dimensions + same specifications.** All three must match.
2. **Style matters.** S-Style ≠ W-Style ≠ BJ-Style ≠ V-Style. These have different tube diameters (S-style uses 1-7/16" OD, W-style uses 1-3/8" OD, BJ-style uses 1-1/4" OD). If a competitor doesn't specify style, check the product page for tube diameter.
3. **Galvanized ≠ painted/black.** These are different products with different prices.
4. **Multi-packs need annotation.** If a competitor sells a 4-pack and SE sells singles, match them but note it. The price comparison should ideally be per-unit.
5. **Don't force matches.** If there's no good match, leave it blank. A wrong match is worse than no match.
6. **Kits/sets vs individual components.** SE sells both individual items and kits (PSV-K prefix usually = kit). Match kits to kits, singles to singles.
7. **Cup Lock and Ring Lock** may match cross-brand if the physical dimensions are the same. Check product pages to verify.

---

## Competitor-Specific Notes

### ScaffoldMart
- Uses "Stack Pin" instead of "Coupling Pin"
- Scaffold frames listed with ScaffoldMart branding in the title
- Has good variety of individual components
- Stair towers use "Deluxe" vs "Standard" naming

### ScaffoldStore
- Prices stored as text strings ("Our Price: $49.80") — need regex extraction
- Uses "Tubular Cross Brace" instead of "Tube Cross Brace"
- "Multi-Function Scaffold" = "Multi-Purpose Scaffold" (Baker style)
- "Ring-Lock" products may correspond to SE's Cup Lock AND Ring Lock lines
- Sells many items as "Sets" (e.g., "Swivel Jack Set")

### USAScaffolding
- Best naming consistency — usually includes dimensions and style
- Uses "|" as separator in titles
- Frequently sells multi-packs (4-Pack, 12-Pack) — need to note this
- Uses unicode quotes and dashes (″ ′ – —)

### ScaffoldSupply
- More generic names — often omits dimensions from title
- Uses "RingLock" (one word) vs SE's "Ring Lock" (two words)
- Has "Cuplock" products that may cross-match
- Skip the "volusiontest" junk row

### BadgerLadder
- Primarily sells kits, towers, and packages — fewer individual components
- Many products are ladders/accessories unrelated to scaffolding
- Expect fewer matches — maybe 30-50 products
- Uses price ranges for some items

### Granite
- Branded product lines: "Amplify" (scaffold frames, braces), "BlackTop" (walkboards), "AXIS" (stages)
- Mostly sells bundles (4-pack, 8-pack)
- Fewest direct 1:1 matches expected — maybe 15-30 products
- Many products are proprietary designs with no direct equivalent

---

## Validation

The `TrackedProducts` tab contains ~130 previously manually matched products. After building the match dataset, compare against these to gauge accuracy. Report:
- How many of the 130 tracked products were matched correctly
- How many were matched incorrectly
- How many were missed

---

## Output Location

Save the final spreadsheet as `Competitor_Price_Comparison.xlsx`.

---

## Technical Notes

- The source Excel file is in the project directory
- Use `openpyxl` or similar for reading/writing Excel
- For web fetching, the product URLs in the spreadsheet are live — you can fetch them to read product descriptions and specs
- Price cleaning: extract numeric value from strings like "Our Price: $49.80", "$129.00 - $159.00" (take first price), etc.
- Handle unicode normalization: ″ → ", ′ → ', – → -, — → -
