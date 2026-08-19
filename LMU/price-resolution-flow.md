# Step 9: Price Resolution — Detailed Flow

## Overview

The price resolver (`LMUDetailsPriceResolver`) determines the upgrade price per passenger by matching flight segments against quota rules and price rules stored in the database. It supports two pricing paths: **special quota pricing** (free/discounted for loyalty members) and **normal pricing** (standard LMUPriceRule lookup).

| Property              | Value                                        |
| --------------------- | -------------------------------------------- |
| Class                 | `LMUDetailsPriceResolver`                    |
| Interface             | `ILMUDetailsPriceResolver`                   |
| Entry Method          | `ResolveAsync()`                             |
| File                  | `Application/Providers/LMUDetailsPriceResolver.cs` |
| Result DTO            | `LMUDetailsPriceResult`                      |

---

## Dependencies

| Service                       | Purpose                                          |
| ----------------------------- | ------------------------------------------------ |
| `ILMUQuotaRepository`        | Resolve quota rules, read used counts            |
| `ILMUPriceRuleRepository`    | Time window match, price lookup (ADT/INF)        |
| `ILMUConfiguration`          | Eligibility window hours, feature flags          |
| `IReferenceDataService`      | Airport IATA → country code (domestic/intl)      |
| `IQuotaRedisService`         | Real-time remaining free quota from Redis        |

---

## Method Signature

```csharp
public async Task<LMUDetailsPriceResult> ResolveAsync(
    IReadOnlyList<LMUDetailsSegment> segments,
    int? totalDurationMinutes,
    bool isBookCabin,
    int adultChildCount = 1,
    bool includeSpecialQuotaPricing = false,
    CancellationToken cancellationToken = default,
    string? recordLocator = null,
    string? userAgent = null,
    string? appVersion = null)
```

| Parameter                 | Purpose                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `segments`                | All flight segments from the reservation                        |
| `totalDurationMinutes`    | Total itinerary duration (fallback when per-segment unavailable)|
| `isBookCabin`             | BookCabin flow → selects `PriceBC` column;否则 `PriceNonBC`     |
| `adultChildCount`         | Adult + child passengers (excludes infants)                     |
| `includeSpecialQuotaPricing` | `false` for `/details`; `true` for `/prepare`, `/payment`    |
| `recordLocator`           | Self-exclusion: Redis ignores this PNR's own held quota         |
| `userAgent` / `appVersion`| App version check for partial-free flow gating                  |

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│  LMUDetailsPriceResolver.ResolveAsync(segments, ...)                   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 0: EARLY EXIT / FALLBACK                                   │   │
│  │ ├─ If segments null/empty → return empty result (no price)       │   │
│  │ ├─ Filter: only ID carrier segments (CarrierCode == "ID")       │   │
│  │ └─ If no ID segments → return empty result                       │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 1: COMPUTE isBookCabinAndSpecialOffers                     │   │
│  │ │                                                                 │   │
│  │ ├─ isBookCabinAndSpecialOffers =                                  │   │
│  │ │   isBookCabin && AppVersionHelper.IsLmuSpecialOffersAppVersion │   │
│  │ │                                                                 │   │
│  │ ├─ AppVersionHelper checks:                                       │   │
│  │ │   ├─ IsLMUSpecialOffersEnable config flag → false = skip       │   │
│  │ │   ├─ Platform detection from User-Agent (android/ios/huawei)   │   │
│  │ │   └─ Version comparison >= minAndroid or minIos                 │   │
│  │ │                                                                 │   │
│  │ ├─ Controls pricing column:                                       │   │
│  │ │   ├─ true  → PriceBC (discounted/loyalty price)                │   │
│  │ │   └─ false → PriceNonBC (standard price)                       │   │
│  │ └─ Also gates special offers flow in Phase 5                      │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 2: ELIGIBILITY FILTERING (per ID segment)                  │   │
│  │ │                                                                 │   │
│  │ │ A segment is ELIGIBLE when ALL conditions pass:                 │   │
│  │ ├─ BusinessClassAvailable == true                                 │   │
│  │ ├─ RemainingSeat is not null                                      │   │
│  │ ├─ DepartureUtc > DateTime.UtcNow (future departure)             │   │
│  │ └─ Current UTC time within eligibility window:                    │   │
│  │    [DepartureUtc − EligibilityWindowHoursBeforeFrom,              │   │
│  │     DepartureUtc − EligibilityWindowHoursBeforeTo]                │   │
│  │    (default: 11h before → 1h before departure)                   │   │
│  │                                                                   │   │
│  │ ❌ Excluded segments: dropped from pricing + quota calculation    │   │
│  │ ❌ Zero eligible segments → return empty result                   │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 3: RECOMPUTE DURATION                                      │   │
│  │ ├─ eligibleTotalDurationMinutes = sum(DurationMinutes)           │   │
│  │ │   for eligible segments only                                    │   │
│  │ └─ If <= 0 and totalDurationMinutes provided → use fallback      │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 4: QUOTA EXCEEDED CHECK                                    │   │
│  │ │                                                                 │   │
│  │ │ For each eligible ID segment:                                   │   │
│  │ ├─ Build flight number (prepend "ID" if missing)                 │   │
│  │ ├─ ResolveRuleAsync(flight, origin, dest, depTime, depDate)     │   │
│  │ │   → returns ResolvedQuotaRule?                                  │   │
│  │ │                                                                 │   │
│  │ ├─ If rule found:                                                 │   │
│  │ │   ├─ Override rule (IsOverride=true):                          │   │
│  │ │   │   └─ GetUsedCountAsync(ruleId, depDate, quotaPeriod)      │   │
│  │ │   └─ Default rule (ApplyToAllFlights):                         │   │
│  │ │       └─ GetUsedCountByKeyAsync(flight, origin, dest, depDate) │   │
│  │ │                                                                 │   │
│  │ │ ❌ If used >= rule.MaxQuota → QuotaExceeded = true → RETURN    │   │
│  │ │    (short-circuits — no pricing performed)                      │   │
│  │ └─────────────────────────────────────────────────────────────────│   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │ (quota available)                         │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 5: SPECIAL QUOTA PRICING (conditional)                     │   │
│  │ │                                                                 │   │
│  │ Entry: includeSpecialQuotaPricing && isBookCabin                  │   │
│  │        && eligibleTotalDurationMinutes > 0                        │   │
│  │                                                                   │   │
│  │ For each eligible segment:                                        │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5a: EARLY SKIP                                              │   │   │
│  │  │ ├─ Missing flight/origin/destination → skip (contribute 0) │   │   │
│  │  │ └─ No departure time or negative duration → skip            │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5b: RE-CHECK ELIGIBILITY WINDOW                             │   │   │
│  │  │ └─ Same 11h-1h check → if outside → skip                   │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5c: DOMESTIC / INTERNATIONAL RESOLUTION                     │   │   │
│  │  │ ├─ Parallel: GetAirportCountryCodeAsync(origin)             │   │   │
│  │  │ │            GetAirportCountryCodeAsync(destination)         │   │   │
│  │  │ ├─ Both must == "ID" → domestic = true                      │   │   │
│  │  │ ├─ Either null → domestic = null → EXCLUDE segment          │   │   │
│  │  │ └─ Mismatch → domestic = false (international)              │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5d: PRICE RULE TIME WINDOW MATCH                            │   │   │
│  │  │ ├─ HasTimeWindowMatchAsync(isDomestic, paxType, duration,  │   │   │
│  │  │ │   depTime, origin, dest)                                  │   │   │
│  │  │ ├─ SQL: LMUPriceRule WHERE IsActive=1, IsDomestic match,   │   │   │
│  │  │ │   PaxType='ADT', effective date range, duration in range, │   │   │
│  │  │ │   origin/dest match (or NULL/ALL),                         │   │   │
│  │  │ │   departureTimeOfDay within [DepartureTimeStart, End]     │   │   │
│  │  │ └─ No match → EXCLUDE segment                               │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5e: QUOTA RULE RESOLUTION                                   │   │   │
│  │  │ ├─ rule = ResolveRuleAsync(flight, origin, dest, time, date)│   │   │
│  │  │ ├─ If rule != null && rule.MaxQuotaSpecial > 0              │   │   │
│  │  │ │   → proceed with special pricing                          │   │   │
│  │  │ └─ Else → fallback to normal pricing for this segment       │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5f: REMAINING FREE QUOTA                                    │   │   │
│  │  │                                                             │   │   │
│  │  │ Redis-first:                                                │   │   │
│  │  │ ├─ _quotaRedis.GetFreeAvailableAsync(flight, date,         │   │   │
│  │  │ │   origin, dest, recordLocator)                            │   │   │
│  │  │ ├─ Returns int? (null if key absent or Redis error)         │   │   │
│  │  │ └─ remainingSpecial = Min(MaxQuotaSpecial, redisAvailable)  │   │   │
│  │  │                                                             │   │   │
│  │  │ DB fallback (when Redis null):                              │   │   │
│  │  │ ├─ Override rule: GetUsedSpecialCountAsync(ruleId, date)    │   │   │
│  │  │ ├─ Default rule: GetUsedSpecialCountByKeyAsync(flight, ...) │   │   │
│  │  │ └─ remainingSpecial = MaxQuotaSpecial − usedSpecial         │   │   │
│  │  │                                                             │   │   │
│  │  │ Cap: remainingSpecial = Max(0, remainingSpecial)            │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5g: SPECIAL COUNT COMPUTATION                               │   │   │
│  │  │                                                             │   │   │
│  │  │ Old flow (app version NOT supported):                       │   │   │
│  │  │ ├─ All-or-nothing:                                          │   │   │
│  │  │ │   adultChildCount <= remainingSpecial                     │   │   │
│  │  │ │   ? adultChildCount (all free)                            │   │   │
│  │  │ │   : 0 (all paid)                                          │   │   │
│  │  │                                                             │   │   │
│  │  │ New flow (app version supported, partial-free):             │   │   │
│  │  │ ├─ maxFreePerPNR = rule.MaxFreeQuotaPerPNR                 │   │   │
│  │  │ ├─ maxFreeQuotaUsed =                                       │   │   │
│  │  │ │   <= -1 → adultChildCount  (unlimited)                    │   │   │
│  │  │ │   == 0  → 0                 (none)                        │   │   │
│  │  │ │   > 0   → min(maxFreePerPNR, adultChildCount) (capped)   │   │   │
│  │  │ └─ specialCount = min(remainingSpecial, maxFreeQuotaUsed)   │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5h: PER-SEGMENT PRICING                                     │   │   │
│  │  │                                                             │   │   │
│  │  │ normalCount = adultChildCount − specialCount                │   │   │
│  │  │                                                             │   │   │
│  │  │ Special passengers:                                         │   │   │
│  │  │ └─ specialCount × rule.SpecialPrice (= 0, always free)     │   │   │
│  │  │                                                             │   │   │
│  │  │ Normal passengers:                                          │   │   │
│  │  │ └─ normalCount × GetPriceAsync(depTime, duration,          │   │   │
│  │  │     isBookCabinAndSpecialOffers, isDomestic, origin, dest)  │   │   │
│  │  │     → returns PriceBC or PriceNonBC based on flag           │   │   │
│  │  │                                                             │   │   │
│  │  │ Infant:                                                     │   │   │
│  │  │ └─ GetInfantPriceAsync(depTime, duration, ...)             │   │   │
│  │  │     (PaxType='INF')                                         │   │   │
│  │  │                                                             │   │   │
│  │  │ Segment total = specialAmount + normalAmount + infantAmount │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  │                                                                   │   │
│  │  ┌────────────────────────────────────────────────────────────┐   │   │
│  │  │ 5i: SEGMENT WITHOUT QUOTA RULE                              │   │   │
│  │  │ └─ Falls back to GetPriceForSegmentAsync() — full normal   │   │   │
│  │  │    pricing for that segment                                 │   │   │
│  │  └────────────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 6: COMBINE SPECIAL RESULTS                                 │   │
│  │ │                                                                 │   │
│  │ ├─ If all segments have results AND currency set                 │   │
│  │ │   AND at least one segment has specialCount > 0:               │   │
│  │ │                                                                 │   │
│  │ ├─ ResolvePerSegmentPriceAsync() → normal/discount display       │   │
│  │ ├─ Compute usableFreeQuota = min(specialCountPerSegment[i])     │   │
│  │ │   for only time-window-matched segments (excludes non-matched) │   │
│  │ ├─ BuildDiscountLabel() → "-25%" or "25%" format                │   │
│  │ │                                                                 │   │
│  │ └─ Return LMUDetailsPriceResult with:                            │   │
│  │    IsSpecialPrice=true, SpecialPricePassengerCount,              │   │
│  │    SpecialPricePassengerCountPerSegment, UsableFreeQuota,        │   │
│  │    DiscountLabel, DiscountLabelForTnc                            │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 7: NORMAL PRICING FALLBACK                                 │   │
│  │ │                                                                 │   │
│  │ Used when:                                                        │   │
│  │ ├─ includeSpecialQuotaPricing == false (i.e., /details)          │   │
│  │ ├─ OR special quota didn't apply (no segments matched)           │   │
│  │ │                                                                 │   │
│  │ ├─ ResolvePerSegmentPriceAsync() → sums per-segment prices       │   │
│  │ └─ Return LMUDetailsPriceResult with IsSpecialPrice=false        │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ PHASE 8: FINAL FALLBACK                                          │   │
│  │ └─ ResolveFallbackDurationOnlyAsync() → empty result (no price) │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Mermaid Sequence Diagram

```mermaid
sequenceDiagram
    participant Caller as DetailsProvider / PaymentProvider
    participant Resolver as LMUDetailsPriceResolver
    participant QuotaRepo as Quota Repository
    participant PriceRepo as PriceRule Repository
    participant RefData as Reference Data API
    participant QuotaRedis as Quota Redis

    Caller->>Resolver: ResolveAsync(segments, duration, isBookCabin, paxCount, includeSpecial)

    Note over Resolver: PHASE 0: Early Exit
    Resolver->>Resolver: Filter ID carrier segments only

    Note over Resolver: PHASE 1: Version Check
    Resolver->>Resolver: isBookCabinAndSpecialOffers =<br/>isBookCabin && AppVersionHelper check

    Note over Resolver: PHASE 2: Eligibility Filtering
    loop Per ID segment
        Resolver->>Resolver: Check: BC available? RemainingSeat? Future? Within 11h-1h window?
    end

    Note over Resolver: PHASE 3: Recompute Duration
    Resolver->>Resolver: Sum DurationMinutes of eligible segments

    Note over Resolver: PHASE 4: Quota Exceeded Check
    loop Per eligible segment
        Resolver->>QuotaRepo: ResolveRuleAsync(flight, origin, dest, time, date)
        QuotaRepo-->>Resolver: ResolvedQuotaRule?
        alt Rule found
            alt Override rule
                Resolver->>QuotaRepo: GetUsedCountAsync(ruleId, depDate, period)
            else Default rule
                Resolver->>QuotaRepo: GetUsedCountByKeyAsync(flight, origin, dest, depDate, period)
            end
            QuotaRepo-->>Resolver: usedCount
            alt used >= MaxQuota
                Resolver-->>Caller: { QuotaExceeded = true }
            end
        end
    end

    opt includeSpecialQuotaPricing && isBookCabin
        Note over Resolver: PHASE 5: Special Quota Pricing
        loop Per eligible segment
            Note over Resolver: 5a-5b: Early skip + re-check window

            Resolver->>RefData: GetAirportCountryCodeAsync(origin)
            Resolver->>RefData: GetAirportCountryCodeAsync(dest)
            RefData-->>Resolver: countryCode1, countryCode2
            Note over Resolver: 5c: Both == "ID" → domestic

            Resolver->>PriceRepo: HasTimeWindowMatchAsync(isDomestic, paxType, duration, depTime, origin, dest)
            PriceRepo-->>Resolver: matchResult
            Note over Resolver: 5d: No match → skip segment

            Resolver->>QuotaRepo: ResolveRuleAsync(flight, origin, dest, time, date)
            QuotaRepo-->>Resolver: rule
            Note over Resolver: 5e: MaxQuotaSpecial > 0?

            alt Redis available
                Resolver->>QuotaRedis: GetFreeAvailableAsync(flight, date, origin, dest, recordLocator)
                QuotaRedis-->>Resolver: remainingSpecial
            else Redis null (DB fallback)
                Resolver->>QuotaRepo: GetUsedSpecialCountAsync / GetUsedSpecialCountByKeyAsync
                QuotaRepo-->>Resolver: usedSpecial
                Note over Resolver: remainingSpecial = MaxQuotaSpecial − usedSpecial
            end

            Note over Resolver: 5g: Compute specialCount<br/>(old: all-or-nothing; new: partial-free)

            Resolver->>PriceRepo: GetPriceAsync(depTime, duration, isBookCabin, isDomestic, origin, dest)
            PriceRepo-->>Resolver: price (PriceBC or PriceNonBC)
            Resolver->>PriceRepo: GetInfantPriceAsync(...)
            PriceRepo-->>Resolver: infantPrice
        end

        Note over Resolver: PHASE 6: Combine Results
        Resolver->>Resolver: usableFreeQuota = min(specialCountPerSegment) for matched segments only
        Resolver->>Resolver: BuildDiscountLabel()
    end

    opt Normal pricing fallback
        Note over Resolver: PHASE 7: Normal Pricing
        loop Per eligible segment
            Resolver->>RefData: GetAirportCountryCodeAsync(origin, dest)
            RefData-->>Resolver: isDomestic
            Resolver->>PriceRepo: HasTimeWindowMatchAsync(...)
            PriceRepo-->>Resolver: match
            Resolver->>PriceRepo: GetPriceAsync() + GetInfantPriceAsync()
            PriceRepo-->>Resolver: {price, currency}
        end
    end

    Resolver-->>Caller: LMUDetailsPriceResult
```

---

## Result DTO — `LMUDetailsPriceResult`

| Property                              | Type                       | Description                                          |
| ------------------------------------- | -------------------------- | ---------------------------------------------------- |
| `QuotaExceeded`                       | `bool`                     | `true` when used ≥ MaxQuota — LMU not available      |
| `PricePerPax`                         | `decimal?`                 | Price per adult/child passenger                      |
| `PricePerInfant`                      | `decimal?`                 | Price per infant (`null` = free)                     |
| `Currency`                            | `string?`                  | e.g. `"IDR"`                                         |
| `IsSpecialPrice`                      | `bool`                     | `true` when free quota price applies                 |
| `SpecialPricePassengerCount`          | `int`                      | Total passengers at special (free) price             |
| `SpecialPricePassengerCountPerSegment`| `IReadOnlyList<int>?`      | Per-segment free count for quota deduction           |
| `SegmentIndicesTimeWindowNotMatched`  | `IReadOnlyList<int>`       | Indices of ID segments that didn't match any filter  |
| `UsableFreeQuota`                     | `int`                      | Bottleneck min of special counts (matched segments)  |
| `RemainingFreeQuotaMin`               | `int?`                     | Min remaining free quota from Redis/DB               |
| `NormalPricePerPax`                   | `decimal?`                 | Sum of PriceNonBC per segment (for display)          |
| `DiscountPricePerPax`                 | `decimal?`                 | Sum of PriceBC per segment (for display)             |
| `DiscountLabel`                       | `string?`                  | e.g. `"-25%"`                                        |
| `DiscountLabelForTnc`                 | `string?`                  | e.g. `"25%"` (without dash prefix)                   |

---

## Domestic vs International Resolution

```
┌───────────────────────────────────────────────────────────┐
│  TryResolveIsDomesticFromAirportsAsync(origin, dest)      │
│                                                           │
│  Parallel:                                                │
│  ├─ GetAirportCountryCodeAsync(origin) → countryCode1     │
│  └─ GetAirportCountryCodeAsync(dest)   → countryCode2     │
│                                                           │
│  Both == "ID" → domestic = true                           │
│  Either null  → domestic = null (unknown, exclude segment)│
│  Mismatch     → domestic = false (international)          │
└───────────────────────────────────────────────────────────┘
```

`IReferenceDataService.GetAirportCountryCodeAsync()`:
- Config: `ReferenceData:LocationListUrl`
- HTTP GET: `{url}?query={iataCode}`
- Reads ISO country code from `data.locations[]`
- Result cached 1 day

---

## Time Window Matching (Price Rule)

SQL query in `LMUPriceRuleRepository.HasTimeWindowMatchAsync()`:

```sql
SELECT * FROM LMUPriceRule
WHERE IsActive = 1
  AND IsDomestic = @isDomestic
  AND PaxType = 'ADT'
  AND EffectiveFrom <= @now AND EffectiveTo >= @now
  AND MinDurationMinutes <= @duration AND MaxDurationMinutes >= @duration
  AND (Origin = @origin OR Origin IS NULL OR Origin = 'ALL')
  AND (Destination = @dest OR Destination IS NULL OR Destination = 'ALL')
```

Then in-memory: check if `departureTimeOfDay >= DepartureTimeStart && <= DepartureTimeEnd` for any matching row.

---

## Quota Rule Resolution

`QuotaRepository.ResolveRuleAsync()` merges two sources:

```
┌──────────────────────────────────────────────────────────────┐
│  Override Rule (LMUQuotaRule table)                           │
│  ├─ Matches: flightNumber + origin + destination             │
│  ├─ IsOverride = true                                        │
│  └─ Priority: takes precedence over default                  │
│                                                              │
│  Default Rule (LMUQuotaConfig table)                         │
│  ├─ ApplyToAllFlights = true                                 │
│  ├─ Matches: origin + destination (any flight)               │
│  └─ IsOverride = false                                       │
└──────────────────────────────────────────────────────────────┘
```

| Field              | Purpose                                                    |
| ------------------ | ---------------------------------------------------------- |
| `MaxQuota`         | Total upgrade capacity (paid + free)                       |
| `MaxQuotaSpecial`  | Free upgrade capacity for loyalty members                  |
| `MaxFreeQuotaPerPNR` | Per-PNR cap: -1=unlimited, 0=none, >0=capped           |
| `QuotaSpecialPeriod` | `Day=0` (per calendar day), `Month=1` (per calendar month)|
| `SpecialPrice`     | Always returns `0` (free)                                  |
| `Currency`         | Always `"IDR"`                                             |

---

## Special Count Computation — Old vs New Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  OLD FLOW (app version NOT supported)                           │
│  ├─ All-or-nothing:                                             │
│  │   adultChildCount <= remainingSpecial                        │
│  │   ? specialCount = adultChildCount   (all passengers free)   │
│  │   : specialCount = 0                 (all passengers pay)    │
│  └────────────────────────────────────────────────────────────── │
│                                                                  │
│  NEW FLOW (app version supported, partial-free)                  │
│  ├─ maxFreePerPNR = rule.MaxFreeQuotaPerPNR                     │
│  │                                                               │
│  │   maxFreePerPNR <= -1  → maxFreeQuotaUsed = adultChildCount  │
│  │   maxFreePerPNR ==  0  → maxFreeQuotaUsed = 0                │
│  │   maxFreePerPNR >   0  → maxFreeQuotaUsed =                  │
│  │                           min(maxFreePerPNR, adultChildCount) │
│  │                                                               │
│  └─ specialCount = min(remainingSpecial, maxFreeQuotaUsed)      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Per-Segment Price Calculation

```
For each eligible segment with quota rule:

  specialAmount = specialCount × rule.SpecialPrice  (= 0)
  normalAmount  = normalCount × GetPriceAsync(...)  (PriceBC or PriceNonBC)
  infantAmount  = infantCount × GetInfantPriceAsync(...)

  segmentTotal = specialAmount + normalAmount + infantAmount

For segments without quota rule:
  → Full normal pricing via GetPriceForSegmentAsync()

Final totals across all eligible segments:
  PricePerPax   = sum(normalAmount per segment) / adultChildCount
  PricePerInfant = sum(infantAmount per segment) / infantCount
```

---

## Edge Cases

| Scenario                              | Behavior                                                 |
| ------------------------------------- | -------------------------------------------------------- |
| No ID segments                        | Return empty result (no price)                           |
| No eligible segments                  | Return empty result (no price)                           |
| Zero duration                         | Use `totalDurationMinutes` fallback                      |
| Quota exceeded (first segment)        | Short-circuit: return `QuotaExceeded=true`, no pricing   |
| Airport country unknown (null)        | Segment excluded from pricing                            |
| Time window not matched               | Segment contributes 0; excluded from `usableFreeQuota`   |
| Redis null (key absent)               | Fall back to DB-based `MaxQuotaSpecial − usedSpecial`    |
| Mixed segments (some special, some not)| Special segments: free for loyalty, paid for others; non-special: full normal price |
| `recordLocator` self-exclusion        | Redis adds back PNR's own held count to avoid double-counting |
| `MaxFreeQuotaPerPNR = -1`            | Unlimited free per PNR (capped by `remainingSpecial`)    |
| `MaxFreeQuotaPerPNR = 0`             | No free quota — all passengers pay normal price          |
| All passengers free (totalAmount=0)   | Caller (Payment) re-resolves without special quota        |

---

## Key Configuration Values

| Config                                      | Purpose                                          |
| ------------------------------------------- | ------------------------------------------------ |
| `EligibilityWindowHoursBeforeFrom`          | Window start: departure minus X hours (default 11)|
| `EligibilityWindowHoursBeforeTo`            | Window end: departure minus Y hours (default 1)  |
| `IsLMUSpecialOffersEnable`                  | Feature flag for special offers flow             |
| `LMUSpecialOffersMinAppVersionAndroid`      | Min Android version for partial-free flow        |
| `LMUSpecialOffersMinAppVersionIos`          | Min iOS version for partial-free flow            |
| `IsDiscountLabelUsingPercentage`            | Discount label format (% vs absolute)           |
| `ReferenceData:LocationListUrl`             | Airport country code API endpoint                |
