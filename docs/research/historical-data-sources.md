# Historical flight data source research

This note records the September 2026 investigation into retrieving historical flight tracks without requiring a Flightradar24 API subscription, token or account authentication.

The target workflow is:

```text
flight number + date
        ↓
find matching historical flight instances
        ↓
show one or more candidates
        ↓
select a flight
        ↓
retrieve timestamped coordinates
        ↓
export KML
```

The main conclusion is that an unauthenticated workflow is feasible for at least about three months of history using Airplanes.live historical replay and trace data. Flightradar24's anonymous endpoints remain useful for recent history but are limited to about seven days.

## Scope and requirements

The desired user input is:

- Flight number, for example `LH732`, `QF27` or `AS61`
- Calendar date

The result should be a list of matching flight instances. Multiple instances may exist on the same date because a single flight number can cover several legs.

Each candidate should expose as much of the following as possible:

- Flight number or ADS-B callsign
- Origin and destination
- Departure time
- Aircraft registration
- Aircraft type
- Source-specific flight or aircraft identifier

After the user selects one candidate, the system needs a historical track containing at least:

- Timestamp
- Latitude
- Longitude
- Altitude where available

KML generation is not the difficult part once these points are available.

## Sources investigated

### Official Flightradar24 API and pyfr24

The upstream `pyfr24` project uses Flightradar24's official API and an API token.

It already provides useful application-level behavior such as:

- Search by flight and date
- Multiple-match selection
- Track retrieval
- KML, CSV and GeoJSON export
- CLI handling and tests

This makes `pyfr24` a useful application foundation, but its official data source does not meet the no-authentication requirement.

The current fork should therefore keep the data-retrieval layer separable from higher-level selection and export logic.

### Unofficial Flightradar24 JSON endpoints

The `igolaizola/fr24` project was used to test Flightradar24's web-facing JSON endpoints.

The relevant endpoints are:

```text
https://api.flightradar24.com/common/v1/flight/list.json
https://api.flightradar24.com/common/v1/flight-playback.json
```

These were tested without credentials.

#### Recent historical lookup

Anonymous search succeeded for recent flights.

For example, querying `LH732` returned a completed flight on September 20, 2026:

```text
FR24 ID:      41c178a0
Route:        EDDF → ZSPD
Registration: D-AIXD
Aircraft:     A359
STD:          2026-09-20 19:30 UTC
ATD:          2026-09-20 19:55:12 UTC
```

Anonymous playback for that flight returned:

```text
HTTP 200
Track points: 2,910
Response size: about 664 KB
```

The first and last parsed coordinates were:

```text
first: lat 50.04755, lon 8.56270
last:  lat 31.15701, lon 121.80597
```

This proves the recent-history chain:

```text
flight number
    ↓
FR24 flight ID
    ↓
historical playback
    ↓
coordinate track
```

#### Multiple same-day flight instances

`AS61` was used to validate multiple same-day matches because the same flight number covers several legs.

For September 14, 2026 the anonymous FR24 endpoint returned four completed legs:

| FR24 ID | Route | Registration | Type | Scheduled departure |
| --- | --- | --- | --- | --- |
| `41a84020` | KSEA → PAJN | N553AS | B738 | 14:03 UTC |
| `41a908c5` | PAJN → PAYA | N553AS | B738 | 17:39 UTC |
| `41a95d42` | PAYA → PACV | N553AS | B738 | 19:29 UTC |
| `41a9b60e` | PACV → PANC | N553AS | B738 | 21:14 UTC |

Playback succeeded for every leg:

| Route | Track points |
| --- | ---: |
| KSEA → PAJN | 502 |
| PAJN → PAYA | 188 |
| PAYA → PACV | 91 |
| PACV → PANC | 328 |

This validates the desired candidate-selection model.

#### Pagination behavior

The flight-list endpoint returns future scheduled flights before older completed flights.

For a multi-leg number such as `AS61`, page 1 can be filled by future schedules before reaching the requested historical date. The September 14 records appeared on page 2.

A historical search implementation must therefore paginate until it has moved past the target date. It cannot assume page 1 is sufficient.

#### Anonymous history limit

The anonymous endpoint enforces a server-side history window.

Tested on September 21, 2026:

| Requested timestamp | Result |
| --- | --- |
| September 21 | HTTP 200 |
| September 20 | HTTP 200 |
| September 15 | HTTP 200 |
| September 14 | HTTP 200 |
| September 13 | HTTP 400 |
| September 1 | HTTP 400 |
| April 22, 2025 | HTTP 400 |

The rejected response identifies `timestamp` as an invalid parameter.

Adding:

```text
filterBy=historic
```

did not extend the window.

The September 14 `LH732` playback endpoint still returned 2,672 points, confirming that both lookup and playback work at the approximate seven-day boundary.

#### gRPC historical trail

The alternative gRPC `HistoricTrail` path was also tested.

It returned:

```text
HTTP 200
response body: 0 bytes
```

It is not currently a useful alternative to the JSON playback endpoint.

#### Conclusion for anonymous FR24 access

Anonymous FR24 web endpoints are usable for recent flights but do not solve the longer-history requirement.

A provider based on them would be useful as a recent-history source:

```text
RecentFR24Provider
    history: approximately seven days
    authentication: none
    metadata quality: high
    track quality: high
```

It should not be treated as the primary historical source.

## Airplanes.live historical data

Airplanes.live provides two complementary public historical data products:

1. Historical replay chunks containing aircraft state, callsign and ICAO hex
2. Per-aircraft full-day trace files containing detailed position history

Both were tested without authentication.

### Per-aircraft historical traces

The trace URL follows this pattern:

```text
https://globe.airplanes.live/globe_history/YYYY/MM/DD/traces/HH/trace_full_HEX.json
```

where:

- `HEX` is the aircraft ICAO 24-bit address
- `HH` is the last two characters of the hex address

Examples were successfully retrieved for history well beyond the FR24 anonymous limit.

#### Approximately 37 days old

For `D-AIXD`, ICAO hex `3c6704`, on August 15, 2026:

```text
HTTP 200
Aircraft: A359
Registration: D-AIXD
Track points: 3,550
Response size: about 703 KB
Observed callsigns:
- DLH712
- DLH713
- DLH732
- DLH768
```

#### Approximately 52 days old

The same aircraft on July 31, 2026:

```text
HTTP 200
Track points: 3,255
Response size: about 640 KB
```

#### Approximately 97 days old

For `VH-ZNB`, ICAO hex `7c8065`, on June 16, 2026:

```text
HTTP 200
Aircraft: B789
Registration: VH-ZNB
Track points: 1,149
Observed callsigns:
- QFA27
- QFA28
```

This demonstrates usable anonymous trace history at roughly three months.

### Isolating a single flight from an aircraft-day trace

A daily aircraft trace may contain multiple flights.

The trace format stores callsign metadata as state changes rather than repeating it on every point. The parser must therefore carry the most recent callsign forward until it changes.

The June 16 `7c8065` trace was segmented this way.

For `QFA27`:

```text
Selected points: 1,004
Start: 2026-06-16 02:29:04 UTC
End:   2026-06-16 17:26:06 UTC
```

The first selected coordinate was near Sydney:

```text
-33.936325, 151.169200
```

The final selected coordinate was near Santiago:

```text
-33.461472, -70.798425
```

The next callsign transition in the same aircraft trace was:

```text
QFA27 → QFA28
```

This proves that a full-day aircraft trace can be reduced to the requested flight instance.

### Historical replay files as the flight-number index

Knowing the aircraft hex is not sufficient because the desired user input is flight number plus date.

Airplanes.live also exposes replay files under:

```text
https://globe.airplanes.live/globe_history/YYYY/MM/DD/heatmap/NN.bin.ttf
```

Despite the `.bin.ttf` suffix, the tested files were raw binary rather than gzip-compressed files.

The replay format contains:

- Timestamp
- Aircraft ICAO hex
- Callsign
- Latitude
- Longitude
- Altitude
- Ground speed
- Other ADS-B state

The files are approximately 30-minute chunks.

For example, chunk `17.bin.ttf` started around 08:30 UTC in the tested archive. A typical day therefore has about 48 chunks.

Sample chunk sizes were approximately 7 to 11 MB.

#### Flight-number to aircraft-hex validation

A narrow group of August 15 replay chunks around the `LH732` flight was decoded while preserving callsign state across chunks.

It resolved:

```text
DLH732 → 3c6704
```

Similarly, June 16 replay chunks around `QFA27` resolved:

```text
QFA27 → 7c8065
```

The June lookup was about 97 days old at the time of testing.

This validates the complete anonymous chain:

```text
flight number + date
        ↓
IATA flight number → ICAO callsign
        ↓
historical replay chunks
        ↓
callsign → aircraft ICAO hex
        ↓
per-aircraft historical trace
        ↓
isolate selected callsign interval
        ↓
timestamped coordinates
        ↓
KML
```

### Replay decoder observations

A third-party tar1090/readsb replay decoder was used during validation:

```text
https://github.com/WeegeeNumbuh1/tar1090-replay-analyzer
```

Two compatibility issues were observed:

1. It assumes a `.ttf` replay file is gzip-compressed. Airplanes.live served the tested files uncompressed.
2. Its callsign-record path packs signed 32-bit values as unsigned without masking. A June replay record triggered a `struct.error`.

The validation harness worked around these issues by:

- Treating downloaded files as raw `.bin`
- Masking affected values with `0xFFFFFFFF` before unsigned packing

These findings should be considered if replay decoding code is incorporated into this project. Reimplementing the small required subset may be preferable to taking the third-party decoder as a dependency.

## ADSB.lol historical data

ADSB.lol was also tested because it exposes a tar1090-style historical trace archive with a layout similar to Airplanes.live.

The tested per-aircraft trace URL pattern is:

```text
https://adsb.lol/globe_history/YYYY/MM/DD/traces/HH/trace_full_HEX.json
```

where:

- `HEX` is the aircraft ICAO 24-bit address
- `HH` is the last two characters of the hex address

No authentication, API key or account was used.

### Successful shorter-horizon trace test

For `D-AIXD`, ICAO hex `3c6704`, on August 15, 2026, approximately 37 days before the validation date:

```text
HTTP 200
Registration: D-AIXD
Aircraft: A359
Track points: 2,191
Observed callsigns:
- DLH712
- DLH713
- DLH732
- DLH768
```

The response was gzip encoded at the HTTP/content level and parsed successfully after decompression.

This is sufficient for the same downstream workflow used with Airplanes.live once the aircraft hex is known:

```text
aircraft hex + date
        ↓
ADSB.lol daily aircraft trace
        ↓
carry forward callsign state
        ↓
isolate the selected flight
        ↓
timestamped coordinates
        ↓
KML
```

### Tested history horizon

The same URL pattern was tested at longer horizons.

| Test | Approximate age | Result |
| --- | ---: | --- |
| D-AIXD on August 15, 2026 | 37 days | HTTP 200, 2,191 points |
| D-AIXD on July 31, 2026 | 52 days | HTTP 404 |
| VH-ZNB on June 16, 2026 | 97 days | HTTP 404 |

For comparison:

| Approximate age | Airplanes.live | ADSB.lol |
| ---: | --- | --- |
| 37 days | available | available |
| 52 days | available | not found |
| 97 days | available | not found |

These tests do **not** establish an exact ADSB.lol retention cutoff. They only show that the tested 37-day trace existed while the tested 52-day and 97-day traces did not.

The actual limit may depend on archive retention, aircraft/date coverage or source availability. It should therefore be probed dynamically rather than hard-coded as a fixed number of days.

### Potential role in the provider stack

ADSB.lol remains useful even if its historical window is shorter than Airplanes.live.

A provider could be modeled as:

```text
ADSBLOLProvider
    authentication: none
    historical aircraft traces: verified at about 37 days
    longer history: not verified
    candidate lookup: still requires a callsign/hex discovery strategy
    track output: compatible with the same normalization and KML pipeline
```

Possible uses include:

- A fallback if Airplanes.live is unavailable
- A second-source check for recent historical tracks
- A preferred source when its data quality or coverage is better for a particular aircraft or region
- A shorter-horizon provider if the requested date falls inside its available archive

Because the trace structure is closely related to the tar1090-style data already needed for Airplanes.live, supporting ADSB.lol should require relatively little additional normalization logic once the provider abstraction exists.

### Remaining ADSB.lol questions

Before relying on ADSB.lol as a formal provider, test:

- The actual retention boundary rather than only the 37, 52 and 97-day samples
- Whether historical replay/index files are available for callsign-to-hex discovery
- Regional and aircraft coverage consistency
- Whether missing traces are caused by retention or by source coverage for a specific day
- Rate limits and acceptable request frequency
- Licensing and terms for derived data

## Flight number and ADS-B callsign mapping

Passenger-facing flight numbers usually use IATA airline codes:

```text
LH732
QF27
UA2151
```

ADS-B callsigns usually use ICAO operator codes:

```text
DLH732
QFA27
UAL2151
```

The lookup layer therefore needs IATA-to-ICAO airline code conversion.

For common carriers this is straightforward, but the implementation should not assume that marketed flight number and ADS-B callsign are always a one-to-one mapping.

Known edge cases include:

- Codeshares
- Regional operators
- Wet leases
- Alternate ATC callsigns
- Repositioning flights
- Non-revenue flights

The search model should allow several callsign candidates where necessary.

## Efficiency considerations

Scanning all historical replay chunks for a date would be inefficient.

At roughly 48 chunks per day and about 7 to 11 MB per chunk, a full-day scan could transfer several hundred MB.

Two approaches can reduce this substantially.

### Search around expected departure time

If schedule information provides an approximate departure time, only nearby replay chunks need to be downloaded.

For example:

```text
scheduled departure: about 19:30 UTC

scan:
18:30
19:00
19:30
20:00
20:30
```

This was sufficient to resolve `DLH732` in the validation test.

### Build a local daily callsign index

For dates that are queried repeatedly, replay data can be streamed once and reduced to a compact index such as:

```json
{
  "QFA27": [
    {
      "hex": "7c8065",
      "first_seen": "02:29:00Z",
      "last_seen": "17:26:00Z"
    }
  ]
}
```

The raw replay chunks do not need to be retained after indexing.

A cache could use one file per date:

```text
cache/
    2026-06-16-flight-index.json
    2026-06-17-flight-index.json
```

Future searches for an indexed date would then avoid downloading replay data again.

## Candidate provider architecture

The research supports separating source-specific extraction from the existing pyfr24 selection and export behavior.

A possible provider model is:

```text
FlightDataProvider
    │
    ├── OfficialFR24Provider
    │      official API
    │      authenticated
    │
    ├── RecentFR24Provider
    │      FR24 web JSON endpoints
    │      anonymous
    │      about seven days verified
    │
    ├── AirplanesLiveProvider
    │      historical ADS-B replay and traces
    │      anonymous
    │      at least about 97 days verified
    │
    └── ADSBLOLProvider
           historical ADS-B traces
           anonymous
           about 37 days verified
           longer history not yet established
```

The provider boundary should expose normalized domain objects instead of raw provider response structures.

For example:

```python
class FlightDataProvider(Protocol):
    def search_flights(
        self,
        flight_number: str,
        date: date,
    ) -> list[FlightCandidate]:
        ...

    def get_track(
        self,
        flight: FlightCandidate,
    ) -> FlightTrack:
        ...
```

The existing higher-level workflow can then remain source-independent:

```text
search
    ↓
show candidates
    ↓
select
    ↓
retrieve normalized track
    ↓
KML / CSV / GeoJSON
```

## Recommended next proof of concept

Before refactoring the main pyfr24 code, implement a small standalone Airplanes.live proof of concept.

Suggested interface:

```bash
python historical_flight_poc.py --flight QF27 --date 2026-06-16
```

Expected output:

```text
Found 1 match:

[0] QFA27 | VH-ZNB | B789 | 02:29–17:26 UTC | 1,004 points
```

The proof of concept should then write:

```text
QF27_2026-06-16.kml
```

This would validate the complete user-facing workflow before introducing provider abstractions into pyfr24.

## Open questions

The following should be tested before treating Airplanes.live as a production-quality provider:

- Maximum available history rather than the approximately 97 days tested so far
- Archive retention policy and whether it is documented
- Coverage consistency across regions
- Missing or changing callsign data during a flight
- Multiple aircraft using the same callsign on the same date
- Codeshare and regional-operator resolution
- Whether route metadata should come from another source or be inferred from track endpoints
- How many replay chunks should be searched when no schedule time is available
- Appropriate local caching and retry behavior
- Licensing and terms applicable to redistribution or derived data

## Validation artifacts

The research was performed on branch:

```text
test/fr24-source-verification
```

Temporary GitHub Actions workflows on that branch include:

- `.github/workflows/fr24-source-verification.yml`
- `.github/workflows/adsb-source-verification.yml`
- `.github/workflows/airplanes-replay-verification.yml`

These workflows are validation artifacts rather than intended production CI and can be removed after the relevant implementation decisions are made.

The validation included live requests to current public endpoints, so availability and behavior should be treated as time-sensitive and retested if implementation occurs materially later.
