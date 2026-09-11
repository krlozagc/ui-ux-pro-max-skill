# Security Notes

This file summarizes the outbound network surface of this repository: which
scripts talk to the network, which services they talk to, and how the
MCP servers declared for the `stack/` template are installed. It complements
[`SECURITY.md`](./SECURITY.md) (vulnerability reporting process) with a
concrete inventory, kept up to date as scripts change.

## Scripts that make outbound network calls

All of the calls below are opt-in: nothing here runs unless the user invokes
the script with the relevant flag and has set the corresponding API key.

| Script | Provider(s) | Trigger | Required key |
| --- | --- | --- | --- |
| `.claude/skills/design/scripts/logo/generate.py` | Gemini (default) | any run | `GEMINI_API_KEY` |
| | Atlas Cloud | `--provider atlas` | `ATLASCLOUD_API_KEY` |
| | MuAPI | `--provider muapi` | `MUAPI_API_KEY` |
| `.claude/skills/design/scripts/icon/generate.py` | Gemini only | any run | `GEMINI_API_KEY` |
| `.claude/skills/design/scripts/cip/generate.py` | Gemini only | any run | `GEMINI_API_KEY` or `GOOGLE_API_KEY` |

- `logo/generate.py` prints an explicit console notice before the first
  Atlas Cloud / MuAPI request, naming the third-party provider and noting it
  requires its own API key, in addition to the module docstring already
  documenting this. `icon/generate.py` and `cip/generate.py` only support
  Gemini, so they have no atlas/muapi path to disclose.
- All three load API keys via the same `load_env()` helper, which reads
  `.env` files in priority order (skill-local `.env`, then
  `~/.claude/skills/.env`, then `~/.claude/.env`) and never overwrites a key
  already present in the environment. No script hardcodes or bundles a key.
- `logo/generate.py`'s HTTP layer validates that any URL it follows
  (redirects and media downloads) is `https://` and resolves to a public,
  non-local address before fetching it (`_validate_public_https_url`),
  guarding against SSRF via a malicious provider response.

`.claude/skills/design-system/scripts/fetch-background.py` does **not**
make an outbound HTTP call itself: `get_background_image()` returns a small
hardcoded list of pre-selected `images.pexels.com` URLs (plus a
`pexels.com/search/...` URL for manual lookup) that get embedded into
generated CSS/HTML. The actual image fetch happens later, client-side, when
that HTML is rendered or screenshotted — not from Python.

`src/ui-ux-pro-max/scripts/search.py` and `core.py` (the BM25/regex search
engine) and every `data/*.csv` file make no outbound network calls.

## Third-party MCP servers (`stack/.mcp.json`)

The `stack/` template's `.claude/settings.json` sets
`enableAllProjectMcpServers: true`, so opening Claude Code in a project built
from this stack loads three third-party MCP servers declared in
`stack/.mcp.json`, each launched via `npx -y <package>@<version>`:

- `@playwright/mcp` — browser automation or the visual-feedback loop
- `chrome-devtools-mcp` — Chrome DevTools Protocol access
- `shadcn` (`mcp` subcommand) — shadcn/ui component search/add

Versions are pinned to an exact release instead of `@latest`, so a future
package release (malicious or simply broken) is not installed and run
without a deliberate version bump and review. `npx` still resolves and
downloads the pinned version from the npm registry the first time it runs
for a given local cache.

## Local Claude Code permissions (`stack/.claude/settings.json`)

The `stack/` template ships a project `permissions.allow` list scoped to the
tools the design-audit workflow actually needs (the design-audit script,
`npm run audit*`, Playwright browser install, and the three MCP servers
above). Review this file before adopting the stack in a project you don't
fully trust, and prefer the narrowest `Read`/`Bash` scopes that still let the
documented workflow (`docs/SETUP.md`, `docs/WORKFLOW.md`) run.

**Known follow-up (not yet applied):** `stack/.claude/settings.json` still
grants `Read(//home/**)` -- absolute-path read access to the entire `/home`
tree, not just the project the stack is installed into. The narrower,
equivalent-behavior fix is:

```diff
- "Bash(npx playwright:*)",
- "Read(//home/**)",
+ "Bash(npx playwright install:*)",
+ "Read(/**)",
```

- `Read(/**)` (single leading slash) anchors at the settings file's own
  directory per Claude Code's permission-path rules, i.e. the project root
  the stack is installed into -- not the whole filesystem.
- `Bash(npx playwright:*)` is narrowed to `Bash(npx playwright install:*)`
  since the only documented use (`scripts/setup.sh`,
  `.github/workflows/design-review.yml`, `docs/SETUP.md`) is
  `npx playwright install [--with-deps] chromium`; the broader wildcard also
  allowed unrelated Playwright subcommands (`test`, `codegen`, etc.).
- `Bash(npm run audit:*)` was left as-is: both `audit` and `audit:file` are
  the project's own npm scripts (see `stack/package.json`), and the
  documented usage passes variable `--url`/`--file` flags after `--`
  (`docs/SETUP.md`), which a non-wildcard rule can't match.

This edit was not applied in this pass because the harness's own guardrails
block edits to any `.claude/settings.json` file as self-modification,
regardless of which repository or tool performs the edit. Apply the diff
above by hand, or re-run this task with that restriction lifted.

## Out of scope for this pass

This note covers the scripts and configuration reviewed during the security
hardening pass that added this file. It does not re-audit skills or scripts
not listed above; see `SECURITY.md` to report anything found elsewhere.
