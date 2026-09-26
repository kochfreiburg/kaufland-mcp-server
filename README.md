# Kaufland MCP Server

**English** · [Deutsch](docs/README.de.md)

**Connect Kaufland to Claude, ChatGPT and Copilot: kaufland Marketplace MCP server: Claude & ChatGPT read your Kaufland seller orders, units, shipments, tickets and storefronts as MCP tools.** Powered by [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

Kaufland MCP Server gives Claude, ChatGPT, Copilot and Cursor 8 tools for Kaufland: kaufland Marketplace MCP server: Claude & ChatGPT read your Kaufland seller orders, units, shipments, tickets and storefronts. Every tool only reads. It runs on AnythingMCP: one click on AnythingMCP Cloud, or self-hosted with Docker. Credentials are stored encrypted and every call is audited.

**Status:** not yet verified against a live system. The adapter follows the vendor's API documentation; please report what you find.  
**Adapter synced:** <!-- synced -->2026-09-26

Maintained by [KOCH Freiburg GmbH](https://www.kochfreiburg.de/), which runs AnythingMCP in production. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Quick start (AnythingMCP Cloud)

1. Sign in at [cloud.anythingmcp.com](https://cloud.anythingmcp.com) and open the [install link](https://cloud.anythingmcp.com/connectors/store?install=kaufland).
2. Enter `KAUFLAND_CLIENT_KEY`, `KAUFLAND_SECRET_KEY` (see [Authentication](#authentication)).
3. Copy the URL of your MCP server under **MCP Servers** and add it to your AI client ([below](#connect-claude-chatgpt-copilot-or-cursor)).

AnythingMCP Cloud is the same open-source code, operated by helpcode.ai in Frankfurt, Germany.

## Self-hosted (Docker)

Needs Docker 24+, openssl and Node 18+.

```bash
git clone https://github.com/kochfreiburg/kaufland-mcp-server.git
cd kaufland-mcp-server
./scripts/install.sh
```

`install.sh` writes `.env` with fresh secrets, starts AnythingMCP, creates the first admin, installs the connector if `KAUFLAND_CLIENT_KEY` and `KAUFLAND_SECRET_KEY` are set in `.env` and creates an MCP API key. Without credentials it prints the install link instead: `http://localhost:3000/connectors/store?install=kaufland`. Then check the whole chain:

```bash
npm install && node scripts/smoke.mjs
```

## Connect Claude, ChatGPT, Copilot or Cursor

- **Claude (claude.ai, Desktop, mobile):** *Customize → Connectors → Add custom connector*, paste your MCP server URL and sign in. Claude connects from Anthropic's cloud, so the URL must be public HTTPS: your AnythingMCP Cloud URL, or your own instance behind TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http kaufland-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) and **VS Code / GitHub Copilot** (`.vscode/mcp.json`, key `servers` instead of `mcpServers`, plus `"type": "http"`):

  ```json
  { "mcpServers": { "kaufland-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** add the public HTTPS URL as a connector (app) in ChatGPT's settings. A `localhost` URL does not work there.

## Tools

8 tools, generated from [`adapter/kaufland.json`](adapter/kaufland.json). **read** tools cannot change anything in the source system.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | What it does | Access |
|---|---|---|
| `kaufland_list_warehouses` | List the seller's warehouses with their id, name and address. | read |
| `kaufland_list_orders` | List orders in a time window with their buyer, storefront, totals and status. | read |
| `kaufland_get_order` | Read one order in full: the buyer, the delivery address, the payment and every order unit with its own price, status and fulfilment state. | read |
| `kaufland_list_order_units` | List individual order units — the level Kaufland actually fulfils, cancels and pays out at. | read |
| `kaufland_list_units` | List the seller's units (offers) with their EAN, condition, price, stock and the storefront each is listed on. | read |
| `kaufland_list_shipments` | List reported shipments with their carrier, tracking number and the order units they cover — the answer to whether something has actually gone out. | read |
| `kaufland_list_tickets` | List customer service tickets with their subject, status and the order they relate to — where a buyer complaint shows up before it becomes a rating. | read |
| `kaufland_list_storefronts` | List the storefronts this seller is active on, with the currency and locale of each. | read |
<!-- tools:end -->

## Example prompts

- Which Kaufland orders from the last two days have not shipped yet?
- Show order 123-456 with every order unit and the delivery address.
- Which order units were cancelled this week, and why?
- Which customer tickets are still open?
- Which of my units on kaufland.de are out of stock?
- What is the price and stock of EAN 4006381333931 on each storefront?

More in [examples/prompts.md](examples/prompts.md).

## Authentication

**Getting credentials**
1. You must already be a Kaufland Marketplace seller. In the **Seller Portal** open **Settings → API keys** and create a key pair.
2. You receive a **Client Key** (public, identifies you) and a **Secret Key** (never sent — it signs the request). Set `KAUFLAND_CLIENT_KEY` and `KAUFLAND_SECRET_KEY`.

**The secret never travels.** Kaufland does not accept a bearer token: every request carries `Shop-Client-Key`, `Shop-Timestamp` and `Shop-Signature`, where the signature is an HMAC-SHA256 over the canonical string

```
<METHOD>\n<FULL URL INCLUDING QUERY>\n<BODY>\n<UNIX TIMESTAMP>\n
```

keyed with the secret. AnythingMCP computes it per request and sends the timestamp it used, so the server can recompute the same string.

**A signature is only valid for a few minutes.** Kaufland rejects a request whose timestamp has drifted, which means a 401 here can be a clock problem rather than a key problem. If every call fails and the keys are definitely right, check the host's time.

**Storefronts are per country.** Kaufland runs `de`, `cz`, `sk`, `pl` and `at` storefronts and most endpoints take a `storefront` parameter. Omitting it gives you the seller's default, which is rarely what a report about one market wants.

**Orders page with offset and limit**, and `ts_created_from` / `ts_created_to` bound the window — both ISO 8601. Ask for the window you mean; the default is short.

**Order units are where the money is.** An order carries `order_units`, each with its own status, price and fulfilment state. A part-cancelled order is normal, so a revenue figure should sum the units that were actually fulfilled rather than the order total.

**Cloud reachability**: sellerapi.kaufland.com is public with a valid certificate.

## Security

- **Read or write is your choice.** All 8 tools only read. Assign the connector to an MCP server whose role whitelists only the tools you want, and the rest are invisible to that client.
- **Credentials** are encrypted with AES-256-GCM and never shown to the model.
- **Response mapping** drops or reshapes fields per tool before they reach the model, e.g. bank details or personal data.
- **Audit log:** every call is recorded with input, output, duration and status, in your own database when self-hosted.
- **SSO, RBAC and SCIM** are included in the self-hosted build.

## FAQ

### Is there a Kaufland Marketplace MCP server?
Yes, this one. It connects the Kaufland Marketplace seller API to Claude, ChatGPT and Copilot through AnythingMCP: 8 tools for orders, order units, units (offers), shipments, tickets, storefronts and warehouses.

### What do I need to connect it?
A Kaufland seller account and an API key pair from the Seller Portal (Settings → API keys): the Client Key and the Secret Key.

### Is the secret key sent anywhere?
No. Kaufland signs every request with HMAC: AnythingMCP sends the Client Key, a timestamp and a signature computed with the Secret Key. The secret itself never leaves your AnythingMCP instance.

### Can the AI change orders or prices?
No. All eight tools only read.

### Why "order units" and not just orders?
Kaufland fulfils, cancels and pays out per unit, so a partly cancelled order is common. The order-unit tool shows what actually happened to each item.

## Troubleshooting

| Problem | Fix |
|---|---|
| `401` / `403` from the vendor | The credentials are wrong or lack rights. Re-enter them on the connector page; the import runs a test call and shows the result. |
| Tools missing in the AI client | The connector is not assigned to the MCP server the client uses. Check **MCP Servers**, then run `node scripts/smoke.mjs`. |
| The host is on your internal network | Self-host AnythingMCP on that network and add the hostname to `SSRF_ALLOWED_HOSTS`, or the outbound guard blocks the call. |
| Works locally, fails on AnythingMCP Cloud | The system must be reachable from the internet with a valid TLS certificate. |

## Related

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [otto-market-mcp-server](https://github.com/kochfreiburg/otto-market-mcp-server): OTTO Market MCP server: connect the otto.de partner API to Claude & ChatGPT. Orders, products, returns, stock and price updates.
- [billbee-mcp-server](https://github.com/kochfreiburg/billbee-mcp-server): Billbee MCP server: connect Billbee order management to Claude & ChatGPT. Orders, products, customers and shipping providers.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): the open-source MCP server and gateway this repository is built on.

## License

AGPL-3.0-only. The adapter definition in `adapter/` comes from AnythingMCP (AGPL-3.0).
