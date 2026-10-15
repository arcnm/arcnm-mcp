# ARCNM MCP server

**Manufacturing should-cost from CAD and drawings, for your agent.** Connect
ARCNM to Claude, GitHub Copilot, Cursor or any MCP client, and it can price a
part from a STEP file or PDF drawing, explain what drives the cost, read the
process plan, compare designs, lot sizes and costing environments, and nest
sheet-metal orders. Every figure traces back to the physics that produced it.

This repository documents the server. ARCNM hosts it, so there is nothing to
install or run.

| | |
|---|---|
| **Endpoint** | `https://api.arcnm.io/mcp-server/mcp` (Streamable HTTP) |
| **Registry** | `io.arcnm/arcnm` in the [MCP Registry](https://registry.modelcontextprotocol.io) |
| **Sign-in** | OAuth with your ARCNM account, or an ARCNM API key as a bearer token |
| **Docs** | [arcnm.io/docs/mcp](https://arcnm.io/docs/mcp) |

## What you need

An [ARCNM](https://arcnm.io) account. The Free plan includes 10 calculations a
month, no card required. Reading results costs nothing; starting a calculation
uses your quota, and `check_quota` tells your agent how much is left.

## Connect

### GitHub Copilot in VS Code

Add the server to `.vscode/mcp.json`:

```json
{
  "servers": {
    "arcnm": {
      "type": "http",
      "url": "https://api.arcnm.io/mcp-server/mcp"
    }
  }
}
```

Start it from the MCP view and sign in to ARCNM in the browser when asked. To
use an API key instead, send it as a header:

```json
{
  "inputs": [
    { "type": "promptString", "id": "arcnm-key", "description": "ARCNM API key", "password": true }
  ],
  "servers": {
    "arcnm": {
      "type": "http",
      "url": "https://api.arcnm.io/mcp-server/mcp",
      "headers": { "Authorization": "Bearer ${input:arcnm-key}" }
    }
  }
}
```

### Claude and Claude Desktop

ARCNM is in the [Claude connector directory](https://claude.ai/directory/arcnm).
Add it there and sign in with your ARCNM account.

### Claude Code

```bash
claude mcp add --transport http arcnm https://api.arcnm.io/mcp-server/mcp
```

Then run `/mcp` in Claude Code to sign in. With an API key:

```bash
claude mcp add --transport http arcnm https://api.arcnm.io/mcp-server/mcp \
  --header "Authorization: Bearer $ARCNM_API_KEY"
```

### Cursor

In `~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project):

```json
{
  "mcpServers": {
    "arcnm": {
      "url": "https://api.arcnm.io/mcp-server/mcp"
    }
  }
}
```

Cursor opens the ARCNM sign-in. With an API key, add
`"headers": { "Authorization": "Bearer ${env:ARCNM_API_KEY}" }`.

### Any other MCP client

Point a Streamable HTTP transport at `https://api.arcnm.io/mcp-server/mcp`. A
client that supports MCP authorization finds the sign-in on its own, from the
server's [protected-resource metadata](https://api.arcnm.io/.well-known/oauth-protected-resource/mcp-server/mcp).
Otherwise, send `Authorization: Bearer <your ARCNM API key>`.

## Sign-in and permissions

**OAuth.** You sign in with your ARCNM account and choose the organization the
client may use. The consent screen lists every permission the client asks for,
and you can untick any of them. The client sees only that organization's data.

**API key.** Create one in ARCNM under **Settings → API keys** and give it only
the scopes your agent needs:

| Scope | Lets the agent |
|---|---|
| `parts:read` | read calculations, costs, process plans and costing environments |
| `parts:write` | price parts (`upload_and_quote`, `quote_part`) and cancel a calculation |
| `uploads:write` | upload files directly, with the `uploads` toolset |

Your agent is only ever shown the tools its credential can use.

## Tools

With no extra configuration, the server lists these 15 tools. Each one carries
its `readOnlyHint` and `destructiveHint`, so your client knows which calls only
read and which ones change something, and can ask you first.

**Price a part**

| Tool | What it does | |
|---|---|---|
| `upload_and_quote` | Prices a new part from its CAD file in one call: a STEP file, or a DXF flat pattern with its sheet thickness and material, plus an optional PDF drawing. Uses quota only if the calculation succeeds. | write |
| `get_upload_link` | A link to the ARCNM upload page, for a file that is on your computer. | read |
| `quote_part` | Prices a part already in ARCNM again: another lot size, material or costing environment, or several environments at once. Uses quota only if the calculation succeeds. | write |

**Follow or stop it**

| Tool | What it does | |
|---|---|---|
| `get_calculation` | One calculation: its status, the time left while it runs and, once priced, its result. | read |
| `list_calculations` | Finds calculations, by part, batch, status or start time. | read |
| `cancel_calculation` | Stops a calculation that is queued or running. A cancelled calculation is never charged. | write |

**Read the result**

| Tool | What it does | |
|---|---|---|
| `explain_cost` | Headline cost and times, the prediction band, the breakdown and the top cost drivers. | read |
| `get_cost_breakdown` | The itemised cost sheet, at any lot size. | read |
| `get_process_plan` | The routing, with work centres and cycle times. | read |

**Improve and compare**

| Tool | What it does | |
|---|---|---|
| `preview_cost` | Re-prices a calculation with factors moved or at another lot size. Uses no quota and writes nothing. | read |
| `optimize_lot_size` | The lot size with the lowest total cost, and the cost-versus-quantity curve. | read |
| `suggest_cost_savings` | Every cost-reduction direction, each with its lever and confidence. | read |
| `compare_calculations` | Two calculations: the price difference. Three to twenty: a ranking. | read |

**Lookups**

| Tool | What it does | |
|---|---|---|
| `list_environments` | The organization's costing environments. | read |
| `check_quota` | The organization's remaining calculation quota. | read |

None of these 15 deletes anything. More tools — direct uploads, parts, ranked
cost drivers, costing environments and what-if adjustments, job nests for
sheet metal, the materials and tooling catalog, calibration, folders — are
listed when your client sends the `X-MCP-Toolsets` header with the toolsets it
wants, or `all`; see [toolsets](https://arcnm.io/docs/mcp#toolsets).

## Prompts

The server also ships 5 prompts that walk an agent through a whole job:
`cost_a_part`, `compare_calculations`, `compare_environments`, `explain_cost`
and `find_cost_drivers`.

## Try it

- *"Quote this STEP file at lot sizes 1, 10 and 100, and tell me where the cost goes."*
- *"Which three things drive the cost of this part, and what would change them?"*
- *"Compare this bracket across our costing environments and tell me where to make it."*
- *"What lot size gives the lowest total cost?"*

## Support

Questions and problems: [hello@arcnm.io](mailto:hello@arcnm.io), or open an
issue here. Documentation: [arcnm.io/docs/mcp](https://arcnm.io/docs/mcp).
Runnable API examples: [arcnm/arcnm-examples](https://github.com/arcnm/arcnm-examples).

ARCNM is built by [rattle mng GmbH](https://arcnm.io/legal-notice) in
Überlingen, Germany.
