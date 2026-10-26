# @pipeworx/emarket-storage

Italian regulated company disclosures direct from eMarket Storage (Teleborsa)
— a document index (title, issuer, timestamp, PDF link), updated same-day,
built to fix the months-long lag in the community ESEF index for current-year
Italian filings.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1879+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `emarket_search_disclosures(company?, azienda_id?, category?, market?, cerca?, date_from?, date_to?, limit?, offset?)`
  — search disclosures by issuer (name, resolved against eMarket's own
  issuer list, or a known azienda_id), EU Transparency Directive category,
  market segment, and/or a publication date window. Returns newest first.
- `emarket_get_disclosure(id, date)` — fetch one disclosure by its Teleborsa
  protocollo id and publication date (both come from a prior search — this
  source has no by-id-only lookup).
- `emarket_search_documents(company?, azienda_id?, category?, date_from?, date_to?, limit?, offset?)`
  — search eMarket Storage's SEPARATE document archive (annual/half-year
  financial reports, ESEF/XBRL ZIP packages), not the press-release stream
  `emarket_search_disclosures` returns. Use this when you need the actual
  filing. `has_xbrl` flags rows carrying a downloadable ESEF/XBRL ZIP.

**All three search parameters above (company/azienda_id on every tool, plus
category/market) are optional.** Omit the issuer entirely to get every
covered issuer's rows for a date window in ONE call — the cheap way to do a
daily check across many issuers (see "One call per day" below).

## Why this pack exists

`esef-filings` proxies `filings.xbrl.org`, a community index whose
current-year coverage for Italian issuers runs months behind (the same
problem `amf-filings` fixes for France). This pack goes to eMarket Storage,
one of Italy's two Officially Appointed Mechanisms (OAMs) for Transparency
Directive disclosures, directly. **Measured live 2026-09-23**: a
`category="104"` (Informazioni privilegiate) search for 2026-09-22..09-24
returned a WEBUILD disclosure published 2026-09-23T17:13 local time (same
day as the query), with its PDF confirmed reachable (HTTP 200).

## One call per day, not one call per issuer

A design partner running a daily check across ~155 issuers was calling once
per issuer and burning through a 50-credit-per-call allowance in ~35 calls
(fleet #3300, 2026-10). **That is unnecessary — every search tool here
already supports a date-window query with NO issuer filter at all**, which
returns every covered issuer's rows for that window in one call:

```
emarket_search_disclosures({date_from: "2026-10-19", date_to: "2026-10-19"})
emarket_search_documents({date_from: "2026-10-19", date_to: "2026-10-19"})
```

Verified live 2026-10-19 — both return every eMarket-covered issuer's rows
for the day, paginating (page_size 24) only if that single day's market-wide
volume exceeds one page, which is rare. A 155-issuer daily check is one (or a
small, bounded few) calls per pack per day, not 155.

## ⚠️ Five things a caller must know

1. **eMarket-covered issuers only — this is not the full Italian market.**
   Italy has a SECOND, separate OAM, **1INFO** (a different keyless JSON API
   covering roughly 314 issuers — see `@pipeworx/oneinfo-storage`). An issuer
   discloses through exactly one of the two. Some well-known names are 1INFO
   issuers and will NOT be found here — confirmed 2026-09-23 by checking
   eMarket Storage's own ~480-company issuer picklist: neither **IREN** nor
   **SOMEC** appear in it. A company search that matches nothing returns an
   explanatory `note`, not a bare error, pointing at `oneinfo-storage`'s
   `oneinfo_search_disclosures` instead. This pack does not
   merge 1INFO data.
2. **This is a document index, not an XBRL-facts API.** Every row is a
   disclosure EVENT: a title, an issuer, a publication timestamp, and a link
   to the underlying PDF. There is no per-fact extraction — no "give me net
   income for FY2025" lookup.
3. **No ISIN or LEI.** Unlike `amf-filings`, eMarket Storage's listing and
   its issuer picklist expose only a company NAME and an internal Teleborsa
   numeric id (`azienda_id`) — no ISIN/LEI field was found anywhere on the
   listing or a per-company page (checked 2026-09-23). Identify issuers by
   name (resolved automatically against the site's own list) or by
   `azienda_id` directly once you have it from a prior search.
4. **`date_to` is INCLUSIVE in this tool — the underlying source's own
   `data_to` is EXCLUSIVE, and that mismatch silently drops rows, not just on
   an exact single-day query.** Verified live 2026-09-24 against the raw
   source: `data_from=2026-09-21&data_to=2026-09-22` returned 13 rows, ALL
   from the 21st — the entire 22nd silently vanished, with no error, no
   empty-result signal, nothing to indicate the caller's "21st through 22nd"
   request was answered wrong. `emarket_search_disclosures` always sends the
   source a `data_to` one calendar day past the caller's inclusive `date_to`
   (a lone `date_from` with no `date_to` means "just that one day"), and the
   response echoes both `date_to` (your inclusive value) and
   `upstream_date_to_exclusive` (the raw bound actually sent) — so a caller
   asking for "the 21st through the 22nd" now correctly gets rows from both
   days. Confirmed locally: the same 21→22 request now returns rows dated
   both 2026-09-21 and 2026-09-22.
5. **Two separate archives on the same site.** `emarket_search_disclosures`
   is the press-release stream; `emarket_search_documents` is the actual
   regulated-filing archive (annual reports, ESEF/XBRL ZIPs). The same
   `category` code on each does NOT return the same rows — verified live
   2026-10-19: Interpump's `category="100"` on `emarket_search_disclosures`
   returns only a results press release, while the same filter on
   `emarket_search_documents` returns "IP - Annual Financial Report 2025"
   plus its ESEF XBRL ZIP. Use the right tool for what you actually need.

## Auth

Keyless. No account, no API key.

## Data sources

- <https://www.emarketstorage.it/it/comunicati-finanziari> — eMarket
  Storage's public disclosure listing (Teleborsa, an EU Transparency
  Directive OAM for Italy). Pipeworx queries it live on every call; there is
  no bulk download or formal API (`/jsonapi` on this Drupal site 404s — it
  is HTML-only), so this pack parses the listing's HTML directly.

## Gotchas

- No formal API. The listing is a plain GET-filterable Drupal View
  (`azienda`, `mercato`, `categoria`, `data_from`/`data_to`, `cerca`). This
  pack parses the rendered HTML with targeted regex — Cloudflare Workers
  have no DOMParser (same approach as `spain-tenders`).
- **The row's `datetime` attribute (on its `time` element) is broken** —
  every row observed carries something like `34Z` (just trailing seconds
  and a "Z", an upstream templating bug), not a usable timestamp. This pack
  parses the human-readable text instead ("DD/MM/YYYY - HH:MM", Italian
  local time) and returns it as `published_at_local` with an explicit
  `timezone: "Europe/Rome"` field — it is NOT converted to UTC, since doing
  that correctly needs DST awareness this pack does not assume.
- **The listing does not expose a category per row** — only the `categoria`
  filter parameter accepts one. `emarket_search_disclosures` therefore
  echoes the requested `category` onto every result when the caller filtered
  by it (that is a true fact about a filtered result, not a guess); on an
  unfiltered search, `category` is honestly `null` rather than fabricated.
- **Fixed page size of 24**, confirmed live (every page but the last was
  exactly 24 rows). There is no total-count field anywhere in the response;
  when a next page exists, this pack reports `has_more: true` and
  `at_least_total` — a true LOWER BOUND (`last_page_index * 24`), never a
  fabricated exact total. Example measured 2026-09-23: SAIPEM narrowed to a
  2-day window returned 1 disclosure; the same issuer over 2025-01-01..
  2026-09-24 returned `at_least_total: 120` (122 exact, confirmed by walking
  to the last page) — the date filter genuinely binds.
- Company-name resolution uses `rankMatches` (exact > prefix > whole-word >
  substring) against eMarket's own issuer picklist, embedded in every
  listing page load. An ambiguous match (multiple issuers tied at the same
  score) returns `candidates` instead of guessing.
- `emarket_get_disclosure` has no direct by-id endpoint on the source — it
  re-fetches the given date's listing (paging up to 8 pages, i.e. up to 192
  disclosures market-wide for that day — measured live, the busiest single
  day found was 3 pages deep) and matches on the protocollo id. Each page
  fetch gets a 10s timeout rather than the shared 25s default, bounding the
  worst case to 80s sequential rather than 375s — a design partner reported
  ~230s stalls in production (fleet #3300) that line up with the old numbers.
- `emarket_search_documents` reuses the same HTML row parser as
  `emarket_search_disclosures`; its ESEF/XBRL rows carry a
  `<span class="icon-esef">` marker before the title text and link to a
  `/sites/default/files/xbrl/….zip` path instead of the usual
  `/comunicati/….pdf` — both are handled, `has_xbrl` reflects which.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "emarket-storage": {
      "url": "https://gateway.pipeworx.io/emarket-storage/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/emarket-storage/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1879+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/emarket_search_disclosures \
  -H 'Content-Type: application/json' \
  -d '{"category":"104","date_from":"2026-09-22","date_to":"2026-09-24"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/emarket_search_disclosures`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "emarket-storage": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-emarket-storage"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-emarket-storage
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Emarket Storage data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
