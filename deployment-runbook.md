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
[ ] Lock Master Branches: Restrict further merges to main.

[ ] Trigger Manual DB Backup: Execute AWS RDS snapshot of the production instance.

aws rds create-db-snapshot --db-instance-identifier prod-db-cluster --db-snapshot-identifier prod-backup-before-v4-2-0

[ ] Notify Stakeholders: Broadcast automated maintenance banner to the web portal.

Phase 2: Deployment Execution (02:15 AM EST)
[ ] Step 1: Run database migration scripts (014_add_bulk_sync_indexes.sql) to add indexes for the bulk sync service.

[ ] Step 2: Deploy the backend API Docker container (api:v4.2.0) to the Kubernetes staging pool, then shift production traffic using a Canary Deployment strategy (10% traffic increment every 5 minutes).

[ ] Step 3: Purge Cloudflare CDN edge cache and deploy the frontend assets (web:v4.2.0) to the S3 bucket.

3. Post-Production Testing Plan (Smoke Tests)
QA engineers will verify live system health immediately following the deployment using the following manual and automated scripts.

⚠️ Important: Ensure all smoke tests pass before increasing canary traffic allocation to 100%.

Automated Verification
[ ] Health Check Endpoint: Verify GET [https://api.platform.com/health](https://api.platform.com/health) returns 200 OK and status healthy.

[ ] Postman Collection Run: Trigger the Prod-Regression-v4.2 suite via Newman to validate core workflows.


