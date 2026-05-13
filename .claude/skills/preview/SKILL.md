---
name: preview
description: Start a live server and open the website preview in Chrome. Use when you want to preview the local site or test changes in the browser.
---

# Website Preview

Start a local development server and open the site in Chrome for previewing.

## Instructions

1. Start live-server in the background on port 8080:
   ```bash
   npx live-server --port=8080 --no-browser
   ```
   Run this in background mode so it doesn't block.

2. Use Chrome MCP tools to open the preview:
   - Call `tabs_context_mcp` with `createIfEmpty: true` to get or create a tab
   - Navigate to `http://localhost:8080`
   - Take a screenshot to confirm the preview loaded

## Notes

- Live-server auto-reloads when files change
- If server is already running on port 8080, just navigate Chrome to the URL
- The server runs in background - use `/tasks` to see running processes
