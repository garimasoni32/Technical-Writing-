# CCAR Market Risk Feed Failure: Incident Runbook & Troubleshooting Guide
When a Comprehensive Capital Analysis and Review (CCAR) Market Risk feed fails, it directly impacts capital adequacy calculations, Stress Testing (9Q forward-looking projections), and regulatory compliance reporting (FR Y-14A/Q). This runbook provides a structured framework to diagnose, isolate, and remediate market risk data feed breaks using an Object-Oriented Data Layer and Python.

1. Triage & Impact Assessment (First 15 Minutes)
Before jumping into the data layer, quickly assess the blast radius to determine if it is a Data Quality (DQ) anomaly or a Hard Upstream Ingestion Failure.

Immediate Questions to Answer:
Asset Classes Impacted: Is the failure isolated to specific desks (e.g., FX, Equities, Fixed Income) or universal?

Risk Metrics Missing: Are we missing standard sensitivities (Greeks), Value-at-Risk (VaR) vectors, or Stress Scenario PnL?

Upstream Status: Did the upstream EOD (End of Day) trading systems sign off?

2. Object Schema Definition
Downstream diagnostic utilities rely on two primary object-relational mappings representing the orchestration tracking layer (FeedLog) and the core risk metrics repository (MarketSensitivities).
```python
from datetime import datetime, date
from decimal import Decimal
from typing import Optional
from database_provider import QzTable, Column, Integer, String, Date, Numeric, DateTime

class FeedLog(QzTable):
    """Object mapping representing the system data-ingestion orchestration ledger."""
    __tablename__ = "feed_log"

    id: Optional[int] = Column(Integer, primary_key=True, autoincrement=True)
    feed_name: str = Column(String(100), nullable=False)
    feed_type: str = Column(String(50), nullable=False)
    business_date: date = Column(Date, nullable=False)
    expected_arrival_time: datetime = Column(DateTime)
    actual_arrival_time: Optional[datetime] = Column(DateTime)
    record_count: int = Column(Integer, default=0)
    status: str = Column(String(20), default="PENDING")


class MarketSensitivities(QzTable):
    """Object mapping representing actual ingested risk exposure fields."""
    __tablename__ = "market_sensitivities"
    __table_args__ = (
        {"unique_constraints": [["business_date", "position_id", "risk_metric"]]},
    )

    id: Optional[int] = Column(Integer, primary_key=True, autoincrement=True)
    business_date: date = Column(Date, nullable=False)
    position_id: str = Column(String(50), nullable=False)
    asset_class: str = Column(String(20), nullable=False)
    risk_factor_type: str = Column(String(50), nullable=False)
    risk_metric: str = Column(String(50), nullable=False)
    metric_value: Decimal = Column(Numeric(18, 6), nullable=False)
    last_updated_ts: datetime = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

```

3. Programmatic Diagnostics: Isolating the Break
Use these object-oriented query blocks within your analysis console to pinpoint where the pipeline ruptured.

Step A: Identify Missing or Empty Feeds
Query the FeedLog object collection to isolate broken or un-arrived system entities.
```python
def check_active_feed_status(context, target_date: date):
    """Queries the data ledger for missing or failed ingestion components."""
    failed_feeds = (
        FeedLog.query(context)
        .filter(FeedLog.business_date == target_date)
        .filter((FeedLog.status == "FAILED") | (FeedLog.record_count == 0))
        .all()
    )
    
    for feed in failed_feeds:
        print(f"[CRITICAL] Feed Failure: {feed.feed_name} | "
              f"Expected: {feed.expected_arrival_time} | Count: {feed.record_count}")
    return failed_feeds
```

Step B: Detect Stale ($T-1$) Data BuffersIf downstream models throw calculation alerts despite an ingestion status of SUCCESS, verify that the active memory layer didn't accidentally consume yesterday's run attributes.
```python
def audit_data_staleness(context, reporting_date: date):
    """Audits record update bounds via model entity properties."""
    summary_metrics = (
        MarketSensitivities.query(context)
        .filter(MarketSensitivities.business_date == reporting_date)
        .all()
    )
    
    # Analyze the record objects programmatically 
    for record in summary_metrics[:5]:
        if record.last_updated_ts.date() < reporting_date:
            print(f"[WARNING] Stale Row Target Detached! "
                  f"Position: {record.position_id} contains T-1 timestamp: {record.last_updated_ts}")
```

4. Automation: Anomaly Verification & Outlier Isolation
   
When files load successfully but breach systemic limits, it usually signifies profile data corruption or extreme value anomalies (e.g., severe curve movements). This class-based routine maps raw input structures into entities, calculates historical variations, and isolates outliers.
```python
import pandas as pd
import numpy as np

class CCARFeedAnalyzer:
    def __init__(self, context):
        self.context = context

    def process_and_validate_feed(self, file_path: str) -> Optional[pd.DataFrame]:
        print(f"[*] Initializing Object Analytics for Feed: {file_path}")
        
        # 1. Ingest Raw Transport Vector
        try:
            df_current = pd.read_csv(file_path)
        except Exception as e:
            print(f"[CRITICAL] Structural File Parse Exception: {str(e)}")
            return None
            
        # 2. Schema Guardrails
        critical_fields = ['position_id', 'asset_class', 'risk_metric', 'metric_value']
        if not all(col in df_current.columns for col in critical_fields):
            print("[CRITICAL] Schema Drift Detected! File mapping definitions mismatched.")
            return None

        # 3. Object-Oriented Baseline Fetching
        # Pull 30-day history as list of objects, convert directly to analytical reference framing
        historical_records = (
            self.context.query(MarketSensitivities)
            .filter(MarketSensitivities.last_updated_ts >= datetime.utcnow() - pd.Timedelta(days=30))
            .all()
        )
        
        if not historical_records:
            print("[WARNING] Zero historical data points extracted from Object Context.")
            return None
            
        df_history = pd.DataFrame([{
            'position_id': r.position_id,
            'risk_metric': r.risk_metric,
            'metric_value': float(r.metric_value)
        } for r in historical_records])

        # 4. Statistical Distribution Aggregations
        df_stats = df_history.groupby(['position_id', 'risk_metric'])['metric_value'].agg(['mean', 'std']).reset_index()
        df_stats.columns = ['position_id', 'risk_metric', 'avg_val', 'std_val']

        # 5. Volatility Profiling (Z-Score Execution)
        merged = pd.merge(df_current, df_stats, on=['position_id', 'risk_metric'], how='left')
        merged['std_val'] = merged['std_val'].replace(0, np.nan)
        merged['z_score'] = (merged['metric_value'] - merged['avg_val']) / merged['std_val']

        # Isolate entries exceeding 4 Standard Deviations
        anomalies = merged[merged['z_score'].abs() > 4.0]
        if not anomalies.empty:
            print(f"\n[ALERT] Found {len(anomalies)} risk anomalies breaching Z-Score Limits (>4.0):")
            return anomalies[['position_id', 'asset_class', 'risk_metric', 'metric_value', 'z_score']]

        print("[SUCCESS] CCAR target data objects passed distribution checks.")
        return None
```

5. Root Cause Playbook (Common Scenarios)

| Symptoms	| Likely Root Cause |	Object-Layer Remediation Action |
|-------------:|------------------------------|--------------------------------|
|Diagnostics return 0 entries for today's context date; source file cannot be located. |	Upstream EOD Batch Delay	| Query upstream task monitors. Coordinate with middle-office operations to verify structural curve compilation states.|
|Verification pipeline outputs Schema Drift Detected.|	Upstream System Release	A source model attribute was added or altered without altering the data contract.| Adjust your QzTable column configuration mappings to map around the change.
|Z-score alerts trigger across Interest Rate Greeks.	|Bad Curve Ingestion / Missing Staging Data	| Zero-value discount factors due to broken benchmark parameters (e.g., SOFR). Cascade and clone yesterday's curve instances to restore calculations.|
|Operational engine generates context timeout warnings.	| Database Lock / High Concurrency |	Resource starvation inside the persistence connection layer. Kill competing reporting threads using application session teardowns. |

6. Fail-Safe Resolution Protocol

   If structural file remediation cannot be established within your regulatory SLA reporting window, initiate the CCAR Business Continuity Protocol:1. Object Proxy FallbackIf approved by the Market Risk Methodology board, programmatically duplicate the previous business day's exposure entities ($T-1$) to serve as an emergency data proxy.
  ```python 
   def execute_emergency_proxy_fill(context, missing_date: date, fallback_date: date):
    """Clones market risk entities from fallback to missing date window."""
    historical_snapshot = (
        MarketSensitivities.query(context)
        .filter(MarketSensitivities.business_date == fallback_date)
        .all()
    )
    
    proxy_buffer = []
    for old_record in historical_snapshot:
        proxy_record = MarketSensitivities(
            business_date=missing_date,
            position_id=old_record.position_id,
            asset_class=old_record.asset_class,
            risk_factor_type=old_record.risk_factor_type,
            risk_metric=old_record.risk_metric,
            metric_value=old_record.metric_value
        )
        proxy_buffer.append(proxy_record)
        
    with context.begin_transaction():
        # Atomically insert proxy tracking matrix via the entity context block
        context.bulk_save_objects(proxy_buffer)
        context.commit()
    print(f"[SUCCESS] Ingested {len(proxy_buffer)} proxy records for processing date: {missing_date}")
```

2. Audit Tracking Ledger Signature
   
Log the incident inside the central CCAR Data Quality Ledger. Specify the percentage of total Risk-Weighted Assets (RWA) impacted by the fallback data to maintain compliance with Federal Reserve SR 15-18 and SR 11-7 validation rules.



