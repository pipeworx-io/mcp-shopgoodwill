# ShopGoodwill Auctions

Search active auctions, closed auctions with realized sale prices, and per-item
detail from [shopgoodwill.com](https://shopgoodwill.com) — the online auction
site run by Goodwill member organizations (170+ regional sellers listing
cameras, jewelry, watches, art, electronics, instruments and collectibles).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1669+ live data sources.

The closed-auction search is the comps value: it returns what lots actually
sold for (final hammer price and bid count), reaching back roughly 90 days.

## Tools

| Tool | What it returns |
|------|-----------------|
| `shopgoodwill_search` | Active listings for a keyword query: current bid, bid count, buy-now price, category, end time, listing URL. Optional price bounds, category id, buy-now-only filter, pagination (max 40/page). |
| `shopgoodwill_sold_search` | Closed auctions for a keyword query with the final price at close. Rows with `num_bids > 0` sold at `final_price`; zero-bid rows did not sell. `days_back` up to the archive's ~90-day reach. |
| `shopgoodwill_item` | Full detail for one lot by `item_id`: description, current/minimum bid, bid history (bidder names masked by the site), seller Goodwill organization, shipping/handling, images, end time. |

Timestamps carry no timezone marker and are US Pacific (site convention).

## Auth

None. The search and item-detail endpoints answer without credentials.

## Data sources

- ShopGoodwill buyer API: `https://buyerapi.shopgoodwill.com/api/Search/ItemListing`
  (search, active and closed) and
  `https://buyerapi.shopgoodwill.com/api/itemDetail/GetItemDetailModelByItemId/{id}`
  (item detail) — the same JSON backend the shopgoodwill.com web app uses.
- Request shapes documented by the community client
  [scottmconway/shopgoodwill-scripts](https://github.com/scottmconway/shopgoodwill-scripts).

### Upstream quirks (measured 2026-09-05)

- A wrong or incomplete search body returns **zero rows with a clean 200** —
  `lowPrice`/`highPrice` must be `"0"`/`"999999"`-style strings, never empty.
- `pageSize` is silently **ignored** — every page is exactly 40 rows (ask 5 or
  100, get 40, no error); pagination is via `page`. The tools' `per_page` is
  applied client-side.
- The closed-auction archive reaches back **~90 days** regardless of the
  `closedAuctionDaysBack` value requested.
- A nonexistent item id returns a 404 with an empty body.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "shopgoodwill": {
      "url": "https://gateway.pipeworx.io/shopgoodwill/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/shopgoodwill/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1669+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/shopgoodwill_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"nikon","per_page":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/shopgoodwill_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "shopgoodwill": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-shopgoodwill"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-shopgoodwill
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Shopgoodwill data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
