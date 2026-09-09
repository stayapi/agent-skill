# StayAPI — Full Tool & Endpoint Catalog

Every capability, grouped by platform. Each row gives the MCP tool name and the REST equivalent under `https://api.stayapi.com`. All REST calls take the `X-API-Key` header; all MCP calls go through the connected `stayapi` server. One successful data call = one quota unit on either transport.

## Contents

- [Booking.com](#bookingcom)
- [Airbnb](#airbnb)
- [Google Hotels](#google-hotels) · [Google Reviews](#google-reviews) · [Google Travel](#google-travel)
- [TripAdvisor](#tripadvisor)
- [Accor (ALL)](#accor-all)
- [Radisson](#radisson)
- [Marriott Bonvoy](#marriott-bonvoy)
- [Hilton](#hilton)
- [WeHotel (Jin Jiang)](#wehotel-jin-jiang)
- [MakeMyTrip India](#makemytrip-india)
- [OpenTable](#opentable)
- [Agoda](#agoda) · [Trip.com](#tripcom) · [Meta (geocoding)](#meta-geocoding)
- [URL vs ID — which to pass](#url-vs-id--which-to-pass)
- [Quota, billing, and errors](#quota-billing-and-errors)
- [Connecting an MCP client](#connecting-an-mcp-client)

---

## Booking.com

| MCP tool | REST | What it does |
|---|---|---|
| `booking_lookup_destination` | `GET /v1/booking/destinations/lookup?query=` | Free-form name → signed `dest_id`. First step before search. Query bare city names. |
| `booking_search_hotels` | `GET /v1/booking/search?dest_id=` | Hotel list for a destination + stay window. `dest_id` is signed — keep the minus. |
| `booking_hotel_details` | `GET /v2/booking/hotel/details?hotel_id=` | Compact one-hotel details. v2 takes `hotel_id` only; resolve URLs first. |
| `booking_hotel_rooms` | `GET /v1/booking/hotel/rooms?hotel_id=` | Room types + bookable rate blocks, incl. `availability_summary.total_available_rooms`. |
| `booking_hotel_reviews` | `GET /v1/booking/hotel/reviews?hotel_id=` | Paginated reviews (max 25/page). `sort=recent_desc` for chronological backfill. |
| `booking_hotel_photos` | `GET /v2/booking/hotel/photos?hotel_id=` | Full gallery; signed URLs at three sizes. Never rebuild photo URLs from ids. |
| — | `GET /v1/booking/hotel/url-to-id?url=` | Canonical hotel URL → numeric `hotel_id`. Real-browser fetch: 5–40 s; cache the result. |

REST additionally exposes `hotel/reviews/summary`, `hotel/reviews/scores`, `hotel/facilities`, `hotel/prices`, and `search_by_url` — see https://stayapi.com/docs. The three REST review endpoints return empty success data for valid zero-review hotels and `NOT_FOUND` for nonexistent IDs.

Canonical URL shape (anything else is a 400): `https://www.booking.com/hotel/{cc}/{slug}.html`

## Airbnb

| MCP tool | REST | What it does |
|---|---|---|
| `airbnb_listing_details` | `GET /v1/airbnb/listing/{id}/details` or `/v1/airbnb/listing/details-from-url?url=` | Listing details. `check_in`/`check_out` required. URL parsing is instant. |
| `airbnb_listing_reviews` | `GET /v1/airbnb/listing/reviews/{id}` or `/v1/airbnb/listing/reviews-from-url?url=` | Reviews, up to 50/page, `sort_by=BEST_QUALITY\|MOST_RECENT`. |
| `airbnb_search_listings` | — | ⚠️ Stub — returns `feature_unavailable`, not billed. No Airbnb location search yet. |

REST also has per-listing `pricing`, `calendar`, `cancellation-policy`, `payment-methods`, and `extract-id`.

## Google Hotels

| MCP tool | REST | What it does |
|---|---|---|
| `google_hotels_search` | `GET /v1/google_hotels/search?location=&check_in=&check_out=` | Priced availability list. `min_rating` works; no upstream max-price filter — cap prices client-side. |

## Google Reviews

| MCP tool | REST | What it does |
|---|---|---|
| `google_reviews_for_place` | `GET /v1/google_reviews/reviews?data_id=` (by id) · `GET /v1/google_reviews/search-and-review?query=` (by name) | Google Maps reviews for any place type (hotels, restaurants, attractions). `sort_by`: `most_relevant` (default), `newest`, `highest_rating`, `lowest_rating`. `next_page_token` pagination works only on the `data_id` path — capture `data_id` from the first response. |

This is the Maps place-review surface. For a hotel, `google_travel_reviews` (below) is an alternative with topic/text filters — use it when you already hold an `entity_token`; use this one to answer "Google reviews for X". Reviews can be rating-only (`text: null`), and edited reviews sort by edit time while `iso_date` keeps the original post date.

## Google Travel

Deep per-hotel data from Google Travel. Everything keys off an `entity_token` — get one from `google_travel_resolve` (name → candidates) or from a `google_travel_search` result card, then fan out. Tokens are long opaque strings; pass them exactly as returned.

| MCP tool | REST | What it does |
|---|---|---|
| `google_travel_resolve` | `GET /v1/google_travel/resolve` | Hotel name → `entity_token` candidates. First step. |
| `google_travel_search` | `GET /v1/google_travel/search` | Free-text search → ~20 hotel cards (rating, price, class, token each); dates, guest-rating threshold (3.5/4.0/4.5), currency, pagination. |
| `google_travel_summary` | `GET /v1/google_travel/summary` | Overall rating, review count, star histogram, per-topic sub-scores. |
| `google_travel_info` | `GET /v1/google_travel/info` | Identity/metadata: address, coordinates, phone, check-in/out times, website, Maps ids. |
| `google_travel_photos` | `GET /v1/google_travel/photos` | Photo gallery with dimensions. |
| `google_travel_prices` | `GET /v1/google_travel/prices` | Nightly rate, usual-price band, partner booking offers. |
| `google_travel_similar` | `GET /v1/google_travel/similar` | Similar hotels + rentals nearby, each with its own token to pivot to. |
| `google_travel_nearby` | `GET /v1/google_travel/nearby` | Nearby POIs (attractions, transit, airports, restaurants) with drive/walk times. |
| `google_travel_reviews` | `GET /v1/google_reviews/travel/reviews` | Google-authored reviews: text `query` filter, `topic` (from summary's `topics[].category`), `sort`, token pagination. Keep filters identical across pages. |

## TripAdvisor

| MCP tool | REST | What it does |
|---|---|---|
| `tripadvisor_geo_search` | `GET /v1/tripadvisor/geo-search` | Place name → `geo_id`. First step before area search. |
| `tripadvisor_search_hotels` | `GET /v1/tripadvisor/search` | Hotels in a `geo_id` area; each carries a `location_id`. |
| `tripadvisor_hotel_details` | `GET /v1/tripadvisor/hotel/details/{location_id}` or `…/hotel/details-from-url?url=` | One hotel's details. URL→ID is instant. |
| `tripadvisor_hotel_reviews` | `GET /v1/tripadvisor/hotel/reviews/{location_id}` or `…/hotel/reviews-from-url?url=` | Paginated reviews with `language`. |
| `tripadvisor_hotel_prices` | `GET /v1/tripadvisor/hotel/prices/{location_id}` or `…/hotel/prices-from-url?url=` | Live provider offers (Booking, Agoda, …) for a required stay window. |

## Accor (ALL)

Covers Sofitel, Fairmont, Novotel, Ibis, and other Accor brands. Search takes `latitude`+`longitude` (+`radius_km`) or a `destination_slug` — get coordinates from `meta_coordinates_lookup` when needed.

| MCP tool | REST | What it does |
|---|---|---|
| `accor_search` | `GET /v1/accor/search` | Properties near coordinates / for a slug, each with its best available offer and `hotel_id`. |
| `accor_hotel_details` | `GET /v1/accor/hotel/details` | Full property content: contact, descriptions, room catalogue, facilities, media. |
| `accor_hotel_amenities` | `GET /v1/accor/hotel/amenities` | Amenity categories/services with fee + location metadata. |
| `accor_hotel_reviews` | `GET /v1/accor/hotel/reviews` | Guest reviews + hotel responses — capped at 20 total (upstream limit). |
| `accor_hotel_rooms` | `GET /v1/accor/hotel/rooms` | Bookable room offers grouped into types with rate plans and policies. |
| `accor_hotel_photos` | `GET /v1/accor/hotel/photos` | Full photo/video gallery, public URLs at every size. |
| `accor_price_calendar` | `GET /v1/accor/hotel/price-calendar` | Per-date cheapest offer, window up to 60 days. |

## Radisson

| MCP tool | REST | What it does |
|---|---|---|
| `radisson_search` | `GET /v1/radisson/search` | Search by destination or hotel name → `hotel_id`s (no prices). |
| `radisson_hotel_details` | `GET /v1/radisson/hotel/details` | Identity, description, location, contact, services, main photo. |
| `radisson_hotel_amenities` | `GET /v1/radisson/hotel/amenities` | Services, hotel/location types, tags. |
| `radisson_hotel_reviews` | `GET /v1/radisson/hotel/reviews` | TripAdvisor summary + review sample as published upstream. |
| `radisson_hotel_url_to_id` | `GET /v1/radisson/hotel/url-to-id` | `radissonhotels.com/…/hotels/{slug}` URL → `hotel_id`. |
| `radisson_hotel_rooms` | `GET /v1/radisson/hotel/rooms` | Static room types + media — **no live availability**. |
| `radisson_hotel_photos` | `GET /v1/radisson/hotel/photos` | Full categorized gallery. |
| `radisson_price_calendar` | `GET /v1/radisson/hotel/price-calendar` | Public lowest-price calendar, up to 60 days. |
| `radisson_hotel_rates` | `GET /v1/radisson/hotel/rates` | Live lowest cash rate for one stay, optional member rates. |
| `radisson_hotel_room_rates` | `GET /v1/radisson/hotel/room-rates` | Exact live room groups with prices and booking conditions. |

## Marriott Bonvoy

Brand.com search and live room rates with Bonvoy award pricing. No URL resolver: `marriott_bonvoy_search` returns the `property_id` codes (e.g. `NYCXR`) that `marriott_bonvoy_rooms` takes.

| MCP tool | REST | What it does |
|---|---|---|
| `marriott_bonvoy_search` | `GET /v1/marriott/bonvoy/search` | Properties near `latitude`/`longitude` for a stay: `property_id`, brand, rating, distance, lowest cash price, `points_per_night`. 20/page via `offset`; `sort_by` `DISTANCE` or `PRICE`; optional `corporate_code`. |
| `marriott_bonvoy_rooms` | `GET /v1/marriott/bonvoy/rooms` | Room types and rate plans for one `property_id` and stay, cash and award. **`points` and `per_night` are the check-in night only** — use `points_total` (whole stay, matches marriott.com) or `total` (cash incl. taxes/fees), or sum `nightly_rates` (one `{date, cash, points, free_night}` per night). A room can list several award plans at different prices; key on `rate_plan_code`. |

## Hilton

Hilton supports undated property discovery, canonical URL resolution, and dated
starting cash rates. Search itself returns no cash rates or availability. Rooms
returns one cash offer per available room type, not full rate-plan or Hilton
Honors award inventory.

| MCP tool | REST | What it does |
|---|---|---|
| `hilton_search` | `GET /v1/hilton/search` | Search by a place query or coordinate pair. Returns a location match and hotel metadata, including a seven-character `hotel_code`; `limit` is 1–150 and defaults to 20. |
| `hilton_hotel_url_to_id` | `GET /v1/hilton/hotel/url-to-id` | Parse a canonical HTTPS Hilton property or booking URL into its uppercase seven-character `hotel_code`. |
| `hilton_rooms` | `GET /v1/hilton/rooms` | Dated room types and one starting cash rate each for `hotel_code`; future `check_in`/`check_out`, `adults` 1–4 (default 2), `currency` USD/GBP (default USD). Always one room and no children. `total` is exact upstream stay total, not `per_night × nights`. |

## WeHotel (Jin Jiang)

Jin Jiang, Metropolo, Campanile, Golden Tulip, Lavande, Vienna, 7 Days, and other WeHotel brands. Search takes a WeHotel city code (e.g. `AR04567` for Shanghai) plus stay dates; everything else keys off the `property_id` a search row or URL resolution returns.

| MCP tool | REST | What it does |
|---|---|---|
| `bestwe_search_hotels` | `GET /v1/bestwe/search` | Dated city search → properties with lowest advertised nightly price and `property_id`. Optional lat/lng center; `sort`: `recommended`/`distance`/`price_asc`/`price_desc`; ≤20/page. |
| `bestwe_hotel_url_to_id` | `GET /v1/bestwe/hotel/url-to-id` | Canonical `hotel.bestwehotel.com` HotelDetail URL → `property_id`. |
| `bestwe_hotel_details` | `GET /v1/bestwe/hotel/details` | Property content: policies, photos, facilities, room metadata. |
| `bestwe_hotel_reviews` | `GET /v1/bestwe/hotel/reviews` | Paginated guest reviews with scores, photos, tags, and hotel replies (≤20/page). |
| `bestwe_hotel_rooms` | `GET /v1/bestwe/hotel/rooms` | Dated room types with public and member rates and cancellation rules. |

## MakeMyTrip India

Use a MakeMyTrip city code and dated stay for search. Search returns the numeric `hotel_id` for details and reviews. Prices are INR listing prices; prefer the tax-and-fee-inclusive amount when the response provides it. Room inventory is not yet available.

| MCP tool | REST | What it does |
|---|---|---|
| `makemytrip_search_hotels` | `GET /v1/makemytrip/search` | Dated India city search. Takes `city_code`, future check-in/check-out, adults, one room, optional child ages, limit, and an opaque cursor. Reuse a cursor only with identical search inputs and limit. |
| `makemytrip_get_hotel_details` | `GET /v1/makemytrip/hotel/details` | Hotel description, amenities, location, and public photos by numeric `hotel_id` and stay. |
| `makemytrip_get_hotel_reviews` | `GET /v1/makemytrip/hotel/reviews` | Reviews by numeric `hotel_id`; start with `next_ota=MMT` and continue only while `source_changed` is false and `next_start` is present. |

## OpenTable

Restaurant data. Search is coordinate-based — resolve place names with `meta_coordinates_lookup` first.

| MCP tool | REST | What it does |
|---|---|---|
| `opentable_search_restaurants` | `GET /v1/opentable/search` | Restaurants near lat/lng for a date/time/party size. |
| `opentable_restaurant_url_to_id` | `GET /v1/opentable/restaurant/url-to-id` | `/r/{slug}` URL → numeric `restaurant_id` (no browser — fast). |
| `opentable_restaurant_details` | `GET /v1/opentable/restaurants/{restaurant_id}` | Full normalized profile: identity, hours, photos, feature flags. |
| `opentable_restaurant_menu` | `GET /v1/opentable/restaurant/menu` | Every published structured menu with items and prices. |
| `opentable_restaurant_reviews` | `GET /v1/opentable/restaurant/reviews` | Paginated reviews (≤25/page), `newest`/`highest`/`lowest`. |
| `opentable_restaurant_availability` | `GET /v1/opentable/restaurants/{restaurant_id}/availability` | Is one exact time bookable for date + party size. |
| `opentable_restaurant_availability_slots` | `GET /v1/opentable/restaurants/{restaurant_id}/availability/slots` | Nearby bookable `HH:MM` slots around a preferred time. |
| `opentable_private_dining_restaurants` | `GET /v1/opentable/private-dining` | Private-dining venues near lat/lng with capacity + direct contact; `with_facets=true` reveals cuisine UUID filters. |

## Agoda

| MCP tool | REST | What it does |
|---|---|---|
| `agoda_hotel_url_to_id` | `GET /v1/agoda/hotel/url-to-id` | Agoda hotel URL → numeric `hotel_id`. Costs one quota unit (real page fetch) — cache the id. First step. |
| `agoda_hotel_prices` | `GET /v1/agoda/hotel/prices` | Live dated room availability, lowest whole-stay price, per-night/stay tax breakdowns, breakfast, cancellation, and inventory. A sold-out stay is a successful empty result. |
| `agoda_hotel_reviews` | `GET /v1/agoda/hotel/reviews/{hotel_id}` | Reviews aggregated across every provider Agoda syndicates. 20/page; `sorting`: `7` recent (default), `6` highest, `5` lowest. |
| `agoda_hotel_review_comments` | `GET /v1/agoda/hotel/reviews/{hotel_id}/comments` | Same shape, one provider only — `provider_id` defaults to `3038` (Agoda's own). |

## Trip.com

| MCP tool | REST | What it does |
|---|---|---|
| `tripcom_hotel_reviews` | `GET /v1/tripcom/hotel/reviews/{hotel_id}` | Paginated reviews (`page_size` 1–50). Drives a real browser — several seconds per call; use large pages. |

## Meta (geocoding)

| MCP tool | REST | What it does |
|---|---|---|
| `meta_coordinates_lookup` | `GET /v1/meta/coordinates-lookup` | Place name → candidate locations with `latitude`/`longitude`, type, country. Feeds OpenTable and Accor search. |

---

## URL vs ID — which to pass

- **ID always wins** when you have one: no resolution step, no extra latency or quota.
- Booking, Airbnb, and TripAdvisor single-item tools accept `url` directly. Airbnb and TripAdvisor URL parsing is instant; **Booking URL resolution takes 5–40 s** (real browser).
- Radisson, OpenTable, Agoda, and WeHotel keep resolution explicit: call their `*_url_to_id` tool once, then use the id everywhere. Agoda's resolver costs a quota unit; OpenTable's is fast (no browser).
- Cache every resolved id for the rest of the session — they're stable.

## Quota, billing, and errors

- Billable: every successful data `tools/call` / REST data request — one unit each. Not billed: MCP housekeeping (`initialize`, `tools/list`), the `airbnb_search_listings` stub, unknown tool names.
- REST errors: RFC 7807 `application/problem+json`; only 2xx bodies are data.
- MCP structured errors and how to react:

| Error | Meaning | Reaction |
|---|---|---|
| `invalid_input` | Missing/bad parameter (`field` says which) | Fix the call, don't retry as-is |
| `url_resolution_failed` | URL didn't yield an id | Check canonical URL format; ask user or try the id path |
| `upstream_error` | The platform itself errored | Retry once after a few seconds |
| `no_results` | Valid query, nothing matched | Report honestly; loosen the query |
| `rate_limited` | Scraper queue saturated | Back off `retry_after` seconds, retry once or twice |
| `quota_exhausted` | Monthly quota used up | Stop. Tell the user the `reset_at` time; suggest upgrading |
| `feature_unavailable` | Not built (Airbnb search) | Offer an alternative platform; votes go to info@stayapi.com |

Auth failures surface as HTTP 401/403 (JSON-RPC `-32001` on MCP) — the key is missing, wrong, or the account is blocked.

## Connecting an MCP client

Full guide with screenshots and troubleshooting: https://stayapi.com/docs/mcp. Replace `YOUR_API_KEY` interactively — never write the key into a file you might commit.

**Claude Code** (note: `--header` must come after the name and URL; open a new session afterwards):

```bash
claude mcp add --scope user --transport http stayapi https://api.stayapi.com/mcp --header "X-API-Key: YOUR_API_KEY"
```

**Cursor** — `~/.cursor/mcp.json`, then restart Cursor:

```json
{
  "mcpServers": {
    "stayapi": {
      "type": "http",
      "url": "https://api.stayapi.com/mcp",
      "headers": { "X-API-Key": "YOUR_API_KEY" }
    }
  }
}
```

**Claude Desktop** — `claude_desktop_config.json` (macOS: `~/Library/Application Support/Claude/`; Windows: `%APPDATA%\Claude\`), then fully quit (⌘Q) and relaunch. The no-space `X-API-Key:YOUR_API_KEY` form is deliberate — Claude Desktop re-tokenizes args containing spaces:

```json
{
  "mcpServers": {
    "stayapi": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://api.stayapi.com/mcp", "--header", "X-API-Key:YOUR_API_KEY"]
    }
  }
}
```

ChatGPT custom connectors are **not** supported (they require OAuth). Batched JSON-RPC requests are rejected — one tool call at a time.
