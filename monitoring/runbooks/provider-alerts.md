# Provider Alert Runbook

## Overview
This runbook covers provider-level alerts for error rate, latency, and cache hit rate.

## Ownership
* Primary: Platform Observability (Slack: #platform-observability)
* Secondary: Provider Integrations (Slack: #provider-integrations)
* Escalation: On-call SRE, then Engineering Manager for Platform

---

## Provider error rate high
Anchor: `#provider-error-rate-high`

### How to verify provider status
1. Check provider status dashboard (internal status page) for incident banners or degradation.
2. Query recent error spikes:
   * `sum(rate(provider_requests_total{status=~"5.."}[5m])) by (provider)`
   * `sum(rate(provider_requests_total[5m])) by (provider)`
3. Inspect request logs for upstream error codes (timeouts, 5xx, rate-limit).

### Mitigations
* Temporarily disable the affected provider in routing config.
* Reduce request timeouts or retry budget to prevent queue buildup.
* Enable fallback provider or circuit breaker.
* Coordinate with provider support if errors are upstream.

### Escalation
* If sustained >15m or customer-impacting: page on-call SRE.
* If provider acknowledges incident: keep Product/Support informed and update status page.

---

## Provider latency p95 high
Anchor: `#provider-latency-p95-high`

### How to verify provider status
1. Check p95 latency per provider:
   * `histogram_quantile(0.95, sum(rate(provider_request_duration_seconds_bucket[5m])) by (le, provider))`
2. Compare current latency to historical baseline dashboard.
3. Review saturation metrics (queue depth, retry rate).

### Mitigations
* Shift traffic to faster providers.
* Lower timeout or concurrency for the impacted provider.
* Enable cache or relaxed latency SLO in routing.

### Escalation
* If p95 > 2s for 20m: page on-call SRE.
* If customer SLAs at risk: notify Engineering Manager for Platform.

---

## Provider cache hit rate low
Anchor: `#provider-cache-hit-rate-low`

### How to verify provider status
1. Validate cache hit ratio:
   * `sum(rate(provider_cache_requests_total{result="hit"}[15m])) by (provider)`
   * `sum(rate(provider_cache_requests_total[15m])) by (provider)`
2. Check for recent cache invalidations or config changes.
3. Inspect cache node health (evictions, memory pressure).

### Mitigations
* Roll back recent cache configuration changes.
* Increase cache TTLs for frequently requested keys.
* Scale cache tier or enable read-through fallback.

### Escalation
* If cache hit rate < 70% for 30m: page on-call SRE.
* If throughput is degraded: notify Provider Integrations lead.
