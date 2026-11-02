# Publishing pwned.coffee to the Official MCP Registry (DNS auth)

Server name: **`coffee.pwned/pwned-coffee`**. File: [`server.json`](server.json) in this folder, valid against the `2025-12-11` schema (checked 2 Nov 2026 with `jsonschema`).

**Everything below runs on Alex's Mac. The private key never leaves it, never goes in this repo, and never goes on the box.**

Sources (read 2 Nov 2026):
- Registry repo: https://github.com/modelcontextprotocol/registry
- Authentication (GitHub vs DNS vs HTTP): https://github.com/modelcontextprotocol/registry/blob/main/docs/modelcontextprotocol-io/authentication.mdx
- Quickstart (install `mcp-publisher`, publish, verify): https://github.com/modelcontextprotocol/registry/blob/main/docs/modelcontextprotocol-io/quickstart.mdx
- Remote servers (`remotes`, `streamable-http`): https://github.com/modelcontextprotocol/registry/blob/main/docs/modelcontextprotocol-io/remote-servers.mdx
- Schema: https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json

## Why DNS, not GitHub or HTTP

- Domain auth (DNS or HTTP) gives the `coffee.pwned/*` namespace, the reverse-DNS form of pwned.coffee. That is what makes the name match the remote URL. The registry does not check the URL's domain itself (remote-servers doc, "Ownership Verification").
- GitHub auth would give `io.github.<alex>/*`. The org namespace `io.github.pwned-coffee/*` doesn't work with `mcp-publisher login github` today (issue #1468, named in the auth doc).
- DNS needs no Worker code: one TXT record in the Cloudflare dashboard. HTTP auth (`/.well-known/mcp-registry-auth`) is the alternative. It needs a Worker route, and the registry's fetch from Google Cloud must get past Cloudflare bot rules without a redirect. That is an optional track phase (see Track C, Phase C7).

## What changed from the draft in `marketing/listings/listings-plan.md`

| Field | Draft | Now | Why |
| --- | --- | --- | --- |
| `description` | "Live exploited vulnerabilities, updated hourly, with fix status, fixed versions and lockfile checks." | "Live security incidents, updated hourly: fix status, fixed versions, lockfile and repo checks." (94 chars) | Most tickets on the board aren't exploited (live board 2 Nov: 38 of 168 exploited, 120 `none_known`), so "exploited vulnerabilities" overclaims. Wording follows llms.txt ("Live security incidents"). Also names check_repo. Max 100. |
| `websiteUrl` | `https://pwned.coffee/` | `https://pwned.coffee/mcp` | The /mcp page is the setup docs. GUESS: either is valid; change back if preferred. |
| `icons` | none | favicon.svg and icon-192.png (both 200 on 2 Nov) | Optional in the schema; directories that read the registry can show it. |
| `repository` | none | none (on purpose) | The schema describes it as the server's *source code* repo. The docs repo isn't source code, so leaving it out avoids implying it's open source. |

Everything else ($schema 2025-12-11, name, title, version 1.0.0 = live `serverInfo.version`, one `streamable-http` remote) was already right.

## Steps for Alex

### 1. Install the tools (Mac)

```bash
brew install mcp-publisher openssl@3
mcp-publisher --help
```

macOS's built-in `openssl` is LibreSSL and fails with `Algorithm Ed25519 not found`. The docs say to use OpenSSL 3, so the commands below call it by path.

```bash
OSSL="$(brew --prefix openssl@3)/bin/openssl"   # /opt/homebrew/opt/openssl@3/bin/openssl on Apple Silicon
"$OSSL" version                                  # must say OpenSSL 3.x
```

### 2. Make the keypair (Mac only)

Keep it somewhere outside any git repo:

```bash
mkdir -p ~/.config/pwned-coffee-registry && chmod 700 ~/.config/pwned-coffee-registry
cd ~/.config/pwned-coffee-registry
"$OSSL" genpkey -algorithm Ed25519 -out key.pem
chmod 600 key.pem

PUBLIC_KEY="$("$OSSL" pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
echo "v=MCPv1; k=ed25519; p=${PUBLIC_KEY}"
```

The last line prints the TXT value. Only the public key goes in DNS.

### 3. Add the TXT record in the Cloudflare dashboard

Go to Cloudflare dashboard → pwned.coffee → DNS → Records → **Add record**:

| Field | Value |
| --- | --- |
| Type | `TXT` |
| Name | `@` (the apex, `pwned.coffee`) |
| Content | `v=MCPv1; k=ed25519; p=<PUBLIC_KEY from step 2>`, typed without surrounding quotes |
| TTL | Auto |

- It **must be on the apex**, not `_mcp-auth` or `_mcp-registry`. The auth doc says a selector record fails with a generic signature error.
- Other apex TXT records (SPF, verification records) can stay. The registry tries every MCPv1 record.
- If you rotate the key later, delete the old MCPv1 record, because a stale one is tried first.

Check that it has propagated:

```bash
dig TXT pwned.coffee +short | grep MCPv1
# "v=MCPv1; k=ed25519; p=...="
```

### 4. Log in with DNS auth

Run this from the folder that holds `server.json`, i.e. the docs repo checkout's `registry/` folder:

```bash
cd ~/code/pwned.coffee/registry          # wherever the docs repo is cloned
PRIVATE_KEY="$("$OSSL" pkey -in ~/.config/pwned-coffee-registry/key.pem -noout -text | grep -A3 "priv:" | tail -n +2 | tr -d ' :\n')"
mcp-publisher login dns --domain "pwned.coffee" --private-key "${PRIVATE_KEY}"
unset PRIVATE_KEY
```

- The output includes `Expected proof record:`. It must match the TXT content exactly.
- GUESS: `mcp-publisher` saves its registry token as a `.mcpregistry_*` file in the current folder. The docs repo's `.gitignore` already ignores `*.pem` and `.mcpregistry_*`. Run `git status` before any commit.

### 5. Validate, then publish

```bash
mcp-publisher validate      # checks server.json without publishing
mcp-publisher publish
# ✓ Successfully published
# ✓ Server coffee.pwned/pwned-coffee version 1.0.0
```

- The registry is in preview, and "breaking changes or data resets may occur".
- Each remote URL can belong to only one server name. A search on 2 Nov found nothing using `https://pwned.coffee/mcp`.
- To publish a change later, bump `version` (e.g. `1.0.1`), then `login` and `publish` again.

### 6. Verify the listing

```bash
curl -s "https://registry.modelcontextprotocol.io/v0.1/servers?search=coffee.pwned/pwned-coffee" | python3 -m json.tool | head -40
```

Expect `"name": "coffee.pwned/pwned-coffee"`, the `streamable-http` remote and version `1.0.0`. Today the search returns `{"servers":[],"metadata":{"count":0}}` (checked 2 Nov 2026).

Downstream: mcp.directory says it auto-discovers from the registry, and PulseMCP says it will pick up registry entries when it reopens (listings-plan rows 1, 13, 29). Check mcp.directory about a week after publishing.

## If DNS doesn't work: HTTP auth (needs a deploy)

Write `v=MCPv1; k=ed25519; p=<PUBLIC_KEY>` (plain text, same public key) to a file. Track C Phase C7 serves it at `https://pwned.coffee/.well-known/mcp-registry-auth` from a `[vars]` value, with status 200, `text/plain`, no redirect, and no bot challenge.

Then run:

```bash
mcp-publisher login http --domain "pwned.coffee" --private-key "${PRIVATE_KEY}"
```

Check what the registry will see:

```bash
curl -sS -H 'Accept: text/plain' -A 'mcp-registry/1.0' https://pwned.coffee/.well-known/mcp-registry-auth
```
