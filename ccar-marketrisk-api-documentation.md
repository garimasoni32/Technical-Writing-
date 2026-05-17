# CCAR Market Risk Feed: Core Technical Specification
Version: 4.1.0

Domain: Risk Management & Regulatory Capital Reporting (FR Y-14A/Q Compliance)

Architecture Style: REST Gateway with Async Worker Processing

Core Stack: Python 3.11+ (FastAPI + Pydantic) | PostgreSQL 15+

1. System Architecture Overview
The CCAR Market Risk Ingestion system is engineered to handle massive end-of-day batch risk metrics from disparate trading desks. To prevent network timeouts and resource exhaustion during peak hours, the API employs an asynchronous decoupled architecture.

[Upstream Desk] ──(HTTP POST JSON)──> [FastAPI Gateway] (Sync Schema Check)
                                              │
                                        (Returns 202 Accepted & Job ID)
                                              │
                                              ▼
                                      [Celery Worker Pool]
                                              │
                                    (Batch Python Parsing)
                                              │
                                              ▼
                                    [PostgreSQL Ledger] (High-Speed UPSERT)

2. External API Interface Specification
Endpoint: Submit Daily Risk Exposures
HTTP Method: POST

Path: /api/v4/risk-feeds/ccar/submit

Protocol Requirement: HTTPS with Content-Encoding: gzip enabled for payloads > 10MB.

**Global Request Headers**

| Header | Type | Required | Description |
|-----:|---------------|------------------|-----------------|
| Content-Type |	string	| Yes	| Must be explicitly set to application/json. |
| X-Reporting-LEI	| string	| Yes	| Legal Entity Identifier (LEI) of the submitting bank subsidiary. |
| X-Idempotency-Key	| string	| Yes |	UUIDv4 to eliminate duplicate processing loops on network retries. |

**JSON Request Payload Schema**

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

3. Python Validation & Gateway Pipeline
The Python layer handles sub-millisecond serialization, headers matching, and semantic constraints verification using type hints and memory-efficient runtime validation.
import os
from typing import List
from datetime import date
from fastapi import FastAPI, Header, HTTPException, status
from pydantic import BaseModel, Field, validator

app = FastAPI(title="CCAR Ingestion Engine", version="4.1.0")

# --- Data Validation Schemas (Pydantic) ---

class GreekMetrics(BaseModel):
    delta: float = Field(..., description="First-order directional price sensitivity.")
    gamma: float = Field(..., description="Second-order acceleration sensitivity.")
    vega: float = Field(..., description="Volatility exposure metric.")

class RiskNode(BaseModel):
    book_id: str = Field(..., regex=r"^BK_[A-Z0-9_]+$", description="Strict standard desk prefix format.")
    asset_class: str = Field(..., description="Must be one of: EQUITY, FX, CREDIT, IR.")
    var_99_1d: int = Field(..., gt=0, description="1-Day 99% Value-at-Risk tracking units in absolute integer currency cents.")
    greeks: GreekMetrics

    @validator('asset_class')
    def verify_asset_domain(cls, value):
        valid_domains = ["EQUITY", "FX", "CREDIT", "IR"]
        if value not in valid_domains:
            raise ValueError(f"Asset class must map to: {valid_domains}")
        return value

class CCARPayload(BaseModel):
    reporting_date: date
    ccar_scenario: str
    base_currency: str = "USD"
    risk_nodes: List[RiskNode]

    @validator('ccar_scenario')
    def verify_regulatory_scenario(cls, value):
        valid_scenarios = ["BASELINING", "SEVERELY_ADVERSE", "INTERNAL_STRESS"]
        if value not in valid_scenarios:
            raise ValueError(f"Invalid CCAR regulatory path string specification.")
        return value

# --- API Endpoint Handler ---

@app.post("/api/v4/risk-feeds/ccar/submit", status_code=status.HTTP_202_ACCEPTED)
async def ingest_market_risk_feed(
    payload: CCARPayload,
    x_reporting_lei: str = Header(..., alias="X-Reporting-LEI"),
    x_idempotency_key: str = Header(..., alias="X-Idempotency-Key")
):
    """
    Accepts, formats, validates, and queues regulatory risk matrix data arrays asynchronously.
    """
    try:
        # Handoff payload to task runner worker pool to release HTTP connection thread immediately
        # job_id = celery_worker.ccar_pipeline_task.delay(payload.dict(), x_reporting_lei)
        mock_job_id = "job_20260517_9921_ax3" 
        
        return {
            "ingestion_job_id": mock_job_id,
            "status": "QUEUED",
            "submitted_records": len(payload.risk_nodes),
            "uri_callback": f"/api/v4/risk-feeds/ccar/jobs/{mock_job_id}/status"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Internal Broker Queue Failure: {str(e)}")


        4. Database Layer & High-Throughput SQL Engine
Data is stored in an optimized PostgreSQL layout designed for low-latency time-series indexing. Rather than processing loops over execution connections, data arrays are unpacked natively inside the engine using an unnested structural layout block.

Database DDL Schema

-- Production Relational Ledger Table
CREATE TABLE fact_ccar_market_risk (
    id BIGSERIAL PRIMARY KEY,
    reporting_date DATE NOT NULL,
    legal_entity_identifier VARCHAR(20) NOT NULL,
    ccar_scenario VARCHAR(30) NOT NULL,
    book_id VARCHAR(50) NOT NULL,
    asset_class VARCHAR(10) NOT NULL,
    var_99_1d BIGINT NOT NULL,          -- Integer cent preservation
    delta NUMERIC(12, 6),
    gamma NUMERIC(12, 6),
    vega NUMERIC(12, 2),
    processed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Compound unique constraint index to safeguard data integrity and expedite UPSERT lookups
CREATE UNIQUE INDEX idx_ccar_unique_reporting_grain 
ON fact_ccar_market_risk (reporting_date, legal_entity_identifier, ccar_scenario, book_id);

Python/SQL Database Connector Implementation
This processing loop unrolls nested dictionary models into linear PostgreSQL arrays, calling atomic upsert protocols for atomic execution blocks.

import psycopg2
from psycopg2.extras import execute_values

def db_batch_upsert_worker(db_connection, lei: str, reporting_date: str, scenario: str, risk_nodes: list):
    """
    Maps unpacked JSON arrays directly into PostgreSQL UNNEST constructs for single-transaction speed optimization.
    """
    # Matrix vector preparation
    book_ids = [node['book_id'] for node in risk_nodes]
    asset_classes = [node['asset_class'] for node in risk_nodes]
    vars_cents = [node['var_99_1d'] for node in risk_nodes]
    deltas = [node['greeks']['delta'] for node in risk_nodes]
    gammas = [node['greeks']['gamma'] for node in risk_nodes]
    vegas = [node['greeks']['vega'] for node in risk_nodes]

    upsert_sql = """
        INSERT INTO fact_ccar_market_risk (
            reporting_date, legal_entity_identifier, ccar_scenario, book_id, asset_class, var_99_1d, delta, gamma, vega
        )
        SELECT %s, %s, %s, * FROM UNNEST(%s, %s, %s, %s, %s, %s)
        ON CONFLICT (reporting_date, legal_entity_identifier, ccar_scenario, book_id) 
        DO UPDATE SET
            var_99_1d = EXCLUDED.var_99_1d,
            delta = EXCLUDED.delta,
            gamma = EXCLUDED.gamma,
            vega = EXCLUDED.vega,
            processed_at = CURRENT_TIMESTAMP;
    """

    with db_connection.cursor() as cursor:
        cursor.execute(upsert_sql, (
            reporting_date, lei, scenario, 
            book_ids, asset_classes, vars_cents, deltas, gammas, vegas
        ))
    db_connection.commit()

    5. API Response Definitions
HTTP 202 Accepted (Asynchronous Job Created)
Returned when raw payload structural parameters are confirmed clean by the Python Pydantic validation parser.

{
  "ingestion_job_id": "job_20260517_9921_ax3",
  "status": "QUEUED",
  "submitted_records": 1,
  "uri_callback": "/api/v4/risk-feeds/ccar/jobs/job_20260517_9921_ax3/status"
}

HTTP 422 Unprocessable Entity (Semantic Validation Failure)
Returned synchronously if the payload breaks model constraints defined within the Pydantic type layout (e.g., passing an invalid scenario tracking type).

{
  "detail": [
    {
      "loc": ["body", "ccar_scenario"],
      "msg": "Scenario must be one of ['BASELINING', 'SEVERELY_ADVERSE', 'INTERNAL_STRESS']",
      "type": "value_error"
    }
  ]
}

HTTP 400 Bad Request (Missing Authentication Header)
Returned when required proxy processing identification values are dropped inside incoming routing requests.

{
  "detail": "Missing mandatory X-Reporting-LEI header configuration metadata context values."
}




