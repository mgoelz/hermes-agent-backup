# Worked Example: German Insolvency Announcements Portal (JSF)

Session-proven recipe for `neu.insolvenzbekanntmachungen.de/ap/` — generalizes
to other JSF/Mojarra government portals (handelsregister-, insolvenz- ports on
the same justiz.de infrastructure).

## Portal structure

- **Stack**: JSF/Mojarra, session in URL (`;jsessionid=...`), ViewState hidden field.
- **Search page**: `/ap/suche.jsf`. Form `frm_suche` with selects bound to code
  lists: `frm_suche:lsom_bundesland:codelist:scl_bundesland:mysom` (NO_CODE or
  BW/BY/BE/...), `frm_suche:lsom_gegenstand:codelist:mysom` (EROEFF = openings,
  plus ABWMASS, SICHMASS, ENT_VERF, ...), date inputs
  `frm_suche:ldi_datumVon:datumHtml5` / `...datumBis...` (ISO `YYYY-MM-DD`).
- **Results**: `/ap/ergebnis.jsf`, table `#tbl_ergebnis`; per-row detail button
  is an `input[type=image]` inside a per-row form `tbl_ergebnis:<N>:frm_detail`.
- **Detail text**: opens a `window.open` POPUP pointing at `/ap/text.xhtml`;
  content lives in `<pre id="veroefftext">`. The button's Mojarra AJAX call
  (`mojarra.ab(... 'msgs frm_text:ihd_text' ...)`) only updates a hidden field;
  replicating it via `requests` returns just a new ViewState (~200 bytes) no
  matter the parameter permutation. Do NOT try — go straight to the browser.
- **Limits**: >1000 hits per query → "Ihre Suche ergab zu viele Treffer";
  also reduced search granularity after 2 weeks (InsBekV). Query ONE DAY at a
  time; for private insolvencies a day can exceed 1000 → add Bundesland.

## Working script skeleton (Playwright, node)

```js
const { chromium } = require('playwright');
const browser = await chromium.launch({ headless: true,
  args: ['--no-sandbox', '--disable-dev-shm-usage'] });
const ctx = await browser.newContext({ locale: 'de-DE', userAgent: '...' });
const page = await ctx.newPage();

// popup interception FIRST
const popupTexts = [];
ctx.on('page', async pop => { /* wait #veroefftext, push innerText, close */ });

await page.goto(BASE + 'suche.jsf', { waitUntil: 'domcontentloaded' });
await page.evaluate(({ v, b }) => {
  const setVal = (el, val) => { /* native setter + input+change events */ };
  setVal(document.querySelector('input[name="frm_suche:ldi_datumVon:datumHtml5"]'), v);
  setVal(document.querySelector('input[name="frm_suche:ldi_datumBis:datumHtml5"]'), b);
  setVal(document.querySelector('select[name="frm_suche:lsom_gegenstand:codelist:mysom"]'), 'EROEFF');
}, { v: von, b: bis });
await Promise.all([page.waitForLoadState('domcontentloaded'),
                   page.click('input[name="frm_suche:cbt_suchen"]')]);
await page.waitForSelector('#tbl_ergebnis');

// rows in one evaluate; buttons in DOM order map 1:1 to row indices
const rows = await page.evaluate(() => { /* map trs -> {datum, az, gericht, schuldner, sitz} */ });
const buttons = await page.$$('form[id$="frm_detail"] input[type="image"]');
const rowIdx = await Promise.all(buttons.map(b => b.getAttribute('name')))
    .then(names => names.map(n => +n.match(/tbl_ergebnis:(\d+):/)[1]));

// click per row; poll until popupTexts.length grows (40 x 250ms budget)
```

Full working version lives on the machine at
`/home/agentuser/insolvenzradar/scrape_pw.js` (scraper) and
`/home/agentuser/insolvenzradar/dossiers.py` (classification + digest pipeline).

## Data volumes (measured, Aug 2026)

- Per day (all Germany, Gegenstand=EROEFF): 345-525 announcements total,
  24-45 with a legal form in the name (companies) — ~36 companies/day,
  ~13k/year (market context: ~24k company insolvencies incl. late publication).
- Detail-text retrieval cost: roughly 3-8 s per row; cap per-run details
  (e.g. 10-12) to keep a daily cron run under 5 min.

## Value-add pipeline on top

1. **Company filter**: legal-form keywords (GmbH, UG, AG, KG, OHG, e.K., GbR, SE, Ltd).
2. **Classification**: ordered regex signatures on name+text (Immobilien first,
   then Gastronomie, Handel, Handwerk/Bau, Logistik, IT, Pflege; else Sonstige).
3. **Contact extraction** (layered regex, fallback strategies — formulations
   differ per court):
   - Verwalter: try "Zum Insolvenzverwalter wird ernannt/bestellt" → next line(s);
     else "Insolvenzverwalter(in) ist:" → until blank line; else single line.
     Truncate before Straße/Str./Weg/Platz patterns and at "Tel:".
   - Tel `Tel(?:efon)?\.?\s*:`, E-Mail, HRB/HRA number, opening date.
4. **Digest**: Markdown/Telegram with Immobilien pinned first; store raw JSON +
   derived dossiers JSON in `data/` (dedupe glob patterns!).

## Operation notes

- Run from `/home/agentuser/insolvenzradar` (node_modules with playwright).
- Host is aarch64: Playwright chromium works; Puppeteer's Chrome-for-Testing
  binary does not (x86-64 ELF).
- Respect the portal: one search session per day-window, sleep between runs,
  friendly UA, only public data (§ 9 InsO publications).
