# @pipeworx/meteoswiss

Swiss weather and federal geodata from MeteoSwiss and swisstopo on
data.geo.admin.ch — find SwissMetNet automatic weather stations, radar, pollen,
ICON forecasts and climate normals, plus elevation and topographic layers, and
get the direct CSV, GeoTIFF and NetCDF download URLs for each.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `meteoswiss_collections(contains?, scope?, prefix?, limit?)` — MeteoSwiss
  datasets by default (`ch.meteoschweiz.*`); `scope: "all_swiss"` opens the whole
  513-collection federal catalogue (swisstopo, BAFU, Agroscope).
- `meteoswiss_collection(collection_id)` — one collection in full. The
  SwissMetNet description is where the resolution and recency codes are defined.
- `meteoswiss_search(collections?, bbox?, datetime?, ids?, limit?, max_assets?)`
  — which stations or tiles cover an area, with their coordinates, human names
  and direct download URLs.

## Auth

Keyless, metadata *and* assets. Everything is a public HTTPS download on
data.geo.admin.ch.

## Data sources

- <https://data.geo.admin.ch/api/stac/v1> — STAC API 1.0, the Swiss
  Confederation's open geodata catalogue. MeteoSwiss OGD is the
  `ch.meteoschweiz.` prefix.

Shared client: `shared/src/stac.ts`, also used by `planetary-computer`,
`earth-search` and `copernicus-dataspace`.

## Traps

- **An item's `datetime` is when the file was published, not when the weather
  happened**, and this produces a wrong answer rather than an error. One item is
  one *station*, not one observation: station `abo` (Adelboden) carries
  `datetime: 2026-09-17T…` while its assets cover 1980 to today, and the
  collection's temporal extent is a few milliseconds wide. "No station data
  before 2026" is a misreading of a correct payload. The measurements are inside
  the CSV assets.
- **The asset filename encodes resolution and recency.** In
  `ogd-smn_<station>_<res>_<age>`: res is `t` = 10-minute, `h` = hourly,
  `d` = daily, `m` = monthly, `y` = yearly; age is `now` (since midnight),
  `recent` (this year to yesterday) or `historical_YYYY-YYYY` (ten-year blocks).
  MeteoSwiss asks that you take the coarsest resolution you need rather than
  aggregating 10-minute files.
- **Station ids are three-letter abbreviations** (`abo`, `ber`, `sma`), not place
  names. The human name is each item's `title` ("Bern / Zollikofen (BER)").
- A default-scope `meteoswiss_search` with no `collections` searches the whole
  federal catalogue, so it mixes weather stations with topographic tiles.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "meteoswiss": {
      "url": "https://gateway.pipeworx.io/meteoswiss/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/meteoswiss/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/meteoswiss_collections \
  -H 'Content-Type: application/json' \
  -d '{"contains":"station","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/meteoswiss_collections`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-meteoswiss` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/meteoswiss/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "meteoswiss": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-meteoswiss"]
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
ask_pipeworx({ question: "your question about Meteoswiss data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
