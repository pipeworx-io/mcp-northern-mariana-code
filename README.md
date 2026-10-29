# @pipeworx/northern-mariana-code

Commonwealth Code of the Northern Mariana Islands (CNMI statutes) — section
text by citation (e.g. `"6 CMC § 1101"`), and topic/keyword search over
official catchlines. Fleet #2892, one of six jurisdictions (DC + five
territories) with no code pack as of 2026-10-09.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nmi_statute(citation)` — text of one Commonwealth Code section by
  citation (e.g. `"6 CMC 1101"`, murder; `"1 CMC 101"`, defines "Northern
  Mariana Islands"), with the official catchline and title context. Tries
  the official site live first, falls back to an Internet Archive Wayback
  Machine capture of the SAME official PDF when the live site is
  unreachable (it has been, since around 2026-10 — see "The current outage"
  below). Every response carries `route` (`"live"` or `"wayback"`),
  `data_as_of`, and a `staleness_warning` on the fallback route.
- `nmi_search(query, limit?)` — topic/keyword search over official section
  catchlines across all 10 titles (e.g. `"murder"`, `"tax"`). Not full-text
  search — see "Known gaps".

## Auth

Keyless.

## Data sources

- `https://cnmilaw.org/pdf/cmc_section/T{title}/{section}.pdf` — the CNMI
  Law Revision Commission's own per-section PDF (e.g. Title 6 § 1101 is
  `.../T6/1101.pdf`). `nmi_statute` fetches this live, per request, with an
  8s timeout, and extracts the matching section's text.
- `https://web.archive.org/web/{timestamp}if_/{that same PDF URL}` — the
  Wayback Machine's own crawl of that identical official PDF, used only
  when the live fetch above fails. One request resolves to the nearest
  actual capture via Wayback's own redirect (`/web/{today}if_/{url}` 302s
  to the closest real snapshot) — no separate CDX lookup per call; the CDX API
  rate-limits aggressively under repeated querying (measured directly while
  building this pack: the same query that succeeds once starts 503ing
  within a handful of calls).
- `mcps/northern-mariana-code/scripts/bake-index.mjs` fetches the title
  listing page (`cnmilaw.org/cmc.php`, or its most recent readable Wayback
  capture) once, offline, and bakes a citation+catchline index into
  `src/nmi-index-data.ts` for `nmi_search` and for `nmi_statute`'s
  "citation exists but is unreachable" vs. "no such citation" distinction —
  the full set of ~7,100 sections is too large to scan per search query.

## The current outage — read this before assuming the live route is dead

**cnmilaw.org is unreachable right now (verified live 2026-10-09), but this
looks temporary, not a shutdown.** Checked three ways:

1. A plain residential HTTPS request times out / gets no response.
2. A `wrangler dev --remote` Worker edge probe — i.e. not a geo-block — gets
   a Cloudflare `522` ("origin unreachable") on both `cnmilaw.org/` and
   `cnmilaw.org/pdf/cmc_section/T6/1101.pdf`.
3. The bare-HTTP vhost (no TLS) resolves to a different backend entirely: a
   Network Solutions registrar page reading *"This domain has been
   temporarily disabled. If you own this domain, please log in to your
   account manager to re-enable the domain."* — a registrar-level
   suspension (lapsed renewal, most likely), not a bot wall or a CAPTCHA.

The Wayback Machine's own crawler was still getting live `200` PDFs from
this exact path as recently as **2026-02-17** (confirmed via its CDX index),
so this is a recent lapse, not a long-dead site. **`nmi_statute` tries
cnmilaw.org live first on every call** (8s timeout) — if the registration
is restored, calls start routing `"route": "live"` again with no code
change needed. There was no need to build a baked copy of the statutory
text itself (unlike `arkansas-code`'s 2012 PRO scan): the Wayback Machine
already holds its own crawl of the identical official PDFs, so falling back
to a live per-request fetch of that memento is still "calling", not
"copying", under root `CLAUDE.md`'s proxy-vs-mirror rule.

## Capabilities

- **Citation lookup** — `nmi_statute`, live-first with a Wayback fallback.
- **Topic/keyword search** — `nmi_search`, over catchlines only.
- **Amendments/enactment history** — embedded in the statutory text itself
  (a `Source: PL ...` line at the end of most sections) but not parsed into
  a separate field — CNMI PDFs do not carry a structured history node the
  way Arkansas's XML scan does.
- **Historical version** (text as it read before a given amendment) — NOT
  AVAILABLE from any free source found.

## Parsing note

Section PDFs are extracted with the shared `extractPdfText()` helper
(`shared/src/pdf-text.ts`) — the same one `kentucky-code` and
`north-dakota-code` use. The catchline index parses `cmc.php`'s own anchor
text (an anchor tag linking to `.../T6/1101.pdf` with text "§ 1101. Murder."), stripping the
leading section number the `href` already carries.

**Refresh path:** `node scripts/bake-index.mjs > src/nmi-index-data.ts` —
tries cnmilaw.org live first, falls back to the most recent readable
Wayback capture of `cmc.php` automatically. Re-run occasionally to pick up
newly-enacted sections; `nmi_statute` itself never depends on this file
being current, since it always fetches statutory text live or from Wayback
per call.

## Known gaps

- `nmi_search` matches official catchlines only, not full statutory text —
  a topic discussed inside a section's body but absent from its catchline
  will not be found.
- While cnmilaw.org is down, every `nmi_statute` call that succeeds does so
  via Wayback and carries whatever date that section happened to be last
  crawled — individual sections' snapshot dates vary (seen during testing:
  2024-08-13 for one section, 2026-02-17 for another), not a single fixed
  cutoff the way `arkansas-code`'s 2012 scan is. Always check `data_as_of`
  per response rather than assuming a pack-wide date.
- A citation whose title/section both parse but that is in neither the live
  site nor Wayback AND not in the baked index is reported as
  `citation_not_found`; one that IS in the baked index but unreachable on
  both routes is reported as a loud `upstream_down` error, never a false
  `found: false`.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "northern-mariana-code": {
      "url": "https://gateway.pipeworx.io/northern-mariana-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/northern-mariana-code/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/nmi_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"6 CMC 1101"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nmi_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-northern-mariana-code` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/northern-mariana-code/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "northern-mariana-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-northern-mariana-code"]
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
ask_pipeworx({ question: "your question about Northern Mariana Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
