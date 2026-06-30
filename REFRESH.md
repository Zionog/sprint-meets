# Refresh runbook — Dutch Meet Finder

How to re-run the data update for `index.html`. Everything is a **hand-curated static file**;
there is no live fetching. A refresh = re-pull data, edit `index.html`, commit.

Companion files: [`athletes-top30.md`](athletes-top30.md) (followed athletes + profiel IDs),
[`geo-cache.json`](geo-cache.json) (city → `[lat,lng]`, reuse to skip geocoding).

---

## Golden rules (learned the hard way)

1. **Use the user's Chrome (Claude-in-Chrome MCP), never `curl`/Bash for atletiek.nu.**
   The site is behind **Cloudflare** — `curl` gets a challenge page. Run all fetches with
   `javascript_tool` on a tab already on `https://www.atletiek.nu/...` (same-origin, and the
   user's session is logged in, which the calendar feed needs).
2. **`javascript_tool` results truncate (~1 KB).** Don't return big blobs. Either return
   compact pipe-delimited lines in **chunks of ~12 records**, or stash full data in a
   `window.__VAR` and export it in slices.
3. **The result filter blocks any output containing URLs / `?a=b&` query strings.** Parse in
   the page and return only clean fields (id, name, city, date) — never raw HTML or hrefs.
4. **Use top-level `await`** in `javascript_tool` (the tool awaits the last expression).
   An `async`-IIFE that returns a Promise comes back as `{}`. For sync DOM work use a plain
   `(function(){ ... })()` IIFE.
5. **Avoid `const`/`let` at top level** — calls share one global scope, so re-running a snippet
   throws "Identifier already declared". Wrap in an IIFE, or assign to `window.*`. Also `top`
   is reserved (it's `window.top`) — don't name a var `top`.
6. **Build edits with a Python script** keyed by meet id; then check `<div>` open/close balance
   and single `<style>`/`<script>` counts before committing.
7. **Push fails from here** (`could not read Username`). Commit anyway (user wants commits);
   the user's IDE sync pushes, or they run `git push origin main`.

---

## Endpoints & IDs (all on atletiek.nu, same-origin fetch with header `X-Requested-With: XMLHttpRequest`)

| Purpose | URL |
|---|---|
| Rankings (read with `get_page_text`) | `/ranglijst/nederlandse-ranglijst/2026/outdoor/senioren-mannen/{100m,200m}/` |
| Athlete search → profiel id | `/feeder.php?page=search&do=athletes&name=<surname>` → HTML w/ `/atleet/profiel/<id>/` links; disambiguate by club |
| Athlete upcoming meets | `/atleet/profiel/<profielId>/` → parse the wedstrijden table; keep rows with date > today, drop result rows (contain `m/s` / a time) |
| Meet entry list | `/wedstrijd/atleten/<meetId>/` → scan `<tr>`; cells = [#, Name, Club, **Events**, Startgroup, Category] |
| Meet info / entrant count | `/wedstrijd/main/<meetId>/` → regex `(\d+)\s*deelnemer` |
| Calendar feed | `/feeder.php?page=search&do=events&country={NL|BE}&event_soort[]=out&predefinedSearchTemplate=0&startDate=<epochSec>&endDate=<epochSec>&onderdelen[]=5&onderdelen[]=6&onderdelen[]=7` |

- **Event IDs:** 100m = **5**, 200m = **6**, 400m = **7**.
- Calendar `onderdelen[]` **ANDs** the events (asking for 5+6+7 ⇒ only meets with *all three* ⇒ ~empty).
  **Query each event id separately and UNION the results.**
- Calendar dates are **unix seconds**; use `predefinedSearchTemplate=0` for a custom range.
- Calendar `<tr>` parse: `.datumCol` = date `DD MON YYYY` (Dutch abbrev JAN FEB MRT APR MEI JUN JUL AUG SEP OKT NOV DEC),
  `.eventnaam` = name (often **duplicated** "X X" by responsive markup — collapse with `/^(.+?)\s+\1(\s|$)/`),
  city = the "Org, City" cell (take text after the last comma; strip trailing status/`deelnemer`).
- **Germany (`country=DE`) returns 0** — atletiek.nu doesn't host German meets. The "all nearby"
  list is effectively **NL + BE**. (Wetzlar-type meets live on their own sites.)
- Geocoding: `https://nominatim.openstreetmap.org/search?format=json&limit=1&city=<city>&country=<Netherlands|Belgium>`
  — works from the browser (CORS ok). Throttle ~700 ms; ≤ ~29 per `javascript_tool` call (45 s limit).
  **Check `geo-cache.json` first; only geocode cities not already in it, then add new ones back.**

---

## Steps

### 1. Rankings (5 min)
Open both ranglijst pages, `get_page_text`, read the **top 30** of each (legal-wind main table).
Diff against `athletes-top30.md`. If unchanged → skip step 2. If changed → note new/dropped names.

### 2. Profile IDs — only for *new* top-30 names
For each new name: `feeder.php ... do=athletes&name=<surname>`, pick the match by **full name + club**
(watch surnames that are common first names — search a distinctive token instead; e.g. "Aldrich" not "Jack").
Update the table in `athletes-top30.md`.

### 3. Per-meet registrations (the reliable signal)
For every curated meet id, fetch `/wedstrijd/atleten/<id>/` and scan rows for the top-30 names
(see harness below). Keep those entered in **100m/200m** (the app's focus). This drives the
"🏃 Following" names. Update each meet's `.athlete-names` in `index.html`.

### 4. Discover new curated meets (optional)
Sweep top-30 athletes' `/atleet/profiel/<id>/` upcoming meets; any meet not already listed is a
candidate to add as a curated row. NOTE: some athletes have **duplicate/near-empty profiles**
(e.g. Samir Cullimore) — profile sweep can under-report, so step 3 (entry lists) is authoritative.

### 5. "All nearby" list + map pins
Compute `startDate`/`endDate` (unix sec) for **today → +4 weeks**. For each country in {NL, BE}
and each event in {5,6,7}: fetch the calendar feed, parse `<tr>` → `{id,name,city,country,date,events}`,
union by id, drop already-curated ids and cancelled meets (`AFGELAST`). Geocode cities via
`geo-cache.json` (+ Nominatim for misses). Rebuild the `ALLMEETS` array in `index.html`.

### 6. Entrant counts (popularity sort)
For each curated meet: `/wedstrijd/main/<id>/` → `deelnemer` count → update `data-pop` on the row.

### 7. Build & commit
Apply edits with a Python script (key by meet id). Verify `<div>` balance + single `<style>`/`<script>`.
`git add -A && git commit`. Mention push will need the user's terminal / IDE sync.

---

## Reusable browser harness (paste into `javascript_tool`, then call per need)

```js
// Run once on a www.atletiek.nu tab to define helpers on window.
window.H = (function(){
  const X={headers:{'X-Requested-With':'XMLHttpRequest'}};
  const MM={JAN:'01',FEB:'02',MRT:'03',APR:'04',MEI:'05',JUN:'06',JUL:'07',AUG:'08',SEP:'09',OKT:'10',NOV:'11',DEC:'12'};
  async function doc(u){ const r=await fetch(u,X); return new DOMParser().parseFromString(await r.text(),'text/html'); }
  return {
    // scan a meet entry list for names in `list`; returns "Name [events]" lines
    scanMeet: async (id,list)=>{ const D=await doc('/wedstrijd/atleten/'+id+'/'); const o=[];
      D.querySelectorAll('tr').forEach(tr=>{ const c=[...tr.querySelectorAll('td')].map(x=>x.innerText.replace(/\s+/g,' ').trim());
        if(!c.length)return; const t=c.join(' | '); for(const n of list){ if(t.includes(n)){o.push(n+' ['+(c[3]||'')+']');break;} } }); return o; },
    // calendar sprint meets for a country over [startSec,endSec], union of 100/200/400
    calendar: async (country,startSec,endSec)=>{ const acc={};
      for(const ond of [5,6,7]){ const D=await doc('/feeder.php?page=search&do=events&country='+country+'&event_soort%5B%5D=out&predefinedSearchTemplate=0&startDate='+startSec+'&endDate='+endSec+'&onderdelen%5B%5D='+ond);
        D.querySelectorAll('tr').forEach(tr=>{ const dc=tr.querySelector('.datumCol'),en=tr.querySelector('.eventnaam'); if(!dc||!en)return;
          const dm=dc.textContent.replace(/\s+/g,' ').match(/(\d{1,2}) ([A-Z]{3}) (\d{4})/); if(!dm)return;
          const a=tr.querySelector('a[href*="/wedstrijd/"]'),im=a?a.getAttribute('href').match(/wedstrijd\/[a-z]+\/(\d+)/):null,id=im?im[1]:en.textContent.trim();
          let nm=en.textContent.replace(/\s+/g,' ').trim().replace(/^(.+?)\s+\1(\s|$)/,'$1');
          let city=''; tr.querySelectorAll('td,span,div').forEach(c=>{const tx=c.textContent.replace(/\s+/g,' ').trim(); if(!city&&/^[^,]+,\s*[A-Za-zÀ-ÿ.\- ]+$/.test(tx)&&!/deelnemer|Inschrijven|Gesloten/.test(tx))city=tx.split(',').pop().trim();});
          if(!acc[id])acc[id]={i:id,n:nm,ci:city,co:country,d:dm[3]+'-'+MM[dm[2]]+'-'+String(dm[1]).padStart(2,'0'),e:[]};
          const EV={5:'100',6:'200',7:'400'}; if(!acc[id].e.includes(EV[ond]))acc[id].e.push(EV[ond]); }); }
      return Object.values(acc); },
    geocode: async (city,cc)=>{ const C={NL:'Netherlands',BE:'Belgium'};
      const r=await fetch('https://nominatim.openstreetmap.org/search?format=json&limit=1&city='+encodeURIComponent(city)+'&country='+C[cc]); const j=await r.json();
      return j[0]?[+(+j[0].lat).toFixed(4),+(+j[0].lon).toFixed(4)]:null; }
  };
})(); 'H ready';
```

Then e.g. `await H.scanMeet('44647', NAMES)` / `await H.calendar('NL', s, e)` and **export results
in chunks of ~12** (pipe-joined) to stay under the truncation limit.

## Adapted, more-efficient strategy for next time
- **Skip step 2 unless rankings changed** (diff vs `athletes-top30.md`).
- **Geocode from `geo-cache.json` first** — last run, all 57 cities were already known, so a refresh
  needs ~0 new geocodes unless new cities appear.
- **Define `window.H` once**, then one short call per meet/country; export compactly. Avoids the
  trial-and-error that cost most of the time (Cloudflare, truncation, param names) — all now known.
- The slowest parts were *discovering* the endpoints and the data round-trip. Both are solved here,
  so a full refresh should be ~10–15 focused browser calls + one Python build.
