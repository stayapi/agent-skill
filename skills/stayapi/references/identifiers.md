# StayAPI identifiers and resolvers

Resolve once, preserve the exact returned identifier, then use it on the provider's
next call. IDs are provider-specific; a search card is not a confirmed booking.

| You have | Resolve with | Reuse as |
|---|---|---|
| Booking city/area | `booking_lookup_destination` with the bare city name | Signed `dest_id` for `booking_search_hotels`; adding a region can bias results toward HOTEL matches. |
| Booking hotel URL | `GET /v1/booking/hotel/url-to-id`, or pass URL to supported detail/review/room tools | Numeric `hotel_id`; browser-backed, 5–40 seconds. |
| Airbnb destination | `airbnb_search_listings(location=...)` | First-page listing cards only; no property-name lookup. |
| Airbnb URL | Pass URL directly | `listing_id`; local parse. |
| Expedia destination/name | REST `/v1/expedia/destinations/lookup`, then `/v1/expedia/hotel/lookup` | `region_id`, then numeric `property_id` for rates/reviews. |
| Vrbo destination | REST `/v1/vrbo/destinations/lookup`, then dated `/v1/vrbo/search` | Internal review `property_id` and distinct public `listing_id`. |
| Google Travel hotel name | `google_travel_resolve` or `google_travel_search` | Opaque `entity_token` for the other Google Travel tools. |
| Google Maps place | `google_reviews_for_place(query=...)` | Returned `data_id` for review pagination. |
| Place coordinates | `meta_coordinates_lookup` | Latitude/longitude for OpenTable and Accor. |
| TripAdvisor place/URL | `tripadvisor_geo_search` or pass URL | `geo_id` / `location_id`. |
| Agoda, Radisson, OpenTable, WeHotel URL | Provider `*_url_to_id` tool | Provider-specific ID. |
| HolidayCheck hotel name/URL | `holidaycheck_hotels_search` / `holidaycheck_hotel_url_to_id` | Lowercase UUID. |
| Marriott property | `marriott_bonvoy_search` by coordinates | `property_id` for rooms. |
| Hilton URL/place | `hilton_hotel_url_to_id` or `hilton_search` | Seven-character `hotel_code`. |
| Otelpuan name/URL | `otelpuan_search_hotels` or rooms/url-to-id route | Numeric review ID or canonical rooms URL, depending on operation. |
