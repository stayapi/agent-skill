# StayAPI — Full Tool & Endpoint Catalog

Every capability, grouped by platform. Each row gives the MCP tool name and the REST equivalent under `https://api.stayapi.com`. All REST calls take the `X-API-Key` header; all MCP calls go through the connected `stayapi` server. One successful data call = one quota unit on either transport.

## Contents

- [Booking.com](#bookingcom)
- [Airbnb](#airbnb)
- [Expedia](#expedia) · [Vrbo](#vrbo)
- [Google Hotels](#google-hotels) · [Google Reviews](#google-reviews) · [Google Travel](#google-travel)
- [TripAdvisor](#tripadvisor)
- [HolidayCheck](#holidaycheck)
- [Accor (ALL)](#accor-all)
- [Radisson](#radisson)
- [Marriott Bonvoy](#marriott-bonvoy)
- [Hilton](#hilton)
- [WeHotel (Jin Jiang)](#wehotel-jin-jiang)
- [MakeMyTrip India](#makemytrip-india)
- [Otelpuan](#otelpuan)
- [OpenTable](#opentable)
- [Agoda](#agoda) · [Trip.com](#tripcom) · [Meta (geocoding)](#meta-geocoding)
- [Display (ChatGPT hotel cards)](#display-chatgpt-hotel-cards)
- [URL vs ID — which to pass](#url-vs-id--which-to-pass)
- [Quota, billing, and errors](#quota-billing-and-errors)
- [Connecting an MCP client](#connecting-an-mcp-client)

---

## Booking.com

| MCP tool | REST | What it does |
|---|---|---|
| `booking_lookup_destination` | `GET /v1/booking/destinations/lookup?query=` | Free-form name → signed `dest_id`. First step before search. Query bare city names. |
| `booking_search_hotels` | `GET /v1/booking/search?dest_id=` | Hotel list for a destination + stay window. `dest_id` is signed — keep the minus. `total_results` is Booking's real count; only offsets 0–1000 are served — page via `has_next_page`. Prices depend on `country_market` (default `US`) — keep it the same across search/rooms/prices. |
| `booking_hotel_details` | `GET /v2/booking/hotel/details?hotel_id=` | Compact one-hotel details. REST `data.property_type` (MCP top-level `property_type`) is Booking's classification, e.g. `HOTEL`, `APARTMENT`, `HOLIDAY_HOME`, `CHALET`, or `null` when unavailable; accept future strings. v2 takes `hotel_id` only; resolve URLs first. |
| `booking_hotel_rooms` | `GET /v1/booking/hotel/rooms?hotel_id=` | Room types + bookable rate blocks, incl. `availability_summary.total_available_rooms`. Use the search's `country_market` so prices match. |
| `booking_hotel_reviews` | `GET /v1/booking/hotel/reviews?hotel_id=` | Paginated reviews (max 25/page). `sort=recent_desc` for chronological backfill. |
| `booking_hotel_photos` | `GET /v2/booking/hotel/photos?hotel_id=` | Full gallery; signed URLs at three sizes. MCP: `limit` (default 20, `0` = all) + `size` to trim; `metadata.photo_count` = full total. Never rebuild photo URLs from ids. |
| — | `GET /v1/booking/hotel/url-to-id?url=` | Canonical hotel URL → numeric `hotel_id`. Real-browser fetch: 5–40 s; cache the result. |

REST additionally exposes `hotel/reviews/summary`, `hotel/reviews/scores`, `hotel/facilities`, `hotel/prices`, and `search_by_url` — see https://stayapi.com/docs. The three REST review endpoints return empty success data for valid zero-review hotels and `NOT_FOUND` for nonexistent IDs.

Canonical URL shape (anything else is a 400): `https://www.booking.com/hotel/{cc}/{slug}.html`

## Airbnb

Airbnb `listing_id` values in REST and MCP responses are decimal strings, including nested listing data and error extensions, preserving all digits of 19-digit IDs. Keep IDs as strings in clients. REST paths accept the full decimal text. MCP accepts positive decimal strings (preferred) or legacy integer numbers up to `9007199254740991`; larger numbers, floats, and booleans are rejected. For example, pass `listing_id="1234567890123456789"`, never a JavaScript `Number` conversion.

| MCP tool | REST | What it does |
|---|---|---|
| `airbnb_search_listings` | `GET /v1/airbnb/search?location=` | First-page destination listings. Optional paired dates; undated results omit prices and neither mode verifies availability. No property-name lookup or pagination. |
| `airbnb_listing_details` | `GET /v1/airbnb/listing/{id}/details` or `/v1/airbnb/listing/details-from-url?url=` | Listing details. `check_in`/`check_out` required. URL parsing is instant. |
| `airbnb_listing_reviews` | `GET /v1/airbnb/listing/reviews/{id}` or `/v1/airbnb/listing/reviews-from-url?url=` | Reviews, up to 50/page, `sort_by=BEST_QUALITY\|MOST_RECENT`. |
| `airbnb_listing_calendar` | `GET /v1/airbnb/listing/{id}/calendar` or `/v1/airbnb/listing/calendar-from-url?url=` | Daily availability from this month; `months=1..12` (default 12). No price quote. |

REST also has per-listing `pricing`, `cancellation-policy`, `payment-methods`, and `extract-id`.

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

Deep per-hotel data from Google Travel. Everything keys off an `entity_token` — get one from `google_travel_resolve` (name → candidates) or from a `google_travel_search` result card, then fan out. Tokens are opaque strings, including short `Cg…` and longer hotel/rental tokens; pass them exactly as returned. Tokens from similar can also be reused. Similar returns null for unsupported tokens; resolve by name instead.

| MCP tool | REST | What it does |
|---|---|---|
| `google_travel_resolve` | `GET /v1/google_travel/resolve` | Hotel name → `entity_token` candidates. First step. |
| `google_travel_search` | `GET /v1/google_travel/search` | Free-text search → ~20 hotel cards (rating, price, class, token each); dates, guest-rating threshold (3.5/4.0/4.5), currency, pagination. |
| `google_travel_summary` | `GET /v1/google_travel/summary` | Overall rating, review count, star histogram, per-topic sub-scores. |
| `google_travel_info` | `GET /v1/google_travel/info` | Identity/metadata: address, coordinates, phone, check-in/out times, website, Maps ids. |
| `google_travel_photos` | `GET /v1/google_travel/photos` | Photo gallery with dimensions. |
| `google_travel_prices` | `GET /v1/google_travel/prices` | Nightly rate, usual-price band, partner booking offers. Optional paired `check_in`/`check_out` (YYYY-MM-DD, today/future, checkout later), `adults` 1–10 (default 2); omitted dates use Google defaults. |
| `google_travel_similar` | `GET /v1/google_travel/similar` | Similar hotels + rentals nearby, each with its own token to pivot to. |
| `google_travel_nearby` | `GET /v1/google_travel/nearby` | Nearby POIs (attractions, transit, airports, restaurants) with drive/walk times. |
| `google_travel_reviews` | `GET /v1/google_reviews/travel/reviews` | Google-authored reviews: text `query` filter, `topic` (from summary's `topics[].category`), `sort`, token pagination. Keep filters identical across pages. |

## TripAdvisor

| MCP tool | REST | What it does |
|---|---|---|
| `tripadvisor_geo_search` | `GET /v1/tripadvisor/geo-search` | Place name → `geo_id`. First step before area search. |
| `tripadvisor_search_hotels` | `GET /v1/tripadvisor/search` | Hotels in a `geo_id` area; each carries a `location_id`. |
| `tripadvisor_hotel_details` | `GET /v1/tripadvisor/hotel/details/{location_id}` or `…/hotel/details-from-url?url=` | One hotel's details, including highlighted amenities when available (not a complete list). No `contact.website` or `num_rooms`. URL→ID is instant. |
| `tripadvisor_hotel_reviews` | `GET /v1/tripadvisor/hotel/reviews/{location_id}` or `…/hotel/reviews-from-url?url=` | Paginated reviews with `language`. At most 20 per page; `per_page` above 20 is treated as 20. |
| `tripadvisor_hotel_prices` | `GET /v1/tripadvisor/hotel/prices/{location_id}` or `…/hotel/prices-from-url?url=` | Live provider offers (Booking, Agoda, …) for a required stay window. |

## HolidayCheck

Search by hotel name or resolve a supported hotel URL, then use its UUID directly.
Search is fixed at ten provider-returned candidates per page; `available_count`
is not an exhaustive total. The resolver is local and strips query/fragment; it
does not confirm hotel existence. Reviews are fixed at ten per page.
`original_language` and `language` are HolidayCheck-supplied locales and do not
guarantee translation.

| MCP tool | REST | What it does |
|---|---|---|
| `holidaycheck_hotels_search` | `GET /v1/holidaycheck/hotels/search` | Hotel-name matches with UUIDs for follow-up calls; `query` is trimmed 1–200 characters, `page` starts at 1, 10/page. `available_count` is provider-returned, not exhaustive. |
| `holidaycheck_hotel_url_to_id` | `GET /v1/holidaycheck/hotel/url-to-id` | Extract a lowercase hotel UUID from supported `.de`/`.com`/`.at`/`.ch` `hi` or `hr` hotel URL; no upstream fetch. |
| `holidaycheck_hotel_details` | `GET /v1/holidaycheck/hotel/details` | Source-grounded property content, contact, descriptions, amenities, check-in/out, and provider summary by UUID. No prices or photo gallery. |
| `holidaycheck_hotel_reviews` | `GET /v1/holidaycheck/hotel/reviews` | One review page plus provider aggregate by UUID; `page` starts at 1, `sort` is `most_relevant` or `recent_desc`, 10/page. No language, page-size, or owner-response controls. |

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
| `radisson_hotel_room_rates` | `GET /v1/radisson/hotel/room-rates` | Raw room groups plus `normalized_rates`: single-room cash alternatives and redeemable nightly/stay points. `reward_rate.points` is the first night; `points_total` sums dated quotes. Final redemption cash payable is unverified; inspect status fields and retain raw offers. |

## Marriott Bonvoy

Brand.com search, live room rates and a per-date price calendar, with Bonvoy award pricing. No URL resolver: `marriott_bonvoy_search` returns the `property_id` codes (e.g. `NYCXR`) that `marriott_bonvoy_rooms` and `marriott_bonvoy_price_calendar` take.

| MCP tool | REST | What it does |
|---|---|---|
| `marriott_bonvoy_search` | `GET /v1/marriott/bonvoy/search` | Properties near `latitude`/`longitude` for a stay: `property_id`, brand, rating, distance, lowest cash price, `points_per_night`. 20/page via `offset`; `sort_by` `DISTANCE` or `PRICE`; optional `corporate_code`. Without `corporate_only` a hotel lacking a code rate shows its regular price; `corporate_only=true` (requires `corporate_code`) keeps only hotels with a rate under the code, priced at it. |
| `marriott_bonvoy_rooms` | `GET /v1/marriott/bonvoy/rooms` | Room types and rate plans for one `property_id` and stay, cash and award. **`points` and `per_night` are the check-in night only** — use `points_total` (whole stay, matches marriott.com) or `total` (cash incl. taxes/fees), or sum `nightly_rates` (one `{date, cash, points, free_night}` per night). A room can list several award plans at different prices; key on `rate_plan_code`. |
| `marriott_bonvoy_price_calendar` | `GET /v1/marriott/bonvoy/price-calendar` | Lowest rate per check-in date for one `property_id`, `date_from`..`date_to` inclusive (≤ 35 days apart, so ≤ 36 dates), each priced for `nights` (1–9). Every date is listed; `available: false` means no rate (prices null). `per_night` = average nightly, `stay_total` = stay, both before taxes/fees, in the property's own currency (no conversion). `corporate_code` returns ONLY rates under the code (unknown code → all dates unavailable). `use_points=true` → `points_total` for the whole stay, cash null; not combinable with `corporate_code`. Unknown property → 404. |

## Hilton

Hilton supports undated property discovery, canonical URL resolution, and dated
starting cash rates plus full-points rewards. Search itself returns no prices or
availability. Rooms returns up to one starting cash offer plus standard/premium
reward offers per available room type, not exhaustive cash plans or Points & Money.

| MCP tool | REST | What it does |
|---|---|---|
| `hilton_search` | `GET /v1/hilton/search` | Search by a place query or coordinate pair. Returns a location match and hotel metadata, including a seven-character `hotel_code`; `limit` is 1–150 and defaults to 20. |
| `hilton_hotel_url_to_id` | `GET /v1/hilton/hotel/url-to-id` | Parse a canonical HTTPS Hilton property or booking URL into its uppercase seven-character `hotel_code`. |
| `hilton_rooms` | `GET /v1/hilton/rooms` | Dated room types with starting cash `rates` and full-points standard/premium `reward_rates` for `hotel_code`; future `check_in`/`check_out`, `adults` 1–4 (default 2), `currency` USD/GBP (default USD). Always one room and no children. `total` is exact upstream cash stay total, not `per_night × nights`. Reward `points` is the check-in night; `points_total` is the stay total with dated `nightly_rates`. Empty rewards means none returned; cash `rates` may be empty for an award-only room. Membership/login flags restrict booking. No Points & Money. `availability_status` is `available` or `unavailable`; verified unavailable returns success, an explanatory `availability_message`, empty `room_types`, and top-level `total: 0` (one credit). `native_currency` may be null for unavailable results; unknown availability fails. |

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

## Otelpuan

| MCP tool | REST | What it does |
|---|---|---|
| `otelpuan_search_hotels` | `GET /v1/otelpuan/search?query=` | Text autocomplete → hotel IDs, names, canonical URLs, and nullable locations. It returns hotel suggestions only and is not exhaustive destination inventory; use its ID directly with `otelpuan_hotel_reviews`. |
| `otelpuan_hotel_url_to_id` | `GET /v1/otelpuan/hotel/url-to-id?url=` | Full Otelpuan root-slug URL → stable numeric `hotel_id`, identity, page aggregate 0–10 rating, and review count. Query strings and fragments are ignored; nested paths and trailing slashes are invalid. |
| `otelpuan_hotel_rooms` | `GET /v1/otelpuan/hotel/rooms?url=&check_in=&check_out=&adults=&child_ages=` | Dated room offers by canonical Otelpuan URL; ISO dates, adults 1–10 (default 2), optional comma-separated child ages (0–16, max 4). Whole-stay and per-date provider prices remain separate. No currency selector, calendar, booking action, or numeric-ID input. |
| `otelpuan_hotel_reviews` | `GET /v1/otelpuan/hotel/reviews?hotel_id=&page=` | Hotel ID → up to 10 reviews per one-based page. Start with `otelpuan_hotel_url_to_id` only when you have a URL, then reuse its ID. The ID-only payload does not include hotel name, URL, or aggregate rating, so those are null. `stay_date` is month-level where supplied; `review_date` is always null because upstream has no publication date. |

## OpenTable

Restaurant data. Search is coordinate-based — resolve place names with `meta_coordinates_lookup` first.

| MCP tool | REST | What it does |
|---|---|---|
| `opentable_search_restaurants` | `GET /v1/opentable/search` | Restaurants near lat/lng for a date/time/party size. |
| `opentable_restaurant_url_to_id` | `GET /v1/opentable/restaurant/url-to-id` | `/r/{slug}` URL → numeric `restaurant_id` (reads the live page — a few seconds uncached, then cached). |
| `opentable_restaurant_details` | `GET /v1/opentable/restaurants/{restaurant_id}` | Full normalized profile: identity, hours, photos, feature flags. |
| `opentable_restaurant_menu` | `GET /v1/opentable/restaurant/menu` | Every published structured menu with items and prices. |
| `opentable_restaurant_reviews` | `GET /v1/opentable/restaurant/reviews` | Paginated reviews (≤25/page), `newest`/`highest`/`lowest`. |
| `opentable_restaurant_availability` | `GET /v1/opentable/restaurants/{restaurant_id}/availability` | Is one exact time bookable for date + party size. |
| `opentable_restaurant_availability_slots` | `GET /v1/opentable/restaurants/{restaurant_id}/availability/slots` | Nearby bookable `HH:MM` slots around a preferred time. |
| `opentable_private_dining_restaurants` | `GET /v1/opentable/private-dining` | Private-dining venues near lat/lng with capacity and the venue's private-dining page (the event contact's name/phone/email are REST-only); `with_facets=true` reveals cuisine UUID filters. `min_capacity` scans 250 venues per call; if `capacity_filter_truncated`, call again with `scan_offset=next_scan_offset`. |

## Agoda

| MCP tool | REST | What it does |
|---|---|---|
| `agoda_search_hotels` | `GET /v1/agoda/search` | Dated city_id search (REST alternatively accepts a strict Agoda search URL). Required check_in/check_out, optional occupancy/child ages/currency/page; fixed page_size 10 may return extra cards. Source nightly/stay totals; no inferred has_more. |
| `agoda_hotel_details` | `GET /v1/agoda/hotel/details?hotel_id=` | Date-free source address, metadata, description, available amenities, photos and useful_information: nullable number_of_rooms (declared total, not availability), year_opened (not construction), check-in/out text and groups [{name,items:[{label,value}]}]. Street can be null. ID 1–2147483647; explicit is_obfuscated masks full/street/coordinates. Resolve name/URL first. Collections not exhaustive; missing hotel is no_results. |
| `agoda_hotel_lookup` | `GET /v1/agoda/hotel/lookup?query=` | Hotel-name suggestions, optionally including location. Query 2–200 trimmed characters, no controls; select a candidate hotel_id. No pagination, prices, availability, or exhaustive-match guarantee. |
| `agoda_hotel_url_to_id` | `GET /v1/agoda/hotel/url-to-id` | Agoda hotel URL → numeric `hotel_id`. Costs one quota unit (real page fetch) — cache the id. Use hotel_lookup for names. |
| `agoda_hotel_prices` | `GET /v1/agoda/hotel/prices` | MCP `children_ages`: JSON integer array, up to nine ages 0–17; REST: CSV. Explicit ages override legacy `children` (default age 8); omitted/blank REST retains count, MCP `[]` means no children. Effective occupancy is echoed in `search`. Live dated room availability, lowest whole-stay price, per-night/stay tax breakdowns, breakfast, cancellation, and inventory. A sold-out stay is a successful empty result. |
| `agoda_hotel_reviews` | `GET /v1/agoda/hotel/reviews/{hotel_id}` | Reviews aggregated across every provider Agoda syndicates; each review has nullable `provider_id`/`provider`. Requested page size 20; actual counts can differ; `sorting`: `7` recent (default), `6` highest, `5` lowest. |
| `agoda_hotel_review_comments` | `GET /v1/agoda/hotel/reviews/{hotel_id}/comments` | Same shape; `provider_id` defaults to `332` (Agoda); `3038` is Booking.com. Agoda currently returns mixed providers regardless — filter on each review's `provider_id`. |

The requested page size is 20; returned counts can differ. A page interleaves per-provider streams, each in the requested order, so the mixed page is not globally sorted. Agoda's own (332) stream can have an occasional review out of place. For date-range backfills, group by `provider_id` and stop per provider, not at the first old review on the page, after reading one more page past that provider's cutoff; keep `provider_id == 332` for Agoda-native reviews. Advertised total counts do not guarantee that every review is accessible.

Agoda review responses include nullable `owner_response` objects with nullable `text`, `published_date`, and `author` strings. Reply dates retain source formatting. `review_score` preserves the source aggregate (`overall`, `score_text`, `review_count`), never a page average; missing aggregates can be `{}` or null, including provider comments.

## Trip.com

| MCP tool | REST | What it does |
|---|---|---|
| `tripcom_search_hotels` | `GET /v1/tripcom/hotels/search` | Positive Trip.com `city_id` and required `check_in`/`check_out`; USD default, adults 2/rooms 1 (1–6), limit 10 (1–20). Optional `hotel_name` (trimmed 1–200 characters, no control characters) requests a native name keyword within the required city/stay; returned `hotel_name` is normalized or null. First page only, no children. Nullable `price`: displayed advertised amount has unspecified tax inclusion; use source `total_amount` with its label and `notes` for local charges, never multiply rounded display prices. Returned hotel IDs feed reviews.  Selected-room names/beds/displayed guests/labels and hotel-card context are source descriptions, not policy or capacity guarantees. Crossed-out prices are source values. |
| `tripcom_hotel_reviews` | `GET /v1/tripcom/hotel/reviews/{hotel_id}` | Paginated reviews (`page_size` 1–50) through the async comment-info API and residential proxy. Per-review `source`: `tripcom`, `ctrip`, `expedia`, `hotelscom`, `tripadvisor`, or `unknown`. Aggregate Expedia ratings do not prove row provenance; upstream credential fields are removed.  `total_reviews` is the hotel aggregate; `pageable_reviews` mirrors `totalCountForPage` for the current feed and does not promise all aggregate rows are retrievable. A positive aggregate with zero pageable rows, or a later out-of-range page, returns an empty successful page. Missing or malformed upstream payloads return an upstream error, not an empty success. |

Ctrip destination lookup is REST-only: `GET /v1/ctrip/destinations/lookup?query=Hangzhou&country=China`. The optional `country` is an exact English or localized country name. Ambiguous cities return 409 with candidates; regions return 422 with typed municipality suggestions. Supply `country` to help resolve a place within the intended country. Parent-place matches require the original name after case, accent, punctuation and spacing normalization, plus an explicit matching country; similar hotel names and wrong-country places are not substitutes. Exact supported places can return their containing Ctrip search city with `resolution=parent_city`; `name`/`city_id` describe that search city, not necessarily a legal municipality. `matched_place_name`, `matched_place_type`, and nullable `matched_place_id` preserve the matched place on the result and suggestions. Direct city matches, including verified full native aliases such as Sevilla/Seville, take priority over place-parent matches and use `resolution=city` with null matched-place fields. Short-name hierarchy ambiguity stays 409: use the full `Castellón de la Plana` name for city 41132 rather than treating `Castellón` as an unambiguous municipality. Exact cities take precedence over same-name provinces; province errors carry `region_type=province`, `destination_id`, and null `city_id`. Conflicting parent cities remain 409; hotel-only or unmatched names remain 404. Select a verified city ID before hotel search; lowering `limit` does not force a match. `country_en` is the provider-supplied English country name or null on the selected city and suggestions. A 503 `LOOKUP_CAPACITY_EXCEEDED` includes `Retry-After`; wait before retrying. Per-user rate limits remain 429.

## Meta (geocoding)

| MCP tool | REST | What it does |
|---|---|---|
| `meta_coordinates_lookup` | `GET /v1/meta/coordinates-lookup` | Place name → candidate locations with `latitude`/`longitude`, type, country. Feeds OpenTable and Accor search. |

## Display (ChatGPT hotel cards)

| MCP tool | REST | What it does |
|---|---|---|
| `show_hotels` | — | Renders hotels you already fetched as visual cards + compare view in MCP Apps clients (ChatGPT). Pass name, photo_url, price/currency/price_per, rating/rating_max, review_count, address, url, provider. Fetches nothing; not billed. |

---

## URL vs ID — which to pass

- **ID always wins** when you have one: no resolution step, no extra latency or quota.
- Booking, Airbnb, and TripAdvisor single-item tools accept `url` directly. Airbnb and TripAdvisor URL parsing is instant; **Booking URL resolution takes 5–40 s** (real browser).
- Radisson, OpenTable, Agoda, and WeHotel keep resolution explicit: call their `*_url_to_id` tool once, then use the id everywhere. HolidayCheck also supports hotel-name search, which returns UUIDs directly; its URL resolver remains local and makes no hotel-existence request. Agoda's resolver costs a quota unit.
- Cache every resolved id for the rest of the session — they're stable.

## Quota, billing, and errors

- Billable: every successful data `tools/call` / REST data request — one unit each. Not billed: MCP housekeeping (`initialize`, `tools/list`), the `show_hotels` display tool, and unknown tool names.
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
| `feature_unavailable` | A requested feature is not built | Offer an alternative supported workflow |

Auth failures surface as HTTP 401/403 (JSON-RPC `-32001` on MCP) — the key is missing, wrong, or the account is blocked.

## Connecting an MCP client

Full guide with screenshots and troubleshooting: https://stayapi.com/docs/mcp. Replace `YOUR_API_KEY` interactively — never write the key into a file you might commit.

**Claude (web, desktop, mobile)** — no key: **Settings → Connectors → Add custom connector**, URL `https://api.stayapi.com/mcp`, then **Connect**, sign in to StayAPI, and **Allow**. The connector signs in with OAuth and syncs across Claude apps; the user revokes it under Connected apps on https://stayapi.com/user/mcp.

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

**Claude Desktop, key-based alternative** — `claude_desktop_config.json` (macOS: `~/Library/Application Support/Claude/`; Windows: `%APPDATA%\Claude\`), then fully quit (⌘Q) and relaunch. The no-space `X-API-Key:YOUR_API_KEY` form is deliberate — Claude Desktop re-tokenizes args containing spaces, and `mcp-remote` is pinned because unpinned builds throw `Unexpected content type: null`:

```json
{
  "mcpServers": {
    "stayapi": {
      "command": "npx",
      "args": ["-y", "mcp-remote@0.1.18", "https://api.stayapi.com/mcp", "--header", "X-API-Key:YOUR_API_KEY"]
    }
  }
}
```

ChatGPT works in developer mode (web, paid plans): chatgpt.com/plugins → **+** → **Create MCP app** → URL `https://api.stayapi.com/mcp`, auth OAuth (leave client ID/secret blank) → approve on stayapi.com → install from personal plugins → use with @StayAPI. Batched JSON-RPC requests are rejected — one tool call at a time.

## Expedia

| MCP tool | REST | What it does |
|---|---|---|
| — | `GET /v1/expedia/destinations/lookup?query=` | Destination suggestions with numeric-string region IDs. |
| — | `GET /v1/expedia/hotel/lookup?query=&region_id=` | Dateless HOTEL name suggestions in region context; property IDs compose with reviews/rates. Verified no matches return HTTP 200, `properties: []`, `count: 0`; upstream failures remain errors. |
| — | `GET /v1/expedia/search?region_id=&check_in=&check_out=` | Dated primary cards. Required ordered dates, adults 1–8, offset 0–1000, limit 1–10; USD only, prices omitted. |
| — | `GET /v1/expedia/hotel/details?property_id=` | Undated verified identity and available metadata. The numeric Expedia property URL may redirect. No dates, prices, or availability. Additional nullable metadata: address.region, telephone, image, property_type, room_count, check_in_time/check_out_time and identity-verified structured_data (original property JSON-LD). Independently confirmed missing properties return 404 NOT_FOUND; empty metadata alone or uncertain upstream failures remain 502. |
| `expedia_hotel_rates` | `GET /v1/expedia/hotel/rates?property_id=&check_in=&check_out=` | Live room offers. MCP `children_ages` is a JSON integer array; REST uses comma-separated ages. Both accept up to six ages from 0–17; omit or use an empty MCP array for adults-only rates. Adults 1–10, ordered nonpast YYYY-MM-DD dates. Currency defaults to USD; other three-letter codes are accepted but conversion is not confirmed. Finite nonnegative priced available inventory or explicit sold-out empties are valid; malformed or contradictory availability is a nonchargeable upstream error. |

## Vrbo

| MCP tool | REST | What it does |
|---|---|---|
| — | `GET /v1/vrbo/destinations/lookup?query=` | Native destination region IDs. |
| — | `GET /v1/vrbo/search?region_id=&check_in=&check_out=` | Dated discovery; same date/adult/page/USD bounds as Expedia. Internal `property_id` is for reviews; public `listing_id` (possibly ending in `ha`) is for URLs. |

For both providers, search responses (and Expedia property-name lookup) report `availability_verified: false`; destination suggestions do not carry that field. Continue only using `next_offset`/`next_limit`, never infer completeness from count. Expedia review items include nullable `owner_response` objects (`text`, `published_date`, `author`, all nullable strings); only the first management reply is returned and its date retains source formatting. Native Vrbo property-name lookup is not available; do not substitute a search engine or fabricate a resolver. Expedia exposes only `expedia_hotel_rates` through MCP; Expedia discovery/details/reviews and all Vrbo operations use REST.

### Booking family rates and labels

`booking_hotel_rooms` and `booking_search_hotels` accept `children` (0–10) and
`children_ages` (comma-separated ages 0–17, exactly one per child). Use
`children=1, children_ages="8"` for a child aged eight. Zero children needs no ages.
Price objects return string `labels` (empty when absent), nullable numeric
`genius_discount_pct`, and boolean `is_mobile_rate`. These describe explicit
anonymous rate markers, never hotel Genius eligibility or sign-in-only offers.
Rooms retain `genius_discount_percentage` and `price.discount_name`.
`rates[].fits_requested_occupancy` is true only when one rate fits the entire
requested party; false/null alternatives must not be used as a full-family quote.
REST prices has the same occupancy and labels contract; its summary is per-room,
not a combined multi-room reservation total. See the public Booking endpoint documentation at https://stayapi.com/docs/endpoints/booking-hotel-rooms.


### Booking point-of-sale options

`booking_search_hotels` and `booking_hotel_rooms` accept optional ISO `market` (ES: es/EUR; US: en-us/USD), `language`, and `currency`. Explicit display overrides win. Legacy `country_market` retains en-us/USD defaults and must match `market` if both are supplied. Other configured defaults and fallback are documented in the Booking endpoint reference. Compare identical occupancy, stay, currency and room/rate; tax-inclusive versus tax-exclusive headlines are not discounts. `effective_market` reports requested_country, nullable targeted_country, language, currency, country_verified=false and verified_country=null. Targeting is best-effort, not verified geography.

For Agoda provider comments, absent aggregates serialize as `review_score: null` in REST; the MCP comments result may omit `review_score`. Never infer it from a page.
