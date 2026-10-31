# <a href="https://claude.ai"><img src="assets/claude-icon.svg" alt="Claude" height="30" align="top"></a> datasentinel-claude-artifact

Example **Claude artifacts** for the [Datasentinel](https://www.datasentinel.io) PostgreSQL
monitoring platform: self-contained HTML dashboards that query the Datasentinel **MCP server**
live from inside the page, through the viewer's own `datasentinel-*` connectors in Claude.

## Requirements and limits

- **A paid Claude plan** (Pro, Max, Team or Enterprise). Artifacts that call connectors are not
  available on the Free plan. On Team and Enterprise plans an Owner can turn them off:
  **Organization settings → Artifacts**, and **Organization settings → Capabilities → Enable
  artifact connectors**.
- **Claude Code or the Claude desktop app, to publish.** Claude Code v2.1.209 or later, or the
  desktop app 1.13576.0 or later, signed in with a claude.ai account (not an API key). An
  artifact created in a claude.ai chat conversation runs on a different runtime, without
  `claude.use`, and cannot run this page.
- **A Claude connector whose name starts with `datasentinel-`** (e.g. `datasentinel-mcp-connector`),
  pointing to your platform's MCP URL (`https://<your-platform>/mcp`), connected and signed in.
  Viewers need a connector with one of the names the artifact was published with. See
  [Adding the connector](#adding-the-connector).
- **The MCP server must be reachable from Anthropic's cloud**, over HTTPS with a publicly
  trusted certificate. Claude calls connectors from Anthropic's infrastructure, not from your
  device, in every Claude client. A platform reachable only on a private network or a VPN is
  therefore not reachable as it stands: see [Private platforms](#private-platforms). A local MCP
  server declared in Claude Desktop does not help: it is called from your workstation, and this
  page only calls connectors, except through the experimental route described under
  [Private MCP server of the desktop app](#private-mcp-server-of-the-desktop-app-experimental).
- **Viewers must be signed in, in your organization.** Each viewer's calls run through their own
  connector, after they allow it for the page. Someone who opens a public link without signing
  in to claude.ai, or from outside your organization, sees the page without its data.
- **Time zone.** The page sends times in the browser's local time, and the platform reads them
  in its own. The Dashboard and every instance sub-tab except Real-time are shifted when the
  two differ; the Real-time sub-tab is not affected.

### Private platforms

The platform does not have to be open to the whole internet, only to Claude. Three options, from
the simplest:

1. **Allow Anthropic's outbound addresses in your firewall.** Connector calls come from a
   published, stable range (`160.79.104.0/21` at the time of writing; see
   [Anthropic IP addresses](https://platform.claude.com/docs/en/api/ip-addresses)). Open the MCP
   host to that range only.
2. **Publish only the MCP host through a gateway** (a reverse proxy) with a public name and
   certificate, and add that name as the connector. The gateway can also be restricted to the
   range above.
3. **MCP tunnels.** A small stack inside your network opens an outbound-only connection to
   Anthropic, with no inbound rule and no public endpoint. Research preview, on the Enterprise
   plan, by request: see [MCP tunnels](https://claude.com/docs/connectors/mcp-tunnels/overview).

With options 1 and 2, check the following, which are the usual causes of *"Couldn't reach the MCP
server"* (see [Troubleshoot your connector](https://claude.com/docs/connectors/building/troubleshooting)):

- **Public DNS.** The connector's host name must resolve, in public DNS, to a public IPv4
  address. A name that resolves only inside your network, or to a private address from outside
  (split-horizon DNS), or that has only an IPv6 record, is rejected before any request is sent.
- **Sign-in endpoints.** OAuth discovery and token requests come from the same range as the
  tool calls. Expose the `/.well-known/` discovery paths and the authorization server they
  point to, not only `/mcp`.
- **No redirect to another host.** Use the URL the server answers on. Claude drops the
  credential when it follows a redirect to a different host, and the sign-in fails.

### Private MCP server of the desktop app (experimental)

A fourth route exists for a platform reachable only from your workstation, with no public
endpoint at all. The artifact runtime can declare a **local MCP server of the Claude desktop
app** instead of a connector, named `host:<server>` after the entry in the app's configuration
file; the page then calls it from your workstation, like any MCP server you use in the chat.

Read the caveats first. As of October 2026 the route is **undocumented**: Anthropic describes
it as not routed yet ([issue #89904](https://github.com/anthropics/claude-code/issues/89904)).
The artifact publishes, but whether its calls reach the server depends on your version of the
desktop app. It is announced as **owner-only** and **desktop-app-only**: nobody else can use the
dashboard, and it never loads in a browser tab. The connector route above stays the supported
one; the two can coexist as two separate artifacts, one for you and one for the people you
share with.

What changes, compared with [Publishing the page](#publishing-the-page):

1. **Declare the server in the desktop app**, under **Settings → Developer → Edit Config**
   (`claude_desktop_config.json`). The file takes local (stdio) servers, so the entry runs
   [`mcp-remote`](https://www.npmjs.com/package/mcp-remote), which bridges the platform's HTTP
   MCP endpoint to a local server and runs the OAuth sign-in in your browser (Node.js required).
   Keep the name within letters, digits, `-` and `_`, and restart the app completely:

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
   sign in once when the login page opens. The tools are named
   `mcp__datasentinel-private__<tool>`.
3. **Publish from the desktop app** with the local-server prompt below instead of the connector
   one. It sets `DS_CONNECTORS` to `['host:datasentinel-<name>']` and the `artifact-capabilities`
   block to one `host:` server, and declares it at publish time:

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

   If Claude answers that host servers are not available in this session, your version does
   not offer the route yet: fall back to a connector.
4. **Open the published artifact inside the desktop app**, with the account that published it,
   and allow the server when asked. *"No connector named "host:datasentinel-private" is
   connected."* means the app does not route `host:` calls yet, or the server is not running:
   nothing to fix on the platform side. In a browser tab the page always shows that message,
   because the server lives in the app.

The step-by-step version of this route, next to the connector route, is in
the Datasentinel documentation: [Claude Artifact](https://docs.datasentinel.io/manual/ai-connector/examples/claude-artifact#publish-it-against-your-platform).

## The example: `datasentinel-postgresql-monitor.html`

A single-file (~275 KB, zero build step) **PostgreSQL fleet workload monitor**. Unlike a static
report, the page calls the Datasentinel MCP tools itself every time you load, filter, drill down
or refresh: the data on screen is always fetched at that moment, never baked in.

![Animated walkthrough of the dashboard: instances ranked by load, a row expanded to its top queries, a question answered in the Ask Claude pane, the top query opened down to its execution plan with Claude asked for an index, then the instance's Overview and Real-time sub-tabs](assets/dashboard-demo.svg)

*Animated illustration with simulated data, not a recording.*

### The flow

1. **Pick a cohort** — the page loads the monitored fleet; faceted tag filters
   (environment, application, datacenter, provider, HA cluster, PostgreSQL version) narrow it
   down, with a Top-N-by-load cap (10/20/50/100/all).
2. **Rank by load** — the cohort is ranked by **Average Active Sessions**
   over the chosen time window (5 m / 15 m / 30 m / 1 h / today / custom up to 24 h). Each
   instance row shows a load-profile bar, whose length is relative to the busiest instance and
   whose segments are the wait classes (CPU, I/O, Locks, LWLocks, IPC, Timeout, Client, and
   Other for the events the server groups beyond its top 50), then the severity-coloured
   Average Active Sessions and the dominant wait. Click **Details** to open the instance in its
   own tab, or **Live** to open it on its Real-time sub-tab.
3. **Expand a row** — expanding an instance makes one call for it and shows its
   top 5 queries (share of load, peak concurrency) and its wait breakdown. Nothing runs in the
   background after the ranking: an instance is only queried when you expand or open it.

### Tabs

The page opens on the **Dashboard**. Each instance you open gets its own tab next to it, up to
10 (the least recently used one closes first). An instance tab has six sub-tabs, and keeps what
it has already loaded: switching between instances or sub-tabs does not query the platform again.
Changing the time window reloads the view on screen, and the other tabs when you come back to
them.

| Tab | What it shows |
|---|---|
| **Dashboard** | Cohort KPIs and the ranked instance rows described above |
| Instance → **Overview** | Narrative analysis, wait-class breakdown, top wait events, and the top queries by load merged with the top queries by execution time (click one to drill in) |
| Instance → **Query detail** | Full query text, KPIs (calls, ms/call, cache hit, WAL, temp spill), waits by category, block access and I/O, the **tables the query accesses** (found in its plans and SQL, with size, bloat, scans and indexes), and the captured **execution plans** with a per-plan verdict and multi-plan comparison. Opened without a query, it lists the top queries to choose from |
| Instance → **Top tables** | Top tables and top indexes, ranked by a criterion you choose (reads from memory or disk, scans, rows read, inserts, updates, deletes, HOT updates, dead rows, changes since analyze, size); each table expands to its row counts and vacuum/analyze history, with an automatic comment |
| Instance → **Vacuum & bloat** | Vacuum health (dead rows, position against the autovacuum and autoanalyze triggers, last vacuum and analyze, suggested next step), table and index bloat, transaction ID age against `autovacuum_freeze_max_age`, and the autovacuum settings |
| Instance → **I/O** | `pg_stat_io` by process, object and context (reads, writes, extends, hits, evictions, reuses, fsyncs, writebacks, latencies), who writes the data pages, block I/O by process, I/O wait events, host disk, and the I/O settings |
| Instance → **Real-time** | Live sessions, blocking chains (holders and waiters), maintenance operations in progress, active sessions by wait event, and a 3-minute host and database load panel (CPU, load average, memory, I/O wait, cache hit, connections, WAL) |

Charts are Chart.js 4 (the page's one external dependency, loaded from jsDelivr with subresource
integrity); everything degrades to text if it fails to load. Light/dark theme (following the
system setting until you choose), responsive layout, and per-viewer preferences (theme, window,
filters, pane width) kept in the browser's storage.

### The `datasentinel-*` connectors

The page calls the connectors listed in `DS_CONNECTORS` by display name (e.g.
`datasentinel-mcp-connector`). The file in this repository lists `datasentinel-production-europe`;
the publishing prompt replaces it with your own connectors. The runtime only exposes connectors named in the
page's manifest, so the list is fixed at publish time: the publishing prompt asks which of your
session's `datasentinel-*` connectors to include (see [Publishing it](#publishing-it)); republish to
change it. The header's **Connector** list switches between them and flags any that is not
connected. Each viewer needs a connector with one of these names, and sees the platform it points
to.

### Authentication

The Datasentinel MCP server uses **OAuth 2.1**, handled by the **connector**: you sign in once
with your Datasentinel (Grafana) login and Claude refreshes the token silently. The page holds
**no token** — nothing in the file, the browser or the tool arguments — and every call runs as the
signed-in user, so the platform's RBAC applies. If a connector is missing or needs to sign in
again, the page shows a notice: reconnect it in Claude under **Customize → Connectors**
([claude.ai/customize/connectors](https://claude.ai/customize/connectors)), then press
**Retry**.

## Ask Claude — chat over what's on screen

The page has a built-in chat pane where you can **ask Claude any question** about what you are
looking at: why an instance is busy, which query to optimise first, whether there is lock
contention, what an execution plan means, and so on. 

Claude answers from the data shown in the current tab, so its answers refer to the instances,
queries and numbers on your screen. Each tab offers suggested questions to start from.

It also reads that data the way the `datasentinel` skill would. The prompt carries a condensed
version of the skill's analysis playbook for the current tab (fleet ranking, workload, query,
real-time and locks, I/O, tables and autovacuum), so the answers follow the same rules: findings
ranked by impact, a buffer miss is not a disk read until the host agrees, dead rows are a bloat
proxy rather than a percentage, the root blocker rather than the loudest victim, an index with
zero scans only "appears unused", and the next step named as the tab that holds it.

- **Communication style.** *TECHNICAL* (PostgreSQL vocabulary, evidence and actions) or
  *PLAIN* (plain language, impact on the application), the two communication styles of the
  `datasentinel` skill. The style applies to the whole page, not only to the answers: every
  label, heading and automatic comment switches with it. The default is your last choice, else
  the style of your Datasentinel role (a Data admin gets TECHNICAL, a Developer gets PLAIN),
  else PLAIN.
- **Model.** *Fast*, *Balanced* (the default) or *Most capable*, depending on how much
  reasoning the question needs.

The chat uses your own Claude account; Claude asks for your consent the first time.

## Publishing it

Publishing requires the **`datasentinel` skill** to be installed in Claude (see
[`datasentinel-skill`](https://github.com/datasentinel/datasentinel-skill) for installation and
connector setup). It teaches Claude how to use the Datasentinel tools, which the publishing prompt
relies on to check your connectors, and it is the source of the page's two communication styles
(TECHNICAL and PLAIN) and of the per-tab analysis playbooks the chat follows. The published page itself does not need the skill: viewers only need
a `datasentinel-*` connector.

### Adding the connector

Add one connector per platform, with a name starting with **`datasentinel-`** (e.g.
`datasentinel-mcp-connector`) and the URL `https://<your-platform>/mcp`. Where you add it depends
on your plan (see
[Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/remote-mcp)):

- **Pro or Max**: **Customize → Connectors → Add custom connector**, in your own account.
- **Team or Enterprise**: members cannot add a custom connector. An Owner adds it once under
  **Organization settings → Connectors → Add → Custom → Web**; each member then opens
  **Customize → Connectors** and clicks **Connect**. The name the Owner chose is the same for
  every member, which is what a shared artifact needs.

Then connect it and sign in on the Datasentinel login page.

### Publishing the page

1. Add the connector as described above.
2. Open Claude Code or the Claude desktop app (see
   [Requirements and limits](#requirements-and-limits)) and check the connector works: ask
   *"list my Datasentinel instances"*.
3. Give Claude `datasentinel-postgresql-monitor.html` (attach it, or run Claude Code in the
   folder that holds it) and paste:

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

4. Open the published artifact, allow the connectors when asked, and check the dashboard loads.
   The **Connector** list in the header shows the connectors you chose.

To change the list later, run the same prompt again on the file: the artifact is republished
with the new list.

For a platform reachable only from your workstation, see the experimental
[Private MCP server of the desktop app](#private-mcp-server-of-the-desktop-app-experimental)
route, which uses a different prompt. The same steps, with the connector route beside them, are
in the Datasentinel documentation: [Claude Artifact](https://docs.datasentinel.io/manual/ai-connector/examples/claude-artifact).

### Sharing it

The artifact is private until you share it from the **Share** control in its header. Each person
you share it with needs their own connector with one of the published names, connected and
signed in, and is asked to allow it the first time the page calls it. They see the data their
own Datasentinel account can access. Connector calls only run for signed-in members of your
organization, so sharing the live page takes a Team or Enterprise plan: on Pro and Max the only
sharing is a public link, which shows the page without its data. See
[Share session output as artifacts](https://code.claude.com/docs/en/artifacts#pull-live-data-with-mcp-connectors).

## Make it your own

The artifact is **completely customizable**. It is a single HTML file that Claude wrote, and
Claude can change it for you: open a conversation in Claude, attach the file (or refer to the
artifact you published), describe the change you want in plain words, and ask Claude to publish
the new version. You don't need to read or edit the code yourself, and you can keep iterating
until the page shows exactly what you need.

Not sure what data is available? Ask Claude *"What data can my Datasentinel connector
provide?"* to get an overview of everything the platform exposes (instances,
workload, queries, tables, replication, live sessions, host and PostgreSQL metrics, reports…),
then pick what you want the page to show.

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

If a change needs data the page does not fetch yet (for example replication metrics or
blocking scenarios), Claude adds it to what the page is allowed to call and declares it when
publishing, as described in [Publishing it](#publishing-it).

## Sandbox — try it without your own platform

A public **sandbox platform** is available at `https://demo.datasentinel.io`, monitoring a fleet
of about 20 PostgreSQL instances. It comes with a shared test account — the credentials below are
**public by design** (the account is read-only):

| | |
|---|---|
| Platform | `https://demo.datasentinel.io` |
| MCP server URL (for the connector) | `https://demo.datasentinel.io/mcp` |
| Username | `sandbox` |
| Password | `Mcp-Sandbox-10!` |

Add a connector named e.g. **`datasentinel-sandbox`** with the URL `https://demo.datasentinel.io/mcp`. When you
connect it, the Datasentinel login page opens in your browser: sign in with the username and
password above. Then follow [Publishing it](#publishing-it).

## How the pieces fit

| Piece | Role | Where it lives |
|---|---|---|
| Datasentinel MCP server | The only channel to the platform: instances, workload, queries, live sessions, metrics | Your platform, added in Claude as a `datasentinel-*` connector |
| `datasentinel` skill | Tells Claude *how* to analyse: which tools to call, how to shape the answer | [`datasentinel-skill`](https://github.com/datasentinel/datasentinel-skill) |
| This artifact | An interactive front-end that calls the same tools itself, as the viewer | `datasentinel-postgresql-monitor.html` |

## Related

- [`datasentinel-skill`](https://github.com/datasentinel/datasentinel-skill) — the Claude skill,
  including MCP server configuration instructions.
- [Datasentinel documentation](https://docs.datasentinel.io), and its
  [Claude Artifact](https://docs.datasentinel.io/manual/ai-connector/examples/claude-artifact) page in the AI Connector section:
  the step-by-step publishing guide, both routes, and the first-load troubleshooting.

Claude documentation this page relies on:

- [Share session output as artifacts](https://code.claude.com/docs/en/artifacts) — publishing,
  sharing, connector calls from a page, availability and admin settings.
- [Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/remote-mcp)
  and [Get started with custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)
  — adding a connector by URL, per plan, and the network requirements.
- [Anthropic IP addresses](https://platform.claude.com/docs/en/api/ip-addresses) — the outbound
  range to allow in a firewall.
- [MCP tunnels](https://claude.com/docs/connectors/mcp-tunnels/overview) — reaching a server
  inside a private network.
- [Troubleshoot your connector](https://claude.com/docs/connectors/building/troubleshooting) and
  [Authentication for connectors](https://claude.com/docs/connectors/building/authentication)
  — connection and sign-in failures, OAuth requirements.
