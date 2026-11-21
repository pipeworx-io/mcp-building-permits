# @pipeworx/building-permits

US residential building permits by county, place and state: buildings, housing units and construction value (dollars) for 1-unit, 2-unit, 3-4 unit and 5+ unit structures, monthly, year-to-date and annual, with the publisher's imputation made visible.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2020+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `building_permits_county(county_fips? | state + name?, period_type?, period?, limit?)` - one county, or every county of a state, for one period.
- `building_permits_place(place_id? | state + name, period_type?, period?, limit?)` - one permit-issuing place; a name matching several places returns the candidates instead of guessing.
- `building_permits_state(state?, include_aggregates?, period_type?, period?)` - states (plus US total, 4 regions, 9 divisions on request).
- `building_permits_series(level, id, period_type?, from?, to?, limit?)` - time series for one county, place or state, oldest first.
- `building_permits_movers(level?, period_type?, period?, compare_to?, metric?, structure?, basis?, direction?, state?, min_base?, limit?)` - largest increases or decreases between two periods of the same period type.

## How the data traps are handled

| Trap | What the pack does |
|---|---|
| Monthly, year-to-date and annual are different files | `period_type` is `monthly`, `year_to_date` or `annual`, stored as part of the row key and returned on every row. A call reads exactly one type; nothing sums or compares across types. Year-to-date is the publisher's January-through-month file and includes late reports and corrections to earlier months. |
| County figures include imputed values; they move when late reports arrive | Every county, state and annual/year-to-date place row returns `estimated_including_imputation` and `reported_only` side by side, plus `imputed_units` and `imputed_value_usd`. Monthly place rows carry the publisher's `source_code` (5 = no report, imputed; 9 = no report and no imputation, zeros mean unknown). Every row carries `file_vintage` (date of the publisher file its current values came from) and `revisions` (times the publisher changed the row since first load). |
| Connecticut moved from 8 counties to 9 planning regions | Counties 09001-09015 and planning regions 09110-09190 are different geographies with different FIPS codes. They are stored under their own ids, flagged with `geography_vintage`, and never joined, re-mapped or compared: a series stops where the publisher stopped issuing the id, and movers exclude geographies present in only one period (and count them). |
| Names are not keys | Counties are addressed by 5-digit FIPS, states by 2-digit FIPS, places by `place_id` (state FIPS + the Census 6-digit place id). Name search exists only to find an id and returns candidates when ambiguous. |
| State valuation is in thousands of dollars, county and place in dollars | Normalised to dollars on load. |

Coverage note: before 2022-01 the publisher's monthly and year-to-date county files list only the monthly-reporting counties (about 750 of 3,030) and its northeast place files about 2,400 of 5,500 places. A county or place absent from those periods was not published; that is not zero permits. Annual files cover every jurisdiction.

Place coverage note: a place is a permit-issuing jurisdiction (city, town, township). Some places have no report in a month and no imputation (`source_code` 9); their zeros are unknown, not zero permits.

## Auth

Keyless.

## Data sources

- US Census Bureau, Building Permits Survey (new privately owned residential construction) - county, place and state level files, issued monthly. US federal data, public domain.

The publisher releases the data as bulk text files only. Every response carries `data_as_of` (latest monthly release included) and `refreshed_at`.

Loaded window: monthly and year-to-date from 2020-01 (place year-to-date only the latest 21 months), annual county and state from 2000, annual place from 2010. Earlier years use older file layouts and are not loaded.

## BUDGET

D1 database `pipeworx-building-permits` (binding `BPS_DB`, fleet #4700). Unit prices: Workers Paid, 2026-11 (CLAUDE.md "A copy has a BUDGET"). Row counts measured from the publisher's files on 2026-11-20: county ~3,020 rows per file (~750 before 2022), state 65 (51 states, US total, 4 regions, 9 divisions), place ~20,100 per file (~15,000 before 2022).

| Slice (measured after the load, 2026-11-20) | Periods | Rows written |
|---|---|---|
| County annual | 26 (2000-2025) | 78,738 |
| County monthly | 81 (2020-01..2026-09; ~750 rows/month before 2022, ~3,020 after) | 190,212 |
| County year-to-date | 81 | 190,224 |
| State (annual, monthly, year-to-date; 65 rows each) | 26 + 81 + 81 | 12,262 |
| Place annual | 16 (2010-2025) | 320,987 |
| Place monthly | 81 | 1,339,011 |
| Place year-to-date | 21 (2025-01..2026-09) | 419,817 |
| Geography dimension | | ~23k |
| Partial index (county and state rows only) | | ~0.4M |

- **One-time load:** ~3.0M rows written (2.55M fact + 23k dimension + 0.4M index) x $1.00/M = **~$3.0**, inside the account's 50M rows/month inclusion.
- **Steady state:** storage 247 MB measured x $0.75/GB-mo = **~$0.20/month**. A monthly release is ~52k new rows (county, state and place, monthly and year-to-date) plus ~6k index rows; revisions to the prior three months' files are diffed row by row against stored hashes and only changed rows are written (worst case ~80k). At most ~140k rows/month x $1.00/M = **<= ~$0.15/month**. Reads: indexed point and range reads of tens to a few hundred rows per call at $0.001/M rows, under $0.01/month at any plausible volume.
- **Under both thresholds** ($50 one-time, $25/month): no Needs Bruce item.
- **Refresh:** run `node scripts/building-permits/load-data.mjs` once a month (the publisher posts one release a month). Files whose directory-listing time is unchanged are skipped; changed files are downloaded and diffed per row (`INSERT OR REPLACE` of only new or changed rows, `revisions` incremented). It never drops or reloads a table.
- **Cheaper options rejected:** R2 JSON shards (a place series per object is 40k+ objects at 2.5 paced writes per second, or multi-MB parses per call, and no ranking for movers); Supabase (the billing database, and it reprices the pack into the exclusive class); loading every year of every file (place monthly back to 2000 is ~6M more rows for pre-2020 history); any nightly or DROP-and-reimport refresh.
- **Bindings:** one D1 binding (`BPS_DB`) and zero text bindings: the gateway's 250-slot text-binding limit counts vars and secrets, not D1.

## Notes

- Place monthly files have no reported-only columns, so `reported_only` is null on monthly place rows; use `source_code` instead.
- County, place and state files are separate tabulations: county sums can differ slightly from the state figure and place sums from the county figure.
- Place `population_2000_basis` is the publisher's 2000-census population field and is not updated.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "building-permits": {
      "url": "https://gateway.pipeworx.io/building-permits/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/building-permits/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2020+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/building_permits_county \
  -H 'Content-Type: application/json' \
  -d '{"county_fips":"48201","period_type":"monthly","period":"2026-09"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/building_permits_county`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-building-permits` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/building-permits/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "building-permits": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-building-permits"]
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
ask_pipeworx({ question: "your question about Building Permits data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
