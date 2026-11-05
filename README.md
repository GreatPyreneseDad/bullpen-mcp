# Bullpen MCP server

**Bullpen is an AI sales team that works the phone.** A crew of AI callers, each with its own number and voice, gets through phone trees, hold queues and AI receptionists to a real person, then drops your rep into the live call. Every call writes to Salesforce. Herman, the manager agent, runs the whole bullpen in plain language and asks before anything leaves the building.

This repository documents Bullpen's remote MCP server. There is nothing to install or host: the server runs at

```
https://bullpen.sale/mcp
```

- Transport: Streamable HTTP (stateless, JSON responses)
- Auth: OAuth 2.1 with PKCE and dynamic client registration, or a workspace API key
- Registry: [`sale.bullpen/bullpen`](https://registry.modelcontextprotocol.io/v0/servers?search=sale.bullpen) in the official MCP Registry
- Site: [bullpen.sale](https://bullpen.sale) · Docs: [bullpen.sale/docs](https://bullpen.sale/docs) · For LLMs: [bullpen.sale/llms.txt](https://bullpen.sale/llms.txt)

## Connect

**Claude (claude.ai, desktop, mobile)**
Settings → Connectors → *Add custom connector* → `https://bullpen.sale/mcp`. The first tool that needs your workspace sends you to bullpen.sale to sign in and click **Allow**.

**Claude Code**
```bash
claude mcp add --transport http bullpen https://bullpen.sale/mcp
```

**Cursor, VS Code, and other MCP clients**
```json
{
  "mcpServers": {
    "bullpen": { "url": "https://bullpen.sale/mcp" }
  }
}
```

**With an API key instead of OAuth** (scripts, servers, headless agents): make a key at [bullpen.sale/app/settings/api](https://bullpen.sale/app/settings/api) and send it as a header:
```
Authorization: Bearer bp_live_...
```

## Tools

| Tool | Needs sign-in | What it does |
|---|---|---|
| `bullpen_about` | no | What Bullpen is, what it costs, how an agent gets a workspace |
| `bullpen_signup` | no | Start a paid signup ($25–$500 in credits) with the Machine Payments Protocol; returns the 402 challenge |
| `whoami` | yes | Workspace, member and role the token acts as |
| `credits_status` | yes | Prepaid balance and recent usage |
| `top_up_credits` | yes | Start an MPP credit top-up ($5–$500) |
| `ask_herman` | yes | Talk to Herman, who runs the bullpen: status, companies, campaigns, sending the crew |
| `confirm_herman_action` | yes | Confirm or cancel a call, text or Salesforce update Herman proposed |
| `ask_brain` | yes | Answer from the workspace's own calls, notes and docs, with sources |
| `find_companies` | yes | Find companies to sell to, with main phone lines and websites |
| `research_company` | yes | Who to ask for at a company |
| `get_a_human` | yes | The crew dials, works the menu and hold, gets a real person, then connects you |
| `knock` | yes | Reach a human at a company's main line and leave a short hello |
| `send_text` / `send_email` | yes | Send from the workspace's line or agent inbox |
| `floor_state` | yes | Live and recent crew calls |
| `create_campaign` / `import_campaign_contacts` / `work_campaign` | yes | Run a calling campaign from a CSV |
| `watch_companies` | yes | Watch companies for buying and risk signals |
| `onboarding_status` | yes | What's left before every feature works, in order, with the tool or link for each |
| `turn_on_herman` | yes | Switch Herman on |
| `get_outbound_agreement` / `accept_outbound_agreement` | yes | Read the compliance agreement; accept with the person's typed name, only after they agree |
| `search_numbers` / `buy_number` | yes | Find a line and buy it from credits ($3/month); a crew member goes on it |
| `hire_crew` | yes | Put AI crew members on any free lines |
| `set_mobile` | yes | The phone that rings when the crew reaches a person (yours, or a teammate's) |
| `buy_credits` | yes | Card checkout link for credits |
| `connect_salesforce` / `connect_google` | yes | Approval links the person opens |
| `invite_teammate` / `list_team` | yes | Invite by email; see members and pending invites |
| `open_page` | no | Links into the app, including `floor` to watch live calls |

Every tool carries a `title` and `readOnlyHint` / `destructiveHint` / `openWorldHint` annotations.

## Onboard from your agent

Your agent can take a new workspace all the way to live calls. Ask it to run `onboarding_status` and work the list: switch Herman on, read you the compliance agreement and record your yes, buy a crew line from credits, set your mobile, invite your team. It hands you links for the few steps that happen in your browser (card checkout, Salesforce and Google approval) and a link to the live floor whenever you want to watch.

## Agents can hire a bullpen

An agent with a wallet can sign its principal up with no checkout page:

```
POST https://bullpen.sale/pay/signup?amount=25
Content-Type: application/json

{"owner_email": "ops@yourco.com", "company": "Your Co", "agent_name": "your agent"}
```

1. The first request returns **402** with an MPP `WWW-Authenticate: Payment …` challenge (cards via Stripe Shared Payment Tokens).
2. Pay and retry with `Authorization: Payment …`.
3. The response carries `api_key`, `org_id` and `balance_cents`.

Payable endpoints are described with `x-payment-info` in [bullpen.sale/openapi.json](https://bullpen.sale/openapi.json). An agent can also attach an **AP2** mandate (`X-AP2-Mandate`) to prove its principal authorized the spend; Bullpen checks every hop's limits and signs a receipt verifiable at [`/pay/ap2/jwks`](https://bullpen.sale/pay/ap2/jwks).

## Guardrails

- **A person stands behind outbound.** Research, Herman, the brain, company search and campaigns work right away. Calls and texts unlock only after a workspace admin accepts the Outbound Compliance Agreement and adds a phone number.
- **Herman asks first.** Anything that reaches the outside world comes back as a pending action; nothing goes out until it's confirmed.
- **Calling rules:** company lines only unless the person gave written consent, calling hours in the callee's time zone, opt-outs honored, and the crew says it's an AI when asked.
- **Permissions:** a token acts as the member who approved it, in that workspace, with that member's role. It can't buy numbers, change billing, connect accounts or invite people. Revoke it any time in Settings → API keys.

## Pricing

Usage-based, prepaid credits: $0.30 per crew minute (no extra charge for reaching a human), $0.03 per text, $3/month per number. New workspaces start with $10. Spending tools return a 402 error at zero. Full table: [bullpen.sale/pricing](https://bullpen.sale/pricing).

## Other ways in

- **A2A:** Herman is an A2A agent. Card at [`/.well-known/agent-card.json`](https://bullpen.sale/.well-known/agent-card.json), JSON-RPC at `https://bullpen.sale/a2a`.
- **REST API + CLI:** one POST endpoint, every action documented at [bullpen.sale/docs](https://bullpen.sale/docs). `curl -fsSL https://bullpen.sale/install.sh | sh`

## Discovery documents

| Path | What |
|---|---|
| [`/.well-known/oauth-protected-resource/mcp`](https://bullpen.sale/.well-known/oauth-protected-resource/mcp) | OAuth protected resource metadata (RFC 9728) |
| [`/.well-known/oauth-authorization-server`](https://bullpen.sale/.well-known/oauth-authorization-server) | Authorization server metadata (RFC 8414) |
| [`/.well-known/mcp/server-card.json`](https://bullpen.sale/.well-known/mcp/server-card.json) | MCP server card |
| [`/.well-known/agent-card.json`](https://bullpen.sale/.well-known/agent-card.json) | A2A agent card |
| [`/openapi.json`](https://bullpen.sale/openapi.json) | OpenAPI 3.1 with MPP `x-payment-info` |
| [`/llms.txt`](https://bullpen.sale/llms.txt) | Summary for language models |

## Support

office@roseglass.dev · [bullpen.sale](https://bullpen.sale) · [Privacy](https://bullpen.sale/privacy) · [Terms](https://bullpen.sale/terms)

Bullpen is built by RoseGlass Corp.
