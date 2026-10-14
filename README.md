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
| `parts:read` | read parts, calculations, costs, process plans, environments and job nests |
| `parts:write` | create parts, attach uploads to them, quote, cancel, apply a what-if and run job nests |
| `uploads:write` | upload STEP files and drawings |

Your agent is only ever shown the tools its credential can use.

## Tools

With no extra configuration, the server lists these 31 tools. Each one carries
its `readOnlyHint` and `destructiveHint`, so your client knows which calls only
read and which ones change something, and can ask you first.

**Get a part in**

| Tool | What it does | |
|---|---|---|
| `presign_upload` | Answers where and how to send one file. | write |
| `confirm_upload` | Records that the file arrived and answers the upload's status. | write |
| `get_upload_link` | A link to the ARCNM upload page, for a client that cannot send files. | read |
| `create_part` | Creates a part to hold an upload. | write |
| `attach_upload` | Attaches an upload to a part revision as its CAD, drawing, mesh or RFQ text. | write |
| `list_parts` | Finds parts, by search text or folder. | read |
| `list_part_revisions` | A part's revisions, with the id a quote needs. | read |

**Price it**

| Tool | What it does | |
|---|---|---|
| `check_quota` | The organization's remaining calculation quota. | read |
| `quote_part` | Prices a part revision; several costing environments start a batch. Uses quota. | write |
| `get_calculation` | One calculation: its status and, once priced, its result. | read |
| `cancel_calculation` | Stops a calculation that has not finished. | write |
| `list_calculations` | Finds calculations, by part, batch or status. | read |

**Understand the cost**

| Tool | What it does | |
|---|---|---|
| `explain_cost` | Headline cost and times, the prediction band, the breakdown and the top cost drivers. | read |
| `get_cost_drivers` | The top cost drivers, ranked, each with its cost, share and reason. | read |
| `get_cost_breakdown` | The itemised cost sheet, at any lot size. | read |
| `get_process_plan` | The routing, with work centres and cycle times. | read |
| `suggest_cost_savings` | Every cost-reduction direction, each with its lever and confidence. | read |
| `optimize_lot_size` | The lot size with the lowest total cost, and the cost-versus-quantity curve. | read |

**Ask what-if**

| Tool | What it does | |
|---|---|---|
| `get_cost_factors` | Every factor a costing environment lets you change, with its value and source. | read |
| `preview_cost` | Re-prices a calculation with factors moved or at another lot size. Uses no quota and writes nothing. | read |
| `apply_adjustment` | Plans, and with `apply=true` makes, the changes that make a what-if stick. | write |

**Compare**

| Tool | What it does | |
|---|---|---|
| `compare_calculations` | Two calculations: the price difference. Three to twenty: a ranking. | read |
| `compare_environments` | Parts against costing environments, as one matrix. | read |
| `list_environments` | The organization's costing environments. | read |
| `get_environment` | One costing environment, with its rates and overheads. | read |

**Nest sheet metal**

| Tool | What it does | |
|---|---|---|
| `create_nest_run` | Plans, and with `apply=true` starts, a job nest of sheet-metal parts. | write |
| `start_nest_run` | Starts a job nest kept as a draft. | write |
| `cancel_nest_run` | Stops a job nest that has not ended. | write |
| `get_nest_run` | A job nest's progress, and its result once it has ended. | read |
| `get_nesting_summary` | Nesting runs used and left, and what nesting has saved. | read |
| `export_nest_layout` | The cutting files, as JSON, DXF or SVG. | read |

None of these 31 deletes anything. More tools (uploads, calculations, parts,
environments, the materials and tooling catalog, calibration, folders) are
listed when a client asks for them; see
[toolsets](https://arcnm.io/docs/mcp#toolsets).

## Prompts

The server also ships 5 prompts that walk an agent through a whole job:
`cost_a_part`, `compare_calculations`, `compare_environments`, `explain_cost`
and `find_cost_drivers`.

## Try it

- *"Quote this STEP file at lot sizes 1, 10 and 100, and tell me where the cost goes."*
- *"Which three things drive the cost of this part, and what would change them?"*
- *"Compare this bracket across our costing environments and tell me where to make it."*
- *"What lot size gives the lowest total cost?"*
- *"Nest this week's sheet-metal orders and tell me what it saves."*

## Support

Questions and problems: [hello@arcnm.io](mailto:hello@arcnm.io), or open an
issue here. Documentation: [arcnm.io/docs/mcp](https://arcnm.io/docs/mcp).
Runnable API examples: [arcnm/arcnm-examples](https://github.com/arcnm/arcnm-examples).

ARCNM is built by [rattle mng GmbH](https://arcnm.io/legal-notice) in
Überlingen, Germany.
