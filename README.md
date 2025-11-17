# AiGent → RentSync

**External API Integration Guide**

**Version:** 1.0  
**Last Updated:** November 2025  
**Status:** Draft for Partner Review

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Data Model Overview](#2-data-model-overview)
3. [Daily Sync Flow](#3-daily-sync-flow)
4. [Outbound Payload Examples](#4-outbound-payload-examples)
5. [Authentication](#5-authentication)
6. [Proposed Request Structure](#6-proposed-request-structure)
7. [Error Handling](#7-error-handling)
8. [Questions for RentSync](#8-questions-for-rentsync)
9. [Next Steps](#9-next-steps)

---

## 1. Introduction

### About AiGent

AiGent is a property management automation platform used across multiple residential real-estate portfolios. Our system centralizes:

- Unit availability
- Pricing
- Lead & leasing operations
- Communications and workflow automation

This document describes how AiGent can send daily rent-roll updates to RentSync, containing:

- Unit availability status
- Pricing
- Unit specifications
- Basic building / property information

> **Note:** This is a proposed integration format based on AiGent's existing data structure. The final format will depend on RentSync's requirements.

---

## 2. Data Model Overview

Below are the fields AiGent currently maintains for units and properties. These fields can be included in the rent-roll sync depending on RentSync's needs.

### 2.1 Property / Project

Properties are represented through the `Project` model.

| Field | Type | Description |
|-------|------|-------------|
| `id` | Integer | Unique property ID |
| `name` | String | Property name |
| `project_group_id` | Integer | Portfolio/group ID |
| `full_address` | String | Full property address |
| `is_actived` | Boolean | Whether the property is active |
| `is_lease_up` | Boolean | Whether the property is in lease-up |

### 2.2 Units (Rent-Roll)

Units are represented through the `ProjectRentRollUnit` model.

#### Core Identity Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | Integer | AiGent unit ID |
| `project_id` | Integer | Property ID |
| `unit_code` | String | Unit number (e.g. "101") |
| `building` | String | Building identifier |
| `floor` | Integer | Floor number |
| `floorplan` | String | Floorplan name/code |
| `type` | String | Unit type (e.g., "1 Bed + Den") |

#### Availability Fields

| Field | Type | Description |
|-------|------|-------------|
| `rentroll_status_id` | Integer | Status ID |
| `rentroll_status` | String | Status name (e.g., "Available", "Rented") |
| `when_available` | String | "Immediate", "December", etc. |

> **Note:** AiGent can derive `is_available: true/false` based on `rentroll_status_id`.

#### Pricing

| Field | Type | Description |
|-------|------|-------------|
| `price` | Float | Base monthly rent |
| `price_amortized` | Float (nullable) | Alternative price field if available |

#### Unit Specs

| Field | Type | Description |
|-------|------|-------------|
| `size` | Integer | Unit size (sqft) |
| `bedrooms` | Float | Bedrooms |
| `bathrooms` | Float | Bathrooms |
| `dens` | Integer | Dens |
| `dwelling_type` | String | Apartment, Townhome, Condo |

#### Additional Attributes

| Field | Type | Description |
|-------|------|-------------|
| `exposure` | String | SW, NE, etc. |
| `view` | String | Description of the unit's view |
| `has_garage` | Boolean | Garage included |
| `pets` | Integer | Pets allowed |
| `parking_stalls` | Integer | Parking count |
| `storage_boxes` | Integer | Storage count |

#### Lead Assignment

| Field | Type | Description |
|-------|------|-------------|
| `lead_first_name` | String | Assigned lead first name |
| `lead_last_name` | String | Assigned lead last name |

> **Note:** Included only when a lead is assigned to the unit.

---

## 3. Daily Sync Flow

### 3.1 Schedule

AiGent can send rent-roll updates on a daily schedule via:

- AWS Lambda (EventBridge cron)
- Server cron job
- Background worker

### 3.2 High-Level Steps

1. Identify all active properties configured for RentSync.
2. Fetch all units from each property's rent-roll.
3. Transform the records into the integration payload.
4. POST the data to RentSync's endpoint.
5. Log outcomes and retry failures when needed.

### 3.3 Delivery Modes Supported by AiGent

| Mode | Description |
|------|-------------|
| **Webhook POST** | RentSync provides a URL; AiGent sends data daily |
| **REST Endpoint** | AiGent sends authenticated POST requests |
| **Batch per property** | One POST per property |
| **Single batch for all properties** | One POST containing all projects + units |

---

## 4. Outbound Payload Examples

### 4.1 Example — Single Unit

```json
{
  "unit_id": 12345,
  "project_id": 101,
  "project_name": "Sunset Apartments",
  "property_address": "123 Main Street, Calgary, AB",
  "unit_code": "101",
  "building": "A",
  "floor": 1,
  "floorplan": "1BR-OPEN",
  "type": "1 Bed + Den",
  "dwelling_type": "Apartment",
  "status": {
    "id": 1,
    "name": "Available"
  },
  "availability": {
    "is_available": true,
    "when_available": "Immediate"
  },
  "pricing": {
    "base_rent": 1500.0,
    "price_amortized": 1425.0
  },
  "specifications": {
    "size_sqft": 750,
    "bedrooms": 1.0,
    "bathrooms": 1.0,
    "dens": 1
  },
  "attributes": {
    "exposure": "SW",
    "view": "City View",
    "has_garage": true,
    "pets_allowed": 1
  },
  "lead_assignment": {
    "lead_first_name": "John",
    "lead_last_name": "Doe"
  },
  "sync_timestamp": "2025-01-15T02:00:00Z"
}
```

### 4.2 Example — Full Property (Multiple Units)

```json
{
  "sync_metadata": {
    "sync_id": "sync-2025-01-15-02-00-00",
    "sync_timestamp": "2025-01-15T02:00:00Z",
    "source": "aigent"
  },
  "property": {
    "project_id": 101,
    "project_name": "Sunset Apartments",
    "address": "123 Main Street, Calgary, AB"
  },
  "units": [
    {
      "unit_id": 12345,
      "unit_code": "101",
      "status_name": "Available",
      "is_available": true,
      "base_rent": 1500.0
    },
    {
      "unit_id": 12346,
      "unit_code": "102",
      "status_name": "Rented",
      "is_available": false,
      "base_rent": 2100.0
    }
  ]
}
```

### 4.3 Example — Flat Format (One Row per Unit)

```json
{
  "sync_timestamp": "2025-01-15T02:00:00Z",
  "units": [
    {
      "property_id": 101,
      "unit_id": 12345,
      "unit_code": "101",
      "status_id": 1,
      "status_name": "Available",
      "is_available": true,
      "base_rent": 1500.0,
      "bedrooms": 1.0,
      "bathrooms": 1.0
    }
  ]
}
```

---

## 5. Authentication

AiGent can authenticate outbound requests using any method required by RentSync:

### Supported Methods

#### API Key in Header

```http
X-API-Key: <key>
```

or

```http
Authorization: Bearer <key>
```

#### HMAC Signed Payload

#### OAuth2 Client Credentials

#### Static Token

### Recommended

**API Key in Header**

---

## 6. Proposed Request Structure

### Example URL (to be defined by RentSync)

```
POST https://api.rentsync.com/v1/rent-roll/sync
```

### Example Headers

```http
Content-Type: application/json
X-API-Key: <rentsync-key>
User-Agent: AiGent/1.0
X-Aigent-Sync-ID: sync-2025-01-15
```

### cURL Example

```bash
curl -X POST "https://api.rentsync.com/v1/rent-roll/sync" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <rentsync-key>" \
  -d @payload.json
```

---

## 7. Error Handling

### Retry Logic

AiGent retries on transient failures using exponential backoff:

- Retry after: 1m → 2m → 5m → 10m
- Max: 5 attempts
- Permanent failures are logged for manual review.

### Error Types & Actions

| Status | Meaning | AiGent Action |
|--------|---------|---------------|
| 400 | Invalid payload | Log + no retry |
| 401/403 | Authentication error | Alert + no retry |
| 404 | Endpoint not found | Alert |
| 429 | Rate limit | Retry after delay |
| 500–504 | Server error | Retry with backoff |

---

## 8. Questions for RentSync

To finalize this integration in the best possible way, we kindly request RentSync's feedback on the following points:

### 1. Preferred payload structure

- Nested (property → units) or flat (one unit per row)?
- One request per property or one batch containing all properties?

### 2. Required vs optional fields

- Which fields are mandatory?
- Which fields are optional but recommended?

### 3. Identifiers

- How should AiGent map property IDs and unit codes to RentSync's identifiers?

### 4. Update strategy

- Do you expect full daily snapshots, delta updates, or both?
- How should we represent units that were removed or no longer available?

### 5. Sync frequency

- Once daily? Multiple times per day?
- Preferred timing of sync?

### 6. Authentication method

- API Key? Bearer token? HMAC signature?
- Any specific header requirements?

### 7. Testing environment

- Do you provide a staging/sandbox endpoint?
- Are there rate limits we should be aware of?

### 8. Error handling

- What does a typical success response look like?
- Do you return unit-level validation errors?
- Any retry recommendations?

---

## 9. Next Steps

Once RentSync provides their preferred format and requirements, AiGent will:

1. Finalize the payload schema
2. Implement authentication
3. Build the scheduled sync service
4. Generate test payloads
5. Conduct staging validation
6. Deploy the production integration

---

**End of Document**

**AiGent Integration Team**
