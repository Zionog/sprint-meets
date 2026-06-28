# Dutch Meet Finder 2026 — Project Overview

## What This Is

A private, password-gated dashboard that helps a senior Dutch sprinter (10.60 / 21.97, ~#16 NL 2026) decide which competitions to enter across the Netherlands and Belgium for the summer 2026 season. It is a single-file HTML app hosted on GitHub Pages.

**Live site:** https://zionog.github.io/sprint-meets
**Repository:** https://github.com/Zionog/sprint-meets
**Branch / file:** `main` → `index.html`
**Password:** `1944`

---

## Goal

Give the athlete a fast, mobile-friendly overview of every relevant upcoming meet — with enough detail (track quality, event availability, cost, registration deadline, spike recommendation) to make a registration decision in under 30 seconds per meet.

The dashboard is for personal use, not public. It is password-gated so it can be shared privately with coaches or training partners.

---

## Architecture

| Concern | Decision |
|---|---|
| Hosting | GitHub Pages (free, zero config) |
| Stack | Pure HTML/CSS/JS — zero dependencies, zero build step |
| File count | One file: `index.html` (~46 KB) |
| Auth | Client-side password gate (overlay div, z-index 9999) |
| Deployment | Edit `index.html` on GitHub, commit → live in ~60 s |

There is no backend, no database, no framework. Everything is hardcoded in the HTML file. This is intentional — it keeps the project simple and deployable by anyone with GitHub access.

---

## Password Gate

The gate renders a full-screen overlay on load. It is dismissed by entering the correct password (`1944`). Implementation details:

- `<div id="gate">` fixed, `z-index: 9999`
- `checkPw()` function reads `#pw-input`, compares to `'1944'`
- Enter key (`onkeydown`) and the button (`onclick`) both call `checkPw()`
- On success: `gate.style.display = 'none'`, `body.style.overflow = ''`
- `DOMContentLoaded` listener locks `body.style.overflow = 'hidden'` and focuses the input

**Known past bug (fixed):** A previous GitHub editor injection corrupted the JS, producing `functionfunction checkPw()` (doubled keyword). This was caused by residual content from the old file merging with the new injection. Always use a two-step clear-then-insert approach when updating via the CM6 API.

---

## Tab Structure

The dashboard has three tabs:

### Meets tab
The main view. Lists all upcoming competitions as expandable cards. Each card shows:
- Date (day number + month + day-of-week abbreviation)
- Meet name + country flag emoji
- City, time, event pills (100m / 200m / 400m — green if offered, struck-through gray if not)
- Tags: track tier, WACT level, prize money, registration urgency
- Cost + star rating + registration link
- Expandable detail: track info, spike recommendation, confirmed NL top-13 athletes

### Tracks tab
Reference guide rating every relevant track in NL and Belgium as Fast / Normal / Slow, with notes on surface type and records set there.

### Spikes tab
Guide to 5 sprint spikes (Nike Maxfly 2, Maxfly 1, Adidas SP3, SP2, Ambition) — when to use each based on track tier, with a quick decision rule at the bottom.

---

## Meets Covered (as of June 2026)

| Date | Meet | City | 100m | 200m | 400m | Closes |
|---|---|---|---|---|---|---|
| 4 Jul Sat | Meeting voor Mon (WACT Challenger) | Leuven 🇧🇪 | ✗ | ✓ | ✗ | 30 Jun |
| 9 Jul Thu | Rijnmond Track Series | Rotterdam 🇳🇱 | ✓ | ✗ | ✗ | 7 Jul |
| 10 Jul Fri | PUMA Fast Arms Fast Legs (WACT Challenger) | Wetzlar 🇩🇪 | ✓ | ✓ | ✗ | 3 Jul |
| 11 Jul Sat | Trackmeeting Breda | Breda 🇳🇱 | ✓ | ✗ | ✗ | 3 Jul |
| 11 Jul Sat | AtH Trackmeeting Eindhoven | Eindhoven 🇳🇱 | ✓ | ✓ | ✓ | 10 Jul |
| 11 Jul Sat | Guldensporenmeeting (WACT Bronze) | Kortrijk 🇧🇪 | ✓ | ✓ | ✓ | 7 Jul |
| 12 Jul Sun | Liliane Mandema Memorial ⭐ | Den Haag 🇳🇱 | ✓ | ✗ | ✗ | 4 Jul |
| 15 Jul Wed | Meeting International Province Liège | Liège 🇧🇪 | ✓ | ✗ | ✗ | 8 Jul |
| 18 Jul Sat | Nacht van de Atletiek (WACT Bronze) | Heusden-Zolder 🇧🇪 | ✓ | ✓ | ✓ | 9 Jul |
| 18 Jul Sat | Memorial Cor Aalberts | Eindhoven 🇳🇱 | ✓ | ✓ | ✗ | 16 Jul |
| 25 Jul Sat | ASICS NK Atletiek — Dutch Nationals ⭐ | Hengelo 🇳🇱 | ✓ | ✓ | ✓ | 12 Jul |
| 1 Aug Sat | IFAM Outdoor (WACT Bronze) ⭐ | Oordegem 🇧🇪 | ✓ | ✓ | ✓ | 26 Jul |

---

## CSS Conventions

| Class | Meaning |
|---|---|
| `.ev-pill.ev-yes` | Event offered — green pill |
| `.ev-pill.ev-no` | Event not offered — gray, struck-through |
| `.meet-row.highlight` | Top pick meet — amber left border, amber register button |
| `.tag-fast / .tag-normal / .tag-slow` | Track tier tags |
| `.tag-wact` | WACT-level tag (purple) |
| `.tag-prize` | Prize money tag (yellow) |
| `.tag-urgent` | Deadline closing soon (red) |
| `.date-day` | Day-of-week abbreviation below date (e.g. "Sat") |

---

## How to Update the File

1. Go to https://github.com/Zionog/sprint-meets/edit/main/index.html
2. Make edits directly in the GitHub web editor
3. Click **Commit changes** → confirm → live in ~60 seconds

Or clone the repo and push:
```bash
git clone https://github.com/Zionog/sprint-meets.git
# edit index.html
git add index.html && git commit -m "your message" && git push
```

**If using Claude in Chrome to inject content via the CM6 API**, always use a two-step approach:
```javascript
// Step 1: clear
view.dispatch({ changes: { from: 0, to: view.state.doc.length, insert: '' } });
// Step 2: insert
view.dispatch({ changes: { from: 0, to: 0, insert: newHtml } });
```
Do NOT do a single replace-all dispatch — emoji/surrogate-pair character counting can cause a position mismatch that leaves residual content, corrupting the JavaScript.

---

## Suggested Race Plan

**Rijnmond** Thu 9 Jul → **Mandema** Sun 12 Jul or **AtH Eindhoven** Sat 11 Jul → **Cor Aalberts** Sat 18 Jul → **NK Hengelo** Sat–Sun 25–26 Jul

Peak target: NK (Dutch Nationals), FBK Stadium Hengelo — the best track in the Netherlands.
