# CCAR Market Risk Feed Failure: Incident Runbook & Troubleshooting Guide

When a Comprehensive Capital Analysis and Review (CCAR) Market Risk feed fails, it directly impacts capital adequacy calculations, Stress Testing (9Q forward-looking projections), and regulatory compliance reporting. This runbook provides a structured, tier-based approach to diagnose, isolate, and remediate market risk data feed breaks using Python and SQL.

1. Triage & Impact Assessment (First 15 Minutes)
Before jumping into the code, quickly assess the blast radius to determine if it is a Data Quality (DQ) anomaly or a Hard Upstream Ingestion Failure.

Immediate Questions to Answer:
Asset Classes Impacted: Is the failure isolated to specific desks (e.g., FX, Equities, Fixed Income) or universal?

Risk Metrics Missing: Are we missing standard sensitivities (Greeks), Value-at-Risk (VaR) vectors, or Stress Scenario PnL?

Upstream Status: Did the upstream EOD (End of Day) trading systems sign off?

2. SQL Diagnostics: Isolating the Break
Use these SQL scripts on your staging/ingestion database to identify where the data pipe ruptured.

Step A: Identify Missing or Empty Feeds
Check if files/feeds were recorded in the metadata orchestration layer but contain zero records or failed to load.

SELECT 
    feed_name,
    feed_type,
    business_date,
    expected_arrival_time,
    actual_arrival_time,
    record_count,
    status
FROM ccar_market_risk.feed_log
WHERE business_date = CAST(GETDATE() AS DATE) -- Or your specific batch date
  AND (status = 'FAILED' OR record_count = 0);

  Step B: Locate the Cutoff/Stale DataIf the feed status says "SUCCESS" but down-stream models are throwing errors, you likely have stale/duplicate data from the previous business day ($T-1$).

  SELECT 
    asset_class,
    risk_factor_type,
    COUNT(DISTINCT risk_factor_id) as unique_factors,
    MIN(last_updated_ts) as oldest_record,
    MAX(last_updated_ts) as newest_record
FROM ccar_market_risk.market_sensitivities
WHERE business_date = '2026-05-15' -- Target CCAR reporting date
GROUP BY asset_class, risk_factor_type;

Troubleshooting Note: If newest_record shows yesterday's timestamp, the ETL completed but didn't actually process new $T$ data.3. Python Automation: Data Reconciliation & Volatility SpikesWhen the feed does arrive but fails validation gates, it’s usually due to data corruption, schema drift, or extreme value anomalies (e.g., bad yield curve shifts).Use this Python script to ingest the raw feed file, check for critical data gaps, and flag abnormal variance compared to a historical baseline.

import pandas as pd
import numpy as np
import pyodbc # or sqlalchemy

def troubleshoot_ccar_feed(file_path, conn_str):
    print(f"[*] Initializing Analysis for Feed: {file_path}")
    
    # 1. Load Inbound Feed
    try:
        df_current = pd.read_csv(file_path)
    except Exception as e:
        return f"[CRITICAL] Failed to parse feed file. Possible corruption or Schema Drift: {str(e)}"
        
    # 2. Schema & Nullity Gate
    critical_cols = ['position_id', 'asset_class', 'risk_metric', 'metric_value']
    missing_cols = [col for col in critical_cols if col not in df_current.columns]
    if missing_cols:
        return f"[CRITICAL] Schema Drift Detected! Missing columns: {missing_cols}"
        
    null_counts = df_current[critical_cols].isnull().sum()
    if null_counts.sum() > 0:
        print(f"[WARNING] Null values found in critical fields:\n{null_counts[null_counts > 0]}")
    
    # 3. Fetch Historical Baseline from DB for Outlier Detection (Z-Score)
    query = """
        SELECT position_id, risk_metric, AVG(metric_value) as avg_val, STDEV(metric_value) as std_val 
        FROM ccar_market_risk.historical_sensitivities
        WHERE business_date >= DATEADD(day, -30, GETDATE())
        GROUP BY position_id, risk_metric
    """
    
    conn = pyodbc.connect(conn_str)
    df_history = pd.read_sql(query, conn)
    conn.close()
    
    # 4. Merge and Flag Anomalies
    merged = pd.merge(df_current, df_history, on=['position_id', 'risk_metric'], how='left')
    
    # Avoid division by zero for fixed/low-risk metrics
    merged['std_val'] = merged['std_val'].replace(0, np.nan) 
    merged['z_score'] = (merged['metric_value'] - merged['avg_val']) / merged['std_val']
    
    # Threshold: Flag anything outside 4 Standard Deviations for CCAR investigation
    anomalies = merged[merged['z_score'].abs() > 4.0]
    
    if not anomalies.empty:
        print(f"\n[ALERT] Found {len(anomalies)} risk metric anomalies exceeding Z-Score threshold (>4 std dev):")
        print(anomalies[['position_id', 'asset_class', 'risk_metric', 'metric_value', 'z_score']].head(10))
        return anomalies
        
    print("[SUCCESS] Data reconciliation and anomaly checks passed.")
    return None

# Example Execution:
# anomalies_df = troubleshoot_ccar_feed('T_market_risk_feed.csv', 'DRIVER={SQL Server};SERVER=risk_db;DATABASE=ccar;UID=usr;PWD=pwd')

4. Root Cause Playbook (Common Scenarios)

| Rank | THING-TO-RANK |
|-----:|---------------|

CCAR Market Risk Feed Failure: Incident Runbook & Troubleshooting Guide
When a Comprehensive Capital Analysis and Review (CCAR) Market Risk feed fails, it directly impacts capital adequacy calculations, Stress Testing (9Q forward-looking projections), and regulatory compliance reporting. This runbook provides a structured, tier-based approach to diagnose, isolate, and remediate market risk data feed breaks using Python and SQL.

1. Triage & Impact Assessment (First 15 Minutes)
Before jumping into the code, quickly assess the blast radius to determine if it is a Data Quality (DQ) anomaly or a Hard Upstream Ingestion Failure.

Immediate Questions to Answer:
Asset Classes Impacted: Is the failure isolated to specific desks (e.g., FX, Equities, Fixed Income) or universal?

Risk Metrics Missing: Are we missing standard sensitivities (Greeks), Value-at-Risk (VaR) vectors, or Stress Scenario PnL?

Upstream Status: Did the upstream EOD (End of Day) trading systems sign off?

2. SQL Diagnostics: Isolating the Break
Use these SQL scripts on your staging/ingestion database to identify where the data pipe ruptured.

Step A: Identify Missing or Empty Feeds
Check if files/feeds were recorded in the metadata orchestration layer but contain zero records or failed to load.

SQL
SELECT 
    feed_name,
    feed_type,
    business_date,
    expected_arrival_time,
    actual_arrival_time,
    record_count,
    status
FROM ccar_market_risk.feed_log
WHERE business_date = CAST(GETDATE() AS DATE) -- Or your specific batch date
  AND (status = 'FAILED' OR record_count = 0);
Step B: Locate the Cutoff/Stale Data
If the feed status says "SUCCESS" but down-stream models are throwing errors, you likely have stale/duplicate data from the previous business day (T−1).

SQL
SELECT 
    asset_class,
    risk_factor_type,
    COUNT(DISTINCT risk_factor_id) as unique_factors,
    MIN(last_updated_ts) as oldest_record,
    MAX(last_updated_ts) as newest_record
FROM ccar_market_risk.market_sensitivities
WHERE business_date = '2026-05-15' -- Target CCAR reporting date
GROUP BY asset_class, risk_factor_type;
Troubleshooting Note: If newest_record shows yesterday's timestamp, the ETL completed but didn't actually process new T data.

3. Python Automation: Data Reconciliation & Volatility Spikes
When the feed does arrive but fails validation gates, it’s usually due to data corruption, schema drift, or extreme value anomalies (e.g., bad yield curve shifts).

Use this Python script to ingest the raw feed file, check for critical data gaps, and flag abnormal variance compared to a historical baseline.

Python
import pandas as pd
import numpy as np
import pyodbc # or sqlalchemy

def troubleshoot_ccar_feed(file_path, conn_str):
    print(f"[*] Initializing Analysis for Feed: {file_path}")
    
    # 1. Load Inbound Feed
    try:
        df_current = pd.read_csv(file_path)
    except Exception as e:
        return f"[CRITICAL] Failed to parse feed file. Possible corruption or Schema Drift: {str(e)}"
        
    # 2. Schema & Nullity Gate
    critical_cols = ['position_id', 'asset_class', 'risk_metric', 'metric_value']
    missing_cols = [col for col in critical_cols if col not in df_current.columns]
    if missing_cols:
        return f"[CRITICAL] Schema Drift Detected! Missing columns: {missing_cols}"
        
    null_counts = df_current[critical_cols].isnull().sum()
    if null_counts.sum() > 0:
        print(f"[WARNING] Null values found in critical fields:\n{null_counts[null_counts > 0]}")
    
    # 3. Fetch Historical Baseline from DB for Outlier Detection (Z-Score)
    query = """
        SELECT position_id, risk_metric, AVG(metric_value) as avg_val, STDEV(metric_value) as std_val 
        FROM ccar_market_risk.historical_sensitivities
        WHERE business_date >= DATEADD(day, -30, GETDATE())
        GROUP BY position_id, risk_metric
    """
    
    conn = pyodbc.connect(conn_str)
    df_history = pd.read_sql(query, conn)
    conn.close()
    
    # 4. Merge and Flag Anomalies
    merged = pd.merge(df_current, df_history, on=['position_id', 'risk_metric'], how='left')
    
    # Avoid division by zero for fixed/low-risk metrics
    merged['std_val'] = merged['std_val'].replace(0, np.nan) 
    merged['z_score'] = (merged['metric_value'] - merged['avg_val']) / merged['std_val']
    
    # Threshold: Flag anything outside 4 Standard Deviations for CCAR investigation
    anomalies = merged[merged['z_score'].abs() > 4.0]
    
    if not anomalies.empty:
        print(f"\n[ALERT] Found {len(anomalies)} risk metric anomalies exceeding Z-Score threshold (>4 std dev):")
        print(anomalies[['position_id', 'asset_class', 'risk_metric', 'metric_value', 'z_score']].head(10))
        return anomalies
        
    print("[SUCCESS] Data reconciliation and anomaly checks passed.")
    return None

# Example Execution:
# anomalies_df = troubleshoot_ccar_feed('T_market_risk_feed.csv', 'DRIVER={SQL Server};SERVER=risk_db;DATABASE=ccar;UID=usr;PWD=pwd')
4. Root Cause Playbook (Common Scenarios)

| Symptoms|	Likely Root Cause	| Remediation Action|
|------------:|-----------------------------|------------------------------|
|SQL query returns 0 records for today's date; source file is missing.	| Upstream EOD Batch Delay	| Check upstream batch schedulers (e.g., Airflow, Autosys). Ping the respective asset class Middle Office IT desk to confirm if trade matching or curve generation is running late. |
| Python script outputs Schema Drift Detected.	 | Upstream System Release |	A column was renamed or added by an upstream system without updating the downstream CCAR data contract.  Map the missing fields manually in the staging ETL view to unblock the feed. |
| Z-score alerts trigger massively across Interest Rate Greeks.	| Bad Curve Ingestion / Missing Staging Data	| Usually happens when a benchmark curve (like SOFR) fails to construct properly, resulting in 0 or Null discount factors, blowing up sensitivities. Roll back to yesterday’s curve as a proxy if approved by Risk Management. |
| pyodbc database connection timeout errors.	| Database Lock / High Concurrency	| CCAR calculation windows often experience heavy database load. Run sp_who2 or checking active block locks in SQL to kill orphaned sessions. |

5. Fail-Safe Resolution ProtocolIf the feed cannot be programmatically restored within the regulatory reporting SLA window, initiate the CCAR Business Continuity Protocol:Proxy Ingestion: If approved by the Market Risk Methodology team, run an emergency SQL script to clone the previous business day's risk factors ($T-1$) to serve as a proxy for the missing assets, scaling by any macro indices if required:

 INSERT INTO ccar_market_risk.market_sensitivities (business_date, position_id, asset_class, risk_metric, metric_value)
SELECT '2026-05-16', position_id, asset_class, risk_metric, metric_value
FROM ccar_market_risk.market_sensitivities
WHERE business_date = '2026-05-15';

Audit Logging: Record the incident in the CCAR Data Quality Ledger, explicitly noting the percentage of total Risk-Weighted Assets (RWA) impacted by the proxy data to satisfy Federal Reserve SR 15-18 / SR 11-7 validation requirements.

