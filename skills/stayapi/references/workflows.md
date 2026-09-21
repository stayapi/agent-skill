# StayAPI — Worked Examples for the Core Workflows

Each workflow shows the MCP tool sequence and the equivalent REST calls. REST examples assume:

```bash
BASE="https://api.stayapi.com"
# key comes from the environment — never inline the literal
AUTH=(-H "X-API-Key: $STAYAPI_API_KEY")
```

Dates in examples are placeholders — always use dates in the future, and pass the user's real dates when they gave any.

## Contents

1. [Destination lookup → hotel search (Booking)](#1-destination-lookup--hotel-search-booking)
2. [Hotel details & photos (Booking)](#2-hotel-details--photos-booking)
3. [Guest reviews & date-range backfill (Booking)](#3-guest-reviews--date-range-backfill-booking)
4. [Google Hotels search](#4-google-hotels-search)
5. [Google reviews for a place](#5-google-reviews-for-a-place)
6. [Airbnb listing details & reviews](#6-airbnb-listing-details--reviews)
7. [Hilton property → dated cash and full-points rewards](#7-hilton-property--dated-cash-and-full-points-rewards)

---

## 1. Destination lookup → hotel search (Booking)

Use when the user names a city/area and wants a hotel list. Two steps: resolve the destination, then search it.

**MCP**: `booking_lookup_destination(query="Phuket")` → `booking_search_hotels(dest_id=-3233180, checkin=..., checkout=...)`

**REST**:

```bash
# Step 1 — free-form name → dest_id. Use the bare city name (no region/state suffix).
curl -sS "${AUTH[@]}" "$BASE/v1/booking/destinations/lookup?query=Phuket"
# → {"success": true, "dest_id": -3233180, "dest_type": "CITY", ...}
# A "suggestions" array may accompany the resolved dest_id — the top-level dest_id is already
# the top match, so proceed with it; surface the alternatives only if the user's intent is
# genuinely unclear (e.g. "Springfield").

# Step 2 — search that destination for a stay window.
curl -sS "${AUTH[@]}" "$BASE/v1/booking/search?dest_id=-3233180&dest_type=CITY\
&checkin=2026-09-11&checkout=2026-09-13&adults=2&rooms=1\
&rows_per_page=25&offset=0&currency=USD"
```

Key parameters for `/v1/booking/search`:

- `dest_id` (required, **keep the minus sign**), `dest_type` (default `CITY`)
- `checkin`/`checkout` `YYYY-MM-DD` — optional, but prices need dates
- `adults` (default 2), `children` + `children_ages` ("5,8"), `rooms`
- `rows_per_page` (1–100, default 25) + `offset` for pagination
- `currency`, `language`

**Reading the search response** — the hotel rows are nested at `data.hotels` (with `data.pagination` beside them), not at the top level. Traps observed in live responses:

- The envelope's top-level `hotel_id` is always `null` on search — per-hotel ids live inside `data.hotels[]`. Hand those to details / rooms / reviews / photos below.
- `price.amount` is the **total for the whole stay**, not a nightly rate — reporting it as per-night is a silent, high-impact error. It can also carry float artifacts (`439.9400000000001`); show `price.display` or round before presenting.
- Results arrive in Booking's default "recommended" ranking and there is **no sort parameter**. For "top 3 by price/rating", over-fetch (`rows_per_page` up to 100) and sort client-side — and say which ranking you used.
- Use `data.pagination.has_next_page` to page; don't quote `total_results` as the destination's inventory (it tracks the requested page size, not the real total).

## 2. Hotel details & photos (Booking)

Use `hotel_id` when you have it. Only fall back to URL resolution when the user gave a URL and nothing else.

**MCP**: `booking_hotel_details(hotel_id="1331780")` · `booking_hotel_photos(hotel_id="1331780")` — both also accept `url=` directly (slower).

**REST**:

```bash
# From a URL? Resolve once, cache the id (slow: real-browser fetch, 5–40 s).
curl -sS "${AUTH[@]}" \
  "$BASE/v1/booking/hotel/url-to-id?url=https://www.booking.com/hotel/us/the-line-la.html"
# → {"hotel_id": "...", ...}

# Compact details (v2, hotel_id only — it rejects url=):
curl -sS "${AUTH[@]}" "$BASE/v2/booking/hotel/details?hotel_id=1331780"

# Full photo gallery — signed, ready-to-use URLs at three sizes:
curl -sS "${AUTH[@]}" "$BASE/v2/booking/hotel/photos?hotel_id=1331780"

# Room types + bookable rate blocks for a stay window:
curl -sS "${AUTH[@]}" "$BASE/v1/booking/hotel/rooms?hotel_id=1331780&checkin=2026-09-11&checkout=2026-09-13"
```

Booking photo URLs are signed per photo — return them as-is; never reconstruct an image URL from a photo id.

## 3. Guest reviews & date-range backfill (Booking)

**MCP**: `booking_hotel_reviews(hotel_id="1331780", sort="recent_desc", page=1, per_page=25)`

**REST**:

```bash
curl -sS "${AUTH[@]}" \
  "$BASE/v1/booking/hotel/reviews?hotel_id=1331780&page=1&per_page=25&sort=recent_desc"
```

Parameters:

- `page` (from 1) + `per_page` (max 25); response carries `has_next_page`
- `sort`: default is Booking's "most relevant" ordering; `recent_desc` (newest first), `recent_asc`, `score_desc`, `score_asc`
- `language`: 2-letter ISO code to filter review language (e.g. `en`)
- `include_stay_dates=true` adds each review's check-in/out dates

**Date-range backfill pattern** ("all reviews from the last 12 months"): request `sort=recent_desc`, walk `page=1,2,3…` collecting reviews, and stop as soon as a page's oldest `reviewed_date` is older than the cutoff. Relevance order can't do this — only `recent_desc` is chronologically monotonic.

## 4. Google Hotels search

Priced availability from Google Hotels — good default when the user wants a market-wide price scan rather than one platform.

**MCP**: `google_hotels_search(location="Phuket", check_in="2026-09-11", check_out="2026-09-13")`

**REST**:

```bash
curl -sS "${AUTH[@]}" "$BASE/v1/google_hotels/search?location=Phuket\
&check_in=2026-09-11&check_out=2026-09-13&adults=2&currency=USD&min_rating=4.0"
```

- `location` free-text; `check_in`/`check_out` required, must be future dates, checkout after checkin
- `min_rating` 1.0–5.0 works; **there is no max-price filter upstream** — filter the response client-side for price caps

For a deep-dive on one property (review histogram, photo gallery, rate band, similar hotels, nearby POIs), pivot to the `google_travel_*` tools — resolve an `entity_token` first with `google_travel_resolve(hotel_name)` or from a `google_travel_search` result. See [tools.md](tools.md#google-travel).

## 5. Google reviews for a place

**MCP**: `google_reviews_for_place(query="Bellagio Las Vegas")` — or `data_id=` if known.

**REST** — two entry points depending on what you have:

```bash
# Have only a name? search-and-review resolves it and returns the first page + data_id:
curl -sS "${AUTH[@]}" --get "$BASE/v1/google_reviews/search-and-review" \
  --data-urlencode "query=Bellagio Las Vegas"

# Have a data_id (from the call above or a Maps URL)? Paginate with it:
curl -sS "${AUTH[@]}" "$BASE/v1/google_reviews/reviews?data_id=0x89c259af98ec7cbd:0x654b31dc1f9816db&sort_by=newest"
```

Pagination via `next_page_token` works only on the `data_id` path — so for multi-page pulls, always capture the `data_id` from the first response and continue with `/reviews`.

Details that matter for review requests:

- `sort_by`: `most_relevant` (default), `newest`, `highest_rating`, `lowest_rating`. "Recent reviews" requires `sort_by=newest` — the default is relevance order, which will quietly return old reviews.
- **Many Google reviews are rating-only** (`text: null`) — a page of 8 may hold only 2–3 with prose. When the user asks for N snippets, keep paginating until you have N with text.
- **Edited reviews reorder by edit time**, while `iso_date` keeps the *original* post date — a review "edited 6 hours ago" can carry `iso_date: 2018-…`. A paginate-until-cutoff backfill on Google reviews must tolerate these outliers rather than stopping at the first old `iso_date`.
- **Two Google review surfaces exist**: this one (`google_reviews_for_place`) is Google Maps place reviews and works for any place type — hotels, restaurants, attractions. `google_travel_reviews` is Google's hotel-only Travel surface (entity_token-keyed, supports topic filters). For a hotel either works: answer "Google reviews for X" with this one; use the travel one when you're already holding an `entity_token` or need topic/text filtering. See [tools.md](tools.md#google-travel).

## 6. Airbnb listing details & reviews

Airbnb URL→ID is an instant regex — passing a URL costs nothing extra.

**MCP**: `airbnb_listing_details(url="https://www.airbnb.com/rooms/22120898", check_in=..., check_out=...)` · `airbnb_listing_reviews(listing_id=22120898, sort_by="MOST_RECENT")`

**REST**:

```bash
# Details — check_in/check_out are REQUIRED (pricing is date-dependent); pick near-future
# dates if the user doesn't care:
curl -sS "${AUTH[@]}" --get "$BASE/v1/airbnb/listing/details-from-url" \
  --data-urlencode "url=https://www.airbnb.com/rooms/22120898" \
  -d check_in=2026-09-11 -d check_out=2026-09-13 -d adults=2 -d currency=USD

# Same by id: $BASE/v1/airbnb/listing/22120898/details?check_in=...&check_out=...

# Reviews — limit up to 50/page, offset-paginated:
curl -sS "${AUTH[@]}" \
  "$BASE/v1/airbnb/listing/reviews/22120898?limit=50&offset=0&sort_by=MOST_RECENT"
```

- `sort_by`: `BEST_QUALITY` (default) or `MOST_RECENT` — use `MOST_RECENT` for backfills, same stop-at-cutoff pattern as Booking.
- REST also exposes per-listing `pricing`, `calendar`, `cancellation-policy`, and `payment-methods` (same `/v1/airbnb/listing/…` shape) — see https://stayapi.com/docs.
- **No Airbnb search by location** — `airbnb_search_listings` is an unbilled stub. For "find me an Airbnb in X", search Booking / Google Hotels instead and say why.

## 7. Hilton property → dated cash and full-points rewards

Resolve a Hilton URL or search a place, then pass the seven-character
`hotel_code` to dated rooms. The rooms call has a fixed search party: one room,
no children, and 1–4 adults. It returns starting cash offers and full-points
standard/premium reward offers. It does not return all cash plans or Points & Money.

**MCP**: `hilton_hotel_url_to_id(url="https://www.hilton.com/en/hotels/lonhitw-hilton-london-park-lane/")` → `hilton_rooms(hotel_code="LONHITW", check_in="2027-11-12", check_out="2027-11-15", adults=2, currency="GBP")`

**REST**:

```bash
# Or start with /v1/hilton/search?query=London and use a returned hotel_code.
curl -sS "${AUTH[@]}" "$BASE/v1/hilton/rooms?hotel_code=LONHITW\
&check_in=2027-11-12&check_out=2027-11-15&adults=2&currency=GBP"
```

`per_night` is the stay average. Use `total` or `total_formatted` for the
upstream stay total; do not multiply the rounded nightly amount by nights.
`currency` is the requested display currency, while `native_currency` identifies
Hilton's source currency. A `pricing_scope` of `starting_rates` confirms the
cash starting-offer scope. `reward_rates` has explicit `points_total` and dated
`nightly_rates`; `points` is only the check-in night. Either offer list may be
empty, but every returned room has a verified offer. `members_only` and
`login_required` describe booking restrictions; no Hilton login is needed to
retrieve quotes. Never infer mixed-payment costs or fifth-night benefits.
