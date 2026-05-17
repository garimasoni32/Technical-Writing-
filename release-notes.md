# Release Notes: Platform Update (v4.2.0)
**Release Date:** May 2026  
**Target Audience:** Retail Store Managers, IT Administrators, and Partner Developers  

---

## Executive Summary
Version 4.2.0 introduces real-time inventory reconciliation for Direct Store Delivery (DSD) vendors and optimizes barcode scanning speeds on handheld Android devices by 35%. This release also updates our public Inventory API to support bulk-patch requests.

---

## What’s New in This Release?

### 1. Enhanced DSD (Direct Store Delivery) Workflow
Store associates can now reconcile vendor deliveries directly from the mobile app without waiting for end-of-day batch processing.
*   **Impact:** Reduces discrepancies in fresh food inventory (e.g., bakery ingredients).
*   **Action Required:** Ensure mobile devices are updated to app version 4.2.0.

### 2. Barcode Scanner Performance Boost
We optimized the camera-to-SaaS telemetry pipeline to fix a lag issue reported during peak counting hours.
*   **Resolved Issue:** Fixed an issue where rapid consecutive scans caused the app to temporarily freeze on legacy Zebra hardware.

---

## Developer & API Updates (v4.2.0)

> ⚠️ **Deprecation Notice:** The endpoint `POST /api/v1/inventory/single-scan` is now deprecated and will be retired in v5.0.0. Please migrate to the new bulk endpoint detailed below.

### New Endpoint: Bulk Inventory Sync
To support faster inventory counting, we have introduced a bulk upload endpoint to our Inventory API.

*   **Endpoint:** `POST /api/v1/inventory/bulk-sync`
*   **Payload Example:**
```json
{
  "store_id": "STORE-9921",
  "scans": [
    { "barcode": "01234567", "quantity": 5 },
    { "barcode": "76543210", "quantity": 12 }
  ]
}
