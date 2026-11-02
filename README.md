# pwned.coffee

**Live security incidents, updated hourly.** [pwned.coffee](https://pwned.coffee) shows what is affected, whether a fix exists, which versions fix it, and the sources behind each answer. It is built from CISA KEV, GitHub security advisories and security news, with every fix status tied to a source you can open. No account, no API key.

This repository is pwned.coffee's home on GitHub: what it is, how to use it, and where to report a wrong entry. **The site's source code is not open source.** The only public pwned.coffee repositories are this docs repo and [pwned-coffee/check-action](https://github.com/pwned-coffee/check-action), the GitHub Action (MIT).

## MCP server

A remote, read-only MCP server at **`https://pwned.coffee/mcp`** (Streamable HTTP, no authentication, rate limited per IP).

| Tool | What it does |
| --- | --- |
| `todays_specials` | Live incidents right now, optionally filtered by `stack` ids. |
| `lookup_cve` | The incident covering a CVE or GHSA `id`, or a named bug such as HTTP2Shell. |
| `check_lockfile` | Matches a lockfile's `content` (optional `ecosystem`) against incidents that state fixed versions. |
| `check_repo` | Takes the `files` that describe what a project runs (lockfiles, Dockerfiles, workflows and more) and says what needs fixing. |
| `whats_new` | Incidents created or updated `since` a point in time (`24h`, `7d` or a timestamp), optional `stack`. |
| `diff_since` | Every correction, new ticket and fix-status change `since` a point in time, each with its source. |

Prompts: `protect_my_repo` and `morning_check`.

**Claude Code**

```sh
claude mcp add --transport http pwned-coffee https://pwned.coffee/mcp
```

**Cursor** (`.cursor/mcp.json`), and other agents that take the same JSON (Windsurf, Gemini CLI, Zed and more):

```json
{
  "mcpServers": {
    "pwned-coffee": { "url": "https://pwned.coffee/mcp" }
  }
}
```

**Codex**

```sh
codex mcp add pwned-coffee --url https://pwned.coffee/mcp
```

Then ask your agent to run `check_repo` on your project, or use the `protect_my_repo` prompt.

[`registry/server.json`](registry/server.json) is its entry for the [Official MCP Registry](https://registry.modelcontextprotocol.io) (`coffee.pwned/pwned-coffee`).

## Check a lockfile from a terminal or CI

```sh
curl -sS -F f=@package-lock.json https://pwned.coffee/api/check
```

You get a text receipt by default. Send `Accept: application/json` for JSON. Add `?fail=1` to get a 422 when something is affected.

In GitHub Actions:

```yaml
- uses: pwned-coffee/check-action@v1   # pin it to a commit SHA
```

The Action sends the repo's lockfiles, Dockerfiles, .nvmrc and workflows to `/api/check` and fails the job when packages need fixing. See [check-action](https://github.com/pwned-coffee/check-action) for its inputs.

## Feeds and API

| Endpoint | Returns |
| --- | --- |
| `https://pwned.coffee/incidents.json` | Live incidents. `?status=past` for older ones, `?stack=<ids>` to filter, `?since=24h` (or an ISO timestamp) for only what changed. |
| `https://pwned.coffee/changes.json` | Every correction, new ticket and fix-status change since `?since=` (default 7 days, at most 90), each with its source. Pass `next_since` back as `since` next time. |
| `https://pwned.coffee/feed.xml` | RSS of the live board, hottest first, same filters. |
| `https://pwned.coffee/i/<id>` | One incident page. |
| `https://pwned.coffee/cve/<id>` | Redirects a CVE or GHSA id to the incident that covers it (404 when no ticket covers it). |
| `https://pwned.coffee/menu/<stack>` | Live incidents, history and status for one product. Also as `/menu/<stack>.json` and `/menu/<stack>/llms.txt`. |
| `https://pwned.coffee/today.json` | Today's board as one small JSON (CORS open). |
| `https://pwned.coffee/grade/<stack>` | A grade from A to F. Badge: `/badge/<stack>.svg` (`?grade=1` for the grade). |
| `https://pwned.coffee/breaches`, `/supply-chain` | Topic pages. |
| `https://pwned.coffee/health`, `/data-health.json` | Data health and the board's rules. |
| `https://pwned.coffee/corrections` | Every automatic correction, with its source. |
| `https://pwned.coffee/llms.txt` | A plain-text guide for LLMs and agents, with every field and stack id. |

Every JSON feed and `feed.xml` sends a weak ETag and answers 304 to a matching `If-None-Match`.

Add `?stack=nextjs,php` (any stack ids from [llms.txt](https://pwned.coffee/llms.txt)) to filter.

## Report a wrong entry

Found a wrong fix status, fixed version, product, CVE or stack tag, or an incident that's missing?

- **[Open a wrong-entry report](../../issues/new?template=wrong-entry.yml)** with the ticket URL or CVE id, what's wrong, and a source link.
- Or email **chef@pwned.coffee**.

A fix status on pwned.coffee only comes from a source we store and link, so a link to the vendor advisory, CVE record, GitHub advisory or release notes is what lets us correct it. Corrections show up in the public [correction log](https://pwned.coffee/corrections).

Issues are public. A security problem with pwned.coffee itself goes to **security@pwned.coffee** ([security.txt](https://pwned.coffee/.well-known/security.txt)), not to an issue.

## Privacy

Lockfiles and other files you paste or upload to `/protect`, `/api/check`, the MCP tools or the GitHub Action are read in memory to answer that one request. They aren't stored and their contents aren't written to logs. Details: [pwned.coffee/about#privacy](https://pwned.coffee/about#privacy).

## Not security advice

pwned.coffee is a summary of public sources. Confirm against the vendor advisory linked on each incident page.

## License

The text in this repository is licensed under [CC BY 4.0](LICENSE). This covers the docs only, not the pwned.coffee site, its code or its data.
