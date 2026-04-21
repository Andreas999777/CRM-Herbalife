# Testing HBL-Connect

HBL-Connect is a single-file HTML/CSS/JS CRM (`index.html`, ~2800 lines) backed by Firebase Firestore. Follow-up lifecycle uses milestones Dag 1 / Dag 3 / Dag 7 / Week 2 / Week 3+ with Tanita body-composition measurements at each.

## Running locally

```bash
cd ~/repos/HBL-Connect && nohup python3 -m http.server 8080 > /tmp/httpd.log 2>&1 &
disown
curl -s -o /dev/null -w 'HTTP:%{http_code}\n' http://localhost:8080/index.html
```

Open http://localhost:8080/index.html in Chrome. Login is Firebase-backed; if the user already logged in once in this browser profile, the session is persisted and you don't need credentials again.

## Coach switching (for test isolation)

Each coach has its own Firestore sub-collection. Switch without manual click via the console:

```js
const s = document.getElementById('coach-selector');
s.value = 'Thalyssa';           // or 'Andreas'
s.dispatchEvent(new Event('change', { bubbles: true }));
```

Thalyssa has ~7 clients with intake data and active follow-up milestones — best for data-rich tests. Andreas typically has 1 client and is ideal for regression checks.

## Verifying Firestore writes

The page attaches an `onSnapshot` listener that keeps `window.clients[id]` in sync with Firestore. To check what was saved, read from that in-memory cache instead of fetching:

```js
// After typing in a Tanita input, wait ~1s for the snapshot to round-trip, then:
clients[currentClientId].tanita_d1    // {"gew": "72,4"}
```

**Note on async consoles:** the environment's JS console returns the last expression synchronously. For async checks, store the result on a `window._xxx` variable, wait with a `sleep`, then read it back:

```js
window._out = null;
navigator.serviceWorker.getRegistrations().then(r => {
  window._out = r[0] ? r[0].active.state + '|' + r[0].scope : 'none';
});
// later:
window._out    // 'activated|http://localhost:8080/'
```

## Known pitfall — snapshot listener resets UI

The `onSnapshot` callback calls `loadClientFiche()` on every Firestore write. Historical bug: this reset the active tab to Evaluatie AND overwrote every input value — including the one the user was currently typing in — so only the first character of a Tanita measurement actually persisted.

Fix (commit `b6a4f8f` on PR #5) relies on two guards:
1. `lastLoadedClientId !== currentClientId` — reset tab ONLY on actual client change.
2. `if (el && el !== document.activeElement)` — skip overwriting focused inputs.

When testing input fields on any fiche-tab, always verify:
- Tab stays active (`document.querySelector('.tab-btn.active').textContent`).
- Full value is saved (`clients[id].tanita_d1.gew` or similar — not truncated to first char).
- Chart / derived UI re-renders with the new value.

## WhatsApp template URLs

`openWaTemplate(id)` calls `window.open(url, '_blank')`. To capture the URL without actually opening WhatsApp:

```js
const orig = window.open;
let captured;
window.open = (u) => { captured = u; return null; };
openWaTemplate('d3');
window.open = orig;
decodeURIComponent(captured);
```

Phone normalization: leading `0` is replaced with `32` (BE). Numbers stored without the leading `0` will NOT get a country-code prefix — this is a data issue, not a code bug.

## PWA verification (service worker + cache)

```js
window._sw = null;
navigator.serviceWorker.getRegistrations().then(r => {
  const a = r[0]?.active;
  window._sw = (a?.state || 'none') + '|' + (a?.scriptURL || '');
});
caches.keys().then(k => window._cacheNames = k.join(','));
caches.open('hbl-connect-v1').then(c => c.keys())
  .then(ks => window._cacheUrls = ks.map(k => new URL(k.url).pathname).join('|'));
```

Expected after full registration:
- SW state: `activated`, script: `http://localhost:8080/sw.js`
- Cache `hbl-connect-v1` contains: `/`, `/index.html`, `/manifest.json`, `/icon-192.png`, `/icon-512.png`, `/icon-512-maskable.png`

If SW isn't registering on first load (page loaded before `load` event fired), manually trigger:

```js
navigator.serviceWorker.register('sw.js');
// wait ~3s for activation
```

## Dashboard field-name conventions

The dashboard reads `startDate`, `c.name`, `isActive` from the client record — NOT the legacy `startdate` / `c.fiche.name` / `active`. Regressions here caused Thalyssa's dashboard to appear empty in PR #4 — double-check any new dashboard logic against the actual field names on existing documents.

## Test coverage checklist

For any PR touching client-fiche or follow-up flows, verify:

- [ ] Sidebar badges update when counts change (0 → hidden, >0 → visible with `warn` or `danger` class).
- [ ] Typing in any `#<tab>-t-<field>` input keeps the active tab AND persists the full value.
- [ ] Progress chart adds a data point after Tanita save (not just delta card).
- [ ] WhatsApp templates substitute `{naam}` and `{why}` correctly.
- [ ] Regression: switch to a quieter coach (Andreas) and verify dashboard numbers are unchanged vs baseline.
- [ ] PWA: no regressions in SW registration or cache entries after adding/removing files from the app shell.

## Devin secrets needed

None. Firebase auth is handled through the browser login (user logs in once; session persists). No API keys or external services need to be provisioned by Devin.
