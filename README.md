# StayAPI Agent Skill

An [agent skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches AI assistants to fetch **live hotel, vacation-rental, and restaurant data** through [StayAPI](https://stayapi.com): Booking.com, Airbnb, Google Hotels, Google Travel, Google Reviews, TripAdvisor, Agoda, Trip.com, Accor (ALL), Radisson, Marriott Bonvoy, Hilton, WeHotel (Jin Jiang), MakeMyTrip India, and OpenTable.

With the skill installed, prompts like these just work:

> *"Find the top hotels in Lisbon for Sep 11–13 with prices"*
> *"Get the 20 most recent guest reviews for this Booking.com hotel"*
> *"What do Google reviews say about the Bellagio?"*
> *"Find Hilton hotels in London and get starting cash room rates for my dates"*
> *"Which OpenTable restaurants near the Eiffel Tower have a table for 4 tonight?"*

The skill guides the assistant to the right StayAPI capability — over the [MCP server](https://stayapi.com/docs/mcp) when connected, or the plain REST API otherwise — and encodes the gotchas that trip up agents (signed destination IDs, stay-total vs nightly prices, per-platform sort parameters, URL-vs-ID resolution costs).

## Requirements

A StayAPI API key — sign up at [stayapi.com](https://stayapi.com) and copy it from the dashboard. The skill never asks you to paste the key into files or chats; it reads the `STAYAPI_API_KEY` environment variable or uses your already-configured MCP connector.

## Install

### Claude Code (plugin marketplace)

```
/plugin marketplace add stayapi/agent-skill
/plugin install stayapi@stayapi
```

### Claude Code / any skills-aware agent (manual)

Copy `skills/stayapi/` into your skills directory, e.g.:

```bash
git clone https://github.com/stayapi/agent-skill.git
cp -R agent-skill/skills/stayapi ~/.claude/skills/stayapi
```

### Claude.ai / Claude Desktop

Download `stayapi.skill` from the [latest release](https://github.com/stayapi/agent-skill/releases/latest) and upload it under **Settings → Capabilities → Skills**.

## What's inside

```
skills/stayapi/
├── SKILL.md                  # triggers, auth rules, transport selection, core workflow guide
└── references/
    ├── workflows.md          # worked examples: destination lookup, search, details, reviews,
    │                         # Google Hotels, Airbnb — MCP sequences + copy-paste REST curl
    └── tools.md              # full MCP tool catalog and corresponding REST endpoints
```

The skill contains documentation only — all data comes from your StayAPI account, billed one quota unit per successful call, same as the REST API.

## Versioning

Releases are tagged (`v0.1.0`, …) with a packaged `.skill` file attached. The skill is developed and validated alongside the StayAPI codebase; this repository is its public distribution home.

## Links & support

- Documentation: https://stayapi.com/docs
- MCP setup guide: https://stayapi.com/docs/mcp
- Support: info@stayapi.com

Licensed under [MIT](LICENSE).
