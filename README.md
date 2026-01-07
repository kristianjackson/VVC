# Vehicle Valuation Connector (VVC)

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
