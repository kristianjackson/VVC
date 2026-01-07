# Provider integration plan

## 1. Authentication methods and required credentials per provider

| Provider | Auth method | Required credentials | Notes |
| --- | --- | --- | --- |
| Provider A (example: OAuth2) | OAuth 2.0 client credentials | `client_id`, `client_secret`, `token_url` | Use short-lived access tokens for all API calls. Cache token until expiry. |
| Provider B (example: API key) | Static API key | `api_key`, optional `account_id` | Send via header `Authorization: ApiKey <api_key>`. |
| Provider C (example: HMAC) | HMAC signature | `api_key`, `api_secret` | Sign canonical request (method + path + timestamp). Include `X-Signature` + `X-Timestamp`. |

> Replace the placeholder providers above with the actual providers in scope and their documented auth requirements.

## 2. Adapter modules per provider with standardized output fields

Define one adapter module per provider. Each module exports a single `fetch_valuation()` function that returns a standardized response shape.

**Standard output contract (JSON):**

```json
{
  "provider": "string",
  "request_id": "string",
  "timestamp": "ISO-8601",
  "valuation": {
    "amount": 0,
    "currency": "USD",
    "condition": "excellent|good|fair|poor",
    "trim": "string",
    "mileage": 0,
    "source": "string"
  },
  "raw": {}
}
```

**Adapter module responsibilities**

- Accept a normalized request DTO (VIN, mileage, condition, location, etc.).
- Invoke the provider API using provider-specific auth.
- Map provider response into the standardized output contract.
- Return both `valuation` and `raw` for traceability.
- Raise errors using a shared error type (`ProviderError`) with `code`, `message`, and `provider`.

## 3. Normalized valuation schema and mapping

**Normalized schema (canonical fields)**

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `valuation.amount` | number | yes | Numeric valuation in `currency`. |
| `valuation.currency` | string | yes | ISO 4217 currency code. |
| `valuation.condition` | enum | yes | `excellent`, `good`, `fair`, `poor`. |
| `valuation.trim` | string | no | Provider-reported trim or normalized trim. |
| `valuation.mileage` | number | yes | Odometer reading in miles. |
| `valuation.source` | string | yes | Provider name or product line. |

**Mapping rules**

- Convert provider-specific condition labels to canonical enum values.
- Normalize currency to ISO 4217 (e.g., `US$` → `USD`).
- If provider supports multiple valuation types (trade-in, retail, private party), select the configured type and set `valuation.source` accordingly.
- Preserve any provider-specific fields in `raw` for audit/debugging.

## 4. Data quality rules

**Currency**

- Must be valid ISO 4217.
- If provider returns unsupported currency, mark response as invalid and surface error `unsupported_currency`.

**Condition**

- Map provider conditions to the canonical enum.
- If missing, infer from request; otherwise set to `unknown` and flag as a data-quality warning.

**Trim**

- Prefer provider-reported trim; fallback to request trim.
- If trim is absent, allow `null` and flag as `missing_trim` warning.

**Mileage**

- Must be non-negative integer.
- If mileage is missing, reject with `missing_mileage` error.
- If mileage is implausible (e.g., > 500,000), accept but flag as `outlier_mileage` warning.

## 5. Testing approach with sandbox/staging credentials

- Maintain a `test_credentials.yaml` template listing each provider and required fields.
- For providers offering sandbox/staging, store credentials in a secret manager and load via environment variables.
- For providers without sandbox support, mock responses using fixtures captured from production (redact PII).
- Write contract tests that validate the standardized output contract.
- Write mapping tests that verify provider response → normalized schema conversions.
- Include a small set of “golden” fixtures per provider to detect schema drift.

**Example test matrix**

| Test type | Description | Tooling |
| --- | --- | --- |
| Contract tests | Validate standardized output fields and types. | Jest/Pytest (repo standard) |
| Mapping tests | Verify field mapping and normalization. | Unit tests per adapter |
| Auth tests | Token refresh / signature creation. | Unit tests with mocked HTTP |
| Integration tests | End-to-end calls to sandbox. | Scheduled or manual |

