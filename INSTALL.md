# Install — troubleshooting

The happy path is in [README.md](README.md). This file covers the things that go wrong.

## Verify install

```bash
claude mcp list
```

Expected output — all three connected:

```
playwright:  npx -y @playwright/mcp@latest               - ✓ Connected
a11y:        npx -y a11y-mcp                             - ✓ Connected
lighthouse:  npx -y lighthouse-mcp                       - ✓ Connected
```

If any show `✗ Failed`, the most common causes:

| Symptom | Cause | Fix |
|---|---|---|
| `command not found: npx` | Node not on PATH | Install Node 20+ |
| `EACCES` on `~/.npm` | First `npx` run can't write its cache | `mkdir -p ~/.npm && chmod -R u+w ~/.npm` |
| `MCP server failed to start` after restart | Stale config from a previous version | `claude mcp remove <name> -s user` then re-add |

## Windows + read-only working directory

If your first Playwright tool call fails with:

```
EPERM: operation not permitted, mkdir '.playwright-mcp'
```

…it means your Claude Code working directory isn't writable for the user that the MCP server runs as. Playwright MCP tries to create a `.playwright-mcp/` cache dir in cwd; if that fails, every subsequent call fails too.

Fix — re-add Playwright with an explicit `--output-dir` pointing somewhere writable:

```powershell
claude mcp remove playwright -s user
claude mcp add playwright -s user -- npx -y "@playwright/mcp@latest" `
  --output-dir "C:\Users\<you>\.playwright-mcp-cache"
```

Then **restart Claude Code** so the MCP server picks up the new args. Verify with `claude mcp list`.

## Lighthouse MCP fails with EPERM on Windows

Symptom — Lighthouse runs but reports:

```
EPERM, Permission denied: '\\?\C:\Users\<you>\AppData\Local\Temp\lighthouse.NNNNNNN'
```

Cause — Windows Defender (or another AV) is holding a handle on the Chromium temp profile while Lighthouse tries to clean it up. The audit itself usually completed; only the post-run cleanup failed.

The skill knows to fall back to the `lighthouse` CLI directly via `npx`. If it doesn't (or you want to run it yourself):

```powershell
npx --yes lighthouse@12.8.2 "<url>" `
  --form-factor=mobile `
  --throttling-method=simulate `
  --output=json `
  --output-path=./audit/lighthouse-prod.json `
  --quiet `
  --chrome-flags="--headless=new --no-sandbox"
```

## First run is slow

The Playwright MCP downloads a Chromium build (~150 MB) on first invocation. Lighthouse and axe also pull dependencies. Budget 60–90 seconds for the very first tool call. After that, runs are fast.

## "I want to run the smoke test in CI"

The smoke test the skill generates uses `@playwright/test` (the test runner), not the MCP. To run it locally or in CI:

```bash
npm install -D @playwright/test
npx playwright install chromium
npx playwright test tests/smoke.spec.ts
```

In CI, set `BASE_URL` to your staging URL:

```bash
BASE_URL=https://staging.myapp.com npx playwright test tests/smoke.spec.ts
```

The test creates a unique `audit-{ISO-timestamp}@example.com` account each run so re-runs don't collide.

## Authenticated screens aren't showing up in Lighthouse / axe results

Expected — the a11y and Lighthouse MCPs hit URLs without a session cookie. They only see the public landing page. The skill flags this in the report so you know what wasn't audited.

To audit logged-in screens with axe + Lighthouse, you need framework-specific glue that pre-seeds an auth token. Roughly:

1. Authenticate via the API (Supabase, Auth0, Clerk, Firebase, …) and get a JWT.
2. Use Playwright's `context.addInitScript()` to write the token into `localStorage` or `IndexedDB` before the page loads.
3. Run axe / Lighthouse against that pre-authenticated context.

Out of scope for this skill — but if you wire it up, a PR with the recipe is welcome.

## Nuke and restart

If something is in a really weird state:

```bash
claude mcp remove playwright  -s user
claude mcp remove a11y        -s user
claude mcp remove lighthouse  -s user
claude mcp add playwright  -s user -- npx -y @playwright/mcp@latest
claude mcp add a11y        -s user -- npx -y a11y-mcp
claude mcp add lighthouse  -s user -- npx -y lighthouse-mcp
```

Then restart Claude Code.
