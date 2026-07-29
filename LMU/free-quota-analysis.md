# Free Quota Analysis

## Free Quota in Redis

### Redis Keys

| Key Pattern | Purpose |
|-------------|---------|
| `LMU:q:{flightNumber}:{yyyy-MM-dd}:f` | Daily free quota counter |
| `LMU:q:{flightNumber}:{yyyy-MM-dd}:b` | Daily business quota counter |
| `LMU:q:{flightNumber}:m:{yyyy-MM}:f` | Monthly free quota counter |
| `LMU:q:{flightNumber}:m:{yyyy-MM}:b` | Monthly business quota counter |
| `LMU:res:{orderId}` | Reservation metadata (JSON, with TTL) |
| `LMU:res:pnr:{recordLocator}` | PNR → orderId mapping |
| `LMU:res:details:{recordLocator}:{carrierCode}` | Details-page free hold metadata |

### Quota Granularity

Quota is tracked **per flight + per date**, not per passenger or per PNR. The `LMU:q:*` keys are simple integer counters — atomically decremented on reservation and incremented on release.

### No Per-Passenger Tracking

The current system does **not** track which specific passenger consumed free quota:

- `LMUQuotaUsage` table has: `QuotaRuleId`, `OrderId`, `FlightNumber`, `IsSpecialPrice` — no passenger column
- `LMUOrderPassenger` table has: `FirstName`, `LastName`, `MemberId`, `LoyaltyId` — no `IsFree`/`IsSpecial` flag
- The system only knows order X consumed N free slots (`specialPricePassengerCount`), not which passengers got them

---

## Free Quota Release Lifecycle

### When Quota Gets Released Back

| Scenario | Timing | Mechanism |
|----------|--------|-----------|
| Payment flow fails (order insert error) | Immediate | `ReleaseForOrderAsync(orderId)` called explicitly (`LMUPaymentProvider.cs:1071,1080`) |
| User abandons payment (stale RESERVED) | After `QuotaStaleReservedMinutes` (default 30 min) | `QuotaCronBackgroundService` → `StaleReservedCleanupService.CleanupStaleReservedAsync()` → `ReleaseForOrderAsync` + mark CANCELLED |
| New attempt for same PNR | On retry | `ReserveForOrderAsync` checks `LMU:res:pnr:{recordLocator}`, releases old reservation if different orderId |
| External cron marks order FAILED | Immediate (synchronous) | `PUT /api/LMU/cron/UpdateOrder` with `status: "FAILED"` releases via `ReleaseForOrderFromUsageAsync` |

### Important: Redis Reservation TTL vs Quota Counter

The reservation key `LMU:res:{orderId}` has a TTL, but the **quota counter keys do not auto-rollback on TTL expiry**. Quota release always requires explicit action (either immediate on failure or via the background cleanup job).

### Config Values

| Config | Default | Purpose |
|--------|---------|---------|
| `QuotaDeductionIntervalMinutes` | 2 | **Dead config** — never used in code |
| `QuotaStaleCleanupIntervalMinutes` | 5 | How often the background cron wakes up to check for stale RESERVED orders |
| `QuotaStaleReservedMinutes` | 30 | How old a RESERVED order must be before quota is released and status → CANCELLED |

### How to Manually Release Quota

**Option 1 — Redis CLI:**
```
INCRBY LMU:q:{flightNumber}:{yyyy-MM-dd}:f {count}
```

**Option 2 — Mark order FAILED via cron endpoint (releases synchronously):**
```
PUT /api/LMU/cron/UpdateOrder
{ "orderId": "...", "status": "FAILED" }
```

**Option 3 — Reduce `QuotaStaleReservedMinutes` and wait for background cron (default 5 min interval).**
