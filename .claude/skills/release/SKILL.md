---
name: release
description: Release the TSWIM 2026 website to production via GitHub Pages. Use this skill when the user wants to deploy, release, push to production, go live, ship changes, or publish the site. Also trigger when the user says "release" or asks to push changes to the live site. Handles cache busting, committing, and pushing to main.
---

# Release to GitHub Pages

This project deploys automatically when changes are pushed to `main`. This skill ensures cache busting is done correctly before pushing.

## Step 1: Check working tree status

```bash
git status
git log --oneline -3
```

If there are uncommitted changes, commit them (use `/ray-commit` conventions) — no need to ask first. Releasing is the user's go-ahead to commit and push.

## Step 2: Identify changed CSS/JS files

Compare the current `HEAD` against what's on the remote to find which CSS/JS files changed:

```bash
git diff origin/main..HEAD --name-only -- '*.css' '*.js'
```

Also check for uncommitted CSS/JS changes:

```bash
git diff --name-only -- '*.css' '*.js'
```

If no CSS/JS files changed at all, skip to Step 4.

## Step 3: Bump cache versions

The site uses query string versioning on CSS/JS links in `index.html` to bust browser caches. Each file has its own version.

Current files and their link patterns:

| File | Pattern | Referenced in |
|---|---|---|
| `assets/css/main.css` | `main.css?v=X.Y.Z` | `index.html` |
| `assets/css/schedule.css` | `schedule.css?v=X.Y.Z` | `index.html` |
| `assets/js/main.js` | `main.js?v=X.Y.Z` | `index.html` |
| `assets/js/ev.min.js` | `ev.min.js?v=X.Y.Z` | `index.html`, `register-earlybird.html`, `register-late.html` |

For each changed CSS/JS file, find its current version and increment the patch number (e.g., `1.0.1` → `1.0.2`). Bump the version in **every** file that references it (e.g., `ev.min.js` lives in three files — keep them in sync). To find all references: `grep -rn "ev.min.js" *.html`.

Only bump versions for files that actually changed — don't bump everything.

After bumping, if the version bump itself is the only uncommitted change, amend the previous commit rather than creating a new one. If there are other uncommitted changes mixed in, create a separate commit: `chore: bump cache versions for release`.

## Step 4: Push to main

Push automatically — do **not** ask for confirmation. Invoking this skill is the user's explicit go-ahead to deploy.

```bash
git push origin main
```

This triggers GitHub Pages deployment automatically.

## Step 5: Verify deployment

Check the GitHub Pages deployment status:

```bash
gh api repos/tswim2026/tswim2026.github.io/pages/builds --jq '.[0] | {status, created_at}'
```

If `gh` is not available or the check fails, just inform the user that changes have been pushed and deployment should happen within a few minutes.

## Important notes

- Never push without checking for cache busting first — stale CSS/JS is the most common issue after deploys
- The `index.html` content itself doesn't need cache busting (HTML is not cached as aggressively by browsers)
- If only `index.html` changed (no CSS/JS), no version bumps are needed
- Push automatically without asking for confirmation — invoking the release skill IS the go-ahead. (This deploys to the live conference website, so still report clearly what was pushed.)
