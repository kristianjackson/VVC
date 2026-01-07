# Vehicle Valuation Connector (VVC)

## 1. Authentication methods and required credentials per provider

| Provider | Auth method(s) | Required credentials | Notes |
| --- | --- | --- | --- |
| **Provider: Autovalue** | API key header | `AUTOVALUE_API_KEY` | Send as `X-Api-Key` header. |
| **Provider: MarketLens** | OAuth 2.0 client credentials | `MARKETLENS_CLIENT_ID`, `MARKETLENS_CLIENT_SECRET`, `MARKETLENS_TOKEN_URL` | Fetch `access_token` then send as `Authorization: Bearer <token>`. |
| **Provider: DealerPulse** | Basic auth | `DEALERPULSE_USERNAME`, `DEALERPULSE_PASSWORD` | Send as HTTP basic auth. |
| **Provider: QuoteStream** | API key + partner ID | `QUOTESTREAM_API_KEY`, `QUOTESTREAM_PARTNER_ID` | `X-Api-Key` + `X-Partner-Id` headers. |
| **Provider: ValuSight** | mTLS + API key | `VALUSIGHT_CLIENT_CERT`, `VALUSIGHT_CLIENT_KEY`, `VALUSIGHT_API_KEY` | Provide client cert/key; also `X-Api-Key`. |

> Replace provider names/credentials with the actual vendor set in production. The table is the canonical place to document changes.

## 2. Adapter modules per provider (standardized outputs)

Create one adapter per provider and standardize the output contract so upstream services always receive the same shape.

**Adapter module convention**

```
/adapters
  autovalue_adapter.py
  marketlens_adapter.py
  dealerpulse_adapter.py
  quotestream_adapter.py
  valusight_adapter.py
```

**Standardized adapter output** (all adapters must emit this shape)

```
{
  "provider": "<provider_name>",
  "requested_vin": "<vin>",
  "requested_region": "<region_code>",
  "request_id": "<provider_request_id>",
  "valuation": {
    "value": 12345.67,
    "currency": "USD",
    "valuation_type": "retail|trade|private_party|auction",
    "as_of": "2024-01-01T00:00:00Z"
  },
  "vehicle": {
    "vin": "<vin>",
    "year": 2020,
    "make": "Toyota",
    "model": "Camry",
    "trim": "SE",
    "mileage": 45000,
    "condition": "good",
    "body_style": "sedan",
    "drivetrain": "FWD",
    "fuel_type": "gasoline",
    "transmission": "automatic"
  },
  "metadata": {
    "source": "<provider_name>",
    "confidence": 0.85,
    "warnings": ["string"],
    "raw_response_ref": "<optional storage key>"
  },
  "errors": []
}
```

## 3. Normalized valuation schema mapping

Map each provider response into a single normalized schema used by downstream services.

**Normalized valuation schema**

| Field | Type | Description |
| --- | --- | --- |
| `provider` | string | Provider name. |
| `requested_vin` | string | VIN submitted by caller. |
| `requested_region` | string | Region/state/market code. |
| `request_id` | string | Provider request/transaction ID. |
| `valuation.value` | number | Numeric valuation value. |
| `valuation.currency` | string | ISO 4217 code (e.g., `USD`). |
| `valuation.valuation_type` | string | `retail`, `trade`, `private_party`, `auction`. |
| `valuation.as_of` | string (ISO 8601) | Provider valuation timestamp. |
| `vehicle.vin` | string | VIN from provider (if returned). |
| `vehicle.year` | number | Model year. |
| `vehicle.make` | string | Make. |
| `vehicle.model` | string | Model. |
| `vehicle.trim` | string | Trim. |
| `vehicle.mileage` | number | Mileage (miles). |
| `vehicle.condition` | string | Normalized condition. |
| `vehicle.body_style` | string | Body style. |
| `vehicle.drivetrain` | string | Drivetrain. |
| `vehicle.fuel_type` | string | Fuel type. |
| `vehicle.transmission` | string | Transmission. |
| `metadata.source` | string | Provider name. |
| `metadata.confidence` | number | Confidence score (0-1). |
| `metadata.warnings` | string[] | Provider warning strings. |
| `metadata.raw_response_ref` | string | Optional storage reference. |
| `errors` | array | Normalized error objects. |

**Example field mapping**

| Normalized field | Autovalue | MarketLens | DealerPulse | QuoteStream | ValuSight |
| --- | --- | --- | --- | --- | --- |
| `valuation.value` | `price.retail` | `valuation.amount` | `values.retail_value` | `valuation.value` | `result.price` |
| `valuation.currency` | `price.currency` | `valuation.currency_code` | `values.currency` | `valuation.currency` | `result.currency` |
| `valuation.as_of` | `price.as_of` | `valuation.timestamp` | `values.as_of_date` | `valuation.as_of` | `result.as_of` |
| `vehicle.trim` | `vehicle.trim` | `vehicle.trim_name` | `vehicle.trim` | `vehicle.trim` | `vehicle.trim_level` |
| `vehicle.mileage` | `vehicle.mileage` | `vehicle.miles` | `odometer.miles` | `vehicle.mileage` | `vehicle.mileage` |

> Keep provider-specific mapping tables in the adapter module documentation if fields diverge beyond this table.

## 4. Data quality rules

Apply validation and normalization rules before emitting the standardized output:

1. **Currency normalization**
   - Convert to ISO 4217 currency codes.
   - Reject/flag unsupported currencies.
2. **Condition normalization**
   - Normalize condition values to: `excellent`, `good`, `fair`, `poor`.
   - Map provider-specific conditions accordingly.
3. **Trim normalization**
   - Trim strings must be non-empty; title-case canonical forms.
   - If trim is missing, set `"unknown"` and add a warning.
4. **Mileage validation**
   - Must be numeric and non-negative.
   - If provider sends km, convert to miles and record warning.
5. **VIN validation**
   - 17-character VIN; reject I/O/Q characters.
   - If provider returns a different VIN, include both `requested_vin` and `vehicle.vin` with a warning.
6. **Valuation range checks**
   - Value must be > 0 and < 1,000,000 (configurable upper bound).
   - If out of range, mark as error and exclude valuation.
7. **As-of timestamp**
   - Must be ISO 8601; if missing, fill with request timestamp and warning.
8. **Regional consistency**
   - If provider’s region/market differs from request, add warning and record `metadata.warnings`.

## 5. Testing approach (sandbox/staging credentials)

1. **Provider sandbox usage**
   - If a provider offers sandbox/staging endpoints, store them in config as `*_BASE_URL`.
   - Keep sandbox credentials in a secrets manager and load via environment.
2. **Contract tests**
   - Use sample responses (recorded fixtures) to verify adapter output shape.
   - Validate normalization logic against all data quality rules.
3. **Integration tests**
   - Run live sandbox calls nightly to validate auth flows and field mapping.
   - Capture provider request IDs and ensure `request_id` is populated.
4. **Failure-mode tests**
   - Simulate missing fields, unknown trim, invalid mileage, and currency mismatch.
   - Assert `errors` populated and `metadata.warnings` emitted.
5. **Credential rotation checks**
   - Add a periodic test to validate credentials before expiry (if provider supports).

---

If you need provider-specific examples or want this spec expanded into a full integration guide, add real provider response samples and update the mapping table accordingly.
