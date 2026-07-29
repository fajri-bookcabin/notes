# Redis / Valkey Key Reference

> Last updated: auto-generated from codebase audit.
> Redis/Valkey is used for **real-time quota counters**, **short-lived reservation holds**, and **PNR/details caching**.
> Reference data (locations, route whitelist, etc.) is cached in-memory (`IMemoryCache`), **not** Redis.

---

## LMU-Owned Redis Keys

All keys created by this service are prefixed with `LMU:` so they can be filtered via `LMU:*`.

### 1. Details Cache

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:Details:{recordLocator}:{carrierCode}` | `String` (JSON) | `CacheTtlMinutes` (default **15 min**) | Cached LMU details response including eligibility, segments, pricing, passenger data, and wording triples. Language is applied at read time so `details→en` and `payment→id` still hit the same cache entry. |

**Set by:** `LMUDetailsProvider`  
**Read by:** `LMUDetailsProvider.GetDetailsFromCacheAsync`, `UpdateCacheTimerAsync`, `SetCachedContactDetailsAsync`  
**Deleted by:** `LMUDetailsProvider.RemoveDetailsFromCacheAsync`

---

### 2. Itinerary Member Profiles Cache

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:ItineraryMemberProfiles:{itineraryId}` | `String` (JSON) | `CacheTtlMinutes` (default **15 min**) | Cached InspireNetz loyalty member profiles for an itinerary. Avoids repeated calls to the points API across the details → payment flow. |

**Set by:** `LMUDetailsProvider`, `LMUPaymentProvider`  
**Read by:** `LMUDetailsProvider`, `LMUPaymentProvider`

---

### 3. Retrieve Reservation Cache (GoLMU)

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:RetrieveReservation:{airlineCode}:{recordLocator}` | `String` (JSON) | `CacheTtlMinutes` (default **15 min**) | Cached GoLMU reservation payload (raw API response). Used to avoid re-fetching the same PNR from the GoLMU API within a short window. |

**Set by:** `LMUDetailsProvider.SetGoLmuReservationCacheAsync`  
**Read by:** `LMUDetailsProvider.GetGoLmuReservationFromCacheAsync`, `LMUPaymentProvider`

---

### 4. Quota Counter (Daily)

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:q:{flightNumber}:{yyyy-MM-dd}:{b\|f}` | `String` (integer) | **None** (persistent until evicted or manually deleted) | Real-time available quota counter for a specific flight + departure date.  <br>• `b` = **business** seats  <br>• `f` = **free** (special-price) seats  <br>Value is initialized from `LMUQuotaRule` / `LMUQuotaUsage` DB aggregates on first access, then decremented on reservation and incremented on release. |

**Set by:** `QuotaRedisService.EnsureKeyInitializedAsync` (lazy init from DB)  
**Updated by:** `QuotaRedisService.TryReserveAsync` (DECR), `QuotaRedisService.ReleaseAsync` (INCR)  
**Read by:** `QuotaRedisService.TryReserveAsync`

---

### 5. Quota Counter (Monthly / Special Period)

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:q:{flightNumber}:m:{yyyy-MM}:{b\|f}` | `String` (integer) | **None** | Same as daily quota counter but bucketed by **month** when `QuotaPeriod.Month` is configured in `LMUQuotaRule`. Used for flights with monthly special quotas. |

**Set / Updated / Read by:** `QuotaRedisService` (same logic as daily keys, via `QuotaPeriodHelper.FormatRedisPeriodSegment`).

---

### 6. Order Reservation Document

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:res:{orderId}` | `String` (JSON) | `reservationTtl` (short, caller-defined) | JSON array of reserved segments per order: `{ Fn, Date, Business, Free, P }`.  <br>Allows idempotent release of quota when an order is cancelled, failed, or paid. |

**Set by:** `QuotaRedisService.ReserveForOrderAsync`  
**Read by:** `QuotaRedisService.ReleaseForOrderAsync`  
**Deleted by:** `QuotaRedisService.ReleaseForOrderAsync`, `QuotaRedisService.ClearReservationForOrderAsync`

---

### 7. PNR → Order Mapping

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:res:pnr:{recordLocator}` | `String` | Same as `LMU:res:{orderId}` | Maps a PNR (record locator) to the **current active orderId**. When a new order is created for the same PNR, the previous order’s quota is automatically released. |

**Set by:** `QuotaRedisService.ReserveForOrderAsync`  
**Read by:** `QuotaRedisService.ReserveForOrderAsync` (to detect duplicate PNR reservations)  
**Deleted by:** `QuotaRedisService.DeletePnrMappingAsync` (called on release / clear)

---

### 8. Order → PNR Mapping

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:res:order:{orderId}:pnr` | `String` | Same as `LMU:res:{orderId}` | Reverse mapping from orderId back to the PNR record locator. Used during cleanup to also delete the `LMU:res:pnr:{recordLocator}` key. |

**Set by:** `QuotaRedisService.ReserveForOrderAsync`  
**Read by:** `QuotaRedisService.DeletePnrMappingAsync`  
**Deleted by:** `QuotaRedisService.DeletePnrMappingAsync`

---

### 9. Details Free Hold

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:res:details:{pnr}:{carrierCode}` | `String` (JSON) | **None** (checked via `ExpiresAtUtc` field) | Free-quota hold created at the `/details` endpoint. Contains a document with segment-level free holds (`{ Fn, Date, Free, P }`) and an `ExpiresAtUtc` timestamp.  <br>• Decrements the free pool immediately when the user views details.  <br>• Consumed (and deleted) when the user proceeds to payment via `ReserveForOrderAsync`.  <br>• Released automatically by `CleanupExpiredDetailsFreeHoldsAsync` cron if expired.  <br>• Also extended when the details cache timer is reset. |

**Set by:** `QuotaRedisService.TryReserveDetailsFreeHoldAsync`  
**Read / Consumed by:** `QuotaRedisService.TryConsumeDetailsFreeHoldMetadataAsync`  
**Deleted by:** `QuotaRedisService.ReleaseDetailsFreeHoldAsync`, `QuotaRedisService.ClearDetailsFreeHoldMetadataAsync`, `QuotaRedisService.CleanupExpiredDetailsFreeHoldsAsync`

---

### 10. Free Quota Ownership (Audit)

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `LMU:q:{flightNumber}:{yyyy-MM-dd\|m:yyyy-MM}:f-owner` | `Hash` | **None** (cleared on hold release; Redis auto-removes empty hashes) | Per-flight per-date mapping of **which PNR is holding how many free seats**. Each hash field is a PNR (record locator), each value is the held free count.  <br>• Written when `/details` creates a free hold (`HINCRBY`).  <br>• Cleared when the hold is released, expired, or consumed by payment (`HINCRBY` negative → `HDEL` at 0).  <br>• Used for audit/debug to answer *"who booked the free quota on flight X?"*. |

**Set by:** `QuotaRedisService.TryReserveDetailsFreeHoldAsync` (after `StringSetAsync` succeeds)  
**Updated by:** `QuotaRedisService.ReleaseDetailsHoldItemsInstanceAsync` (decrement on release), `QuotaRedisService.TryConsumeDetailsFreeHoldMetadataAsync` (decrement on consumption)  
**Read by:** `QuotaRedisService.GetFreeQuotaOwnersAsync` (returns `Dictionary<PNR, freeCount>`)

---

## Key Naming Quick Reference

| Prefix | What it stores |
|--------|----------------|
| `LMU:Details:` | Cached LMU details response per (PNR, carrier) |
| `LMU:ItineraryMemberProfiles:` | Cached loyalty member profiles per itineraryId |
| `LMU:RetrieveReservation:` | Cached GoLMU reservation per (airline, PNR) |
| `LMU:q:` | Real-time quota counters per (flight, date/month, seat type) |
| `LMU:res:` | Order reservation JSON per orderId |
| `LMU:res:pnr:` | PNR → orderId mapping |
| `LMU:res:order:` | OrderId → PNR mapping |
| `LMU:res:details:` | Details-stage free-quota hold per (PNR, carrier) |
| `LMU:q:*:f-owner` | Free-quota ownership hash per (flight, date/month) → PNR → count |

---

## Admin Inspection

The `AdminController` exposes endpoints to inspect these keys directly:

- `GET /api/LMU/admin/cache/keys?pattern=LMU:*` — List all LMU-owned keys.
- `GET /api/LMU/admin/cache/key?key=LMU:Details:ABC123:ID` — Inspect a single key (type, TTL, value).
- `DELETE /api/LMU/admin/cache/key?key=LMU:Details:ABC123:ID` — Delete a single key.

---

## IMemoryCache Keys (Not Redis)

The following keys are **in-memory only** (`IMemoryCache`) and are **not** stored in Redis/Valkey:

| Key Pattern | Purpose |
|-------------|---------|
| `RefData:AirportCountry:v1:{iata}` | Airport → country code mapping (1 day TTL) |
| `RefData:AirlineEquipment:v1` | Airline equipment type → aircraft name map (1 day TTL) |
| `RefData:RefAirlineBaggage` | RefAirlineBaggage entities (1 day TTL) |
| `LMU:RouteWhitelist:HasAnyActive` | Whether any whitelist routes exist (5 min TTL) |
| `LMU:RouteWhitelist:{cc}:{o}:{d}` | Per-route whitelist allow/deny (5 min TTL) |

---

## Notes

- **Redis is the source of truth for real-time availability**, but DB (`LMUQuotaUsage`) remains the source of truth for **committed** (paid) usage.
- Quota counter keys (`LMU:q:*`) are **not time-bound**; they persist until the Redis instance evicts them or an admin deletes them. They are re-initialized from DB on first access after eviction.
- The `Details` cache and `RetrieveReservation` cache share the same TTL (`CacheTtlMinutes`, default 15 minutes). This keeps the PNR data fresh while reducing load on Sabre/GoLMU APIs.
- Sabre SOAP sessions themselves are **not** cached in Redis by this service; session pooling is handled by the external `SoapService` REST proxy.
