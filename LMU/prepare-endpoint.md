# POST /api/LMU/prepare - Endpoint Documentation

## Overview

The `/prepare` endpoint is a critical pre-payment step in the LMU (Last-Minute Upgrade) flow. It validates the reservation, resolves pricing (including loyalty/special quota), and fetches available payment methods before the actual payment transaction occurs.

**Endpoint:** `POST /api/LMU/prepare`  
**Controller:** `ApiController.cs`  
**Provider:** `LMUPaymentProvider.PrepareAsync()`

---

## Request

### Body (`LMUPrepareRequest`)

```json
{
  "recordLocator": "ABC123",
  "passengers": [
    {
      "firstName": "John",
      "lastName": "Doe",
      "passengerNumber": 1,
      "nameId": "1.1"
    }
  ],
  "contactDetails": {
    "title": "Mr",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "+628123456789"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `recordLocator` | string | Yes | PNR (Passenger Name Record) locator |
| `passengers` | array | Yes | List of passengers to upgrade |
| `contactDetails` | object | Yes | Customer contact information |

---

## Response (`LMUPrepareResponse`)

```json
{
  "isSuccess": true,
  "paymentFlag": true,
  "paymentMethods": [...],
  "recordLocator": "ABC123",
  "bookingCode": "ABC123-1",
  "remainingDuration": 3600,
  "currency": "IDR",
  "totalAmount": 1500000,
  "price": {
    "adult": 1500000,
    "infant": 150000
  },
  "freePax": 0,
  "freeQuotaRedeemedNotice": null,
  "message": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `isSuccess` | bool | Whether the prepare operation succeeded |
| `paymentFlag` | bool | `true` = payment required; `false` = free upgrade (zero-price) |
| `paymentMethods` | array | Available payment methods (when `paymentFlag=true`) |
| `recordLocator` | string | The PNR locator |
| `bookingCode` | string | Booking code (set for zero-price fulfillment) |
| `remainingDuration` | int | Seconds remaining before booking expires |
| `currency` | string | Currency code (e.g., "IDR") |
| `totalAmount` | decimal | Total upgrade amount |
| `price` | object | Per-passenger pricing breakdown |
| `freePax` | int | Number of passengers eligible for free upgrade |
| `freeQuotaRedeemedNotice` | object | Notice when free quota unavailable but paid upgrade available |
| `message` | object | Error details (when `isSuccess=false`) |

---

## Error Codes

### Validation Errors (1000–1999)

| Code | Constant | Description |
|------|----------|-------------|
| 1000 | `RequestRequired` | Request body is null or missing required fields |
| 1001 | `RecordLocatorRequired` | Record locator is missing |
| 1002 | `CarrierCodeRequired` | Carrier code is missing |
| 1003 | `EmailInvalid` | Email format is invalid |
| 1004 | `NoPassengersSelected` | No passengers selected for upgrade |
| 1005 | `RecordLocatorNotFound` | Record locator not found in system |

### Processing Errors (2000+)

| Code | Constant | Description |
|------|----------|-------------|
| 2000 | `DetailsFailed` | Failed to retrieve reservation details |
| 2002 | `ZeroPriceFailed` | Zero-price fulfillment failed |
| 2003 | `PaymentMethodsUnavailable` | Payment methods service unavailable |
| 2004 | `NoPaymentMethodsForCurrency` | No payment methods for the currency |
| 2005 | `UnableToRetrievePaymentMethods` | Error fetching payment methods |
| 2006 | `AlreadyBusinessClass` | Passenger already in business class |
| 2007 | `HasAncillaryServicesExcl99` | Has ancillary services (excluding 99) |
| 2008 | `MultipleSegments` | Multiple segments not supported |
| 2009 | `VcrStatusNotEligible` | VCR status not eligible |
| 2010 | `ScheduleChange` | Schedule change detected |

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          POST /api/LMU/prepare                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │  Validate Request     │
                          │  - recordLocator      │
                          │  - contactDetails     │
                          │  - email format       │
                          └───────────┬───────────┘
                                      │
                         ┌────────────┴────────────┐
                         │ Validation Failed?      │
                         │ (error code 1000-1005)   │
                         └────────────┬────────────┘
                               Yes    │    No
                         ┌────────────┴────────────┐
                         │                         │
                         ▼                         ▼
              ┌─────────────────┐    ┌─────────────────────────────┐
              │ Return 400 Bad  │    │ Store Contact Details in    │
              │ Request         │    │ Redis Cache                 │
              └─────────────────┘    └──────────────┬──────────────┘
                                                    │
                                                    ▼
                                      ┌─────────────────────────────┐
                                      │ Fetch Reservation Details   │
                                      │ (GetDetailsAsync with cache)│
                                      └──────────────┬──────────────┘
                                                    │
                                       ┌────────────┴────────────┐
                                       │ Details Failed?         │
                                       │ (BusinessClassFull,     │
                                       │  QuotaExceeded, etc.)   │
                                       └────────────┬────────────┘
                                              Yes   │   No
                                       ┌────────────┴────────────┐
                                       │                         │
                                       ▼                         ▼
                            ┌─────────────────┐    ┌─────────────────────────────┐
                            │ Return Error     │    │ Check Upgrade In Progress   │
                            │ (error 2000+)    │    │ (UpgradeBlockService)       │
                            └─────────────────┘    └──────────────┬──────────────┘
                                                                 │
                                                    ┌────────────┴────────────┐
                                                    │ Upgrade In Progress?    │
                                                    └────────────┬────────────┘
                                                          Yes    │    No
                                                    ┌────────────┴────────────┐
                                                    │                         │
                                                    ▼                         ▼
                                         ┌─────────────────┐    ┌─────────────────────────────┐
                                         │ Return Error     │    │ Validate Passenger Count    │
                                         │ (UpgradeInProgress)│  │ > 0                         │
                                         └─────────────────┘    └──────────────┬──────────────┘
                                                                              │
                                                                 ┌────────────┴────────────┐
                                                                 │ passengerCount <= 0?    │
                                                                 └────────────┬────────────┘
                                                                       Yes    │    No
                                                                 ┌────────────┴────────────┐
                                                                 │                         │
                                                                 ▼                         ▼
                                                      ┌─────────────────┐    ┌─────────────────────────────┐
                                                      │ Return Success  │    │ Match Passengers with       │
                                                      │ PaymentFlag=false│   │ Details (by name)           │
                                                      │ (NoSelection)   │    └──────────────┬──────────────┘
                                                      └─────────────────┘                   │
                                                                                             │
                                                                                             ▼
                                                                              ┌─────────────────────────────┐
                                                                              │ Check Infant Policy         │
                                                                              │ (PreventPartialUpgrade      │
                                                                              │  WithInfant)                │
                                                                              └──────────────┬──────────────┘
                                                                                             │
                                                                                ┌────────────┴────────────┐
                                                                                │ Partial Upgrade + Infant?│
                                                                                └────────────┬────────────┘
                                                                                      Yes   │   No
                                                                                ┌────────────┴────────────┐
                                                                                │                         │
                                                                                ▼                         ▼
                                                                     ┌─────────────────┐    ┌─────────────────────────────┐
                                                                     │ Return Error     │    │ Match Loyalty Profiles      │
                                                                     │ (PartialUpgrade  │    │ for Passengers              │
                                                                     │  NotAllowed)     │    └──────────────┬──────────────┘
                                                                     └─────────────────┘                   │
                                                                                                           │
                                                                                                           ▼
                                                                                             ┌─────────────────────────────┐
                                                                                             │ Resolve Pricing             │
                                                                                             │ (ILMUDetailsPriceResolver)  │
                                                                                             │ - Include special quota if  │
                                                                                             │   loyalty passenger exists  │
                                                                                             └──────────────┬──────────────┘
                                                                                                            │
                                                                                               ┌────────────┴────────────┐
                                                                                               │ QuotaExceeded?           │
                                                                                               └────────────┬────────────┘
                                                                                                     Yes   │   No
                                                                                               ┌────────────┴────────────┐
                                                                                               │                         │
                                                                                               ▼                         ▼
                                                                                    ┌─────────────────┐    ┌─────────────────────────────┐
                                                                                    │ Return Error     │    │ Calculate Total Amount      │
                                                                                    │ (QuotaExceeded)  │    │ (adult/child + infant)      │
                                                                                    └─────────────────┘    └──────────────┬──────────────┘
                                                                                                                       │
                                                                                                          ┌────────────┴────────────┐
                                                                                                          │ totalAmount == 0?       │
                                                                                                          │ (Zero-Price / Free)     │
                                                                                                          └────────────┬────────────┘
                                                                                                                Yes   │   No
                                                                                                          ┌────────────┴────────────┐
                                                                                                          │                         │
                                                                                                          ▼                         ▼
                                                                                           ┌─────────────────────────┐    ┌─────────────────────────────┐
                                                                                           │ Zero-Price Fulfillment  │    │ Check ForceRpOne Override   │
                                                                                           │ (FulfillZeroPriceOrder) │    │ (test mode: amount=1 IDR)   │
                                                                                           │ - Store LMUOrder        │    └──────────────┬──────────────┘
                                                                                           │ - Deduct special quota  │                   │
                                                                                           │ - Call Fulfillment API  │                   │
                                                                                           └──────────────┬──────────┘                   │
                                                                                                          │                         │
                                                     ┌────────────────────────────────────────────────────┼─────────────────────────┘
                                                     │                                                    │
                                                     ▼                                                    ▼
                                          ┌─────────────────┐                             ┌─────────────────────────────────────┐
                                          │ Success!        │                             │ Fetch Payment Methods               │
                                          │ PaymentFlag=false│                            │ (GetPaymentMethodsAsync)            │
                                          │ (Free Upgrade)  │                             │ - ProductType, Currency, Amount     │
                                          └─────────────────┘                             └──────────────┬──────────────────────┘
                                                                                                        │
                                                                                           ┌────────────┴────────────┐
                                                                                           │ Payment Methods Failed?  │
                                                                                           └────────────┬────────────┘
                                                                                                 Yes   │   No
                                                                                           ┌────────────┴────────────┐
                                                                                           │                         │
                                                                                           ▼                         ▼
                                                                                ┌─────────────────────┐    ┌─────────────────────────────┐
                                                                                │ Return 502 Bad      │    │ Filter by Currency           │
                                                                                │ Gateway             │    │ (active + matching currency) │
                                                                                └─────────────────────┘    └──────────────┬──────────────┘
                                                                                                                        │
                                                                                                           ┌────────────┴────────────┐
                                                                                                           │ No Methods for Currency? │
                                                                                                           └────────────┬────────────┘
                                                                                                                 Yes   │   No
                                                                                                           ┌────────────┴────────────┐
                                                                                                           │                         │
                                                                                                           ▼                         ▼
                                                                                              ┌─────────────────────┐    ┌─────────────────────────────┐
                                                                                              │ Return 502 Bad      │    │ Persist Free Quota Reserved │
                                                                                              │ Gateway             │    │ (if special quota eligible) │
                                                                                              └─────────────────────┘    └──────────────┬──────────────┘
                                                                                                                                       │
                                                                                                                                       ▼
                                                                                                                           ┌─────────────────────────────┐
                                                                                                                           │ Return Success              │
                                                                                                                           │ PaymentFlag=true            │
                                                                                                                           │ + PaymentMethods            │
                                                                                                                           └─────────────────────────────┘
```

---

## Key Behaviors

### 1. Contact Details Storage
Contact details (title, firstName, lastName, email, phone) are validated and stored in Redis cache for use at payment time. They are persisted to `LMUOrderContactDetails` only when payment occurs.

### 2. Pricing Resolution
- **Authenticated + Loyalty Passenger**: Special/free quota pricing may apply
- **No Loyalty**: Normal (paid) pricing only
- **Quota Exceeded**: Returns error immediately

### 3. Zero-Price Flow (Free Upgrade)
When `totalAmount = 0` (loyalty eligible):
1. Store `LMUOrder` and `LMUOrderPassenger`
2. Deduct special quota
3. Call Fulfillment API
4. Return `PaymentFlag=false` with `BookingCode`

### 4. Paid Flow
When `totalAmount > 0`:
1. Fetch available payment methods from payment gateway
2. Filter by currency and active status
3. Persist free quota reservation (if applicable)
4. Return `PaymentFlag=true` with filtered `PaymentMethods`

### 5. Free Quota Redeemed Notice
When details indicate special quota is available but pricing resolves to paid (e.g., loyalty slots exhausted), a `FreeQuotaRedeemedNotice` is included to inform the UI.

---

## HTTP Status Codes

| Status | Condition |
|--------|-----------|
| 200 OK | Success (both paid and free upgrade) |
| 400 Bad Request | Validation errors or processing errors |
| 502 Bad Gateway | Payment methods service unavailable |

---

## Dependencies

- `ILMUDetailsProvider` - Reservation details and caching
- `ILMUDetailsPriceResolver` - Pricing computation
- `IPaymentMethodsClient` - Payment method retrieval
- `ILMUQuotaRepository` - Special quota management
- `UpgradeBlockService` - Upgrade blocking checks
- `ILMUFulfillmentClient` - Zero-price fulfillment (GOBC API)

---

## Example Scenarios

### Scenario 1: Paid Upgrade
```
Request: 2 adult passengers, IDR currency
Pricing: IDR 750,000 per passenger
Total:   IDR 1,500,000
Result:  PaymentFlag=true, PaymentMethods=[CreditCard, VirtualAccount, ...]
```

### Scenario 2: Free Upgrade (Loyalty)
```
Request: 1 authenticated loyalty passenger
Pricing: Special quota = 0 (free)
Total:   IDR 0
Result:  PaymentFlag=false, BookingCode="ABC123-1"
Action:  Order created, fulfillment API called
```

### Scenario 3: Free Quota Exhausted
```
Request: 2 passengers, 1 loyalty eligible
Details: Special quota available but slots exhausted
Pricing: Falls back to normal price
Total:   IDR 1,500,000
Result:  PaymentFlag=true, FreeQuotaRedeemedNotice included
```
