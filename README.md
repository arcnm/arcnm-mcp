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
| **No account** | Pay per call at `https://api.arcnm.io/mcp-server/pay/mcp`: [see below](#pay-per-call-no-account) |

## What you need

An [ARCNM](https://arcnm.io) account. The Free plan includes 10 calculations a
month, no card required. Reading results costs nothing; starting a calculation
uses your quota, and `check_quota` tells your agent how much is left. No
account? Your agent can [pay per call](#pay-per-call-no-account) instead.

## Pay per call, no account

ARCNM runs a second MCP server for agents without an ARCNM account. There is no
sign-in: the agent pays for each calculation inside the tool call.

| | |
|---|---|
| **Endpoint** | `https://api.arcnm.io/mcp-server/pay/mcp` (Streamable HTTP, no key) |
| **Registry** | `io.arcnm/arcnm-pay` in the [MCP Registry](https://registry.modelcontextprotocol.io) |
| **Price** | EUR 0.89 per calculation, VAT included. One made part in one costing environment is one calculation |
| **Guide** | [arcnm.io/docs/pay-per-call](https://arcnm.io/docs/pay-per-call) |

Bought and catalogue parts are free. A price check is free and returns a
binding offer that stands for 24 hours. A calculation that fails is never
charged.

| Tool | What it does | Cost |
|---|---|---|
| `price_parts` | Prices one CAD file or several. Unpaid, it answers with the offer as a payment requirement; paid, with the result and a `result_token`. With no file, it lists the environments, parameters, limits and unit price. | EUR 0.89 per made part per environment, when paid |
| `get_result` | A paid purchase: its status while it runs, then the summary and the receipt. | free |
| `preview_cost` | The itemised cost sheet of a paid part or component at another lot size. | free |

**Paying.** The MCP server takes x402: USDC on Base. Your client signs the
offer and calls `price_parts` again with the payment in `_meta["x402/payment"]`.
If your client can't attach a payment, pay the offer's `pay_url` over HTTP and
read the result with `get_result`. A card pays over HTTP only, at
`POST https://api.arcnm.io/api/v1/pay/price`, from 6 calculations per payment.

Add it next to the account server, or on its own, for example in
`.vscode/mcp.json`:

```json
{
  "servers": {
    "arcnm-pay": {
      "type": "http",
      "url": "https://api.arcnm.io/mcp-server/pay/mcp"
    }
  }
}
```

**Which one?** Use the account server when you price parts every day: it runs
on your plan's quota, in your own costing environments, calibrated to your
shop. Pay per call when there is no account: your agent prices in the
environments `price_parts` lists, and pays per calculation.

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

With no extra configuration, the server lists these 18 tools. Each one carries
its `readOnlyHint` and `destructiveHint`, so your client knows which calls only
read and which ones change something, and can ask you first.

**Price a part**

| Tool | What it does | |
|---|---|---|
| `upload_and_quote` | Prices a new part from its CAD file in one call: a STEP file, or a DXF flat pattern with its sheet thickness and material, plus an optional PDF drawing. Uses quota only if the calculation succeeds. | write |
| `get_upload_link` | A link to the ARCNM upload page, for a file that is on your computer. | read |
| `quote_part` | Prices a part already in ARCNM again: another lot size, material or costing environment, or several environments at once. Uses quota only if the calculation succeeds. | write |
| `precheck_part` | Reads a CAD file the way its calculation will, before it starts: how an assembly splits into made parts, catalogue parts and parts left out, how many calculations it starts and how many your plan pays for. Free; start it with `upload_and_quote` and `precheck_id`. | write |
| `get_precheck` | One precheck: its status and, once read, its answer. | read |
| `attach_drawing` | Gives a drawing to the part it belongs to: before a run, on a precheck, or after it, on the part's calculation, which is then priced again. It plans by default; `apply=true` attaches. | write |

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

None of these 18 deletes anything. More tools — direct uploads, parts, ranked
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
