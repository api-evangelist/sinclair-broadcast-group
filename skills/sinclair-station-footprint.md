---
name: sinclair-station-footprint
description: >-
  Retrieve and analyze Sinclair, Inc.'s complete television station footprint — call
  letters, network affiliation, Nielsen DMA and rank, ownership arrangement, digital
  subchannels and station websites — from the only machine-readable publication of it
  that exists. Use when asked which markets Sinclair operates in, which stations carry
  a given network, how many stations Sinclair owns outright versus operates under a
  services agreement, or where a specific call sign sits in the footprint.
api: Sinclair Corporate Content API
base_url: https://sbgi.net/wp-json
operations:
  - getStationMap
  - getRsnMap
generated: '2026-08-12'
method: generated
source: openapi/sinclair-broadcast-group-content-openapi.yml
---

# Sinclair station footprint

Sinclair publishes no API documentation, but its corporate site serves the footprint
as JSON. This is the authoritative machine-readable source for the question "where does
Sinclair operate?"

## Before you start

- No authentication. No API key. Do not send an `Authorization` header.
- Call from the server side. `Access-Control-Allow-Origin` is pinned to
  `https://sbgi.net`, so a browser on any other origin will be blocked by CORS.
- Pace at one request per ten seconds — `robots.txt` declares `Crawl-delay: 10` and it
  is the only pacing number Sinclair publishes. Responses carry `max-age=600`, so
  polling faster than once per ten minutes returns nothing new anyway.
- There are no rate-limit headers. You get no backoff signal. Be conservative.

## Step 1 — Fetch the footprint

Call `getStationMap`:

```
GET https://sbgi.net/wp-json/sbg/v1/station-map
```

This route takes **no parameters**. There is no pagination, no filtering, no field
selection. It returns the entire collection — about 136 KB on 2026-08-12. Fetch it
once and filter locally. Do not call it in a loop.

The response is an array of location records:

```
[ { "location": [37.68888, -97.388134],
    "address":  "316 North West Street, Wichita, KS 67203",
    "stations": [ { "callLetter": "KAAS",
                    "station": "KAAS",
                    "affiliation": "FOX",
                    "dma": "Wichita - Hutchinson, KS",
                    "dmaRank": "70",
                    "stationStatus": "O&O",
                    "stationURL": "https://www.foxkansas.com",
                    "stationLogo": "",
                    "newsletterURL": "",
                    "subStations": [ { "callLetter": "KAAS-2", "affiliation": "MyTV", ... } ] } ] } ]
```

## Step 2 — Flatten before you count

The shape is three levels deep and counting it wrong is the most common mistake:

- **Location records** are keyed by street address, not by market. Two Sinclair
  facilities in one DMA are two records. Do not count locations as markets.
- **Primary stations** live under `stations[]`.
- **Digital subchannels** live under `stations[].subStations[]` — this is where the
  Comet, Charge!, Roar and The Nest multicast networks are carried.

To count markets, take the distinct set of `dma` values, not the length of the array.
Measured on 2026-08-12: 101 location records, 197 primary stations, 365 subchannels,
87 distinct DMAs.

## Step 3 — Read the fields correctly

- `dmaRank` is a **string**, not an integer. Cast before sorting or you will get
  lexicographic order and rank 10 will sort before rank 2.
- `affiliation` is free text, not an enumerated code. Normalize before grouping —
  observed values include `FOX`, `ABC`, `CBS`, `NBC`, `CW`, `MyTV`, `Univision`,
  `Independent`, and Sinclair's own `Roar` and `Comet`.
- `stationStatus` is the ownership arrangement and is the field to use when asked what
  Sinclair actually *owns* versus operates. Observed values: `O&O` (owned and
  operated), `JSA` (joint sales agreement), `LMA` (local marketing agreement), `MSA`
  (management services agreement), `OSA`, and `Services Arrangement`. Only `O&O` is
  ownership. On 2026-08-12 the split was 160 O&O, 15 MSA, 14 JSA, 5 LMA, 2 OSA, 1
  services arrangement.
- `stationLogo` and `newsletterURL` are frequently empty strings, not null. Test for
  truthiness, not for presence.
- `address` is duplicated from the parent location record onto each station.

## Step 4 — Regional sports networks, with a caveat

`getRsnMap` returns 19 territories:

```
GET https://sbgi.net/wp-json/sbg/v1/rsn-map
```

**Do not present this as current.** The payload still carries Bally Sports titles and
`ballysports.com` links. Sinclair's regional sports networks moved to Main Street
Sports Group and then to FanDuel Sports Network; this route has not tracked either
rebrand. Treat it as a historical snapshot and say so when you use it. The `teams`
field is an HTML fragment, not a structured list — parse it, do not render it raw.

## Error handling

Errors are not RFC 9457. The envelope is `{"code", "message", "data": {"status"}}`.
Branch on `code`, never on `message`.

- `rest_no_route` (404) — the path is wrong. Verify against https://sbgi.net/wp-json/.
- `rest_forbidden` (401) — you have wandered onto a privileged route. There is no
  credential that fixes this; return to the public surface.

## What this data cannot tell you

There is no station-to-network entity, no carriage or distribution data, no revenue or
audience figures, and no join between a station and any press release. The `sbg/v1`
and `wp/v2` graphs do not share a key.
