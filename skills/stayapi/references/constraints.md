# StayAPI provider constraints

- Use three-letter uppercase currency codes. Invalid currency fails before the provider call.
- Booking city `dest_id` values are signed. Preserve the minus sign. Hotel URLs must be
  `https://www.booking.com/hotel/{cc}/{slug}.html`; URL resolution is browser-backed.
- Booking search prices are whole-stay totals under `data.hotels`, not nightly amounts.
  Results retain Booking's ranking; over-fetch and sort client-side only when needed.
- Booking families require `children` plus exactly that many comma-separated
  `children_ages` (0–17). Only rates with `fits_requested_occupancy: true` support a
  full-family comparison; price summaries are per room.
- Booking rate labels describe explicit anonymous rate markers. Treat a Genius label as
  evidence that a returned rate applied it, never as evidence of hotel eligibility.
- Airbnb listing IDs must remain decimal strings end to end. Send MCP `listing_id` as a quoted string for 19-digit values; legacy numeric inputs stop at `9007199254740991`.
- Airbnb details require paired dates. Airbnb search is first-page destination discovery,
  with optional date-qualified displayed prices only; it never confirms availability.
- Expedia rates require a numeric `property_id`, ordered nonpast dates, and optional
  `children_ages` as a JSON integer array of at most six ages 0–17. Expedia/Vrbo
  discovery is REST-only; discovery cards omit prices and do not confirm availability.
- Sort names differ: Booking/HolidayCheck `recent_desc`, Airbnb `MOST_RECENT`, Google
  reviews `newest`, Agoda `7`. Check the tool schema before a chronological backfill.
- Page caps: Booking 25, Airbnb 50, HolidayCheck 10, Agoda 20, OpenTable 25, WeHotel 20;
  Accor reviews are capped at 20 total. Trip.com reviews are paginated.
- Booking market parameters select locale/currency defaults; explicit values win and
  legacy `country_market` keeps its defaults. Compare quotes only with the same market,
  currency, occupancy, rate selection, and tax scope. A requested or targeted country
  does not prove a verified network exit there.
