# LMUQuotaRule — `MaxQuota` vs `MaxQuotaSpecial`

## Summary

| Aspect                | `MaxQuota`                              | `MaxQuotaSpecial`                        |
| --------------------- | --------------------------------------- | ---------------------------------------- |
| Purpose               | **Total** paid upgrade quota per flight/route | **Free/special** upgrade quota per flight/route |
| When checked          | Always (every `/details` request)       | Only when: BookCabin + authenticated + quota rule has `MaxQuotaSpecial > 0` |
| Price charged to pax  | Normal price (from LMUPriceRule)        | **Free** (`SpecialPrice = 0`)            |
| Usage counter in DB   | `LMUQuotaUsage` where `IsSpecialPrice = ANY` (all paid+done rows) | `LMUQuotaUsage` where `IsSpecialPrice = 1` only |
| SQL filter difference | `Status IN ('Paid','Done')`             | `Status IN ('Paid','Done') AND IsSpecialPrice = 1` |
| Exceeded behavior     | `LMUDetailsPriceResult.QuotaExceeded = true` → "LMU isn't available" | If `remainingSpecial == 0`, those pax get **normal price** (not blocked) |
| Nullable              | `int` (required, not null)              | `int?` (nullable, `null` = no special quota) |
| Period column         | Uses departure-date-based counting only | Has its own `QuotaSpecialPeriod` (Day or Month) |

---

## Detailed Explanation

### `MaxQuota` — Total Upgrade Quota

This is the **overall cap** on how many passengers can upgrade on a specific flight/route combination within a time window.

**Checked at:** `LMUDetailsPriceResolver.cs:136-170` — runs for every `/details` call.

**Logic:**
```
used = GetUsedCountAsync(ruleId, departureDate)   -- COUNT(*) of LMUQuotaUsage
       WHERE Status IN ('Paid','Done')            -- all completed orders

if (used + adultChildCount > rule.MaxQuota)
    → QuotaExceeded = true
    → response: "LMU isn't available."
```

- Counts **all** completed upgrade orders (both normal and special price) on that flight/route/date.
- When exceeded → the entire request **fails** — no upgrade available at all.

---

### `MaxQuotaSpecial` — Free/Special Upgrade Quota

This is a **separate, smaller cap** for passengers eligible for free upgrades (BookCabin members with loyalty profiles).

**Checked at:** `LMUDetailsPriceResolver.cs:256-267` — only runs when:
1. `includeSpecialQuotaPricing == true` (authenticated request)
2. `isBookCabin == true` (reservation from BOOKCABIN)
3. `rule.MaxQuotaSpecial > 0`

**Logic:**
```
usedSpecial = GetUsedSpecialCountAsync(ruleId, departureDate)
              WHERE Status IN ('Paid','Done') AND IsSpecialPrice = 1

remainingSpecial = Max(0, MaxQuotaSpecial - usedSpecial)

if (passengerCount <= remainingSpecial)
    → passenger gets SPECIAL (free) price
    → SpecialPricePassengerCount += passengerCount
else
    → those passengers get NORMAL price (not blocked)
```

- Counts **only** completed orders where `IsSpecialPrice = 1` (free upgrades).
- When special quota is exhausted → passengers **still can upgrade**, but at **normal price**.
- This quota does **not** block the request — it only controls pricing.

---

## Counting Period (`QuotaSpecialPeriod`)

Both quotas share the same `QuotaSpecialPeriod` column for their time window:

| Value | Enum     | Behavior                                                    |
| ----- | -------- | ----------------------------------------------------------- |
| `0`   | `Day`    | Count usage per exact `DepartureDate` (same calendar day)   |
| `1`   | `Month`  | Count usage per calendar month of departure date (e.g. all July flights) |

> **Note:** `MaxQuota` uses departure-date filtering but does not have its own period column — it always uses the departure date (or month) from the `QuotaSpecialPeriod` setting. Both quotas share this same period setting.

---

## Relationship Flow

```
                   LMUQuotaRule
                   ┌─────────────────────────┐
                   │ MaxQuota: 30            │  ← total paid upgrade slots
                   │ MaxQuotaSpecial: 5      │  ← free upgrade slots (subset)
                   │ QuotaSpecialPeriod: Day │
                   └─────────┬───────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
     ┌────────────────┐           ┌──────────────────┐
     │  MaxQuota check│           │MaxSpecial check  │
     │  (always runs) │           │(BookCabin+auth)  │
     └───────┬────────┘           └────────┬─────────┘
             │                             │
             ▼                             ▼
   ┌──────────────────┐         ┌──────────────────────┐
   │ used >= MaxQuota? │         │ usedSpecial >=        │
   │   YES → BLOCK    │         │   MaxQuotaSpecial?    │
   │   NO  → continue │         │   YES → normal price  │
   └──────────────────┘         │   NO  → free price    │
                                └──────────────────────┘
```

---

## Example

Given a rule for flight `ID123 CGK→DPS`:

| Column             | Value |
| ------------------ | ----- |
| `MaxQuota`         | 30    |
| `MaxQuotaSpecial`  | 5     |
| `QuotaSpecialPeriod` | Day |

| Scenario | `used` (total) | `usedSpecial` | Result |
| -------- | -------------- | ------------- | ------ |
| No orders yet | 0 | 0 | Normal price shown; special available (0 < 5) |
| 2 normal orders | 2 | 0 | Normal price; special still available |
| 3 normal + 2 free | 5 | 2 | Normal price; 3 free slots remaining |
| 28 normal + 2 free | 30 | 2 | **MaxQuota exceeded** → "LMU isn't available" |
| 25 normal + 5 free | 30 | 5 | **MaxQuota exceeded** → blocked |
| 20 normal + 5 free | 25 | 5 | **MaxSpecial exhausted** → normal price for all new pax |

---

## Key Source Files

| File | Lines | Role |
| ---- | ----- | ---- |
| `Application/Models/LMUQuotaRule.cs:14-15` | Model definition | `MaxQuota` and `MaxQuotaSpecial` fields |
| `Application/Models/QuotaPeriod.cs:4-11` | Enum | `Day = 0`, `Month = 1` |
| `Application/Providers/LMUDetailsPriceResolver.cs:136-170` | Quota check | `MaxQuota` check (total quota) |
| `Application/Providers/LMUDetailsPriceResolver.cs:256-267` | Special check | `MaxQuotaSpecial` check (free quota) |
| `Infrastructure/Data/LMUQuotaRepository.cs:134-160` | SQL | `GetUsedCountAsync` — counts all `LMUQuotaUsage` |
| `Infrastructure/Data/LMUQuotaRepository.cs:162-188` | SQL | `GetUsedSpecialCountAsync` — counts `IsSpecialPrice = 1` only |
| `Infrastructure/Data/LMUQuotaRepository.cs:190-222` | SQL | `GetUsedCountByKeyAsync` — fallback by flight/origin/dest |
| `Infrastructure/Data/LMUQuotaRepository.cs:224-250` | SQL | `GetUsedSpecialCountByKeyAsync` — fallback special by key |
