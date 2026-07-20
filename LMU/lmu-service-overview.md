# LMU Service Overview

## What Is This Service?

The **LMU (Last-Minute Upgrade) Service** is an ASP.NET Core 8.0 Web API that enables airline passengers to upgrade from economy to business class on last-minute flights. Built for the BookCabin platform (Lion Air group, carrier code `ID`), it orchestrates the entire upgrade journey — from eligibility checking through payment, fulfillment, receipt generation, and loyalty points accrual.

The service integrates with **Sabre** (global distribution system) for flight reservation and ticketing operations, and coordinates with multiple internal services (payment, itinerary, profile, reference data) and external APIs (InspireNetz loyalty points).

---

## Core Responsibilities

### 1. Upgrade Eligibility & Details

Before a passenger can upgrade, the service validates multiple conditions:

- **Departure window**: Must be between 11 hours and 1 hour before departure
- **Carrier check**: Flight must be operated by carrier `ID`
- **Ticketing status**: Passenger must not be checked in
- **Business class inventory**: At least 2 seats available in business class (C/D/I/Z classes)
- **Quota availability**: Daily or monthly upgrade quota not exceeded
- **Route whitelist**: If enabled, only pre-approved routes are eligible (deny-by-default)
- **No existing upgrade**: No PAID/FAILED/FULFILLED order already exists for the PNR
- **No ancillary services**: Passenger must not have group=99 ancillary services
- **Not already business class**: Passenger must currently be in economy

The details endpoint returns passenger information, segment breakdown, pricing (including special loyalty pricing), baggage weights, and cabin points calculation.

### 2. Pricing & Quota Management

**Two-tier pricing system:**

- **Normal price** (`LMUPriceRule`): Matched by route, departure time window, duration bucket, domestic/international classification, and passenger type (ADT/INF). Separate rates for BookCabin vs non-BookCabin passengers.
- **Special quota price** (`LMUQuotaRule.SpecialPrice`): Discounted or free upgrades available to BookCabin loyalty members, drawn from a separate quota pool (`MaxQuotaSpecial`).

**Quota architecture:**

- **Redis** handles real-time atomic reserve/release (DECR/INCR) to prevent overselling during concurrent payment attempts
- **SQL Server** stores committed usage records (`LMUQuotaUsage`) as the source of truth
- Quota rules are per-flight, with fallback to global `LMUQuotaConfig`
- Quota period can be daily or monthly (`QuotaSpecialPeriod`)
- Stale reservations are automatically cleaned up by a background service

**Total price formula:**

```
total = (pricePerPax × (adultChildCount - specialCount)) + (specialPrice × specialCount) + (pricePerInfant × infantCount)
```

### 3. Payment Processing

The payment flow:

1. Validates the cached session matches fresh details
2. Reserves quota atomically in Redis (all-or-nothing across segments)
3. Calls the payment gateway to initiate payment
4. Creates order records in the database (LMUOrder, LMUOrderPassenger, LMUOrderContactDetails)
5. Generates a unique 10-character booking code
6. Inserts a pending loyalty points record
7. Returns payment URL for redirect

For zero-price orders (free loyalty upgrades), the order is fulfilled immediately without going through the payment gateway.

### 4. Order Fulfillment (Cron-Driven)

After payment initiation, fulfillment is handled asynchronously:

- **PAID status** (via cron callback): Generates receipt PDF → uploads to S3 → commits quota to DB → cleans up Redis reservation
- **FAILED status**: Releases Redis quota → marks DB usage as released → updates order status
- **UpdateItineraryBC**: Updates travel class in the itinerary system for BookCabin orders

### 5. Receipt Generation

- Renders Razor templates (English, Indonesian, Thai locales) to HTML
- Converts HTML to PDF via EvoPdf cloud service
- Uploads PDF to AWS S3
- Stores presigned URL in the order record
- Re-upload available via admin reporting endpoint

### 6. Loyalty Points Processing

- At order creation, a pending loyalty record is inserted with the cabin points amount (price × configured percentage)
- A background service or cron endpoint processes pending records
- Calls the InspireNetz Points API (Digest authentication) to credit loyalty points
- Points are split evenly across loyalty-enrolled passengers
- Failed attempts are tracked for retry

### 7. Reporting & Admin

**Reporting** provides operational visibility:

- Order queries by date range, status, record locator, or loyalty ID
- Price rule CRUD (route-based pricing configuration)
- Quota config/rules/usage CRUD and remaining capacity views
- Manual fulfillment triggers (upgrade, issue ticket, resend email) via GOBC API
- Receipt re-upload for individual orders

**Admin** provides Redis cache management:

- List keys by pattern (default `LMU:*`)
- Inspect individual keys (type, value, TTL)
- Delete keys to force cache refresh

---

## Business Flow Summary

```
Passenger                    LMU Service                    External Systems
    |                              |                              |
    |--- Check Upgrade Status ---->|                              |
    |<-- Already upgraded? --------|                              |
    |                              |                              |
    |--- Check Eligibility ------->|--- CreateSession ----------->| Sabre
    |                             |--- GetReservation ---------->|
    |                             |--- DisplayInventory -------->|
    |<-- Eligible? ----------------|                              |
    |                              |                              |
    |--- Get Details ------------->|--- (cached or fresh) ------->| Sabre
    |<-- Passengers, Price --------|                              |
    |                              |                              |
    |--- Prepare ----------------->|--- Validate + Price -------->|
    |<-- Payment Methods ----------|                              |
    |                              |                              |
    |--- Payment ----------------->|--- Reserve Quota (Redis)     |
    |                             |--- Initiate Payment --------->| Payment Gateway
    |                             |--- Create Order (DB)         |
    |<-- Booking Code -------------|                              |
    |                              |                              |
    |                     [Cron Callback: PAID]                   |
    |                             |--- Generate Receipt --------->| EvoPdf + S3
    |                             |--- Commit Quota (DB)         |
    |                             |--- Process Loyalty ---------->| InspireNetz
    |                             |--- Update Itinerary --------->| Itinerary Service
```

---

## External System Integrations

| System | Purpose | Protocol |
|--------|---------|----------|
| **Sabre SOAP API** | CreateSession, GetReservation, ReadTicketingDocuments, DisplayInventory, UpdateItinerary, CloseSession | REST proxy → SOAP |
| **SQL Server** | Orders, passengers, quota rules/usage, price rules, loyalty tracking, route whitelist, service config | FluentMigrator + ADO.NET |
| **Redis/Valkey** | Details cache, quota counters (atomic DECR/INCR), reservation holds, member profiles cache | StackExchange.Redis |
| **AWS Secrets Manager** | DB connection, Redis connection, payment secrets, PDF key, InspireNetz credentials | HTTPS |
| **Payment Gateway** | Payment initiation and status | REST |
| **GOBC API** | Upgrade fulfillment, ticket issuance, email resend, reservation retrieval | REST |
| **Itinerary Service** | Travel class updates, split itinerary, full itinerary with user info | REST |
| **Profile Service** | Member profiles (loyalty ID, name, title) for loyalty pricing | REST |
| **Reference Data Service** | Airport locations, airline equipment data, baggage weights | REST |
| **InspireNetz Points API** | Loyalty points accrual (Add Sale Entry) | REST (Digest auth) |
| **EvoPdf** | HTML to PDF receipt generation | In-process |
| **AWS S3** | Receipt PDF storage and presigned URL generation | HTTPS |

---

## API Endpoints Summary

### Customer-Facing (`/api/LMU/`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `CheckUpgradeStatus` | GET | Check if PNR has blocking orders |
| `eligibility` | GET | Full eligibility check |
| `details` | POST | Get upgrade details (passengers, segments, price) |
| `GetReservation` | POST | Retrieve reservation by ProviderRecordLocator |
| `prepare` | POST | Validate and resolve pricing, get payment methods |
| `payment` | POST | Initiate payment and create order |
| `UpdateItineraryBC` | POST | Update itinerary travel class (cron-triggered) |
| `GetOrdersByLoyaltyId` | GET | Orders by loyalty ID (last 30 days) |
| `GetOrdersByRecordLocators` | GET | Orders by PNR list |
| `GetOrderByBookingCode` | GET | Single order by booking code |

### Cron / Async (`/api/LMU/cron/`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `orders` | GET | Query orders by various filters |
| `orders/loyalty-summary` | GET | Orders with loyalty IDs (pending queue) |
| `process-loyalty-points` | POST | Process all pending loyalty orders |
| `orders/loyalty-processed` | PUT | Update loyalty processed status |
| `UpdateOrderPassengerDocumentNumbers` | PUT | Update EMD/VCR numbers per passenger |
| `UpdateOrder` | PUT | Update order status (PAID → fulfill, FAILED → release) |

### Reporting (`/api/reporting/`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `reupload/receipt/{orderId}` | POST | Re-generate receipt PDF |
| `orders` | GET | Order reporting (date range, max 7 days) |
| `fulfillment/upgrade/{orderId}` | POST | Manual upgrade trigger |
| `fulfillment/ticket/{orderId}` | POST | Manual issue-ticket trigger |
| `resend/email/{orderId}` | POST | Manual resend email |
| `price-rules` | GET/POST/PUT/DELETE | Price rule CRUD |
| `quota-config` | GET/POST/PUT | Global quota config |
| `quota-rules` | GET/POST/PUT/DELETE | Quota rule CRUD |
| `quota-remaining` | GET | Remaining quota per flight |
| `quota-usage` | GET/POST/PUT/DELETE | Quota usage CRUD |

### Admin (`/api/LMU/admin/`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `cache/keys` | GET | List Redis keys by pattern |
| `cache/key` | GET | Inspect Redis key details |
| `cache/key` | DELETE | Delete a Redis key |

### Health (`/api/Health/`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `ping` | GET | Health check (`{ status: "ok", message: "pong" }`) |

---

## Configuration & Environments

| Environment | `appEnvironment` | AWS Secrets | DB Source |
|-------------|-------------------|-------------|-----------|
| Local | `local` | Skipped | `appsettings.json` / User Secrets |
| Dev (AWS) | `dev` | Loaded | AWS Secrets Manager |
| Production | `prd` | Loaded | AWS Secrets Manager |

Config loading order: `appsettings.json` → environment-specific (`appsettings.dev-aws.json` / `appsettings.prd-aws.json`) → AWS Secrets Manager → User Secrets.

DB and Redis connections are validated at startup — the application exits with code 1 if either fails.
