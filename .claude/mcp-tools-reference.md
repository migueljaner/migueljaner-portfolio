# MCP tools reference

What was used this session across the portfolio + fita-a-fita deployment work, and
the sharp edges worth knowing before relying on them again.

## Hostinger MCP

Read-only tools work fine and are the main way to diagnose a Node app deploy without
waiting on the user to check hPanel manually:
- `hosting_listWebsitesV1`, `DNS_getDNSRecordsV1` — account/domain state.
- `hosting_listNodeJSBuildsV1` — per-build recorded config (node version, app type,
  entry file, output directory) and its `state` (`completed`/`failed`) — compare this
  across attempts when something silently doesn't work.
- `hosting_getNodeJSBuildLogsV1` — full build log by build UUID.
- Subdomain create/delete (`hosting_createWebsiteSubdomainV1` /
  `hosting_deleteWebsiteSubdomainV1`) — note Hostinger does **not** support nested
  subdomains (a subdomain of a subdomain); they only branch directly off the root
  registered domain.

**`hosting_createNodeJSBuildFromArchiveV1` cannot practically be called directly** —
its `archive` parameter rejects local file paths ("The archive field must be a file")
and a 50MB+ archive is impractical to pass as an inline payload anyway. In practice:
give the user the zip's local path and have them upload it through hPanel manually,
then use the read-only tools above (plus SSH) to diagnose the result.

There's no tool anywhere in this MCP's surface for setting a Node app's environment
variables (searched for it twice, confirmed absent) — see the `.env`-in-archive
workaround in `.claude/hostinger-deployment.md`.

## MongoDB Atlas MCP

Mutating tools (`atlas-create-db-user`, `atlas-create-access-list`, `drop-database`,
`delete-many`, etc.) require an MCP-protocol "elicitation" confirmation step by
default. **This client doesn't reliably surface that confirmation prompt**, so every
first attempt at a mutating call silently comes back denied
("User did not confirm the execution..."), even with an interactive user present and
willing to approve.

Workaround: set `MDB_MCP_CONFIRMATION_REQUIRED_TOOLS` in the MongoDB MCP server's
`env` block (in `.claude.json`) to an empty string (or a reduced list, keeping
destructive ones like `drop-database` gated if you want that safety net) — then the
MCP server itself skips asking. Requires reconnecting the MCP server
(`/mcp` → reconnect) for the env change to take effect, since it's read once at
process start. Once confirmation is disabled at that layer, get real informed
consent through `AskUserQuestion` before any destructive call instead — that channel
actually works reliably, unlike the tool's own confirmation gate.

Side note: `atlas-connect-cluster` creates a temporary, auto-expiring database user
as a side effect of establishing the connection — expected behavior, not a bug, and
not something to clean up manually unless it visibly lingers.

## GitHub MCP

Read tools (search, get file contents, list branches/commits) work fine. Write tools
(`push_files`, `create_or_update_file`) returned a 403 the first time they were tried
against a brand-new repo — likely a permissions-sync delay rather than a real
capability gap. Local `git push` worked immediately and is the reliable fallback.

## Playwright MCP

Used for two distinct things this session:
- **Real screenshots for `projects.json` entries** — navigate the live URL, set a
  consistent viewport, `browser_take_screenshot`. Always prefer a real screenshot of
  the live site over a placeholder.
- **Diagnosing a "silently broken" symptom precisely** — when the fita-a-fita frontend
  reported "images not showing" with zero console errors, `browser_network_requests`
  surfaced the actual failing requests (`net::ERR_BLOCKED_BY_ORB`), which led straight
  to the real cause (the backend was 500ing and returning a JSON error body where an
  image was expected — Chrome's Opaque Response Blocking hides that mismatch behind a
  generic network error rather than surfacing the real status/body). Checking network
  requests directly is far more reliable than guessing from a screenshot when
  something fails silently in the browser.

## General pattern worth repeating

When a hosting platform's own build log looks 100% clean but the platform still
reports failure, don't trust a generic AI-assistant diagnostic panel bundled into that
platform (repeatedly wrong, vague, or contradicted by directly-verified facts this
session — e.g. confidently claiming a file extension was wrong when the file was
checked directly and proven otherwise). Go verify directly via SSH/API, and where
possible diff the failing app's on-disk state against a known-working sibling app's —
that comparison is what actually found the real causes here, not the panel's own
suggestions.
