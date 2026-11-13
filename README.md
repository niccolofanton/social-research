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
| Chrome jumps to the front / switches desktop | Normal once per connection unless you use background mode. See below. |
| The connection drops mid-run | The extension disconnects when its last controlled tab closes, or when a page embeds a frame from another extension (Chrome detaches the debugger). Extensions that inject into X/Reddit or also use the debugger API (e.g. other browser-automation extensions) can interfere: try a profile with fewer extensions. |

### Running in the background (keep Chrome on its own desktop)

Verified in the extension source (v0.5.0) and `@playwright/mcp` 0.0.85, Nov 2026.

**Why Chrome comes to the front.** It happens **once per connection**: on the first tool call of a
session, and again after any reconnect. Two things cause it. The server opens the connect page by
launching Chrome with a URL, and the extension then calls `chrome.tabs.update({active:true})` and
`chrome.windows.update({focused:true})`. `browser_tabs select` also raises the window, because it sends
`Page.bringToFront`. Navigating, reading pages and closing tabs don't steal focus.

**Background mode.** Add `--background` to the args, or set `PLAYWRIGHT_MCP_BACKGROUND=1`. It needs the
token and extension ≥ 0.5.0. In background mode:

- the connect page opens in the background;
- the extension doesn't focus the window;
- `bringToFront` becomes a no-op;
- new tabs are created **inactive**.

```bash
claude mcp add playwright -s user -e PLAYWRIGHT_MCP_EXTENSION_TOKEN=<your-token> -e PLAYWRIGHT_MCP_BACKGROUND=1 -- npx @playwright/mcp@latest --extension
```

Caveats:

- **Version:** the upstream docs say Chrome **158+** is needed for a fully background connect page. Older builds may still raise a window in edge cases.
- **No open window:** if the extension's Chrome profile has no open window, Chrome opens a new one, and new windows get activated.
- **App windows:** if the last active window was an app/PWA window, the connect page opens in a new window.

**The trade-off: hidden tabs don't render.** Chrome stops rendering a tab when it's inactive or its
window is covered or on another desktop. On macOS that includes a window on another Space.
`document.visibilityState` becomes `hidden`, and `requestAnimationFrame` and `IntersectionObserver` stop.

- **What still works:** navigating, reading the DOM and every extractor. Reddit (old.reddit.com) works fully.
- **What doesn't:** X's infinite scroll can't load more posts. The skill handles this by paging X
  searches with `until:` dates instead of scrolling.
- **Background mode makes every tab hidden.** Rendering never runs in that mode.
- **No fix from the running browser:** Playwright can't force a page visible over CDP. The only fixes are
  Chrome launch flags: `--disable-backgrounding-occluded-windows` (covered windows only, not
  inactive tabs), `--disable-renderer-backgrounding`, and `--disable-background-timer-throttling`.

| Setup | Chrome stays put | X scrolling | Effort |
|---|---|---|---|
| **Background mode** (recommended) | Yes (rare edge cases before Chrome 158) | No, the skill pages by date | One env var |
| Default (no background) | No, jumps to front once per connection | Only while the window is visible | None |
| Separate Chrome instance (own `--user-data-dir`, logged in, extension installed, launched with the flags above), left on its own desktop | Jumps to that desktop once per connection | Yes | Medium |
| Headless Playwright profile (`--headless --browser chrome --user-data-dir <dir>`, log in once headed) | Fully invisible | Yes | Medium. Not your real Chrome session, possible bot detection, closes after 1 h idle |

Avoid `--remote-debugging-port` on your everyday Chrome profile: Chrome 136+ blocks it on the default
data directory.

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
