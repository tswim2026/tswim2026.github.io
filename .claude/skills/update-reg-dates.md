---
description: Update registration dates (early-bird/late deadlines) across all files that reference them
globs: ["*.html", "assets/js/ev.js"]
---

# Update Registration Dates

When the user asks to change early-bird or late-registration dates, update **all** of these locations:

## 1. `assets/js/ev.js` — date constants (lines 3-4)
```
var EB_END = '2026-06-07T23:59:59+08:00';
var LT_END = '2026-06-21T23:59:59+08:00';
```
Then regenerate `assets/js/ev.min.js` from the updated source and bump `?v=` in all files that load it.

## 2. `index.html` — fee table headers
Search for `data-period="earlybird"` and `data-period="late"` in `<th>` tags — update the "by June X" text.

## 3. `register-earlybird.html` — deadline banner
Search for "Early bird deadline" — update both the English and Chinese date text.

## 4. `register-late.html` — info banner
Search for "Late registration" — update both the English and Chinese date text.

## Checklist
- [ ] `assets/js/ev.js` EB_END
- [ ] `assets/js/ev.js` LT_END
- [ ] `assets/js/ev.min.js` regenerated
- [ ] `?v=` version bumped in index.html, register-earlybird.html, register-late.html
- [ ] `index.html` fee table `<th>` text
- [ ] `register-earlybird.html` deadline banner
- [ ] `register-late.html` info banner
