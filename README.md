# ui-ux-audit — Claude Code skill

A three-stage UI/UX audit pipeline for any running web app: **Playwright walkthrough** at three viewports, **axe-core** WCAG 2.1 AA scan, and **Lighthouse** mobile audit — orchestrated by [Claude Code](https://docs.claude.com/en/docs/claude-code) and producing a single markdown report you can hand to your team.

```
audit ui/ux http://localhost:3000
```

…and 5–10 minutes later you have:

```
./audit/
  ├── report.md            # P0/P1/P2/P3 UX findings, axe table, Lighthouse scores + top 3 fixes
  ├── screenshots/         # full-page PNGs at 1440 / 768 / 375, named in flow order
  └── lighthouse-prod.json # raw Lighthouse output for diffing future runs
./tests/
  └── smoke.spec.ts        # Playwright test re-running the flow, ready to drop in CI
```

---

## When to use it

| Scenario | What you'd get |
|---|---|
| **You're about to ship a release.** | Punch list of regressions a teammate would otherwise find in QA — broken layouts at desktop, dead buttons, console errors, missing focus states. |
| **You inherited a codebase.** | A guided tour with screenshots — the report tells you what the primary flow even *is*, plus where the rough edges live. |
| **You're prepping for accessibility review.** | axe-core findings grouped by severity with rule IDs and *specific* fixes (not "improve contrast" — actual hex codes). |
| **A perf complaint just landed.** | Real Lighthouse numbers from a production build (the skill auto-detects dev servers and rebuilds), with the top 3 highest-payoff fixes called out. |
| **You want CI smoke coverage but never wrote the test.** | A working `tests/smoke.spec.ts` mirroring the actual user flow, with role-based selectors and unique account-per-run logic so re-runs don't collide. |

It is **not** a replacement for: design review, security review, code review, manual cross-browser testing, or talking to a real user. It's an audit *floor* — the things a sharp pair of eyes would catch in 20 minutes, in 5.

---

## Quick start

### 1. Install the skill

```bash
# Mac / Linux
git clone https://github.com/brumley-agents/claude-ui-ux-audit-skill \
  ~/.claude/skills/ui-ux-audit
```

```powershell
# Windows
git clone https://github.com/brumley-agents/claude-ui-ux-audit-skill `
  "$env:USERPROFILE\.claude\skills\ui-ux-audit"
```

### 2. Add the three MCP servers

```bash
claude mcp add playwright  -s user -- npx -y @playwright/mcp@latest
claude mcp add a11y        -s user -- npx -y a11y-mcp
claude mcp add lighthouse  -s user -- npx -y lighthouse-mcp
```

Verify all three show `✓ Connected`:

```bash
claude mcp list
```

### 3. Restart Claude Code

MCP servers are loaded at startup. After restart, the skill is available in every project.

> ⚠️ **First run is slow.** The Playwright MCP downloads a Chromium build (~150 MB) the first time. Lighthouse and axe also pull dependencies on first invocation. After that, runs are fast.

> ⚠️ **Windows + read-only cwd:** if your first Playwright call fails with `EPERM, mkdir '.playwright-mcp'`, see [INSTALL.md](INSTALL.md#windows--read-only-working-directory) for the `--output-dir` fix.

---

## Using it

### Minimum invocation

```
audit ui/ux http://localhost:3000
```

The skill will:

1. Confirm the URL is reachable (curl probe, fail fast if your dev server isn't running).
2. Open the landing page in Playwright at 1440×900 and read the accessibility tree.
3. **Auto-discover** the primary flow from whatever the landing page suggests is the main CTA — sign up, browse, checkout, whatever. No template to fill.
4. Walk it, screenshot every state, then repeat at 768×1024 and 375×812.
5. Run axe-core against every URL visited.
6. Run Lighthouse mobile. If the URL is a dev server (Vite/Expo/Next/CRA), build the production output and re-run so the perf numbers aren't fiction.
7. Write the report and the smoke test.

### Steering the audit

You don't have to take the auto-discovered flow. Anything below works:

```
ux audit https://staging.myapp.com — flow is sign in → create project → invite teammate
```

```
audit ui/ux http://localhost:5173 — focus on the checkout flow,
skip Lighthouse, my dev server is the only thing running here
```

```
a11y audit my app at localhost:8080
```

```
lighthouse audit https://example.com
```

The skill triggers on any of: **audit ui/ux**, **ui audit**, **ux audit**, **ux review**, **a11y audit**, **accessibility audit**, **lighthouse audit**, plus any explicit request to audit a localhost or staging URL.

---

## What the report looks like

```markdown
# {AppName} — UI/UX Audit

**Audited:** 2026-04-29 · **URL:** http://localhost:3000 · **Build:** Vite prod export
**Viewports captured:** 1440×900, 768×1024, 375×812

## 1. UX findings (ranked by impact)

### P0 — Drawer navigation is broken on web at every viewport ≥ 768px
- **What I saw:** … (with screenshot reference)
- **Why it matters:** …
- **Fix:** Configure the drawer with `drawerType: 'permanent'` for widths ≥ 768. …

### P1 — …
### P2 — …

## 2. Accessibility violations (axe-core, WCAG 2.1 A/AA + best-practice)

| # | Rule ID | Severity | Element | Failing detail | Suggested fix |
|---|---|---|---|---|---|
| 1 | autocomplete-valid | Serious | input[autocomplete="password"] | Not a valid token. | Change to `autocomplete="current-password"`. |
| 2 | color-contrast | Serious | .text-white in Sign in button | 3.45:1 (needs 4.5:1) | Darken `#2a9d5f` to `~#218754`. |
…

## 3. Lighthouse — mobile, throttled

| Category | Dev | Prod | Δ |
|---|---|---|---|
| Performance | 43 🔴 | 60 🟡 | +17 |
| Accessibility | 90 🟡 | 90 🟡 | 0 |
…

### Top 3 fixes
1. Cut the JavaScript bundle (`expo-router` lazy routes; drop `@expo/vector-icons` wholesale; …)
2. Defer / subset the Inter fonts
3. Fix the four color-contrast failures + missing meta description
```

Real example output from the audit that produced this skill: see the original report for the LineupCaptain audit.

---

## How findings are ranked

| | Definition | Example |
|---|---|---|
| **P0** | Ship-blocker. Broken feature, dead control, critical accessibility. | Hamburger menu logs `OPEN_DRAWER not handled by any navigator` and silently does nothing. |
| **P1** | Serious UX issue affecting most users. | Form fields render at 1408 px wide on a 1440 px desktop viewport. |
| **P2** | Noticeable but non-blocking. | Roster summary says `1 MEN` instead of `1 MAN` (pluralization bug). |
| **P3** | Polish / nice-to-have. | Active tab affordance is a 5%-darker grey background — works, but invisible at a glance. |

Findings are paired with **specific** fixes. The report says "darken `#2a9d5f` to `~#218754` to clear 4.5:1 against white" — not "improve contrast." If a finding can't get specific, it's not in the report.

---

## What it does NOT do

- **It can't audit auth-gated screens with axe-core or Lighthouse.** Those MCPs hit the URL without a session cookie, so they only see the public landing. The Playwright walkthrough still covers logged-in screens (the skill creates a dummy email/password account on the fly), but for axe/Lighthouse on `/dashboard` you'd need to pre-seed an auth token. Out of scope for this skill — but the report explicitly flags what wasn't audited so you know.
- **It won't fix anything.** The output is a report, not a PR. (Drop the report into a fresh Claude Code session and ask it to apply the P0 fixes — that part works well.)
- **It doesn't hit non-web Expo / RN apps.** This audits the web build only.
- **It doesn't replace cross-browser testing.** The Playwright MCP uses Chromium. If you have Safari-specific bugs, this won't catch them.

---

## Configuration

The skill reads everything from the prompt — no config file. If you find yourself wanting one, the skill is ~300 lines of markdown at [`SKILL.md`](SKILL.md) and is meant to be edited.

If you want to bias the audit toward a specific area, just say so:

> `audit ui/ux http://localhost:3000 — I really care about the checkout flow, less about the marketing pages`

---

## Sharing / contributing

The whole skill is two markdown files. Fork, edit, and either point your friend at your fork or send a PR back here. Particularly welcome:

- **More framework dev-server detection** — the skill currently knows Vite / Expo / Next / CRA. If you use SvelteKit, Astro, Remix, etc., adding the build/serve recipe is a 4-line change in §3 of `SKILL.md`.
- **Auth pre-seeding recipes** — Supabase, Clerk, Auth0, Firebase. Each is its own glue function but follows the same pattern (write a token into IndexedDB or `localStorage` before the audit MCPs boot).
- **More viewport presets** — currently 1440 / 768 / 375. If you target 4K or foldables, adding a preset is one line.

---

## License

MIT. Use it however.

---

## Acknowledgments

Built on three excellent MCP servers:
- [@playwright/mcp](https://github.com/microsoft/playwright-mcp) (Microsoft)
- [a11y-mcp](https://github.com/priyankark/a11y-mcp) (Priyanka)
- [lighthouse-mcp](https://github.com/priyankark/lighthouse-mcp) (Priyanka)
