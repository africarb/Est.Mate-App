# Esti.Mate — Construction Estimating Platform

Single-file React application for South African construction estimating, built as a self-contained HTML file. No build step required — open `src/index.html` directly in a browser, or serve it locally.

## Quick startg d

```bash
# Option 1 — open directly in browser
open src/index.html      # macOS
xdg-open src/index.html  # Linux
start src\index.html     # Windows

# Option 2 — serve with any static server
npx serve src
# or
python3 -m http.server 8000 --directory src
```

Then visit `http://localhost:8000` (or whatever the server prints).

## Project structure

```
estimate-export/
├── README.md                     # This file
├── ARCHITECTURE.md               # How the app is structured internally
├── DEVELOPMENT.md                # Working with this codebase in Claude Code
├── src/
│   └── index.html                # The entire app — HTML + CSS + React (Babel-transpiled in-browser)
└── reference-boq/                # Source data spreadsheets used to build the BoQ items
    ├── Model_BoQ_Rev0.xlsx       # Latest model BoQ with H1/H1,5/H2 hierarchy
    ├── Model_BoQ_Rev02.xlsx      # Concrete trade reference
    ├── Model_BoQ_Rev03.xlsx      # Concrete trade with formwork H1s
    ├── AAQS_model_bills_of_quantities_in_Excel*.xlsx   # AAQS templates
    ├── Suppliers.xlsx            # Supplier pricing source data
    ├── Suppliers_Rev01.xlsx
    └── Building.xlsx
```

## What this app does

A QS (Quantity Surveyor) tool with six tabs:

1. **Home** — landing page with supplier directory, contractor search, featured suppliers
2. **Comparison** — side-by-side contractor rate comparison for any trade's BoQ items, with sticky columns/headers, H1/H1,5/H2 hierarchy, delivery location, and unit/quantity entry
3. **Rate Buildup** — first-principles rate construction per item (materials + labour gang + plant + overheads + profit), with live supplier pricing feeds
4. **Supplier Pricing** — manage supplier price lists with quantity bands and contractor-specific account discounts
5. **Supplier Compare** — compare material prices across suppliers
6. **Build a BoQ** — drag-and-drop BoQ builder with live rate calculation

## Technology

- **React 18** (via CDN, UMD build)
- **Babel Standalone** (in-browser JSX transpilation)
- **No build tools** — everything is in one `<script type="text/babel">` block
- All state managed via React hooks (`useState`, `useEffect`, `useRef`)
- All styling inline + a `<style>` block at the top of the file

## Data model summary

Five core data objects live as top-level `const` declarations in the script:

- `TRADES` — array of trade names (27 trades)
- `BOQ` — `{ trade: [{desc, unit}, ...] }` — Bill of Quantity items per trade
- `TRADE_SECTIONS` — `{ trade: [{before, type, desc}, ...] }` — H1/H1,5/H2/H3 headings for grouping items
- `SUPPLIERS` — array of supplier objects
- `CONTRACTORS` — array of contractor objects with trade specialty
- `MERKEL` — `{ "Trade||A": {materials, labour, plant, overheads, profit} }` — Merkel's rate buildup templates per item (keyed by trade + alpha index where A=item 1, B=2, …, AA=27, etc.)
- `SPD` — supplier pricing data (volume bands)
- `CDX` — contractor discount index

See `ARCHITECTURE.md` for details.
