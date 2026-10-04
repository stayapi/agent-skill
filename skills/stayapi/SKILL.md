---
name: stayapi
description: Fetch live hotel, vacation-rental, and restaurant data through StayAPI (stayapi.com) — Booking.com, Airbnb, Expedia, Vrbo, Google Hotels, Google Travel, Google Reviews, TripAdvisor, Agoda, Trip.com, HolidayCheck, Accor, Radisson, Marriott Bonvoy, Hilton, WeHotel (Jin Jiang), MakeMyTrip India, Otelpuan, and OpenTable. Use whenever the user wants real hotel prices, availability, rooms, photos, guest reviews, review backfills or monitoring, destination or property ID lookups, or restaurant search and reservation data — even if they never mention StayAPI by name. Works through the StayAPI MCP server or plain REST with an API key.
metadata:
  version: 0.1.26
---

# StayAPI — Live Hotel & Travel Data

StayAPI is a hosted API that returns live, structured data from the major travel platforms: Booking.com, Airbnb, Expedia, Vrbo, Google Hotels / Google Travel / Google Reviews, TripAdvisor, Agoda, Trip.com, HolidayCheck, Accor (ALL), Radisson, Marriott Bonvoy, Hilton, WeHotel (Jin Jiang), MakeMyTrip India, Otelpuan, and OpenTable. One successful data call or definitive business negative costs one quota unit, on any platform, over either transport. MCP input/provider/protocol failures do not consume quota.

Canonical links:

- Dashboard & signup: https://stayapi.com (the dashboard shows the API key)
- Documentation hub: https://stayapi.com/docs
- MCP setup guide: https://stayapi.com/docs/mcp
- REST base URL: `https://api.stayapi.com` · MCP endpoint: `https://api.stayapi.com/mcp`
- Support: info@stayapi.com

## Authentication — handle the key carefully

Every call needs a StayAPI API key, sent as the `X-API-Key` header — except through Claude's StayAPI connector, which signs in with OAuth instead. Resolve access in this order:

1. **StayAPI MCP tools already connected** (tool names like `stayapi_*` or `mcp__stayapi__*`): just call them — the connector already carries the credentials (API key or OAuth sign-in), so you never touch a key.
2. **`STAYAPI_API_KEY` environment variable**: use it by reference (`$STAYAPI_API_KEY`), never by value.
3. **Neither present**: in claude.ai, Claude Desktop, or the Claude mobile app, have the user add the StayAPI connector (**Settings → Connectors → Add custom connector**, URL `https://api.stayapi.com/mcp`, then Connect and sign in — no key needed). Otherwise ask the user for their key, or point them to https://stayapi.com to create an account and copy it from the dashboard.

The key is a paid credential tied to the user's quota and billing. Never print it, echo it, log it, commit it to a repository, or paste it into code, config files, or chat output. In shell commands always interpolate the variable; if the user pastes a key literal, put it in an environment variable first and use the variable from then on.

## Two transports

**MCP (preferred when available).** The server at `https://api.stayapi.com/mcp` exposes the capabilities listed with MCP names in the catalog. If StayAPI tools are visible, call them directly — no HTTP assembly, no key handling. To connect Claude Code:

```bash
claude mcp add --scope user --transport http stayapi https://api.stayapi.com/mcp --header "X-API-Key: YOUR_API_KEY"
```

(`--header` must come after the name and URL; a new session is needed to pick the server up.) In Claude on the web, desktop, or mobile, the user adds it under **Settings → Connectors → Add custom connector** with URL `https://api.stayapi.com/mcp` and signs in (OAuth, no key). Cursor and Claude Desktop config snippets: see [references/tools.md](references/tools.md#connecting-an-mcp-client) or https://stayapi.com/docs/mcp.

**REST.** Same data over plain HTTPS — use it when no MCP client is available or when scripting:

```bash
curl -sS -H "X-API-Key: $STAYAPI_API_KEY" \
  "https://api.stayapi.com/v1/booking/destinations/lookup?query=Phuket"
```

## Discover before guessing

For an MCP client, use its automatic tool discovery or standard `tools/list` protocol
before choosing an unfamiliar tool or supplying provider-specific fields. It returns
the current tool names, descriptions, required fields, types, bounds, and defaults
for that account; disabled tools are absent. It is free housekeeping, not a data
request. Read [the discovery workflow](references/agent-discovery.md) when you need a
schema, an error needs correction, or the visible tool set differs from this skill.

Keep this entrypoint for access, the common resolve-then-fetch pattern, and cross-provider
constraints. Load [workflows.md](references/workflows.md) for a core flow and
[tools.md](references/tools.md) only for a provider-specific capability or REST lookup.

## The core pattern: resolve an ID, then fetch

Most tools take a platform-native ID. Resolve once, reuse the returned ID or cursor,
and do not substitute an ID from another provider. Read [identifiers.md](references/identifiers.md)
when the user starts with a name or URL, or when the provider's resolver is unclear.

## Core workflows

The core flows below cover most requests; [references/workflows.md](references/workflows.md) has complete worked examples of each (MCP call sequence + copy-paste REST curl):

1. **Destination lookup → hotel search** (Booking): free-text city → `dest_id` → hotel list with prices and `hotel_id`s.
2. **Hotel details & photos** (Booking): `hotel_id` → compact details / full signed photo gallery.
3. **Guest reviews & date-range backfill** (Booking): paginated reviews; `sort=recent_desc` + paginate-until-cutoff for "all reviews since X".
4. **Google Hotels search**: location + dates → priced availability list.
5. **Google reviews for a place**: name or `data_id` → Google reviews with pagination.
6. **Airbnb destination search, details & reviews**: destination → first-page listing cards, then `listing_id` → details (dates required) and reviews.
7. **HolidayCheck hotel data**: hotel name → UUID search result → details or one fixed ten-review page; `recent_desc` for newest-first review collection.

Everything beyond these — TripAdvisor, Google Travel deep-dives, Accor, Radisson, Marriott Bonvoy, Hilton, WeHotel (Jin Jiang), OpenTable, Agoda, Trip.com, room-level rates, price calendars, private dining — is cataloged in [references/tools.md](references/tools.md). Prefer the connected server's discovered schema for MCP calls; use the catalog for REST equivalents and provider-specific constraints.

## Errors and backoff

REST errors are RFC 7807 `application/problem+json`; MCP tools return structured error dicts (`invalid_input`, `url_resolution_failed`, `upstream_error`, `no_results`, `rate_limited`, `quota_exhausted`, `feature_unavailable`). Two rules matter:

- **Check the tool outcome.** REST uses HTTP status; MCP may carry `isError`, JSON-RPC errors, or a structured helper error inside HTTP 200. Never parse an error as data. MCP invalid-input/provider/protocol failures release the reserved credit; definitive `no_results` remains billable.
- **Respect `retry_after`.** `rate_limited` means back off briefly and retry once or twice. `quota_exhausted` includes `retry_after` and `reset_at` — the monthly quota is gone, so stop calling and tell the user when it resets (or suggest upgrading); retrying in a loop only burns time.

Transient upstream errors (`upstream_error`, HTTP 502/503) are usually worth one retry after a few seconds — these are live scrapes of third-party sites, not a static database.

## Constraints that change answers

Read [constraints.md](references/constraints.md) before reporting a price, handling
children, choosing review sort/pagination, or treating a search result as availability.
Those provider-specific limits prevent common but material mistakes.
