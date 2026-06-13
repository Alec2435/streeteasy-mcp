# streeteasy-mcp

A remote [MCP](https://modelcontextprotocol.io) server that wraps the StreetEasy
GraphQL API so an LLM agent can search and parse NYC rental listings.

It vendors the [`streeteasy-api`](https://github.com/evandcoleman/streeteasy-api)
client (v0.4.0) and exposes it over either **stdio** (local) or **Streamable
HTTP** (remote) transport, so it can be connected to by Claude or any MCP client.

> [!IMPORTANT]
> **StreetEasy blocks datacenter/cloud IPs.** Its API sits behind PerimeterX
> bot-detection that `403`s cloud/datacenter IPs (AWS, GCP, Railway, etc.). The
> HTTP build deploys fine and the MCP layer works, but the upstream
> `search_rentals` / `get_rental_details` calls fail from a datacenter unless you
> do one of:
> - **Run the stdio server locally** from a normal home connection — see
>   [Run as a local MCP server](#run-as-a-local-mcp-server-recommended), or
> - **Route upstream calls through a residential proxy** by setting
>   `STREETEASY_PROXY` — this is what lets the hosted HTTP build (Railway, etc.)
>   reach the API. See [Proxy / bot-detection](#proxy--bot-detection).

## Tools

| Tool | Description |
| --- | --- |
| `search_rentals` | Search active NYC rentals by area, price, beds, baths, amenities, pets. Returns compact listings + `totalCount`, paginated. Each listing includes `leadPhotoUrl` / `photoUrls` and a listing `url`. |
| `get_rental_details` | Full detail for one listing id: description, amenities, pricing history, building info, nearby transit/schools, and resolved media — `media.photoUrls`, `media.floorPlanUrls`, `media.videoLinks` (YouTube/Vimeo), `media.tour3dUrl`. |
| `list_areas` | Look up StreetEasy area names ↔ numeric codes (optionally filtered by a search term). |
| `list_amenities` | List the valid amenity enum tokens. |

`search_rentals` accepts area **names** (`"MANHATTAN"`, `"Williamsburg"`,
`"upper east side"`) or numeric codes, and validates amenity tokens against the
known set.

### Media

Photos resolve to Zillow's CDN (`photos.zillowstatic.com/fp/{key}-se_large_800_400.jpg`),
videos to their provider watch URL (YouTube/Vimeo) plus a thumbnail, and 3D
tours to a direct `tour3dUrl`. All are public — no auth required.

### Not included: contact info & inquiries

Listing agent contact details and "request a tour" inquiries are **not** exposed.
They live behind StreetEasy's contact flow, which is protected by PerimeterX
bot-detection (a "Press & Hold" human check). Automating it would mean evading
bot-detection, so it's intentionally left out — the right pattern is to surface
the listing `url` and let a human submit the tour request in their browser.

## Endpoints

- `POST /mcp` — the MCP Streamable HTTP endpoint (stateless).
- `GET /` and `GET /health` — health checks.

## Run as a local MCP server (recommended)

Runs over stdio from your machine's residential IP — the configuration that
actually reaches StreetEasy.

```bash
npm install
npm run build
# register with Claude Code (uses the stdio entry point):
claude mcp add streeteasy -- node "$(pwd)/dist/stdio.js"
```

Then ask Claude to search rentals. To run the stdio server by hand:

```bash
npm run start:stdio
```

## Run as an HTTP server

```bash
npm install
npm run build
npm start            # listens on $PORT (default 3000), POST /mcp
```

Test it with the MCP SDK client (see `test-client.mjs`):

```bash
MCP_URL=http://localhost:3000/mcp node test-client.mjs
```

## Configuration

| Env var | Purpose |
| --- | --- |
| `PORT` | Port to listen on. Railway sets this automatically. |
| `MCP_AUTH_TOKEN` | Optional. If set, `POST /mcp` requires `Authorization: Bearer <token>`. |
| `STREETEASY_PROXY` | Optional. HTTP/HTTPS proxy for all upstream StreetEasy calls, e.g. `http://user:pass@host:port`. **Required for cloud/datacenter deploys** — use a residential proxy. `HTTPS_PROXY` / `ALL_PROXY` are also honored. |

### Proxy / bot-detection

StreetEasy `403`s datacenter IPs, so any cloud host (Railway included) must send
upstream requests through a **residential proxy**. Set `STREETEASY_PROXY` to a
proxy URL (credentials may be embedded, e.g.
`http://user:pass@host:port`) and all StreetEasy GraphQL traffic is tunnelled
through it. On startup the server logs the proxy in use with credentials
redacted (`Outbound proxy: http://***:***@host:port`).

Rotating residential proxies hand out a fresh exit IP per connection, and a
clean IP isn't guaranteed every time, so the client automatically **retries a
`403` bot-challenge** (up to 3 times when a proxy is set) to land on a good IP.
A local stdio server on a residential connection doesn't need a proxy.

## Deploy on Railway

This repo ships a `Dockerfile`. With the Railway CLI:

```bash
railway login
railway init --name streeteasy-mcp
# Cloud hosts are datacenter IPs — set a residential proxy so calls aren't 403'd:
railway variables --set "STREETEASY_PROXY=http://user:pass@host:port"
railway up
railway domain          # generate a public URL
# optional: railway variables --set MCP_AUTH_TOKEN=<token>
```

## Connect from Claude Code

```bash
claude mcp add --transport http streeteasy https://<your-app>.up.railway.app/mcp
# if you set a token:
claude mcp add --transport http streeteasy https://<your-app>.up.railway.app/mcp \
  --header "Authorization: Bearer <token>"
```
