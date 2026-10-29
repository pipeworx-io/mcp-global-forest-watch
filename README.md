# @pipeworx/global-forest-watch

The Global Forest Watch Data API (WRI / UMD) — the catalogue and query surface
behind GFW: annual tree cover loss and gain, primary forest extent, GLAD and
RADD deforestation alerts, VIIRS/MODIS fire alerts, carbon flux, protected
areas, concessions and Aqueduct water risk.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `gfw_datasets(query?, limit?, page?)` — list or keyword-search the catalogue
  (382 datasets as of 2026-09). With a `query` the whole catalogue is walked,
  capped at 5 upstream pages. Keyless.
- `gfw_dataset(dataset, version?)` — one dataset's metadata (source, citation,
  resolution, coverage, update frequency, cautions, licence) and version list;
  with a version, also that version's status, assets and field/band list.
  Keyless.
- `gfw_query(dataset, sql, version?, limit?, _apiKey)` — read-only SELECT over
  one dataset version. Needs a caller key.

## Auth

Catalogue browsing is keyless and verified so (2026-09-17). The `/query`
endpoint requires an `x-api-key` and answers
`403 "Request is missing valid API key"` without one.

No platform key is provisioned. The intended env var when one lands is
`PLATFORM_GFW_KEY`; sign up at <https://globalforestwatch.org> and mint a key
with `POST /auth/token` then `POST /auth/apikey` on the data API.

`platformKeyEnv` is deliberately NOT declared while the key is unset, for the
same reason as copernicus-cds — see that README's Auth section. Add it in the
same change that lands the secret.

## Data sources

- <https://data-api.globalforestwatch.org/datasets> — catalogue. Paging is
  `page[number]` / `page[size]` (JSON:API style, square brackets); the response
  carries `meta.total_items` and `meta.total_pages`.
- <https://data-api.globalforestwatch.org/dataset/{d}> — dataset metadata.
- <https://data-api.globalforestwatch.org/dataset/{d}/{version}> — version
  metadata. `latest` is a valid version alias, but it **307-redirects** to the
  concrete version, so the fetch must follow redirects. A bare `curl` without
  `-L` sees a 307 with an empty body, which reads as an outage.
- <https://data-api.globalforestwatch.org/dataset/{d}/{version}/fields> —
  columns/bands. Raster datasets report the column name under `pixel_meaning`
  rather than `name`.
- <https://data-api.globalforestwatch.org/dataset/{d}/{version}/query> — SQL.
  The table is always called `data`.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "global-forest-watch": {
      "url": "https://gateway.pipeworx.io/global-forest-watch/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/global-forest-watch/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/gfw_datasets \
  -H 'Content-Type: application/json' \
  -d '{"query":"tree cover loss","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gfw_datasets`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-global-forest-watch` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/global-forest-watch/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "global-forest-watch": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-global-forest-watch"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Global Forest Watch data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
