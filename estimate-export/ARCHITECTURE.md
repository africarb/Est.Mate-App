# Architecture

Esti.Mate is a **single-file React application**. Everything lives in `src/index.html`:

```
src/index.html
├── <style>           — Global CSS (about 70 lines)
├── <body>
│   └── <div id="root"></div>
└── <script type="text/babel">
    ├── Constants & colour tokens
    ├── Static data (TRADES, BOQ, TRADE_SECTIONS, SUPPLIERS, CONTRACTORS, MERKEL, SPD, CDX, PROMOS, FEATURED, LABOUR_ROLES)
    ├── Helper functions (toAlpha, Rf, Rp, refreshSupplierMaterials, getEffectivePrice, getTpl, computeRate, …)
    ├── Components (H2Dropdown, H15Dropdown, LinkModal, SupplierPricingTab, SupplierCompareTab, RateBuildupTab, BoQBuilderTab)
    ├── Main App component (state + tab routing + nav + render)
    └── ReactDOM.createRoot(...).render(<App/>)
```

The file is approximately 1.1 MB and 6300+ lines because all reference data — every BoQ item, every supplier price band, every rate buildup template — is embedded as JS literals.

## Key data shapes

### `BOQ`
```js
{
  "Masonry": [
    { desc: 'Mass brickwork', unit: 'm3' },
    { desc: 'Rectangular shaped piers', unit: 'm3' },
    ...
  ],
  "Concrete, formwork and reinforcement": [ ... ],   // 343 items
  ...
}
```

### `TRADE_SECTIONS`
```js
{
  "Masonry": [
    { before: 0,  type: 'H',   desc: 'BRICKWORK' },                  // H1 heading
    { before: 0,  type: 'H15', desc: 'BRICKWORK FOUNDATIONS' },      // H1.5 sub-heading
    { before: 0,  type: 'H2',  desc: 'Brickwork of NFP bricks ...' },// H2 item group
    { before: 56, type: 'H',   desc: 'BRICKWORK SUNDRIES' },
    ...
  ],
}
```
- `before` is the BoQ item index immediately *after* this heading
- `type` controls render style: `H` (bold uppercase banner), `H15` (italic teal dropdown if multiple), `H2` (italic teal dropdown if multiple), `H3` (hidden in Comparison tab)

### `MERKEL`
```js
{
  "Concrete, formwork and reinforcement||A": {
    materials: [
      { id: 1, desc: 'Cement', qty: 350, unit: 'kg', unitCost: 3.20, wastage: 5 },
      { id: 2, desc: 'Builders sand', qty: 1.0, unit: 'm3', unitCost: 280, wastage: 5,
        supplier: 'AfriSam', suppMatId: 'a1' }    // optional — links to SPD live pricing
    ],
    labour: {
      gang: [
        { id: 1, role: 'Artisan (Bricklayer/Plasterer/Tiler)', count: 1, daily: 640 },
        { id: 2, role: 'Unskilled Labourer', count: 2, daily: 340 }
      ],
      output: 3    // units of work per day per gang
    },
    plant: [
      { id: 1, desc: 'Concrete mixer', hours: 3, rate: 45 }
    ],
    overheads: { materials: 5, labour: 15, plant: 10 },  // %
    profit:    { materials: 10, labour: 10, plant: 10 }  // %
  },
  ...
}
```

The key format is `"<trade>||<alpha>"` where alpha = A for item 0, B for item 1, …, AA for item 26, BA for item 27, etc. Use `toAlpha(ii)` to convert.

Optionally keys can use the compound form `"<trade>|||<h1>|||<h2>|||<ii>"` for section-specific overrides.

### `SUPPLIERS`
```js
[
  { name: 'AfriSam', trade: 'Concrete, formwork and reinforcement',
    ownership: 'Private', value: 'Urban ready-mix dominance.',
    contractors: [...] },
  ...
]
```

### `CONTRACTORS`
```js
[
  { name: 'WBHO',  specialty: 'Masonry' },
  { name: 'JCVLV', specialty: 'Masonry' },
  ...
]
```

### `SPD` (Supplier Pricing Data)
Volume-band-based pricing for materials:
```js
{
  "AfriSam": {
    trade: 'Concrete, formwork and reinforcement',
    materials: [
      { id: 'a1', desc: 'Ready-mix concrete 25MPa', unit: 'm3',
        bands: [
          { upTo: 10,  price: 2400 },
          { upTo: 50,  price: 2300 },
          { upTo: Infinity, price: 2200 }
        ] }
    ]
  },
  ...
}
```

### `CDX` (Contractor Discount Index)
Account-specific % discount each contractor gets per supplier:
```js
{
  "WBHO":  { "PPC Ltd": 5, "Corobrik": 6, "Ocon Brick": 8, ... },
  "JCVLV": { ... }
}
```

## Core helpers

| Function | Purpose |
|---|---|
| `toAlpha(idx)` | Convert 0-based index to A, B, …, Z, AA, AB, … |
| `Rf(n)`, `Rp(n)` | Format a number as Rand currency (full / split rand + cents) |
| `getTpl(trade, ii, ctrName, compQty, h2)` | Get rate buildup template, with compound-key fallback |
| `refreshSupplierMaterials(buildup, ctr, trade, qty)` | Replace material unit costs with live supplier-band prices |
| `getEffectivePrice(supplier, matId, contractor, qty)` | Look up band price + apply contractor discount |
| `computeRate(buildup)` | Compute final rate from a buildup (materials + labour + plant + OH + profit) |
| `compLinkKey(trade, h1, h2, ii)` | Generate the compound key used in `rbLinks` |
| `getItemSection(trade, ii)` | Find which H1 section an item belongs to |

## Comparison tab — table layout

The Comparison tab uses a CSS Grid-like layout via colgroup with these column widths:

| Column | Width | Frozen | Notes |
|---|---|---|---|
| Item | 52 | ✓ left:0 | Letter A, B, … |
| Description | 420 | ✓ left:52 | H1/H1,5/H2 dropdowns render here |
| Link / +BoQ buttons | 110 | ✓ left:472 | |
| Unit | 60 | ✓ left:582 | |
| Qty | 88 | ✓ left:642 | Sticky shadow on right edge |
| Contractor Rate | 108 | scrolls | |
| Contractor Amount | 116 | scrolls | |
| (repeat per contractor) | | | Always 3 contractor columns (blank placeholders if fewer linked) |

Both `thead` rows use `position: sticky; top: 0` and the `tfoot` TOTAL row uses `position: sticky; bottom: 0`. The whole table wraps in a `<div>` with `overflow: auto` and `maxHeight: calc(100vh - thTop - 20px)` — measured dynamically.

## Supplier Compare tab

Identical layout to the Comparison tab — same column widths, same sticky behaviour, same TOTAL row — but rows are material **groups** (`COMPARE_GROUPS`) and columns are **suppliers** (not contractors).

## Recent changes you should know about

These are the most recent material modifications, from oldest → newest:

1. Rebuilt the Masonry trade's `TRADE_SECTIONS` from the Rev01 source (11 H1,5 sub-headings)
2. Added H15 type rendering with dropdown grouping (H15Dropdown component, mirrors H2Dropdown)
3. Rebuilt Concrete trade's `TRADE_SECTIONS` from Rev03 source (13 H1, 12 H1,5, 73 H2)
4. Made H1 headers visually bolder (fontWeight 800, fontSize 12, colour #111827)
5. Always show 3 contractor columns in the Comparison tab — pad with greyed-out blanks if fewer linked
6. Hide the "Auto-selected Contractors" panel when 0 contractors are linked
7. Sticky left columns (Item, Description, Link/BoQ, Unit, Qty) in both Comparison and Supplier Compare tabs
8. Dynamic table `maxHeight` based on measured `thTop`
9. Hide H3 rows in the Comparison tab (return null)
10. Generated 343 first-principles rate buildups for the entire Concrete trade — replaces the previous 6 entries
11. Broke the live supplier pricing link in the Comparison tab — it now uses static `unitCost` from buildups (Rate Buildup tab still gets live pricing)
12. Fixed `rbLinks` value-format bug (objects vs strings) for `.lastIndexOf` calls
13. Fixed `selKey` parsing in Rate Buildup tab to support the compound `"trade|||h1|||h2|||ii"` format
14. Added Unlink (✕) button next to "Linked" pill in Comparison tab

## Known quirks

- The file uses `<script type="text/babel">` so it's transpiled in-browser by `@babel/standalone`. Slow on first paint but no build step.
- Storage is in-memory React state only — refresh = lose all changes. There is no `localStorage` because artifacts in Claude.ai disallow it (but this constraint is irrelevant when running the file directly).
- Mixed colon delimiters in keys: `rbData` uses `::`, `rbLinks` compound key uses both `:::` and `|||` — confusing, but consistent within each system.
