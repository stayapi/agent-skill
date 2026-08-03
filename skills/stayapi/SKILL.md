---
name: stayapi
description: Fetch live hotel, vacation-rental, and restaurant data through StayAPI (stayapi.com) — Booking.com, Airbnb, Google Hotels, Google Travel, Google Reviews, TripAdvisor, Agoda, Trip.com, Accor, Radisson, and OpenTable. Use whenever the user wants real hotel prices, availability, rooms, photos, guest reviews, review backfills or monitoring, destination or property ID lookups, or restaurant search and reservation data — even if they never mention StayAPI by name. Works through the StayAPI MCP server or plain REST with an API key.
metadata:
  version: 0.1.0
---

# StayAPI — Live Hotel & Travel Data

StayAPI is a hosted API that returns live, structured data from the major travel platforms: Booking.com, Airbnb, Google Hotels / Google Travel / Google Reviews, TripAdvisor, Agoda, Trip.com, Accor (ALL), Radisson, and OpenTable. One successful data call costs one quota unit, on any platform, over either transport.

Canonical links:

- Dashboard & signup: https://stayapi.com (the dashboard shows the API key)
- Documentation hub: https://stayapi.com/docs
- MCP setup guide: https://stayapi.com/docs/mcp
- REST base URL: `https://api.stayapi.com` · MCP endpoint: `https://api.stayapi.com/mcp`
- Support: info@stayapi.com

## Authentication — handle the key carefully

Every call needs a StayAPI API key, sent as the `X-API-Key` header. Resolve it in this order:

1. **StayAPI MCP tools already connected** (tool names like `stayapi_*` or `mcp__stayapi__*`): just call them — the connector already carries the key, so you never touch it.
2. **`STAYAPI_API_KEY` environment variable**: use it by reference (`$STAYAPI_API_KEY`), never by value.
3. **Neither present**: ask the user for their key, or point them to https://stayapi.com to create an account and copy it from the dashboard.

The key is a paid credential tied to the user's quota and billing. Never print it, echo it, log it, commit it to a repository, or paste it into code, config files, or chat output. In shell commands always interpolate the variable; if the user pastes a key literal, put it in an environment variable first and use the variable from then on.

## Two transports

**MCP (preferred when available).** The server at `https://api.stayapi.com/mcp` exposes every capability as a tool. If StayAPI tools are visible, call them directly — no HTTP assembly, no key handling. To connect Claude Code:

```bash
claude mcp add --scope user --transport http stayapi https://api.stayapi.com/mcp --header "X-API-Key: YOUR_API_KEY"
```

(`--header` must come after the name and URL; a new session is needed to pick the server up.) Cursor and Claude Desktop snippets: see [references/tools.md](references/tools.md#connecting-an-mcp-client) or https://stayapi.com/docs/mcp.

**REST.** Same data over plain HTTPS — use it when no MCP client is available or when scripting:

```bash
curl -sS -H "X-API-Key: $STAYAPI_API_KEY" \
  "https://api.stayapi.com/v1/booking/destinations/lookup?query=Phuket"
```

## The core pattern: resolve an ID, then fetch

Most tools take a platform-native ID, and there is a resolver for whatever the user actually has. Resolve once, then reuse the ID — IDs are stable, and some URL resolutions are slow or cost a quota unit of their own.

| You have | Resolve with | You get |
|---|---|---|
| City / area name (Booking) | `booking_lookup_destination` | signed `dest_id` for `booking_search_hotels` |
| Booking hotel URL | `GET /v1/booking/hotel/url-to-id` (or pass `url` straight to details/rooms/reviews tools) | numeric `hotel_id` — slow (5–40 s, real browser); cache it |
| Airbnb URL | pass `url` directly — resolution is instant (regex) | `listing_id` |
| Hotel name (Google Travel) | `google_travel_resolve` or `google_travel_search` | `entity_token` for all other `google_travel_*` tools |
| Place name (Google Reviews) | `google_reviews_for_place` with `query` | first page + `data_id` to paginate with |
| Place name → coordinates | `meta_coordinates_lookup` | `latitude`/`longitude` for OpenTable and Accor search |
| TripAdvisor place name / URL | `tripadvisor_geo_search` / pass URL directly | `geo_id` / `location_id` |
| Agoda, Radisson, OpenTable URL | the platform's dedicated `*_url_to_id` tool | numeric ID for that platform's other tools |

## Core workflows

The six flows below cover most requests; [references/workflows.md](references/workflows.md) has complete worked examples of each (MCP call sequence + copy-paste REST curl):

1. **Destination lookup → hotel search** (Booking): free-text city → `dest_id` → hotel list with prices and `hotel_id`s.
2. **Hotel details & photos** (Booking): `hotel_id` → compact details / full signed photo gallery.
3. **Guest reviews & date-range backfill** (Booking): paginated reviews; `sort=recent_desc` + paginate-until-cutoff for "all reviews since X".
4. **Google Hotels search**: location + dates → priced availability list.
5. **Google reviews for a place**: name or `data_id` → Google reviews with pagination.
6. **Airbnb listing details & reviews**: any Airbnb URL or `listing_id` → details (dates required) and reviews.

Everything beyond these — TripAdvisor, Google Travel deep-dives, Accor, Radisson, OpenTable, Agoda, Trip.com, room-level rates, price calendars, private dining — is cataloged in [references/tools.md](references/tools.md): all 55 MCP tools with their REST equivalents and per-platform gotchas. Read it whenever a request goes beyond the six core flows.

## Errors and backoff

REST errors are RFC 7807 `application/problem+json`; MCP tools return structured error dicts (`invalid_input`, `url_resolution_failed`, `upstream_error`, `no_results`, `rate_limited`, `quota_exhausted`, `feature_unavailable`). Two rules matter:

- **A 2xx status is the only success signal.** Never parse an error body as data.
- **Respect `retry_after`.** `rate_limited` means back off briefly and retry once or twice. `quota_exhausted` includes `retry_after` and `reset_at` — the monthly quota is gone, so stop calling and tell the user when it resets (or suggest upgrading); retrying in a loop only burns time.

Transient upstream errors (`upstream_error`, HTTP 502/503) are usually worth one retry after a few seconds — these are live scrapes of third-party sites, not a static database.

## Gotchas that bite agents

- **Booking `dest_id` is signed** — city IDs are negative (Phuket is `-3233180`). Stripping the minus silently returns 0 hotels with `success: true`. Preserve the sign end-to-end.
- **Booking hotel URLs must be canonical**: `https://www.booking.com/hotel/{cc}/{slug}.html`. Search-result URLs are rejected with a 400.
- **Booking destination lookup**: query the bare city name ("Savannah", not "Savannah, GA") — appending a region biases results to HOTEL-type matches. A HOTEL-type `dest_id` is that hotel's `hotel_id`.
- **Prefer IDs over URLs on Booking** — URL resolution drives a real browser (5–40 s). Airbnb URL parsing is instant; Agoda URL resolution costs one quota unit (it fetches the page).
- **Booking search prices are stay totals, not nightly rates**, and hotel rows nest at `data.hotels` (the envelope's top-level `hotel_id` is null on search). Results come in Booking's "recommended" ranking — there is no sort parameter, so "top by price/rating" means over-fetch and sort client-side.
- **Sort parameter names differ per platform** — Booking `sort=recent_desc`, Airbnb `sort_by=MOST_RECENT`, Google reviews `sort_by=newest`, Agoda `sorting=7`. Check the workflow/catalog reference before assuming; the default is usually relevance, which silently fails "most recent" requests.
- **Airbnb details require `check_in` and `check_out`**; pick near-future dates if the user doesn't care.
- **Page-size caps**: Booking reviews 25/page · Airbnb reviews 50/page · Agoda 20/page · OpenTable 25/page · Accor reviews capped at 20 total (upstream limit).
- **`airbnb_search_listings` is a stub** — Airbnb search by location isn't built yet; the tool returns `feature_unavailable` (not billed). Offer Booking, Google Hotels, or TripAdvisor search instead.
- **Trip.com reviews are slow by design** (a real browser defeats bot detection) — expect several seconds per page and keep `page_size` generous to minimize calls.
