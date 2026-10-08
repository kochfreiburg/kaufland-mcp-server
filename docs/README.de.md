# Kaufland MCP Server

[English](../README.md) · **Deutsch**

**Verbinde Kaufland mit Claude, ChatGPT und Copilot: kaufland Marketplace MCP server: Claude & ChatGPT read your Kaufland seller orders, units, tickets, warehouses and storefronts als MCP-Tools.** Basiert auf [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

Kaufland MCP Server gibt Claude, ChatGPT, Copilot und Cursor 7 Tools für Kaufland: kaufland Marketplace MCP server: Claude & ChatGPT read your Kaufland seller orders, units, tickets, warehouses and storefronts. Alle Tools lesen nur. Es läuft auf AnythingMCP: mit einem Klick in AnythingMCP Cloud oder selbst gehostet mit Docker. Zugangsdaten werden verschlüsselt gespeichert, jeder Aufruf landet im Audit-Log.

**Zuletzt geprüft:** 2026-10-08 gegen a live Kaufland seller account on AnythingMCP Cloud (read-only calls of all 7 tools through the production connector).  
**Adapter synchronisiert:** <!-- synced -->2026-10-08

Maintained by [KOCH Freiburg GmbH](https://www.kochfreiburg.de/), which runs AnythingMCP in production. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Schnellstart (AnythingMCP Cloud)

1. Melde dich bei [cloud.anythingmcp.com](https://cloud.anythingmcp.com) an und öffne den [Installationslink](https://cloud.anythingmcp.com/connectors/store?install=kaufland).
2. Trage `KAUFLAND_CLIENT_KEY`, `KAUFLAND_SECRET_KEY` ein (siehe [Authentifizierung](#authentifizierung)).
3. Kopiere die URL deines MCP-Servers unter **MCP Servers** und füge sie in deinen KI-Client ein ([siehe unten](#claude-chatgpt-copilot-oder-cursor-verbinden)).

AnythingMCP Cloud ist derselbe Open-Source-Code, betrieben von helpcode.ai in Frankfurt.

## Selbst gehostet (Docker)

Benötigt Docker 24+, openssl und Node 18+.

```bash
git clone https://github.com/kochfreiburg/kaufland-mcp-server.git
cd kaufland-mcp-server
./scripts/install.sh
```

`install.sh` schreibt `.env` mit neuen Secrets, startet AnythingMCP, legt den ersten Admin an, installiert den Connector, sofern `KAUFLAND_CLIENT_KEY` und `KAUFLAND_SECRET_KEY` in `.env` gesetzt sind, und erzeugt einen MCP-API-Key. Ohne Zugangsdaten gibt es stattdessen den Installationslink aus: `http://localhost:3000/connectors/store?install=kaufland`. Danach die ganze Kette prüfen:

```bash
npm install && node scripts/smoke.mjs
```

## Claude, ChatGPT, Copilot oder Cursor verbinden

- **Claude (claude.ai, Desktop, Mobil):** *Customize → Connectors → Add custom connector*, MCP-Server-URL einfügen und anmelden. Claude verbindet sich aus der Cloud von Anthropic, die URL muss also öffentlich per HTTPS erreichbar sein: deine AnythingMCP-Cloud-URL oder deine eigene Instanz mit TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http kaufland-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) und **VS Code / GitHub Copilot** (`.vscode/mcp.json`, Schlüssel `servers` statt `mcpServers`, dazu `"type": "http"`):

  ```json
  { "mcpServers": { "kaufland-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** die öffentliche HTTPS-URL in den ChatGPT-Einstellungen als Connector (App) hinzufügen. Eine `localhost`-URL funktioniert dort nicht.

## Tools

7 Tools, erzeugt aus [`adapter/kaufland.json`](../adapter/kaufland.json). Tools mit **lesen** können im Quellsystem nichts ändern.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | Funktion | Zugriff |
|---|---|---|
| `kaufland_list_warehouses` | List the seller's warehouses with their id, name and address. | lesen |
| `kaufland_list_orders` | List orders in a time window: id, storefront, creation time and number of units. | lesen |
| `kaufland_get_order` | Read one order in full: the buyer, the billing and shipping address and every order unit with its own price, status and delivery window. | lesen |
| `kaufland_list_order_units` | List individual order units — the level Kaufland actually fulfils, cancels and pays out at. | lesen |
| `kaufland_list_units` | List the seller's units (offers) with their EAN, condition, price, stock and the storefront each is listed on. | lesen |
| `kaufland_list_tickets` | List customer service tickets with their subject, status and the order they relate to — where a buyer complaint shows up before it becomes a rating. | lesen |
| `kaufland_list_storefronts` | List the storefronts (one per country) Kaufland runs, with the currency and locale of each. | lesen |
<!-- tools:end -->

## Beispiel-Prompts

- Welche Kaufland-Bestellungen der letzten zwei Tage sind noch nicht versendet?
- Zeig mir Bestellung 123-456 mit allen Bestelleinheiten und der Lieferadresse.
- Welche Kundentickets sind noch offen?
- Welche meiner Angebote auf kaufland.de sind ausverkauft?
- Auf welchen Storefronts bin ich aktiv?

Weitere (auf Englisch) in [examples/prompts.md](../examples/prompts.md).

## Authentifizierung

**Getting credentials**
1. You must already be a Kaufland Marketplace seller. In the **Seller Portal** open **Settings → API keys** and create a key pair.
2. You receive a **Client Key** (public, identifies you) and a **Secret Key** (never sent — it signs the request). Set `KAUFLAND_CLIENT_KEY` and `KAUFLAND_SECRET_KEY`.

**The secret never travels.** Kaufland does not accept a bearer token: every request carries `Shop-Client-Key`, `Shop-Timestamp` and `Shop-Signature`, where the signature is an HMAC-SHA256 over the canonical string

```
<METHOD>\n<FULL URL INCLUDING QUERY>\n<BODY>\n<UNIX TIMESTAMP>
```

keyed with the secret, with no newline after the timestamp. AnythingMCP computes it per request and sends the timestamp it used, so the server can recompute the same string.

**A signature is only valid for a few minutes.** Kaufland rejects a request whose timestamp has drifted, which means a 401 here can be a clock problem rather than a key problem. If every call fails and the keys are definitely right, check the host's time.

**Storefronts are per country.** Kaufland runs a storefront per country (`de`, `cz`, `sk`, `pl`, `at` and more); `kaufland_list_storefronts` returns the current list. Most endpoints take a `storefront` parameter. Omitting it gives you the seller's default, which is rarely what a report about one market wants.

**Orders page with offset and limit**, and `ts_created_from` / `ts_created_to` bound the window — both ISO 8601. Ask for the window you mean; the default is short.

**Shipments** have no list endpoint in the Seller API: for an order's delivery details call `kaufland_get_order` with `embedded=delivery`.

**Order units are where the money is.** An order carries `order_units`, each with its own status, price and fulfilment state. A part-cancelled order is normal, so a revenue figure should sum the units that were actually fulfilled rather than the order total.

**Cloud reachability**: sellerapi.kaufland.com is public with a valid certificate.

## Sicherheit

- **Lesen oder schreiben entscheidest du.** Alle 7 Tools lesen nur. Weise den Connector einem MCP-Server zu, dessen Rolle nur die gewünschten Tools freigibt; die anderen sieht dieser Client gar nicht.
- **Zugangsdaten** werden mit AES-256-GCM verschlüsselt und nie an das Modell gegeben.
- **Response-Mapping** entfernt oder formt Felder pro Tool, bevor sie das Modell erreichen, etwa Bankdaten oder personenbezogene Daten.
- **Audit-Log:** Jeder Aufruf wird mit Eingabe, Ausgabe, Dauer und Status protokolliert, selbst gehostet in deiner eigenen Datenbank.
- **SSO, RBAC und SCIM** sind in der selbst gehosteten Version enthalten.

## FAQ

### Gibt es einen MCP-Server für den Kaufland Marktplatz?
Ja, diesen hier. Er verbindet die Seller-API von Kaufland über AnythingMCP mit Claude, ChatGPT und Copilot: 7 Tools für Bestellungen, Bestelleinheiten, Angebote, Tickets, Storefronts und Lager.

### Was brauche ich für die Verbindung?
Ein Kaufland-Verkäuferkonto und ein API-Schlüsselpaar aus dem Seller Portal (Einstellungen → API-Schlüssel): Client Key und Secret Key.

### Wird der Secret Key übertragen?
Nein. Kaufland signiert jede Anfrage per HMAC; der Secret Key verlässt deine AnythingMCP-Instanz nie.

### Kann die KI Bestellungen oder Preise ändern?
Nein. Alle acht Tools lesen nur.

## Fehlerbehebung

| Problem | Lösung |
|---|---|
| `401` / `403` vom Hersteller | Zugangsdaten falsch oder ohne Rechte. Auf der Connector-Seite neu eintragen; der Import macht einen Testaufruf und zeigt das Ergebnis. |
| Tools fehlen im KI-Client | Der Connector ist nicht dem MCP-Server zugewiesen, den der Client nutzt. **MCP Servers** prüfen, dann `node scripts/smoke.mjs` ausführen. |
| Das System steht im internen Netz | AnythingMCP in diesem Netz selbst hosten und den Hostnamen in `SSRF_ALLOWED_HOSTS` eintragen, sonst blockiert der Outbound-Guard den Aufruf. |
| Lokal ok, in AnythingMCP Cloud nicht | Das System muss aus dem Internet mit gültigem TLS-Zertifikat erreichbar sein. |

## Verwandte Repositories

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [otto-market-mcp-server](https://github.com/kochfreiburg/otto-market-mcp-server): OTTO Market MCP server: connect the otto.de partner API to Claude & ChatGPT. Orders, products, returns, stock and price updates.
- [billbee-mcp-server](https://github.com/kochfreiburg/billbee-mcp-server): Billbee MCP server: connect Billbee order management to Claude & ChatGPT. Orders, products, customers and shipping providers.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): der Open-Source-MCP-Server und -Gateway, auf dem dieses Repository aufbaut.

## Lizenz

AGPL-3.0-only. Die Adapter-Definition in `adapter/` stammt aus AnythingMCP (AGPL-3.0).
