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

1. **Chrome** logged in to X and Reddit. Google is optional and only used for the Reddit complement.
2. The **Playwright MCP Bridge** extension, from the Chrome Web Store.
3. The **Playwright MCP server in extension mode**, with the extension token so it connects without the approval dialog:

   ```bash
   claude mcp add playwright -s user -e PLAYWRIGHT_MCP_EXTENSION_TOKEN=<your-token> -- npx @playwright/mcp@latest --extension
   ```

   To get the token, open the extension's connect page (the first time a client connects, Chrome shows a
   dialog that says *"Set this environment variable to bypass the connection dialog"*) and copy it from there.
   Restart Claude Code, or reconnect the server, after adding it.

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
