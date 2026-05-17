# CCAR Market Risk Feed: Core Technical Specification
Version: 4.2.0

Domain: Risk Management & Regulatory Capital Reporting (FR Y-14A/Q Compliance)
Architecture Style: REST Gateway with Async Worker Processing
Core Stack: Python 3.11+ (FastAPI + Pydantic) | Enterprise Object-Relational Layer (QzTable Pattern)

1. System Architecture Overview
The CCAR Market Risk Ingestion system utilizes an asynchronous decoupled architecture. The database interaction bypasses raw relational string writing, relying instead on an Object-Relational Data Context that abstracts transaction persistence and row mappings into unified runtime instances.

[Upstream Desk] ──(HTTP POST JSON)──> [FastAPI Gateway] (Sync Pydantic Schema Check)
                                             │
                                       (Returns 202 Accepted & Job ID)
                                             │
                                             ▼
                                     [Celery Worker Pool]
                                             │
                                    (Unpacks Entity Objects)
                                             │
                                             ▼
                                    [Object Data Matrix] ──> [QzTable Persistence Engine]

2. External API Interface Specification
Endpoint: Submit Daily Risk Exposures
HTTP Method: POST

Path: /api/v4/risk-feeds/ccar/submit

Protocol Requirement: HTTPS with Content-Encoding: gzip enabled for payloads > 10MB.

Global Request Headers

| Header |	Type	| Required |	Description |
|--------:|------------------|-----------------------|-----------------------|
|Content-Type |	string |	Yes | 	Must be explicitly set to application/json. |
| X-Reporting-LEI |	string	| Yes	| Legal Entity Identifier (LEI) of the submitting bank subsidiary. |
|X-Idempotency-Key|	string |	Yes	| UUIDv4 to eliminate duplicate processing loops on network retries. |

JSON Request Payload Schema

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
3. Python Validation & Gateway Pipeline
The validation layer utilizes Pydantic data schemas to enforce structural type constraints on intake before mapping data downstream into database model entities.
```python
from typing import List
from datetime import date
from fastapi import FastAPI, Header, HTTPException, status
from pydantic import BaseModel, Field, field_validator

app = FastAPI(title="CCAR Ingestion Engine", version="4.2.0")

# --- Inbound API Request Validation Schemas ---

class GreekMetrics(BaseModel):
    delta: float = Field(..., description="First-order directional price sensitivity.")
    gamma: float = Field(..., description="Second-order acceleration sensitivity.")
    vega: float = Field(..., description="Volatility exposure metric.")

class RiskNode(BaseModel):
    book_id: str = Field(..., json_schema_extra={"pattern": r"^BK_[A-Z0-9_]+$"})
    asset_class: str = Field(..., description="Must be one of: EQUITY, FX, CREDIT, IR.")
    var_99_1d: int = Field(..., gt=0, description="1-Day 99% VaR tracking units in absolute integer currency cents.")
    greeks: GreekMetrics

    @field_validator('asset_class')
    @classmethod
    def verify_asset_domain(cls, value: str) -> str:
        valid_domains = ["EQUITY", "FX", "CREDIT", "IR"]
        if value not in valid_domains:
            raise ValueError(f"Asset class must map to: {valid_domains}")
        return value

class CCARPayload(BaseModel):
    reporting_date: date
    ccar_scenario: str
    base_currency: str = "USD"
    risk_nodes: List[RiskNode]

    @field_validator('ccar_scenario')
    @classmethod
    def verify_regulatory_scenario(cls, value: str) -> str:
        valid_scenarios = ["BASELINING", "SEVERELY_ADVERSE", "INTERNAL_STRESS"]
        if value not in valid_scenarios:
            raise ValueError("Invalid CCAR regulatory path string specification.")
        return value
 ```

API Endpoint Handler
```
@app.post("/api/v4/risk-feeds/ccar/submit", status_code=status.HTTP_202_ACCEPTED)
async def ingest_market_risk_feed(
    payload: CCARPayload,
    x_reporting_lei: str = Header(..., alias="X-Reporting-LEI"),
    x_idempotency_key: str = Header(..., alias="X-Idempotency-Key")
):
    """
    Accepts, formats, and queues regulatory risk matrix data arrays asynchronously.
    """
    try:
        # Serialized dictionary data is handed off to the background task worker pool
        mock_job_id = "job_20260517_9921_ax3" 
        
        return {
            "ingestion_job_id": mock_job_id,
            "status": "QUEUED",
            "submitted_records": len(payload.risk_nodes),
            "uri_callback": f"/api/v4/risk-feeds/ccar/jobs/{mock_job_id}/status"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Internal Broker Queue Failure: {str(e)}")
```

4. Object-Oriented Database Layer & QzTable Engine
Rather than executing raw text strings or tracking connections over manual transactional SQL units, data objects inherit from an active-record declarative model container (QzTable).
```python
Object Schema Definition

from datetime import datetime
from decimal import Decimal
from typing import Optional
from database_provider import QzTable, QzContext, Column, Integer, String, Date, Numeric, DateTime

class FactCCARMarketRisk(QzTable):
    """
    Object-Relational mapping representing the fact_ccar_market_risk storage layer.
    """
    __tablename__ = "fact_ccar_market_risk"
    __table_args__ = (
        # Natural unique grain index pattern for rapid upsert resolution
        {"unique_constraints": [["reporting_date", "legal_entity_identifier", "ccar_scenario", "book_id"]]},
    )

    # Property mappings abstracting individual column attributes
    id: Optional[int] = Column(Integer, primary_key=True, autoincrement=True)
    reporting_date: date = Column(Date, nullable=False)
    legal_entity_identifier: str = Column(String(20), nullable=False)
    ccar_scenario: str = Column(String(30), nullable=False)
    book_id: str = Column(String(50), nullable=False)
    asset_class: str = Column(String(10), nullable=False)
    var_99_1d: int = Column(Integer, nullable=False)  # Integer tracking preserving currency cents
    delta: Optional[Decimal] = Column(Numeric(12, 6))
    gamma: Optional[Decimal] = Column(Numeric(12, 6))
    vega: Optional[Decimal] = Column(Numeric(12, 2))
    processed_at: datetime = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

High-Throughput Object Batch Upsert Implementation
This execution engine handles the transaction implicitly through a session context (QzContext). Instead of unrolling multi-line INSERT ... ON CONFLICT scripts, it calls an explicit state-merge routine natively supported by the object framework layer.
```python
def db_batch_upsert_worker(context: QzContext, lei: str, reporting_date: date, scenario: str, risk_nodes: list):
    """
    Transforms unpacked JSON records into strongly typed FactCCARMarketRisk data object
    instances and pushes them to the DB using native object batch state merging.
    """
    records_to_upsert = []

    for node in risk_nodes:
        # Creating database object instances from the risk payload structures
        risk_record = FactCCARMarketRisk(
            reporting_date=reporting_date,
            legal_entity_identifier=lei,
            ccar_scenario=scenario,
            book_id=node['book_id'],
            asset_class=node['asset_class'],
            var_99_1d=node['var_99_1d'],
            delta=Decimal(str(node['greeks']['delta'])),
            gamma=Decimal(str(node['greeks']['gamma'])),
            vega=Decimal(str(node['greeks']['vega']))
        )
        records_to_upsert.append(risk_record)

    # Open transaction context block to isolate bulk mutations
    with context.begin_transaction():
        # Executes high-velocity bulk save/upsert matching on the table's natural unique constraints
        FactCCARMarketRisk.bulk_upsert(
            session=context,
            records=records_to_upsert,
            conflict_target=['reporting_date', 'legal_entity_identifier', 'ccar_scenario', 'book_id'],
            update_fields=['var_99_1d', 'delta', 'gamma', 'vega', 'processed_at']
        )
```

Analytical Object Queries Example
This displays how downstream analytical calculators extract the structured records using Object-Oriented model properties instead of executing native SQL string select lookups.
``` python
def fetch_high_exposure_books(context: QzContext, target_date: date, scenario: str) -> list[FactCCARMarketRisk]:
    """
    Extracts high-exposure positions exceeding 1 million USD (100,000,000 cents) using Object Queries.
    """
    # Programmatic fluent method querying interface
    query = (
        FactCCARMarketRisk.query(context)
        .filter(FactCCARMarketRisk.reporting_date == target_date)
        .filter(FactCCARMarketRisk.ccar_scenario == scenario)
        .filter(FactCCARMarketRisk.var_99_1d > 100000000)
        .order_by(FactCCARMarketRisk.var_99_1d.desc())
    )
    
    # Executes query and returns collection arrays populated with instantiated table record models
    return query.all()
```

5. API Response Definitions
HTTP 202 Accepted (Asynchronous Job Created)
Returned when raw payload structural parameters are confirmed clean by the Pydantic parser.
```python
{
  "ingestion_job_id": "job_20260517_9921_ax3",
  "status": "QUEUED",
  "submitted_records": 1,
  "uri_callback": "/api/v4/risk-feeds/ccar/jobs/job_20260517_9921_ax3/status"
}
```
HTTP 422 Unprocessable Entity (Semantic Validation Failure)
Returned synchronously if the payload breaks structural constraints defined within the Pydantic layer models.
```python
{
  "detail": [
    {
      "loc": ["body", "ccar_scenario"],
      "msg": "Scenario must be one of ['BASELINING', 'SEVERELY_ADVERSE', 'INTERNAL_STRESS']",
      "type": "value_error"
    }
  ]
}
```
