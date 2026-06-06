# Claude Code instructions

This is a context file for Claude Code working on this project.

## Project at a glance

**Esti.Mate** is a single-file React app for South African construction estimating. Everything lives in `src/index.html` — ~1.1 MB, ~6300 lines, with React+Babel via CDN (no build step).

## Before you edit

Read these in order:
1. `README.md` — what the app does, how to run it
2. `ARCHITECTURE.md` — internal structure, data shapes, recent changes
3. `DEVELOPMENT.md` — common edits, pitfalls, anchor strings for finding things

## Edit workflow

1. Edits are made directly to `src/index.html` — no build step needed
2. After editing, run the syntax check from `DEVELOPMENT.md` to catch obvious errors:
   ```bash
   node -e "const fs=require('fs');const html=fs.readFileSync('src/index.html','utf8');const s=html.indexOf('<script type=\"text/babel\">')+27;const e=html.lastIndexOf('</script>');try{new Function(html.slice(s,e));console.log('OK')}catch(err){console.log('ERR:',err.message)}"
   ```
3. Open `src/index.html` in a browser to verify the change works
4. Check the browser console for runtime errors (JSX errors only surface here)

## Style preferences

- Match the existing terse JS style — inline ternaries, comma-separated `const` declarations, minimal whitespace
- No new build tools, no new dependencies, no module imports
- Keep changes inside `src/index.html` — don't split it across files
- Inline styles are the norm; only add CSS rules in the `<style>` block when needed for `:hover`, `position: sticky`, pseudo-elements, etc.

## Things to be careful with

- **`MERKEL` keys use `toAlpha(ii)`** — item 27 is `AA`, not `Z`
- **`TRADE_SECTIONS.before` indices must match BoQ item positions** — inserting/removing items shifts every following index
- **Sticky table columns** — if you change a column width, update every column's `left:` offset to its right (in `<colgroup>`, `<thead>`, `<tbody>`, and `<tfoot>`)
- **Two delimiters in `rbLinks` keys** — use the `compLinkKey()` helper, don't hand-build them
- **The Comparison tab does NOT use live supplier pricing** — this was a deliberate break. Don't reintroduce `refreshSupplierMaterials()` there
- **H3 rows are hidden in the Comparison tab** — they return `null` by design
- **3 contractor columns always render** — blank placeholders fill gaps when fewer are linked

## Useful Python utilities

If you need to inspect or regenerate data from `reference-boq/`:

```python
# Parse a BoQ Excel sheet to extract H1/H1,5/H2/items
import openpyxl, re
wb = openpyxl.load_workbook('reference-boq/Model_BoQ_Rev03.xlsx', data_only=True)
sheet = wb['Concrete, formwork + Reinfmnt']
for row in sheet.iter_rows(values_only=True):
    marker = (row[0] or '').strip() if row[0] else ''
    desc   = (row[1] or '').strip() if row[1] else ''
    unit   = (row[2] or '').strip() if row[2] else ''
    if marker in ('H1','H1,5','H2','H3') and desc:
        print(f'{marker:5} {desc}')
    elif re.match(r'^\d+$', marker):
        print(f'  ITEM ({unit}) {desc}')
```

```python
# Generate MERKEL key for item index
def toAlpha(idx):
    n, r = idx+1, ""
    while n > 0:
        n -= 1
        r = chr(65 + (n % 26)) + r
        n //= 26
    return r
# toAlpha(0)='A', toAlpha(25)='Z', toAlpha(26)='AA', toAlpha(342)='ME'
```

## Don't do

- Don't add a `package.json`, `webpack.config.js`, or any build configuration
- Don't add `localStorage`/`sessionStorage` — it's blocked in the original artifact context
- Don't split `src/index.html` into multiple files
- Don't `npm install` anything — there's no `node_modules` and nothing needs one
- Don't use TypeScript — the file is plain JSX
