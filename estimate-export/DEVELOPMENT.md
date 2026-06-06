# Development guide

This file is written for Claude Code, but is useful for any developer too.

## Working with `src/index.html`

The entire app is in **one file**. Edits are made directly to `src/index.html`. There is no compile step — refresh the browser to see changes.

## Finding things

The file is 6300+ lines. These anchor strings will land you in the right place fast:

| Looking for… | Search for |
|---|---|
| Top of script | `<script type="text/babel">` |
| Trade name list | `const TRADES=` |
| Bill of Quantities data | `const BOQ=` |
| Section hierarchy (H1/H1,5/H2/H3) | `const TRADE_SECTIONS=` |
| Suppliers | `const SUPPLIERS=` |
| Contractors | `const CONTRACTORS=` |
| Rate buildup templates | `const MERKEL=` |
| Supplier pricing bands | `const SPD=` |
| Contractor account discounts | `const CDX=` |
| Rate computation logic | `function computeRate(` |
| Template lookup with fallback | `function getTpl(` |
| Live supplier price lookup | `function getEffectivePrice(` |
| Comparison tab rendering | `tab==="comparison"` |
| Supplier Compare tab | `function SupplierCompareTab(` |
| Rate Buildup tab | `function RateBuildupTab(` |
| BoQ Builder tab | `function BoQBuilderTab(` |
| Sticky table column CSS | `.comp-table .sc{` |
| Main App function | `function App()` |

## Common edits

### Add a new BoQ item to a trade

Locate the trade in `const BOQ`, add an entry. Item indices auto-increment — the alpha code (A, B, …) is derived via `toAlpha(ii)` so you just add to the array.

```js
"Masonry": [
  { desc: 'Mass brickwork', unit: 'm3' },
  // ... existing items ...
  { desc: 'YOUR NEW ITEM', unit: 'm3' },    // ← new
]
```

If the new item should appear *between* existing items, also update the `before:` indices in `TRADE_SECTIONS["<that trade>"]` — every section heading that came after the insertion point needs `+1`.

### Add a rate buildup template

For item with index `ii=27` in trade `"Foo"`, the key is `"Foo||AB"`:

```js
"Foo||AB": {
  materials: [{ id: 1, desc: 'Thing', qty: 1, unit: 'no', unitCost: 100, wastage: 5 }],
  labour: { gang: [{ id: 1, role: 'Artisan ...', count: 1, daily: 640 }], output: 5 },
  plant: [],
  overheads: { materials: 5, labour: 15, plant: 10 },
  profit:    { materials: 10, labour: 10, plant: 10 }
}
```

If `materials[i]` has `supplier` and `suppMatId` set, the Rate Buildup tab will fetch its `unitCost` live from `SPD`/`CDX` (the Comparison tab will not — that link was deliberately broken).

### Add a new H1,5 sub-heading to a trade

Locate the trade in `TRADE_SECTIONS`, add:

```js
{ before: <item_index>, type: 'H15', desc: 'YOUR SUB-HEADING' }
```

If there are multiple H1,5 entries at the same `before:` index, they auto-group into a dropdown (`H15Dropdown` component).

### Change a colour

Top of the script:
```js
const T   = '#10B981';   // primary teal
const TL  = '#ECFDF5';   // teal light bg
const BD  = '#E5E7EB';   // border default
const GTX = '#16A34A';   // green text (cheapest)
const GBG = '#DCFCE7';   // green bg
```

## Testing changes

```bash
# Serve the file
npx serve src              # → http://localhost:3000
# or
python3 -m http.server 8000 --directory src
```

Then open in browser and check:
1. Page loads without console errors
2. Each tab renders
3. Click into Comparison → pick a trade → quantities update totals
4. Click into Rate Buildup → pick a trade → pick an item → buildup loads

## Syntax check before sharing

Because Babel transpiles in-browser, syntax errors only show up at runtime. Quick way to catch them earlier:

```bash
# Extract the script block and check it parses
node -e "
const fs = require('fs');
const html = fs.readFileSync('src/index.html','utf8');
const s = html.indexOf('<script type=\"text/babel\">') + 27;
const e = html.lastIndexOf('</script>');
const code = html.slice(s, e);
try { new Function(code); console.log('SYNTAX OK'); }
catch(e) { console.log('SYNTAX ERROR:', e.message); }
"
```

Note: this only catches plain-JS syntax errors. JSX-specific errors won't be caught — Babel handles those at runtime.

## Pitfalls

1. **Don't add `localStorage`** — the original file was deployed in a Claude.ai artifact context where browser storage is blocked. Use React state.
2. **MERKEL key format matters** — always use `toAlpha(ii)` to construct keys, never manual letters. For item 27 it's `AA`, not `Z+1`.
3. **`TRADE_SECTIONS` `before:` indices must be accurate** — they refer to BoQ item indices, not row positions. Inserting items shifts everything.
4. **Two delimiters in rbLinks keys** — `::` for simple, `:::` and `|||` for compound. Use the helper `compLinkKey()`.
5. **Sticky columns need matching `left:` offsets** — if you change a column width, update the `left:` of every column to its right in both `<thead>` and `<tbody>`.
6. **Always 3 contractor columns** — even when fewer are linked, blank placeholders fill the gap. Don't change this logic without thinking about layout consistency.
7. **H3 rows return `null`** — they're hidden by design in the Comparison tab. The data is still in `TRADE_SECTIONS`.

## Reference data

The `reference-boq/` folder contains the source spreadsheets used to build the BoQ items and section hierarchy. If you need to refresh the data:

```bash
# Inspect a sheet
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('reference-boq/Model_BoQ_Rev03.xlsx', data_only=True)
for sheet_name in wb.sheetnames:
    sheet = wb[sheet_name]
    print(f'\n=== {sheet_name} ===')
    for row in sheet.iter_rows(values_only=True, max_row=20):
        print(row)
"
```

The convention used in those files:
- Column A: hierarchy marker — `B` (Bill name), `H1` (heading 1), `H1,5` (sub-heading), `H2` (item description group), `H3` (preamble note), or a digit (item number)
- Column B: description
- Column C: unit

That's what the parsing scripts assume.
