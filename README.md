# social-research

A [Claude Code](https://claude.com/claude-code) skill that researches **X (Twitter)** and **Reddit**
by reading **your own logged-in Chrome** through the
[Playwright MCP](https://github.com/microsoft/playwright-mcp) browser extension. It returns a
concise, sourced synthesis: TL;DR, techniques, gotchas, resources, people/communities, coverage.

- **Read-only by design:** no likes, votes, posts, replies, follows or logins. Browser JS is limited to reading the DOM.
- **Small scale, human pace:** ~6 results pages and ~10 threads per platform per request, one tab, waits between pages.
- **Stops on friction:** CAPTCHAs, rate limits and login walls are reported, never bypassed.
- **Tested extractors (Nov 2026):** X search, old.reddit.com feeds, search, subreddit search and threads, plus a Google `site:reddit.com` complement. Claude's built-in WebSearch can't see Reddit.
- **One channel among others:** it's meant to run alongside normal web research, not replace it.

## Requirements

- **Chrome** (or another Chromium browser), logged in to X and Reddit. Google is optional and only used for the Reddit complement.
- **Node.js 18+**, because the server runs with `npx`.
- **Claude Code**.
- The **Playwright MCP** server in *extension mode*, set up as described below.

## Setting up Playwright MCP (extension mode)

In extension mode, the Playwright MCP server doesn't launch a separate, empty browser. It drives
**your real Chrome** through an extension, so it uses your existing logins and cookies.
It can only see the tabs it opens in its own tab group, not your other tabs.

### 1. Install the Chrome extension

Install **[Playwright Extension](https://chromewebstore.google.com/detail/playwright-extension/mmlmfjhmonkocbjadbfplnigmagldckm)**
from the Chrome Web Store.

> Install it in the **Chrome profile that is logged in to X and Reddit**. The skill sees exactly
> the logins of that profile and no others.

### 2. Add the MCP server to Claude Code

```bash
claude mcp add playwright -s user -- npx @playwright/mcp@latest --extension
```

`-s user` makes it available in every project. Alternatively, add it by hand to `~/.claude.json`
under `mcpServers`, or to a project's `.mcp.json`:

```json
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--extension"],
      "env": {}
    }
  }
}
```

### 3. Connect once and copy the token

Start Claude Code and ask it to open any page with Playwright, for example: *"use playwright to open example.com"*.
Chrome then opens the extension's connect page:

- *"claude-code" is trying to connect to the Playwright Extension*, with a warning and a list of tabs.
- A box titled *"Set this environment variable to bypass the connection dialog"*, containing
  `PLAYWRIGHT_MCP_EXTENSION_TOKEN=…`, with **regenerate** and **copy** buttons.

Copy that line. You can click **Allow & select** on a harmless tab, such as the extension's own
"Welcome" tab, to finish this first connection.

### 4. Save the token so the dialog is skipped

Without the token, every new session waits for you to approve the dialog, and an unattended tool
call hangs until it times out. Re-add the server with the token:

```bash
claude mcp remove playwright -s user
```

```bash
claude mcp add playwright -s user -e PLAYWRIGHT_MCP_EXTENSION_TOKEN=<your-token> -- npx @playwright/mcp@latest --extension
```

You can also put it in the `env` object of the JSON config above. **Restart Claude Code** (or
reconnect the server from `/mcp`). A server that is already running doesn't pick up new environment variables.

### 5. Verify

```bash
claude mcp list
```

`playwright` should show as connected. Then ask Claude: *"with playwright, open old.reddit.com and
tell me which user is logged in"*. It should answer right away, without a dialog in Chrome.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| A tool call hangs, or Chrome shows the connect dialog | Token missing or wrong, or the server was started before you added it. Check the `env`, then restart or reconnect the server. |
| Pages open but you're not logged in | The extension is installed in a different Chrome profile. Install it in the logged-in profile. |
| Calls fail with a connection error | Chrome is closed or the extension is disabled. Open Chrome, enable it, and retry. |
| A `.playwright-mcp/` folder appears in your project | Playwright writes page snapshots there on every navigation. Safe to delete; add it to `.gitignore`. |
| You think the token leaked | Click **regenerate** on the connect page, update the `env`, restart Chrome and the server. |

> **Security:** with the token set, any MCP client configured with it can drive your logged-in
> browser without asking. Keep it out of shared configs and repos. This skill is read-only by
> design, but other prompts using the same server may not be.

## Install

### As a plugin (recommended)

In Claude Code:

```
/plugin marketplace add niccolofanton/social-research
/plugin install social-research@social-research
```

### Manually

```bash
git clone https://github.com/niccolofanton/social-research.git
```

```bash
mkdir -p ~/.claude/skills && cp -r social-research/skills/social-research ~/.claude/skills/
```

## Usage

Ask naturally, or call it explicitly:

```
/social-research what are people saying about <project> on X and Reddit?
search Reddit for workarounds to <bug>
check r/<subreddit> for the latest compatibility reports
```

To make Claude use it automatically for research, add something like this to your `~/.claude/CLAUDE.md`:

```markdown
When I ask for research, always do normal web research first, and also use the `social-research`
skill when community experience, opinions, bugs or workarounds matter. Cite every source.
```

## Notes

- Results reflect your **personalized, logged-in** feeds and rankings.
- Each navigation makes Playwright MCP write a snapshot to `.playwright-mcp/` in the working directory. You can delete it.
- If an extractor stops working because a site changed its markup, the skill tells you which one at the end of the run instead of patching itself.
- Personal, small-scale research only. Respect each platform's terms of service.

## License

MIT
