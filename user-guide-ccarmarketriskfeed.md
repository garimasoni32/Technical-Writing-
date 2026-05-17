# User Guide: Reading and Verifying CCAR Market Risk Feeds
## Document Overview

This guide provides Risk Management Analysts, Financial Compliance Officers, and Auditing teams with the non-technical foundations required to access, read, and verify data flowing through the Comprehensive Capital Analysis and Review (CCAR) Market Risk feed.

This document details how the ingestion system reads incoming files, how data is structurally broken down, and how to interpret the generated columns within the final reporting ledger for FR Y-14A/Q regulatory submissions.

1. **How the Ingestion Feed Reads Data**
   
   The CCAR Market Risk system processes end-of-day batch files arriving from diverse global trading desks (e.g., Equity, FX, and Fixed Income).

   Instead of processing rows in isolation, the ingestion engine groups and validates data using a strict four-key natural grain. The system reads incoming data through an "all-or-nothing" validation framework to ensure absolute data integrity.

**The Ingestion Sequence:**

Perimeter Verification: The system checks incoming batch files for fundamental data health (correct date formats, valid subsidiary identifiers, and approved scenario names).

Asynchronous Ingest: Accepted files are immediately assigned a unique tracking job ID, queued, and processed in the background to prevent network drops.

The Overwrite (UPSERT) Rule: If a trading desk submits an updated file for an identical combination of Date, Legal Entity, Scenario, and Book ID, the system automatically overwrites the existing record with the newer values and flags it with a fresh processing timestamp.

2. **Deciphering the Reporting Ledger (Column Directory)**
   
   Once the raw data is successfully processed, it creates a standardized, linear report. Below is the comprehensive directory of the columns generated in the final ledger, their data constraints, and their regulatory meaning.

**Data Grid Directory**

3. **Critical Analytical Conventions (How to Read the Metrics)**

   To prevent severe miscalculations when aggregating files for stress testing reports, analysts must observe the following data transformations:

   1. Value-at-Risk (VaR) Numeric ScalingThe Rule: The column var_99_1d stores values as absolute integer currency cents, completely eliminating the floating-point decimal rounding errors that can corrupt capital calculation aggregates.
     
      The Conversion: To read the true dollar exposure, divide the database value by 100.
     
    Example:

   If the row displays a var_99_1d value of 125000500,
   
      the actual exposure is calculated as:

       $$125,000,500 \div 100 = \$1,250,005.00$$

   2. Directional Risk Sensitivities (The Greeks)Positive vs. Negative Sensitivities:

     Pay close attention to the sign ($+$ or $-$) displayed in the sensitivity columns.

    A negative Vega indicates the portfolio loses value if market volatility spikes, while a positive Delta indicates the portfolio benefits from an upward move in asset pricing.

4. Operational Sign-off Checklist:
  
   Before approving a daily reporting cycle for Federal Reserve review, analysts must confirm the feed meets these baseline criteria:

   [ ] Zero Asset Domain Exceptions: Ensure no asset classes fall outside the permitted EQUITY, FX, CREDIT, or IR values.

   [ ] Temporal Alignment: Verify that the processed_at timestamp matches the expected batch execution window for the current business cycle.

   [ ] Materiality Audit: Flag any book where the converted var_99_1d dollar value exceeds your firm’s designated regulatory exception threshold (e.g., $>\$1,000,000.00$) for immediate sign-off escalation.
