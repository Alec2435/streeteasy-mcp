# streeteasy-mcp

A remote [MCP](https://modelcontextprotocol.io) server that wraps the StreetEasy
GraphQL API so an LLM agent can search and parse NYC rental listings.

It vendors the [`streeteasy-api`](https://github.com/evandcoleman/streeteasy-api)
client (v0.4.0) and exposes it over the **Streamable HTTP** transport, so it can
be deployed as a long-running web service (e.g. on Railway) and connected to by
Claude or any MCP client.

## Tools

| Tool | Description |
| --- | --- |
| `search_rentals` | Search active NYC rentals by area, price, beds, baths, amenities, pets. Returns compact listings + `totalCount`, paginated. |
| `get_rental_details` | Full detail for one listing id: description, amenities, media, pricing history, building info, nearby transit/schools. |
| `list_areas` | Look up StreetEasy area names ↔ numeric codes (optionally filtered by a search term). |
| `list_amenities` | List the valid amenity enum tokens. |

`search_rentals` accepts area **names** (`"MANHATTAN"`, `"Williamsburg"`,
`"upper east side"`) or numeric codes, and validates amenity tokens against the
known set.

## Endpoints

- `POST /mcp` — the MCP Streamable HTTP endpoint (stateless).
- `GET /` and `GET /health` — health checks.

## Run locally

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

## Deploy on Railway

This repo ships a `Dockerfile`. With the Railway CLI:

```bash
railway login
railway init --name streeteasy-mcp
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
