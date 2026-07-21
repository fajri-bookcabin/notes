# `passengers.isBCMember` — `/details` Endpoint

## Overview

`IsBCMember` is a boolean property on each passenger object returned by `POST api/LMU/details`. It indicates whether a passenger is a **BookCabin (BC) loyalty member** — i.e., they have an active loyalty ID registered with the airline's profile system.

---

## Where It Lives

| Location | File | Line |
|----------|------|------|
| DTO definition | `Application/DTOs/LMUDetailsResponse.cs` | 160 |
| Population logic | `Application/Providers/LMUDetailsProvider.cs` | 1954–1993 |
| Background fetch | `Application/Providers/LMUDetailsProvider.cs` | 838–906 |
| Cache key | `LMU:ItineraryMemberProfiles:{itineraryId}` | Redis |

### DTO Definition

```csharp
// LMUDetailsResponse.cs:160
/// <summary>True when this passenger is a BC member (has loyaltyId from
/// BookCabin itinerary member profile matching).</summary>
public bool IsBCMember { get; set; }
```

---

## How It Gets Set — Full Flow

### Step 1: Detect BookCabin Origin

When the Sabre reservation's `receivedFrom` field equals `"BOOKCABIN"`, the response is flagged as a BookCabin booking (`IsBookCabin = true`). This triggers the member profile lookup pipeline.

```
LMUDetailsProvider.cs:288–301
```

### Step 2: Background Member Profile Fetch (Fire-and-Forget)

A non-blocking background task (`FireFetchAndCacheItineraryMemberIds`) is launched:

1. Fetches the full itinerary JSON from the Itinerary Service.
2. Extracts `memberId` values from itinerary passengers (skipping non-travelling passengers).
3. Calls the Profile Service (`GetMemberProfileByMemberIdsAsync`) to get member profiles — each profile contains `LoyaltyId`, `FirstName`, `LastName`, `MemberId`.
4. Caches the profiles in Redis under `LMU:ItineraryMemberProfiles:{itineraryId}` with a TTL.

```
LMUDetailsProvider.cs:838–906
```

### Step 3: Apply Loyalty IDs to Passengers

`ApplyLoyaltyIdsToPassengersFromCacheAsync` is called after passengers are parsed:

1. **Guard conditions** — returns early if:
   - Response is null or has no passengers
   - The booking is **not** BookCabin (`IsBookCabin == false`)
   - No valid `ItineraryId`
2. Reads cached member profiles from Redis.
3. For each passenger, performs a **case-insensitive, trimmed name match** (first + last name) against the cached profiles.
4. If a match is found **and** the matched profile has a **non-empty `LoyaltyId`**, the passenger's `IsBCMember` is set to `true`.
5. Matched profiles are removed from the pool (greedy — each profile matches at most one passenger).

```
LMUDetailsProvider.cs:1954–1993
LMUDetailsProvider.cs:1995–2015 (FindAndRemoveMatchingProfile helper)
```

---

## When Is `IsBCMember = true`?

All three conditions must be met:

| # | Condition | Why |
|---|-----------|-----|
| 1 | `receivedFrom == "BOOKCABIN"` | Only BookCabin reservations trigger member profile lookup |
| 2 | Passenger name matches a cached member profile | Profile must exist for that itinerary's traveller list |
| 3 | Matched profile has a non-empty `LoyaltyId` | Having a profile alone isn't enough — they must have an active loyalty membership |

If **any** condition fails, `IsBCMember` remains `false` (default).

---

## Example Response

```json
{
  "passengers": [
    {
      "name": "JOHN/DOE MR",
      "title": "MR",
      "nameId": 1,
      "isBCMember": true,
      "eligibility": ["SEAT"],
      "ticketNumber": "*****",
      "hasInf": false
    },
    {
      "name": "JANE/DOE MRS",
      "title": "MRS",
      "nameId": 2,
      "isBCMember": false,
      "eligibility": ["SEAT"],
      "ticketNumber": "*****",
      "hasInf": false
    }
  ]
}
```

---

## Key Takeaways

- **`IsBCMember` is only meaningful for BookCabin reservations.** For non-BookCabin bookings, it will always be `false` because the profile lookup is never triggered.
- **Name matching is greedy.** Each cached profile can only match one passenger, preventing duplicate assignments.
- **The property is cached** as part of `LMUDetailsCachedDto` in Redis, so it persists across cache hits.
- **For authenticated cache-hit flows**, `IsBCMember` is re-applied from the cached profiles to ensure consistency.
