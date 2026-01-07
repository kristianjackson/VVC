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
