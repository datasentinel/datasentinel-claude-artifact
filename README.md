# <a href="https://claude.ai"><img src="assets/claude-icon.svg" alt="Claude" height="30" align="top"></a> datasentinel-claude-artifact

Example **Claude artifacts** for the [Datasentinel](https://www.datasentinel.io) PostgreSQL
monitoring platform: self-contained HTML dashboards that query the Datasentinel **MCP server**
live, through the viewer's own `datasentinel-*` connectors in Claude.

## The example: `datasentinel-postgresql-monitor.html`

A single-file **PostgreSQL fleet workload monitor** (~275 KB, no build step). The page calls the
Datasentinel MCP tools itself on every load, filter, drill-down or refresh: the data on screen is
fetched at that moment, never baked in.

![Animated walkthrough of the dashboard: instances ranked by load, a row expanded to its top queries, a question answered in the Ask Claude pane, the top query opened down to its execution plan with Claude asked for an index, then the instance's Overview and Real-time sub-tabs](assets/dashboard-demo.svg)

*Animated illustration with simulated data, not a recording.*

### The flow

1. **Pick a cohort.** Faceted tag filters (environment, application, datacenter, provider, HA
   cluster, PostgreSQL version) narrow the monitored fleet, capped at the Top N by load
   (10/20/50/100/all).
2. **Rank by load.** The cohort is ranked by **Average Active Sessions** over the time window
   (5 m / 15 m / 30 m / 1 h / today / custom up to 24 h). Each row shows a load-profile bar
   (length relative to the busiest instance, one segment per wait class: CPU, I/O, Locks,
   LWLocks, IPC, Timeout, Client, and Other for the events beyond the server's top 50), the
   severity-coloured Average Active Sessions and the dominant wait. **Details** opens the
   instance in its own tab, **Live** opens it on its Real-time sub-tab.
3. **Expand a row.** One call shows its top 5 queries (share of load, peak concurrency) and its
   wait breakdown. Nothing runs in the background: an instance is only queried when you expand
   or open it.

### Tabs

The page opens on the **Dashboard**. Each instance you open gets its own tab, up to 10 (the least
recently used closes first), with six sub-tabs that keep what they loaded: switching does not
query the platform again. Changing the time window reloads the view on screen, and the other
tabs when you come back to them.

| Tab | What it shows |
|---|---|
| **Dashboard** | Cohort KPIs and the ranked instance rows |
| Instance → **Overview** | Narrative analysis, wait classes, top wait events, and the top queries by load merged with those by execution time (click one to drill in) |
| Instance → **Query detail** | Full query text, KPIs (calls, ms/call, cache hit, WAL, temp spill), waits by category, block access and I/O, the **tables the query accesses** (from its plans and SQL, with size, bloat, scans and indexes), and the captured **execution plans** with a per-plan verdict and multi-plan comparison. Opened without a query, it lists the top queries to pick from |
| Instance → **Top tables** | Top tables and indexes, ranked by the criterion you choose (reads from memory or disk, scans, rows read, inserts, updates, deletes, HOT updates, dead rows, changes since analyze, size); each table expands to its row counts, vacuum/analyze history and an automatic comment |
| Instance → **Vacuum & bloat** | Vacuum health (dead rows, position against the autovacuum and autoanalyze triggers, last vacuum and analyze, suggested next step), table and index bloat, transaction ID age against `autovacuum_freeze_max_age`, and the autovacuum settings |
| Instance → **I/O** | `pg_stat_io` by process, object and context (reads, writes, extends, hits, evictions, reuses, fsyncs, writebacks, latencies), who writes the data pages, block I/O by process, I/O wait events, host disk, and the I/O settings |
| Instance → **Real-time** | Live sessions, blocking chains (holders and waiters), maintenance operations in progress, active sessions by wait event, and a 3-minute host and database load panel (CPU, load average, memory, I/O wait, cache hit, connections, WAL) |

Charts use Chart.js 4, the page's one external dependency (loaded from jsDelivr with subresource
integrity); without it, everything degrades to text. Light/dark theme (following the system until
you choose), responsive layout, and per-viewer preferences (theme, window, filters, pane width)
kept in the browser's storage.

### Ask Claude — chat over what's on screen

A built-in chat pane answers **any question** about what you are looking at: why an instance is
busy, which query to optimise first, whether there is lock contention, what an execution plan
means. Claude answers from the data in the current tab, so its answers refer to the instances,
queries and numbers on your screen. Each tab suggests questions to start from.

The prompt carries a condensed version of the `datasentinel` skill's analysis playbook for the
current tab (fleet ranking, workload, query, real-time and locks, I/O, tables and autovacuum), so
the answers follow the skill's rules: findings ranked by impact, a buffer miss is not a disk read
until the host agrees, dead rows are a bloat proxy rather than a percentage, the root blocker
rather than the loudest victim, an index with zero scans only "appears unused", and the next step
named as the tab that holds it.

- **Communication style.** *TECHNICAL* (PostgreSQL vocabulary, evidence and actions) or *PLAIN*
  (plain language, impact on the application), the skill's two styles. It switches every label,
  heading and automatic comment of the page, not only the answers. Default: your last choice,
  else the style of your Datasentinel role (a Data admin gets TECHNICAL, a Developer gets PLAIN),
  else PLAIN.
- **Model.** *Fast*, *Balanced* (the default) or *Most capable*, depending on how much reasoning
  the question needs.

The chat uses your own Claude account, and asks for your consent the first time.

### The `datasentinel-*` connectors

The page calls the connectors listed in `DS_CONNECTORS` by display name. The file in this
repository lists `datasentinel-production-europe`; the publishing prompt replaces it with the
`datasentinel-*` connectors you pick from your session (see [Publishing it](#publishing-it)). The
runtime only exposes connectors named in the page's manifest, so the list is fixed at publish
time: republish to change it. The header's **Connector** list switches between them and flags any
that is not connected. Each viewer needs a connector with one of these names, and sees the
platform it points to.

### Authentication

**OAuth 2.1**, handled by the **connector**: you sign in once with your Datasentinel (Grafana)
login, and Claude refreshes the token silently. The page holds **no token** (not in the file, the
browser or the tool arguments), and every call runs as the signed-in user, so the platform's RBAC
applies. If a connector is missing or must sign in again, the page shows a notice: reconnect it
under **Customize → Connectors** ([claude.ai/customize/connectors](https://claude.ai/customize/connectors)),
then press **Retry**.

## Publishing it

Publishing needs the **`datasentinel` skill** in Claude
([Install the Skill](https://docs.datasentinel.io/manual/ai-connector/install-the-skill)): the
publishing prompt relies on it to check your connectors, and it is the source of the page's two
communication styles and of the chat's per-tab playbooks. Viewers do not need it, only a
`datasentinel-*` connector.

### Adding the connector

Add one connector per platform, with a name starting with **`datasentinel-`** (e.g.
`datasentinel-mcp-connector`) and the URL `https://<your-platform>/mcp`
([Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/remote-mcp)):

- **Pro, Max or Ultra**: **Customize → Connectors → Add custom connector**, in your own account.
- **Team or Enterprise**: members cannot add one. An Owner adds it once under **Organization
  settings → Connectors → Add → Custom → Web**, then each member clicks **Connect** in
  **Customize → Connectors**. Every member gets the name the Owner chose, which is what a shared
  artifact needs.

Then connect it and sign in on the Datasentinel login page.

### Publishing the page

1. With the connector added, open Claude Code or the Claude desktop app (see
   [Requirements and limits](#requirements-and-limits)) and check it works: ask
   *"list my Datasentinel instances"*.
2. Give Claude `datasentinel-postgresql-monitor.html` (attach it, or run Claude Code in its
   folder) and paste:

   ```text
   Publish the attached datasentinel-postgresql-monitor.html as an artifact.

   Step 1 - Datasentinel connectors. List the MCP connectors available in this
   session whose display name starts with "datasentinel-" (from the tool names
   mcp__<connector>__<tool>; ignore a "claude_ai_" prefix on the <connector>
   part, it is not in the display name). Show me a numbered list with, for each
   one, its display name and whether it answers (list its instances once; show
   only how many there are, not their data). Then ask me which ones
   to include in the artifact, and wait for my answer. If there is none, stop
   and tell me to add a connector named datasentinel-<something> in
   Customize > Connectors.

   Step 2 - Adapt the file to my choice, and change nothing else:
   - DS_CONNECTORS in the script: the display names I chose, in the order of
     the list;
   - the "artifact-capabilities" JSON block at the top: one "servers" entry per
     chosen connector, each with all the tools listed there, same order.

   Step 3 - Publish it, declaring the "mcp" capability exactly as in that block
   (ignore its "_comment" key) and the "sample" capability. If the publish
   result reports display names that differ from DS_CONNECTORS, fix
   DS_CONNECTORS to match and publish again. Do not add, embed or ask for any
   token.
   ```

3. Open the published artifact, allow the connectors when asked, and check the dashboard loads.
   The header's **Connector** list shows the connectors you chose.

To change the list later, run the same prompt again on the file: the artifact is republished. For
a platform reachable only from your workstation, see the experimental
[Private MCP server of the desktop app](#private-mcp-server-of-the-desktop-app-experimental)
route, with its own prompt. Both routes are also described step by step in the Datasentinel
documentation: [Claude Artifact](https://docs.datasentinel.io/manual/ai-connector/examples/claude-artifact#publish-it-against-your-platform).

### Sharing it

The artifact is private until you share it from the **Share** control in its header. Each person
you share it with needs their own connector with one of the published names, connected and signed
in, allows it the first time the page calls it, and sees the data their own Datasentinel account
can access. Connector calls only run for signed-in members of your organization, so sharing the
live page takes a Team or Enterprise plan. Anyone who opens a public link without signing in, or
from outside your organization, sees the page without its data; on Pro, Max and Ultra, that is
anyone other than you, including the people you invite by email. See
[Share session output as artifacts](https://code.claude.com/docs/en/artifacts#pull-live-data-with-mcp-connectors).

## Requirements and limits

- **A paid Claude plan** (Pro, Max, Ultra, Team or Enterprise): artifacts that call connectors are
  not available on the Free plan. On Team and Enterprise plans an Owner can turn them off, under
  **Organization settings → Artifacts** and **Organization settings → Capabilities → Enable
  artifact connectors**.
- **Claude Code v2.1.209 or later, or the Claude desktop app 1.13576.0 or later, to publish**,
  signed in with a claude.ai account (not an API key). A legacy artifact, made in a chat before
  September 16, 2026, runs on a different runtime without `claude.use` and cannot run this page.
- **A `datasentinel-*` connector** to `https://<your-platform>/mcp`, connected and signed in, for
  you and for each viewer: see [Adding the connector](#adding-the-connector) and
  [Sharing it](#sharing-it).
- **An MCP server reachable from Anthropic's cloud**, over HTTPS on port 443 (Claude cannot
  connect to another port, such as `:8443`) with a publicly trusted certificate. Claude calls
  connectors from Anthropic's infrastructure in every Claude client, never from your device, so a
  platform reachable only on a private network or a VPN needs one of the
  [Private platforms](#private-platforms) options. A local MCP server declared in Claude Desktop
  does not help: this page only calls connectors, except through the experimental
  [Private MCP server of the desktop app](#private-mcp-server-of-the-desktop-app-experimental)
  route.
- **Time zone.** The page sends times in the browser's time zone, and the platform reads them in
  its own: when they differ, the Dashboard and every instance sub-tab except Real-time are shifted.

### Private platforms

The platform does not have to be open to the whole internet, only to Claude. Three options, from
the simplest:

1. **Allow Anthropic's outbound addresses in your firewall**, and open the MCP host to them only:
   a published, stable range (`160.79.104.0/21` and `153.61.208.0/20` at the time of writing; see
   [Anthropic IP addresses](https://platform.claude.com/docs/en/api/ip-addresses)).
2. **Publish only the MCP host through a gateway** (a reverse proxy) with a public name and
   certificate, and add that name as the connector. The gateway can also be restricted to the
   same range.
3. **MCP tunnels** (research preview, on the Enterprise plan, by request: see
   [MCP tunnels](https://claude.com/docs/connectors/mcp-tunnels/overview)). A small stack inside
   your network opens an outbound-only connection to Anthropic, with no inbound rule and no public
   endpoint. The Datasentinel sign-in endpoints share the `/mcp` host, so the Owner sets the
   connector's **Tunnel OAuth configuration**: the browser reaches `/authorize` on the platform's
   own name, and Claude reaches `/token` and `/register` through the tunnel
   ([Authenticate to MCP servers behind a tunnel](https://claude.com/docs/connectors/mcp-tunnels/oauth)).
   Not validated with Datasentinel yet: test the sign-in and a tool call first.

With options 1 and 2, these are the usual causes of *"Couldn't reach the MCP server"* (see
[Troubleshoot your connector](https://claude.com/docs/connectors/building/troubleshooting)):

- **Public DNS.** The host name must resolve, in public DNS, to a public IPv4 address. A name that
  resolves only inside your network, or to a private address from outside (split-horizon DNS), or
  that has only an IPv6 record, is rejected before any request is sent.
- **Sign-in endpoints.** OAuth discovery and token requests come from the same range as the tool
  calls: expose the `/.well-known/` discovery paths and the authorization server they point to,
  not only `/mcp`.
- **No redirect to another host.** Claude drops the credential when it follows one, and the
  sign-in fails: use the URL the server answers on.
- **Port 443.** The connector URL must not name another port.

### Private MCP server of the desktop app (experimental)

A fourth route, for a platform reachable only from your workstation: the artifact runtime can
declare a **local MCP server of the Claude desktop app** instead of a connector, named
`host:<server>` after its entry in the app's configuration file. The page then calls it from your
workstation, like any MCP server you use in the chat.

The route is **not supported by Anthropic**: its
[artifacts documentation](https://code.claude.com/docs/en/artifacts#pull-live-data-with-mcp-connectors)
states that a published page cannot call local MCP servers, and the runtime still describes
`host:` calls as not routed yet ([issue #89904](https://github.com/anthropics/claude-code/issues/89904)).
As of November 2026 the result depends on your version of the Claude app: some versions refuse the
publish, others publish the artifact but expose none of the server's tools to it. It is announced
as **owner-only** and **desktop-app-only**: nobody else can use the dashboard, and it never loads
in a browser tab. The connector route stays the supported one; both can coexist as two artifacts,
one for you and one for the people you share with.

What changes, compared with [Publishing the page](#publishing-the-page):

1. **Declare the server in the desktop app**, under **Settings → Developer → Edit Config**
   (`claude_desktop_config.json`). The file takes local (stdio) servers, so the entry runs
   [`mcp-remote`](https://www.npmjs.com/package/mcp-remote), which bridges the platform's HTTP
   MCP endpoint to a local server and runs the OAuth sign-in in your browser (Node.js required).
   Keep the name within letters, digits, `-` and `_`, then restart the app completely:

   ```json
   {
     "mcpServers": {
       "datasentinel-private": {
         "command": "cmd",
         "args": ["/c", "npx", "-y", "mcp-remote", "https://private-mcp.datasentinel.io/mcp"]
       }
     }
   }
   ```

   That is the Windows form. On macOS and Linux, set `"command": "npx"` and drop `"/c"`.
2. **Check it works in a chat of the desktop app**: ask *"list my Datasentinel instances"* and
   sign in once when the login page opens. The tools are named `mcp__datasentinel-private__<tool>`.
3. **Publish from the desktop app** with this prompt instead of the connector one. It sets
   `DS_CONNECTORS` to `['host:datasentinel-<name>']` and the `artifact-capabilities` block to one
   `host:` server, and declares it at publish time:

   ```text
   Publish the attached datasentinel-postgresql-monitor.html as an artifact,
   against my local MCP server starting with "datasentinel-" (its tools are named
   mcp__datasentinel-<name>__<tool>). It is not a claude.ai connector: declare
   it as the host server "host:datasentinel-<name>".

   Step 1 - Check it answers: list its instances once, and show me only how many
   there are. If the server is not available in this session, stop and tell me.

   Step 2 - Adapt the file, and change nothing else:
   - DS_CONNECTORS in the script: exactly ['host:datasentinel-<name>'];
   - the "artifact-capabilities" JSON block at the top: one "servers" entry,
     with "server": "host:datasentinel-<name>" and all the tools listed there,
     same order.

   Step 3 - Publish it, declaring the "mcp" capability exactly as in that block
   (ignore its "_comment" key) and the "sample" capability. Show me the publish
   result: it should report the host server and its number of tools. Do not add,
   embed or ask for any token.
   ```

   If Claude answers that host servers are not available in this session, your version does not
   offer the route yet: fall back to a connector.
4. **Open the published artifact inside the desktop app**, with the account that published it, and
   allow the server when asked. Neither message below has a fix on the platform side:
   - *"No connector named "host:datasentinel-private" is connected."*: the app does not route
     `host:` calls yet, or the server is not running. In a browser tab the page always shows it,
     because the server lives in the app.
   - *"This page is not allowed to call this Datasentinel tool"*, although you allowed the server:
     the app reports it as connected but offers none of its tools to the artifact, a regression
     reported after the desktop app update of November 2026 (the same tools keep working in the
     chat).

## Make it your own

The artifact is **completely customizable**: it is a single HTML file that Claude wrote, and
Claude can change it for you. Attach the file (or refer to the artifact you published), describe
the change in plain words, and ask Claude to publish the new version. You don't need to read or
edit the code, and you can keep iterating until the page shows exactly what you need. Not sure
what data is available? Ask *"What data can my Datasentinel connector provide?"* for an overview
of everything the platform exposes (instances, workload, queries, tables, replication, live
sessions, host and PostgreSQL metrics, reports…).

Some examples of what you can ask:

- *"Open the dashboard on the last hour instead of the last 15 minutes, and show the top 20
  instances by default."*
- *"Add a column with the PostgreSQL version next to each instance name."*
- *"Only show production instances; hide the environment filter."*
- *"Add the replication lag of each replica to the Real-time sub-tab."*
- *"Add a Blocking sub-tab with the blocking scenarios of the time window."*
- *"Use our company colours and put our logo in the header."*
- *"Translate the whole page into French."*
- *"Remove the I/O and Vacuum & bloat sub-tabs; we only need the workload and the live view."*

If a change needs data the page does not fetch yet (for example replication metrics or blocking
scenarios), Claude adds it to what the page is allowed to call and declares it when publishing,
as in [Publishing it](#publishing-it).

## Sandbox — try it without your own platform

A public **sandbox platform** monitors about 20 PostgreSQL instances, with a shared, read-only
test account whose credentials are **public by design**:

| | |
|---|---|
| Platform | `https://demo.datasentinel.io` |
| MCP server URL (for the connector) | `https://demo.datasentinel.io/mcp` |
| Username | `sandbox` |
| Password | `Mcp-Sandbox-10!` |

Add a connector named e.g. **`datasentinel-sandbox`** with the MCP server URL, sign in with this
account when the Datasentinel login page opens, then follow [Publishing it](#publishing-it).

## How the pieces fit

| Piece | Role | Where it lives |
|---|---|---|
| Datasentinel MCP server | The only channel to the platform: instances, workload, queries, live sessions, metrics | Your platform, added in Claude as a `datasentinel-*` connector |
| `datasentinel` skill | Tells Claude *how* to analyse: which tools to call, how to shape the answer | [Install the Skill](https://docs.datasentinel.io/manual/ai-connector/install-the-skill) |
| This artifact | An interactive front-end that calls the same tools itself, as the viewer | `datasentinel-postgresql-monitor.html` |

## Related

- [Datasentinel documentation](https://docs.datasentinel.io/ai-connector), and its
  [Claude Artifact](https://docs.datasentinel.io/manual/ai-connector/examples/claude-artifact)
  page in the AI Connector section: the step-by-step publishing guide, both routes, and the
  first-load troubleshooting.

Claude documentation this page relies on:

- [Share session output as artifacts](https://code.claude.com/docs/en/artifacts): publishing,
  sharing, connector calls from a page, availability and admin settings.
- [Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/remote-mcp)
  and [Get started with custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp):
  adding a connector by URL, per plan, and the network requirements.
- [Anthropic IP addresses](https://platform.claude.com/docs/en/api/ip-addresses): the outbound
  ranges to allow in a firewall.
- [MCP tunnels](https://claude.com/docs/connectors/mcp-tunnels/overview): reaching a server inside
  a private network.
- [Troubleshoot your connector](https://claude.com/docs/connectors/building/troubleshooting) and
  [Authentication for connectors](https://claude.com/docs/connectors/building/authentication):
  connection and sign-in failures, OAuth requirements.
