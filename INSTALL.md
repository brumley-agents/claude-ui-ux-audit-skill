# UI/UX Audit Skill — Install Guide

A skill for [Claude Code](https://docs.claude.com/en/docs/claude-code) that runs a three-stage UI/UX audit on any running web app: Playwright walkthrough at three viewports, axe-core a11y scan, and Lighthouse mobile audit. Produces a markdown report, screenshots, and a Playwright smoke test ready to drop in CI.

## What it does

Hand it a URL (`audit ui/ux http://localhost:3000`), it gives you back:

- `./audit/report.md` — UX findings ranked P0/P1/P2/P3, axe violations table, Lighthouse scores
- `./audit/screenshots/` — PNGs of every flow step at 1440 / 768 / 375 px
- `./tests/smoke.spec.ts` — Playwright test re-running the primary flow

If the URL is a dev server (Vite/Expo/Next), it auto-detects and re-runs Lighthouse against a production build so the perf numbers aren't fiction.

## Install

### 1. Install the skill

Drop this folder into your user-scope skills directory:

| OS | Path |
|---|---|
| macOS / Linux | `~/.claude/skills/ui-ux-audit/` |
| Windows | `%USERPROFILE%\.claude\skills\ui-ux-audit\` |

The folder needs to contain `SKILL.md` (and this file is fine to leave in too). After Claude Code restart, the skill is available in every project.

### 2. Install the three MCP servers

The skill orchestrates three MCPs. Add all three at user scope so they work across projects:

```bash
claude mcp add playwright  -s user -- npx -y @playwright/mcp@latest
claude mcp add a11y        -s user -- npx -y a11y-mcp
claude mcp add lighthouse  -s user -- npx -y lighthouse-mcp
```

Verify:

```bash
claude mcp list
```

Should show all three as `✓ Connected`. Restart Claude Code if any aren't picked up yet.

### 3. (Windows only) Playwright cache directory fix

If your Claude Code working directory is read-only, the first Playwright tool call fails with:

```
EPERM: operation not permitted, mkdir '.playwright-mcp'
```

Re-add Playwright with a writable `--output-dir`:

```powershell
claude mcp remove playwright -s user
claude mcp add playwright -s user -- npx -y "@playwright/mcp@latest" --output-dir "C:\path\to\writable\dir\.playwright-mcp"
```

Then restart Claude Code so the MCP server picks up the new args.

### 4. (Optional) Install Playwright in your project for CI

The smoke test the skill generates uses `@playwright/test` (not the MCP). To actually run it:

```bash
npm install -D @playwright/test
npx playwright install chromium
npx playwright test tests/smoke.spec.ts
```

## Use it

```
audit ui/ux http://localhost:3000
```

Or:

```
ux audit https://staging.myapp.com — flow is sign up → create project → invite teammate
```

If you don't specify a flow, the skill auto-discovers one from the landing page's primary CTA.

## Known limitations

- **The a11y and Lighthouse MCPs don't carry session cookies.** If your app gates its real screens behind auth, those MCPs only see the public landing page. The Playwright walkthrough still covers authenticated screens — you just won't get axe/Lighthouse numbers for them. Pre-seeding a session is doable but needs framework-specific glue (e.g. write a Supabase JWT into IndexedDB before the MCPs boot); not in scope for this skill.

- **Lighthouse MCP can EPERM on Windows with Defender** when cleaning up its Chromium temp profile. The audit usually completed; the skill knows to fall back to the `lighthouse` CLI directly via `npx`.

- **Dev-server Lighthouse numbers are fiction.** Hot-reload, no minification, on-demand transforms. The skill detects dev-server URLs and re-runs against a production build (`expo export`, `vite build`, etc.) — but if your build process is unusual, it'll ask before guessing.

## Sharing this skill

Just zip the `ui-ux-audit/` folder and send it. Or push to a Git repo and have your friend clone it into `~/.claude/skills/`. The skill is self-contained — no external dependencies beyond the three MCPs above.
