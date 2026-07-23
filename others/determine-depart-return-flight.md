# Segment Journey Classification (Departure vs Return)

## Flowchart

```mermaid
flowchart TD
    A["input: segments [] = all SOAP reservation segments"] --> B{"len(segments) ≤ 1?"}
    B -- yes --> C["return (segments, nil)\nall = departure, no return"]
    B -- no --> D{"first.DepartureAirport ==\nlast.ArrivalAirport?"}
    D -- no (one-way) --> C
    D -- yes (round-trip candidate) --> E["findReturnSplitIndex()\nscan adjacent segment pairs"]

    E --> F["i = 1, tzCache = {}"]

    F --> G{"i < len(segments)?"}
    G -- no --> C

    G -- yes --> H["get timezone of\nsegments[i-1].ArrivalAirport\nvia locationClient.Lookup()"]
    H --> I["get timezone of\nsegments[i].DepartureAirport\nvia locationClient.Lookup()"]

    I --> J["parse segments[i-1].ArrivalDateTime\nin prev airport timezone"]
    J --> K["parse segments[i].DepartureDateTime\nin next airport timezone"]

    K --> L{"parse OK\nto both?"}
    L -- no --> M["i++"]
    L -- yes --> N{"nextDeparture - prevArrival\n> 12 hours?"}

    N -- no --> M
    N -- yes --> O["return i as split index"]

    M --> G

    O --> P{"splitIdx valid?\n(0 < i < len)"}

    P -- yes --> Q["return (segments[:splitIdx], segments[splitIdx:])\ndeparture = first half\nreturn = second half"]
    P -- no --> C
```

## Summary

**Location:** `internal/service/details.go:400-456`

The function `groupSegmentsForJourney` classifies all reservation segments into **departure** and **return** journeys:

1. **Single/one segment** → all departure, no return.
2. **One-way** (first segment departure ≠ last segment arrival airport) → all departure, no return.
3. **Round-trip** → scan adjacent segment pairs. For each pair, resolve airport timezones via the location client, parse arrival/departure times, and **split at the first gap > 12 hours** between the previous segment's arrival and the next segment's departure.

If no gap > 12 hours is found, all segments are treated as departure (no return).

## Callers

| Caller | File | Purpose |
|--------|------|---------|
| `GetDetails` | `details.go:148` | Build departure + optional return journey addons |
| `prepare_validate` | `prepare_validate.go:40` | Validate ancillaries against correct segment group |
| `buildOrderRequest` | `prepare_order.go:198` | Build order payload with trip direction |
