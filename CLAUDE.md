# TSWIM 2026 — Claude Code Notes

## Registration Date Changes

When early-bird or late-registration dates change, **update all these places**:

1. `assets/js/ev.js` (dev-only, gitignored) — `EB_END` and `LT_END` constants
2. `assets/js/ev.min.js` (committed, no `?dev=`) — regenerate from ev.js with `?dev=` stripped
3. `register-earlybird.html` — deadline banner text (search "Early bird deadline")
4. `register-late.html` — info banner text (search "Late registration")
5. `index.html` — fee table `<th>` headers (search `data-period="earlybird"` and `data-period="late"`)

## Registration Files

| File | Fee table column | Form value |
|---|---|---|
| `register-earlybird.html` | Early Bird | `earlybird` |
| `register-late.html` | Late Registration | `late` |
| `register-onsite.html` | On-site Registration | `onsite` |

## Date-Check JS

Two versions of the script exist:

- **`ev.js`** — dev source with `?dev=YYYY-MM-DD` param for date simulation. **Not committed** (gitignored under `assets/`). Keep locally for testing.
- **`ev.min.js`** — production minified version. **Force-committed** (`git add -f`). No `?dev=` bypass — uses real system date only.

The script handles:
- **Index page**: highlights the active fee column, sets Register button to correct form or disables it
- **Early-bird page**: redirects to `register-late.html` if early-bird period is over
- **Late page**: redirects to `index.html#registration` if late period is over

### Local testing with `?dev=`

Only works with the dev source `ev.js`. Swap the script tag temporarily:
```html
<!-- <script src="assets/js/ev.min.js?v=1.0.1"></script> -->
<script src="assets/js/ev.js"></script>
```

Then use:
```
index.html?dev=2026-06-01     → early bird period
index.html?dev=2026-06-08     → late period
index.html?dev=2026-06-22     → closed
```

## Server-Side Price Enforcement

The Apps Script `OnSubmit.gs` ignores the client-sent `registrationType` and determines it from the server clock via `isEarlyBird(config)`. Even if a user navigates directly to the early-bird form after the deadline, they will be charged the correct (late) price.

## Regenerating ev.min.js

1. Edit `ev.js` (the dev source with `?dev=`)
2. Create `ev.min.js` by minifying ev.js **with `?dev=` logic removed**
3. `git add -f assets/js/ev.min.js`
4. Bump `?v=` version in `index.html`, `register-earlybird.html`, `register-late.html`
