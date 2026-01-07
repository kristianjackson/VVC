# Vehicle Valuation Consolidation (VVC) — PRD (Concise)

## 1) User journey (VIN → validation → provider lookups → report)
1. **VIN entry**
   - User enters a VIN in the VVC UI and submits.
   - System performs basic format checks (length 17, disallow I/O/Q, checksum if available).
2. **Validation & enrichment**
   - Decode VIN to confirm make/model/year/trim and normalize vehicle attributes.
   - If invalid or ambiguous, present user with corrections or error.
3. **Provider lookups**
   - Use normalized vehicle attributes to query valuation providers (KBB, NADA/J.D. Power, others).
   - Apply provider-specific parameter mapping (trim, options, mileage, condition, location).
   - Respect provider rate limits and licensing terms.
4. **Consolidated report view**
   - Display a summary view with provider valuations, ranges, timestamps, and confidence notes.
   - Show attribution required by each provider.
   - Provide export/share options and an audit log reference.

## 2) Required data providers & availability/licensing constraints
### Required
- **Kelley Blue Book (KBB)**  
  - **API availability**: Commercial data feeds and licensing available; public APIs are limited and not for automated valuation use.  
  - **Licensing/usage**: Requires paid license; attribution and display rules apply; may limit caching, redistribution, and usage in competing products.
- **NADA / J.D. Power**  
  - **API availability**: Commercial automotive valuation data products available via J.D. Power licensing.  
  - **Licensing/usage**: Paid license with strict usage terms and attribution; often requires authenticated access, rate limits, and audit compliance.

### Optional/Alternate
- **Black Book**  
  - **API availability**: Commercial vehicle valuation data products.  
  - **Licensing/usage**: Paid license; restrictions on redistribution and display.  
- **Manheim (MMR)**  
  - **API availability**: Commercial data access, often restricted to approved partners.  
  - **Licensing/usage**: Paid license; access controls and use-case approval.  
- **Edmunds**  
  - **API availability**: Limited public APIs and paid partnerships.  
  - **Licensing/usage**: Licensing varies; confirm attribution and caching rules.  

> **Action**: Legal/procurement must confirm each provider’s current API availability, contract terms, and allowable use-cases before implementation.

## 3) Non-functional requirements (NFRs)
- **Rate limits**
  - Respect provider-specific limits; implement request throttling, backoff, and queueing.
  - Cache responses within licensing allowances to reduce calls.
- **Uptime/availability**
  - Target ≥99.5% system availability; degrade gracefully if a provider is unavailable.
  - Provide partial results with clear “provider unavailable” indicators.
- **PII handling**
  - VINs may be considered sensitive; store minimal data, encrypt in transit and at rest.
  - No sharing beyond licensed providers; log access strictly.
- **Audit & traceability**
  - Record request parameters, provider responses (or hashes if retention restricted), timestamps, and user identity.
  - Provide immutable audit trail for compliance and billing disputes.

## 4) Compliance checklist (licensed valuation data)
- **License verification**
  - Confirm license coverage (geography, vehicle types, use-case).
  - Validate restrictions on caching, redistribution, and derivative analytics.
- **Attribution**
  - Display provider branding and attribution text per contract.
  - Preserve required logos/trademarks and “as-of” timestamps.
- **Data retention**
  - Follow provider retention rules; purge or anonymize data when required.
- **Access controls**
  - Restrict access to licensed users; enforce authentication and authorization.
- **Audit readiness**
  - Provide audit logs and usage reports on demand.

## 5) PRD scope & constraints (concise)
### In scope
- VIN entry, validation, and decoding.
- Provider integrations for KBB and NADA/J.D. Power (phase 1).
- Consolidated report view with attribution and audit metadata.
- Rate limiting, caching (within license), and partial-result handling.

### Out of scope (initial)
- Real-time auction feeds (e.g., Manheim MMR) unless licensed.
- Predictive valuation analytics beyond provider data.
- Consumer-facing public API without licensing agreements.

### Constraints & assumptions
- Provider contracts must be executed before production launch.
- Usage must comply with each provider’s attribution and display rules.
- System must support provider outages and still deliver partial results.
