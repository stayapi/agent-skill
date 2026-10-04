# StayAPI MCP discovery

Use discovery when the requested provider or operation is unfamiliar, a parameter is
ambiguous, or the client shows fewer tools than this skill describes. Do not guess a
tool name, parameter spelling, count limit, or child-occupancy shape.

## MCP workflow

1. Let the MCP client discover tools automatically, or request the standard
   `tools/list` protocol method when the client exposes it. It is authenticated
   housekeeping and does not consume a StayAPI quota unit.
2. Choose an advertised tool and read its `description` and `inputSchema`. The schema
   identifies required fields, JSON types, bounds, and defaults. The list reflects the
   account's enabled tools, so an omitted tool is not available in that connection.
3. Make one correctly shaped `tools/call`. Save returned provider IDs and cursors for
   the next call instead of resolving the same item again.
4. Interpret a structured result before retrying. `invalid_input` means change the
   request; honor `rate_limited.retry_after` when supplied; `upstream_error` may justify
   one delayed retry; `no_results` is a valid negative result.

Example recovery, using the schema for `expedia_hotel_rates`:

```text
tools/call expedia_hotel_rates(property_id="10507369", check_in="2027-06-10", check_out="2027-06-10")
→ {"error":"invalid_input","field":"check_in/check_out","message":"Check-in must not be in the past and check-out must follow it."}

tools/list
→ expedia_hotel_rates.inputSchema says property_id is a numeric string;
  check_in/check_out are required YYYY-MM-DD strings; adults is 1–10;
  children_ages is an optional JSON array of at most six integer ages 0–17.

tools/call expedia_hotel_rates(property_id="10507369", check_in="2027-06-10", check_out="2027-06-12", adults=2, children_ages=[])
```

The corrected example shows input shape only; a live rate result remains subject to
availability and provider response. For an Expedia name, use the REST discovery path
in [tools.md](tools.md#expedia) to obtain a numeric `property_id` first.

## REST fallback

REST has no equivalent generic per-account discovery endpoint. Use
[tools.md](tools.md) to map the operation to a REST path, then consult the endpoint's
OpenAPI schema or public documentation before calling it. Treat non-2xx RFC 7807
responses as errors and correct their reported field/constraint before retrying.

## Local verification helper

From a source checkout, `python scripts/inspect_mcp_schema.py --tool <name>` prints
the registered MCP schema without contacting StayAPI or an upstream provider. It is a
development aid: it shows every locally registered tool and cannot apply an account's
tool preferences. Connected agents should use remote `tools/list` instead.
