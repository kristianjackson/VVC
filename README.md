# VIN Vehicle Report: User Journey, Schema, and API Notes

## 1. User Journey: VIN Entry ➜ Validation ➜ Data Fetching ➜ Report Rendering

1. **VIN Entry**
   - User enters a 17-character VIN into the UI input field.
   - Client-side formatting trims whitespace and uppercases the VIN.

2. **Validation**
   - **Client-side validation**: check length (17), allowed characters (A–Z, 0–9, excluding I/O/Q), and checksum if supported.
   - **Server-side validation**: re-validate the VIN, reject invalid inputs with a structured error response (see error model below).

3. **Data Fetching**
   - Backend orchestrates requests to multiple data providers (e.g., KBB, NADA).
   - Each provider request returns either:
     - **Success** with provider-specific payload, or
     - **Failure/partial** with an error code and status.
   - Backend consolidates provider responses into a single, normalized report schema.

4. **Report Rendering**
   - Frontend receives a consolidated report object and renders:
     - Core valuation data (KBB, NADA).
     - Metadata (VIN, timestamps).
     - Data-source status indicators.
   - Missing data surfaces as “Unavailable” with source-specific status details.

---

## 2. Consolidated Report Schema

The backend returns a single, normalized report document. Example structure:

```json
{
  "vin": "1HGCM82633A004352",
  "reportId": "rpt_01HXYZ123",
  "createdAt": "2024-06-01T12:34:56Z",
  "updatedAt": "2024-06-01T12:35:10Z",
  "values": {
    "kbb": {
      "amount": 15250,
      "currency": "USD",
      "asOf": "2024-06-01T12:34:58Z"
    },
    "nada": {
      "amount": 14900,
      "currency": "USD",
      "asOf": "2024-06-01T12:35:02Z"
    }
  },
  "metadata": {
    "year": 2003,
    "make": "Honda",
    "model": "Accord",
    "trim": "EX",
    "bodyStyle": "Sedan",
    "drivetrain": "FWD",
    "engine": "2.4L I4"
  },
  "timestamps": {
    "requestedAt": "2024-06-01T12:34:56Z",
    "completedAt": "2024-06-01T12:35:10Z"
  },
  "sources": {
    "kbb": {
      "status": "success",
      "statusReason": null,
      "latencyMs": 1200
    },
    "nada": {
      "status": "success",
      "statusReason": null,
      "latencyMs": 1800
    }
  }
}
```

### Schema Fields (required/optional guidance)

- **vin** *(string, required)*: validated 17-character VIN.
- **reportId** *(string, required)*: unique report identifier.
- **createdAt / updatedAt** *(ISO-8601 string, required)*: server-side timestamps.
- **values** *(object, required)*:
  - **kbb** *(object, optional)*: KBB valuation data.
  - **nada** *(object, optional)*: NADA valuation data.
  - Each value object includes:
    - **amount** *(number, required if object exists)*.
    - **currency** *(string, required if object exists)*.
    - **asOf** *(ISO-8601 string, optional)*: when provider data was last updated.
- **metadata** *(object, optional)*: decoded VIN vehicle attributes.
- **timestamps** *(object, required)*:
  - **requestedAt / completedAt** *(ISO-8601 string, required)*.
- **sources** *(object, required)*:
  - **kbb / nada** *(object, required)*:
    - **status** *(string, required)*: `success | partial | unavailable | error`.
    - **statusReason** *(string | null)*: provider error or missing data reason.
    - **latencyMs** *(number, optional)*.

---

## 3. Handling Missing or Partial Data

When a provider is unavailable or returns partial data:

- **Keep the report valid**: return the consolidated report object even if one provider fails.
- **Omit value objects**: if KBB/NADA data is missing, exclude the corresponding `values.kbb` or `values.nada`.
- **Populate source status**: set `sources.<provider>.status` to `unavailable` or `error`, and include a `statusReason`.
- **Frontend behavior**:
  - Render “Unavailable” for missing provider values.
  - Use status to show a warning badge or tooltip.
  - Avoid blocking the entire report view when one source fails.

---

## 4. API Error/Status Model and Response Format Notes

### Error Model

Use a consistent error envelope for validation and server/provider failures:

```json
{
  "error": {
    "code": "VIN_INVALID",
    "message": "VIN must be 17 characters and exclude I, O, Q.",
    "details": {
      "field": "vin",
      "reason": "checksum_failed"
    }
  },
  "status": 400,
  "timestamp": "2024-06-01T12:34:56Z",
  "requestId": "req_01HXYZABC"
}
```

### Status Codes and Notes for Frontend

- **200 OK**: consolidated report is available (even if partial).
- **400 Bad Request**: invalid VIN format/checksum.
- **404 Not Found**: VIN valid but no vehicle data exists.
- **422 Unprocessable Entity**: VIN valid but report cannot be generated due to missing required decoded fields.
- **502/503**: upstream provider outages; still return 200 with partial data when possible, otherwise return error envelope.

Frontend should:

- Treat **200** as success even if `sources.*.status` is not `success`.
- Use `sources` for source-level UI badges and tooltips.
- Use `error.code` for user-friendly error messaging and fallback UX.
# Vehicle Valuation Connector (VVC)

This document is the source of truth for provider integration guidance; if merge conflicts occur, keep this content and reconcile any provider-specific additions here.

## 1. Authentication methods and required credentials by provider

| Provider | Auth method | Required credentials | Notes |
| --- | --- | --- | --- |
| **Provider: Example VIN/Valuation A** | API key in header | `API_KEY` | Send `Authorization: Bearer <API_KEY>` for all requests. Rotate keys quarterly. |
| **Provider: Example Valuation B** | OAuth 2.0 Client Credentials | `CLIENT_ID`, `CLIENT_SECRET`, `TOKEN_URL` | Obtain access token, then send `Authorization: Bearer <access_token>`. |
| **Provider: Example Auction C** | HMAC signature | `ACCESS_KEY`, `SECRET_KEY`, `SIGNING_REGION` | Sign requests with timestamped HMAC; include `X-Access-Key` and `X-Signature`. |

> Replace the example providers with your actual vendor names once onboarded. Maintain credentials in your secret manager and reference them via environment variables.

## 2. Adapter modules per provider with standardized output fields

Each provider should have its own adapter module that returns a consistent object. Suggested interface:

```
interface ValuationAdapter {
  getValuation(input: ValuationRequest): Promise<NormalizedValuation>;
}
```

**Standardized output fields (`NormalizedValuation`):**

| Field | Type | Description |
| --- | --- | --- |
| `provider` | string | Provider identifier (e.g., `valuation_a`). |
| `request_id` | string | Provider request/trace ID when available. |
| `as_of` | string (ISO-8601) | Timestamp when valuation was produced. |
| `currency` | string (ISO-4217) | Currency code (e.g., `USD`). |
| `base_value` | number | Base valuation figure. |
| `adjustments` | array | Array of `{type, amount, description}` adjustments. |
| `final_value` | number | Total after adjustments. |
| `confidence` | number | 0–1 confidence score if provided. |
| `condition` | string | Normalized condition (`excellent`, `good`, `fair`, `poor`). |
| `trim` | string | Normalized trim name. |
| `mileage` | number | Mileage used for valuation. |
| `vin` | string | VIN when provided. |
| `raw` | object | Raw response for auditing. |

## 3. Mapping provider response fields to a normalized valuation schema

Use explicit mapping per provider. Below is a sample mapping template to be filled per integration.

### Example mapping: Provider A

| Provider field | Normalized field | Transform |
| --- | --- | --- |
| `valuation.amount` | `base_value` | Parse as float. |
| `valuation.currency` | `currency` | Uppercase ISO-4217. |
| `vehicle.miles` | `mileage` | Integer. |
| `vehicle.trim` | `trim` | Normalize to canonical trim name. |
| `valuation.condition` | `condition` | Map to `excellent/good/fair/poor`. |
| `valuation.adjustments[]` | `adjustments` | Map `{code, value}` → `{type, amount}`. |
| `meta.requestId` | `request_id` | String. |
| `meta.timestamp` | `as_of` | ISO-8601. |

### Example mapping: Provider B

| Provider field | Normalized field | Transform |
| --- | --- | --- |
| `price.base` | `base_value` | Number. |
| `price.total` | `final_value` | Number. |
| `price.currencyCode` | `currency` | Uppercase ISO-4217. |
| `vehicle.conditionGrade` | `condition` | 5→excellent, 4→good, 3→fair, else poor. |
| `vehicle.trimName` | `trim` | Normalize. |
| `vehicle.odometer` | `mileage` | Integer. |

## 4. Data quality rules

| Rule | Description | Handling |
| --- | --- | --- |
| Currency | Must be valid ISO-4217. | Reject or flag if missing/invalid. |
| Condition | Must map to `excellent/good/fair/poor`. | Use fallback mapping; flag unknown. |
| Trim | Must be a recognized trim for the make/model/year. | Normalize via lookup table; if unknown, keep raw but flag. |
| Mileage | Must be non-negative and plausible (0–400,000). | Clamp/flag outliers; record original. |
| VIN | 17 chars, alphanumeric excluding I/O/Q. | If invalid, skip VIN-based lookups and flag. |
| Valuation | `final_value` must be >= 0; `base_value` must be present. | Reject if missing base; flag negative values. |

## 5. Testing approach with sandbox/staging credentials

1. **Provider sandbox usage**
   - Prefer sandbox endpoints and credentials if provided by the vendor.
   - Maintain separate `*_SANDBOX` environment variables for credentials.
2. **Contract tests**
   - Record sample payloads and expected normalized outputs per provider.
   - Validate mapping logic and required fields.
3. **Schema validation**
   - Enforce the `NormalizedValuation` schema using JSON Schema or runtime validators.
4. **Credential verification tests**
   - For each provider, add a lightweight health check call in CI (if allowed).
   - If sandbox is unavailable, mock responses in CI and run live checks in a secured staging environment.
5. **Regression tests**
   - Store anonymized fixtures to detect changes in provider responses.

## Next steps

- Replace example providers with real vendor names and endpoints.
- Create adapter modules and mapping tables in codebase.
- Implement data quality validation and schema enforcement.
