## Production Release Plan: Core Platform Update (v4.2.0)
### Execution Date: May 26, 2026

Author: Garima Soni 

Target Window: 02:00 AM - 04:00 AM EST (Low-traffic maintenance window)

Release Coordinator: @DevOps-Lead

1. Inventory of Updated Artifacts & Files
The following code repositories and configuration manifests are included in this release bundle:

| Repository / Component | Impacted Files / Modules | Image/Artifact Tag | 
|-----:|---------------|---------------------|
| inventory-api-service | controllers/v1/bulk_sync.go models/transactions.go middleware/rate_limiter.go | docker.io/upshop/api:v4.2.0 |
| merchant-web-portal | src/components/UploadWizard.jsx src/hooks/useBulkUpload.js | static-build/web:v4.2.0 |
| database-migrations | migrations/ddl/014_add_bulk_sync_indexes.sql | db-migrate:v14 |

2. Execution Runbook (Push Plan to Production)

This sequence must be executed chronologically by the deployment team.

Phase 1: Pre-Deployment Tasks (T-Minus 30 Minutes)
* Lock Master Branches: Restrict further merges to main.

* Trigger Manual DB Backup: Execute AWS RDS snapshot of the production instance.

aws rds create-db-snapshot --db-instance-identifier prod-db-cluster --db-snapshot-identifier prod-backup-before-v4-2-0

* Notify Stakeholders: Broadcast automated maintenance banner to the web portal.

Phase 2: Deployment Execution (02:15 AM EST)

Step 1: Run database migration scripts (014_add_bulk_sync_indexes.sql) to add indexes for the bulk sync service.

Step 2: Deploy the backend API Docker container (api:v4.2.0) to the Kubernetes staging pool, then shift production traffic using a Canary Deployment strategy (10% traffic increment every 5 minutes).

Step 3: Purge Cloudflare CDN edge cache and deploy the frontend assets (web:v4.2.0) to the S3 bucket.

3. Post-Production Testing Plan (Smoke Tests)
QA engineers will verify live system health immediately following the deployment using the following manual and automated scripts.

⚠️ Important: Ensure all smoke tests pass before increasing canary traffic allocation to 100%.

Automated Verification
* Health Check Endpoint: Verify GET [https://api.platform.com/health](https://api.platform.com/health) returns 200 OK and status healthy.
* Postman Collection Run: Trigger the Prod-Regression-v4.2 suite via Newman to validate core workflows.


Manual Verification Matrix

| Test Case ID | Description | Expected Result | Status |
|-----:|---------------|---------------------|----------------------|
| TC-01 | Attempt single scan upload | Payload parses successfully, returns 200 OK. | Pending |
| TC-02 | Submit bulk array with 1 invalid barcode | Gateway handles partial failure, returns 207 Multi-Status. | Pending |
| TC-03 | Rapidly ping endpoint 65 times in 1 minute | Rate limiter triggers, blocks request with 429 Too Many Requests. | Pending |

4. Monitoring & Batch Health Optimization
Once live, the DevOps team will monitor batch processing behaviors for 60 minutes via Datadog and CloudWatch.

Key Performance Indicators (KPIs) to Watch
HTTP Error Rates: Monitor for spikes in 5xx responses (Threshold: > 0.5% of total traffic).

DB CPU Utilization: Ensure database read latency remains under 80ms during the bulk transaction processing.

Redis Memory Usage: Ensure the new token-bucket rate limiter does not exhaust available cluster memory cache.

[Datadog Dashboard Monitor Alert Settings]
IF avg(last_5m):aws.rds.cpu_utilization{env:production} > 85% 
THEN CRITICAL ALERT -> Page On-Call Backend Engineer

5. Rollback Plan (Failover Sequence)
If any smoke test fails, or if database CPU utilization exceeds 90% for more than 3 minutes, the Release Coordinator will call a Hard Rollback.

[Rollback Trigger] ──> 1. Divert Traffic to Old Pods (v4.1.9) ──> 2. Revert Cloudflare CDN Assets ──> 3. Revert DB Migration Script

Steps to Rollback:
Divert Traffic: Force the Kubernetes ingress controller to immediately reroute 100% of traffic back to the previous stable pod deployment (api:v4.1.9).

kubectl rollout undo deployment/api-service-prod

Revert Frontend UI: Point the S3 bucket routing configuration to the cached web:v4.1.9 directory.

Database Down-Migration: Only execute if data corruption is imminent. Run the teardown DDL script to safely drop the newly created index without locking tables:

DROP INDEX CONCURRENTLY idx_bulk_transactions;

Post-Mortem Ticket: Open a P1 Jira ticket detailing the root cause of the deployment failure.





