# Release Notes: CCAR Market Risk Feed System (v4.2.0)
## Release Date: May 2026

Target Audience: Market Risk Analysts, Regulatory Compliance Officers, and Risk Infrastructure Developers

## Executive Summary
Version 4.2.0 introduces real-time ingestion capabilities for Comprehensive Capital Analysis and Review (CCAR) risk data and transitions our core persistence layer from legacy raw SQL execution scripts to an optimized Object-Oriented Database platform (QzTable framework). This update resolves systemic queue timeouts during high-velocity End-of-Day (EOD) data dumps, guaranteeing deterministic sub-second record mapping and automatic upserts for FR Y-14A/Q compliance streams.

## What’s New in This Release?
1. Object-Oriented Data Matrix & QzTable Integration
The database tier has been entirely decoupled from raw relational string execution. Data frames and inbound streaming arrays are now managed via strongly typed data object instances, automating memory lifecycle management and database transactional sessions.

Impact: Eliminates SQL injection vectors and structural query formatting exceptions. Incorporates a unified object context manager (QzContext) that enforces automated entity tracking and thread-safe batch processing.

Action Required: Upstream desk scripts writing directly to database connections must switch to the programmatic object-oriented interface.

2. High-Throughput Bulk Ingestion Performance Boost
By leveraging native structural arrays unpacked directly inside the database engine via the object-relational abstraction layer, we have optimized high-velocity batch data ingestion speeds.

Resolved Issue: Fixed a race condition where consecutive multi-book trade sensitivity matrices caused database deadlocks and pyodbc connection timeout exceptions during peak reporting hours. Ingestion is now handled via an optimized object state-merge engine.

## Developer & API Updates (v4.2.0)
⚠️ Deprecation Notice: The atomic endpoint POST /api/v3/risk-feeds/ccar/single-node is now deprecated and will be permanently retired in v5.0.0. All downstream systems must migrate to the asynchronous bulk-processing gateway detailed below.

New Endpoint: Submit Daily Risk Exposures
To guarantee stability during massive regulatory stress testing calculations, we have introduced a decoupled, asynchronous, bulk-upload endpoint utilizing Pydantic semantic validations and background worker tasks.

Endpoint: POST /api/v4/risk-feeds/ccar/submit

Protocol Requirement: HTTPS with Content-Encoding: gzip enabled for payloads exceeding 10MB.

Mandatory Headers:

X-Reporting-LEI: The 20-character Legal Entity Identifier of the bank subsidiary.
```python
{
  "reporting_date": "2026-05-15",
  "ccar_scenario": "SEVERELY_ADVERSE",
  "base_currency": "USD",
  "risk_nodes": [
    {
      "book_id": "BK_GLOBAL_EQ_LONG_02",
      "asset_class": "EQUITY",
      "var_99_1d": 125000500,
      "greeks": {
        "delta": 0.8421,
        "gamma": 0.0112,
        "vega": -4500.25
      }
    }
  ]
}
```

X-Idempotency-Key: UUIDv4 token to safeguard against duplicate message ingestion on network retries.

Payload Example:
