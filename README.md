# SAP BTP to Snowflake Data Integration using Fivetran (Batch Processing)

## Project Overview

This project demonstrates a data integration pipeline that extracts financial data from SAP BTP (HANA Cloud) and loads it into Snowflake using Fivetran's batch processing capabilities. The solution implements a dedicated, always-on warehouse architecture in Snowflake to ensure consistent data loading performance.

**Project Context:** Research Student Implementation (9 months experience via contractor)  
**Integration Type:** Batch-based ETL Pipeline  
**Source System:** SAP BTP (HANA Cloud)  
**Target System:** Snowflake Data Warehouse  
**Integration Tool:** Fivetran Connector  

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SAP BTP (HANA Cloud)                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Finance Schema (FINANCE)                                     │  │
│  │  - Document Headers (BKPF)                                    │  │
│  │  - Line Items (BSEG)                                          │  │
│  │  - G/L Master Data (SKA1)                                     │  │
│  │  - Custom Views                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                     Read-Only User (FIVETRAN_HANA_USER)             │
│                              │                                       │
└──────────────────────────────┼───────────────────────────────────────┘
                               │
                               │ SSL/TLS (Port 443)
                               │ Batch Sync (Daily/Hourly)
                               ▼
                  ┌────────────────────────┐
                  │                        │
                  │      FIVETRAN          │
                  │   (ETL Orchestration)  │
                  │                        │
                  │  - SAP HANA Connector  │
                  │  - Batch Scheduling    │
                  │  - Schema Mapping      │
                  │                        │
                  └────────────────────────┘
                               │
                               │ Secure Connection
                               │ Auto Schema Management
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           SNOWFLAKE                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  FIVETRAN_WH_DEDICATED (Always-On Warehouse)                 │  │
│  │  - Auto Suspend = 0 (Never suspends)                         │  │
│  │  - Configurable Size (SMALL/MEDIUM)                          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Database: FIVETRAN_DB                                        │  │
│  │    └── Schema: FINANCE_RAW (Landing Zone)                    │  │
│  │         ├── BKPF (Raw Table)                                  │  │
│  │         ├── BSEG (Raw Table)                                  │  │
│  │         ├── SKA1 (Raw Table)                                  │  │
│  │         └── _FIVETRAN_* (Metadata Tables)                    │  │
│  │                                                               │  │
│  │    └── Schema: FINANCE_RAW_V (Clean Views - Optional)        │  │
│  │         └── Deduplicated/Transformed Views                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Flow Sequence

1. **Source Configuration**: SAP HANA Cloud with read-only user credentials
2. **Extraction**: Fivetran connects to HANA via secure SQL (SSL on port 443)
3. **Transfer**: Batch-mode data extraction (scheduled: daily/hourly)
4. **Loading**: Data lands in Snowflake raw schema with Fivetran metadata
5. **Validation**: Row counts and freshness checks in Snowflake
6. **Consumption**: Optional clean views for downstream BI tools

---

## Prerequisites

### Required Access and Credentials

| Component | Requirements |
|-----------|-------------|
| **SAP BTP** | • Access to SAP BTP Cockpit<br>• SAP HANA Cloud instance permissions<br>• Ability to create database users and grant SELECT privileges |
| **Snowflake** | • Snowflake account with ACCOUNTADMIN or similar privileges<br>• Ability to create warehouses, databases, users, and roles |
| **Fivetran** | • Active Fivetran account<br>• Permission to create connectors and destinations |

### Data Scope for POC

Start with a limited set of finance tables (3-5 objects recommended):
- **BKPF**: Accounting document headers
- **BSEG**: Accounting document line items
- **SKA1**: G/L account master data
- Custom curated reporting views (optional)

### Network Considerations

- If organization uses IP allowlisting, Fivetran IP ranges must be added to SAP BTP network security rules
- Ensure SSL/TLS is enabled for HANA Cloud connections
- Confirm port 443 (or configured SQL port) is accessible from Fivetran's network

---

## Implementation Steps

### Phase 0: Pre-Implementation Checklist

Before starting the implementation, ensure:
- ✅ SAP BTP (HANA Cloud) login credentials available
- ✅ Snowflake account credentials available
- ✅ Fivetran account credentials available
- ✅ Identified 3-5 finance tables/views for initial sync
- ✅ Network security team notified (if IP allowlisting required)

---

### Phase 1: SAP BTP (HANA Cloud) Preparation

#### 1.1 Access Database and Open SQL Console

**Steps:**
1. Log in to **SAP BTP Cockpit**
2. Navigate to **Instances and Subscriptions**
3. Open your **SAP HANA Cloud** instance
4. Click **Open in SAP HANA Cloud Central**
5. Launch **Database Explorer**
6. Select your database and open **SQL Console**

#### 1.2 Create Read-Only User for Fivetran

Create a dedicated user with minimal privileges (read-only access to required schemas):

**User Configuration:**
- Username: `FIVETRAN_HANA_USER`
- Access Level: SELECT only on FINANCE schema (or specific tables)
- Security: Strong password required

**Privileges Granted:**
- SELECT on SCHEMA FINANCE
- SELECT on underlying schemas (if using calculation views)

**Security Note:** This user has read-only access to prevent any unintended data modifications from the integration tool.

#### 1.3 Document Connection Parameters

Record the following for Fivetran configuration:

| Parameter | Example Value | Notes |
|-----------|---------------|-------|
| **Host** | `xxxxxx.hana.ondemand.com` | HANA Cloud hostname |
| **Port** | `443` | Standard secure SQL port |
| **User** | `FIVETRAN_HANA_USER` | Created in step 1.2 |
| **Password** | `<secure_password>` | Strong password set |
| **SSL** | `Yes` | Always use SSL/TLS |

**Network Security:** If your organization requires IP allowlisting, coordinate with network administrators to add Fivetran's IP ranges for your region.

---

### Phase 2: Snowflake Preparation (Dedicated Warehouse)

#### 2.1 Access Snowflake Worksheet

**Steps:**
1. Log in to **Snowflake** web interface
2. Navigate to left menu → **Worksheets**
3. Create new SQL Worksheet (click **+** button)

#### 2.2 Execute Snowflake Setup Configuration

**Components Created:**

**a) Dedicated Role for Fivetran**
- Role Name: `FIVETRAN_ROLE`
- Purpose: Isolate Fivetran permissions

**b) Dedicated Warehouse (Always-On Configuration)**
- Warehouse Name: `FIVETRAN_WH_DEDICATED`
- Size: `SMALL` (adjustable based on performance)
- **Auto Suspend: 0** (never auto-suspends - always running)
- Auto Resume: `TRUE` (can restart if manually stopped)
- Initially Suspended: `FALSE` (starts immediately)

**Rationale for Always-On Warehouse:**
- Eliminates cold-start delays during batch loads
- Ensures predictable load performance
- Suitable for scheduled batch processing patterns
- Can be scaled up/down without downtime

**c) Database and Schema Structure**
- Database: `FIVETRAN_DB`
- Schema: `FINANCE_RAW` (landing zone for raw data)

**d) Dedicated User for Fivetran**
- Username: `FIVETRAN_USER`
- Default Role: `FIVETRAN_ROLE`
- Default Warehouse: `FIVETRAN_WH_DEDICATED`
- Default Namespace: `FIVETRAN_DB.FINANCE_RAW`

**e) Permissions Granted to Role**
- USAGE on warehouse, database, and schema
- CREATE TABLE, CREATE VIEW, CREATE STAGE, CREATE FILE FORMAT on schema
- Role assigned to `FIVETRAN_USER`

#### 2.3 Verify Warehouse Status

**Validation Steps:**
1. Navigate to top bar → **Admin** → **Warehouses**
2. Locate `FIVETRAN_WH_DEDICATED`
3. Confirm status shows **Running** (green indicator)
4. Verify **Auto Suspend = 0**
5. If suspended, click **Resume**

#### 2.4 Document Snowflake Connection Details

Record the following for Fivetran configuration:

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Account** | `xy12345.eu-central-1` | Snowflake account identifier |
| **User** | `FIVETRAN_USER` | Created in step 2.2 |
| **Password** | `<secure_password>` | Strong password set |
| **Warehouse** | `FIVETRAN_WH_DEDICATED` | Always-on warehouse |
| **Database** | `FIVETRAN_DB` | Landing database |
| **Schema** | `FINANCE_RAW` | Raw data schema |
| **Role** | `FIVETRAN_ROLE` | Isolated role |

---

### Phase 3: Configure Snowflake Destination in Fivetran

#### 3.1 Add Destination

**Steps:**
1. Log in to **Fivetran** dashboard
2. Navigate to left menu → **Destinations**
3. Click **+ Add destination**
4. Select **Snowflake** from available destinations

#### 3.2 Configure Connection Parameters

Enter the Snowflake connection details documented in Phase 2.4:

**Configuration Fields:**
- Account identifier
- Username and password
- Warehouse, database, schema, and role

**Connection Test:**
- Click **Save & Test**
- Wait for "Connection successful" confirmation
- If connection fails, verify credentials and network connectivity

---

### Phase 4: Configure SAP HANA Source Connector (Batch Mode)

#### 4.1 Add SAP HANA Connector

**Steps:**
1. In Fivetran dashboard → **Connectors**
2. Click **+ Add connector**
3. Search for **SAP HANA**
4. Select the SAP HANA connector

#### 4.2 Enter HANA Connection Credentials

Configure the connector with SAP BTP connection details from Phase 1.3:

**Configuration Fields:**
- Host: HANA Cloud hostname
- Port: 443 (or configured port)
- User: `FIVETRAN_HANA_USER`
- Password: User password
- Use SSL: **Enabled**
- Destination: Select Snowflake destination created in Phase 3

**Connection Test:**
- Click **Save & Test**
- Wait for successful connection confirmation
- Click **Continue** to proceed

#### 4.3 Select Finance Objects for Synchronization

**Schema Selection Process:**
1. On the Schemas/Tables screen, expand the **FINANCE** schema
2. Toggle **ON** for 3-5 initial tables (recommended for POC):
   - ✅ BKPF (Accounting document headers)
   - ✅ BSEG (Accounting document line items)
   - ✅ SKA1 (G/L account master data)
   - ✅ Additional curated reporting views (as needed)
3. Click **Save** or **Save & Continue**

**Note:** Starting with a limited set allows for faster initial sync and easier validation. Additional tables can be added incrementally.

#### 4.4 Configure Sync Frequency (Batch Mode)

**Sync Schedule Configuration:**
1. Navigate to connector **Setup** tab
2. Locate **Sync Frequency** setting
3. Select **Daily** for initial POC phase
4. Leave advanced settings at default values
5. Click **Save**

**Frequency Options:**
- **Daily**: Recommended for POC and stable production environments
- **Hourly**: Can be enabled after successful validation for fresher data
- Custom intervals available based on business requirements

**Note:** Batch processing with daily frequency balances data freshness with system resource utilization.

#### 4.5 Initiate Historical Sync

**Initial Load Process:**
1. On the connector page, click **Start Initial Sync** (or **Start Sync**)
2. Monitor progress bar for sync status
3. Wait for completion (duration depends on data volume)
4. Verify status changes to **Synced** with timestamp

**Initial Sync Characteristics:**
- Performs full historical load of selected tables
- May take longer than incremental syncs
- Establishes baseline for future incremental updates

---

### Phase 5: Validation in Snowflake

#### 5.1 Verify Data Landing

**Validation Queries:**

**a) List Loaded Tables**
```sql
SHOW TABLES IN SCHEMA FIVETRAN_DB.FINANCE_RAW;
```

**b) Sample Data Inspection**
```sql
SELECT * FROM FIVETRAN_DB.FINANCE_RAW.BKPF LIMIT 50;
```

**c) Row Count Validation**
```sql
SELECT TABLE_NAME, ROW_COUNT
FROM FIVETRAN_DB.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'FINANCE_RAW'
ORDER BY ROW_COUNT DESC;
```

**Expected Results:**
- All selected tables appear in `FINANCE_RAW` schema
- Sample queries return data matching SAP source
- Row counts align with expected volumes

#### 5.2 Data Freshness Verification (Optional)

Validate that the most recent data has been loaded:

**Sample Freshness Check:**
```sql
SELECT MAX(POSTING_DATE) AS latest_posting_date
FROM FIVETRAN_DB.FINANCE_RAW.BKPF;
```

**Validation Criteria:**
- Maximum date should reflect recent business transactions
- Date should be within expected range based on sync schedule

**Fivetran Metadata:**
- Each table includes `_FIVETRAN_SYNCED` timestamp column
- Indicates when the record was last synchronized
- Useful for tracking data lineage

---

### Phase 6: Performance Optimization and Tuning

#### 6.1 Dedicated Warehouse Management

**Current Configuration:**
- Warehouse: `FIVETRAN_WH_DEDICATED`
- Auto Suspend: **0** (always running)
- Current Size: `SMALL`

**Performance Tuning:**

**Scale Up (for faster loading):**
```sql
ALTER WAREHOUSE FIVETRAN_WH_DEDICATED SET WAREHOUSE_SIZE = 'MEDIUM';
```

**Scale Down (to reduce costs):**
```sql
ALTER WAREHOUSE FIVETRAN_WH_DEDICATED SET WAREHOUSE_SIZE = 'SMALL';
```

**Sizing Recommendations:**
- **XSMALL/SMALL**: Suitable for POC and small data volumes (<1M rows)
- **MEDIUM**: Recommended for production with moderate volumes (1M-10M rows)
- **LARGE**: For high-volume scenarios or time-sensitive loads

#### 6.2 Cost Monitoring

**Warehouse Usage Query:**
```sql
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE WAREHOUSE_NAME = 'FIVETRAN_WH_DEDICATED'
ORDER BY START_TIME DESC;
```

**Monitoring Metrics:**
- Credits consumed per hour
- Average execution times
- Peak usage periods

**Cost Optimization Strategy:**
- Monitor usage patterns during POC
- Right-size warehouse based on actual performance requirements
- Consider scheduled warehouse scaling for predictable load patterns

---

### Phase 7: Data Quality Layer (Optional)

#### 7.1 Create Clean Views Schema

**Purpose:** Provide deduplicated, type-safe views for downstream BI tools while preserving raw data integrity.

**Schema Creation:**
```sql
CREATE OR REPLACE SCHEMA FIVETRAN_DB.FINANCE_RAW_V;
```

#### 7.2 Deduplication Pattern

**Example: Document Header Deduplication**
```sql
CREATE OR REPLACE VIEW FIVETRAN_DB.FINANCE_RAW_V.BKPF_CLEAN AS
SELECT *
FROM (
  SELECT t.*,
         ROW_NUMBER() OVER (
           PARTITION BY DOCUMENT_NUMBER
           ORDER BY _FIVETRAN_SYNCED DESC NULLS LAST
         ) AS rn
  FROM FIVETRAN_DB.FINANCE_RAW.BKPF t
)
WHERE rn = 1;
```

**Deduplication Logic:**
- Partitions by business key (e.g., DOCUMENT_NUMBER)
- Selects most recently synced record
- Handles potential duplicates from incremental loads

**Benefits:**
- Raw tables remain unchanged (audit trail preserved)
- Clean views provide single version of truth
- Easy to add data type transformations or business logic

---

## Troubleshooting Guide

### Common Issues and Resolutions

#### Issue 1: Fivetran Cannot Connect to HANA

**Symptoms:**
- Connection test fails in Fivetran
- "Unable to reach host" error

**Resolution Steps:**
1. Verify host, port, and SSL settings are correct
2. Confirm `FIVETRAN_HANA_USER` credentials
3. Check if Fivetran IP addresses are allowlisted in SAP BTP network security
4. Test connectivity using SAP HANA Database Explorer
5. Review SAP HANA Cloud instance status (ensure it's running)

---

#### Issue 2: Tables Not Visible in Fivetran

**Symptoms:**
- Expected tables don't appear in Fivetran schema selection
- Schema appears empty

**Resolution Steps:**
1. Verify SELECT grants on schema to `FIVETRAN_HANA_USER`
2. In Fivetran connector → **Schemas** → Click **Rescan**
3. Toggle tables **ON** after rescan completes
4. Click **Save**
5. For calculation views, ensure SELECT on underlying schemas is granted

---

#### Issue 3: Slow Initial Sync Performance

**Symptoms:**
- Initial sync taking excessive time
- Connector stuck at low percentage completion

**Resolution Steps:**
1. Temporarily scale warehouse to MEDIUM during initial load:
   ```sql
   ALTER WAREHOUSE FIVETRAN_WH_DEDICATED SET WAREHOUSE_SIZE = 'MEDIUM';
   ```
2. Monitor sync progress in Fivetran dashboard
3. After initial sync completes, scale back to SMALL if appropriate
4. Consider reducing table count for POC if performance issues persist
5. Check SAP HANA resource utilization (may need to optimize queries)

---

#### Issue 4: Data Type Mapping Issues

**Symptoms:**
- Unexpected data types in Snowflake
- Date/time fields formatted as strings

**Resolution Steps:**
1. Keep raw tables as-is (Fivetran manages schema)
2. Create views in `FINANCE_RAW_V` schema with explicit CAST operations
3. Document data type mappings for downstream consumers
4. Adjust BI tool connections to use clean views instead of raw tables

---

## Project Checklist

### POC Implementation Checklist

Use this checklist to track implementation progress:

**SAP BTP Configuration:**
- [ ] Created `FIVETRAN_HANA_USER` with SELECT privileges on finance schema
- [ ] Documented HANA connection parameters (host, port, SSL, credentials)
- [ ] Verified network connectivity (IP allowlisting if required)

**Snowflake Configuration:**
- [ ] Created `FIVETRAN_WH_DEDICATED` with Auto Suspend = 0
- [ ] Created `FIVETRAN_DB` database and `FINANCE_RAW` schema
- [ ] Created `FIVETRAN_USER`, `FIVETRAN_ROLE`, and assigned grants
- [ ] Verified warehouse is running (green status indicator)

**Fivetran Configuration:**
- [ ] Added Snowflake Destination (connection test successful)
- [ ] Added SAP HANA Connector (connection test successful)
- [ ] Selected 3-5 finance tables/views for initial sync
- [ ] Configured Daily sync frequency
- [ ] Started Initial Sync

**Validation:**
- [ ] Verified tables landed in `FIVETRAN_DB.FINANCE_RAW` schema
- [ ] Executed row count validation queries
- [ ] Performed data freshness spot-checks
- [ ] Confirmed sync status shows "Synced" in Fivetran

**Optional Enhancements:**
- [ ] Created `FINANCE_RAW_V` schema for clean views
- [ ] Implemented deduplication logic in views
- [ ] Configured warehouse usage monitoring
- [ ] Documented data type mappings

---

## Alternative Configuration: SAP S/4HANA (ODP) Source

### When to Use This Variant

If your source system is **SAP S/4HANA with Operational Data Provisioning (ODP)** instead of direct HANA tables:

### Configuration Differences

**Phase 2 (Snowflake):** Identical setup - follow all steps in Phase 2 for dedicated warehouse configuration.

**Phase 4 (Fivetran Connector):** Use different connector type:

1. **Connector Selection:**
   - In Fivetran → Connectors → Add connector
   - Choose **SAP ERP** or **SAP S/4HANA (ODP)** connector

2. **Connection Parameters:**
   - SAP Application Server/Host
   - Instance number
   - Client
   - Technical user with ODP access permissions
   - Password

3. **Data Source Selection:**
   - Search and select finance extractors (examples):
     - `0FI_GL_14` (G/L line items)
     - `0FI_AP_4` (Accounts payable)
     - `0FI_AR_4` (Accounts receivable)

4. **Sync Configuration:**
   - Sync Frequency: **Daily** for POC
   - Start Initial Sync
   - Validate in Snowflake following Phase 5 steps

**Note:** ODP extractors provide pre-configured data structures optimized for analytics, potentially reducing transformation requirements.

---

## Future Enhancements and Next Steps

### Short-Term (Post-POC)

1. **Expand Table Coverage:**
   - Add additional finance tables beyond initial 3-5 objects
   - Include master data tables (cost centers, profit centers)
   - Incorporate custom Z-tables if applicable

2. **Increase Sync Frequency:**
   - Move from Daily to Hourly sync for near real-time reporting
   - Evaluate business requirements for data freshness

3. **Optimize Performance:**
   - Fine-tune warehouse sizing based on actual load patterns
   - Implement scheduled scaling (larger during business hours)

### Medium-Term

4. **Implement Change Data Capture (CDC):**
   - Transition from batch to log-based CDC for real-time updates
   - Reduces data transfer volumes (only changed records)
   - Improves data freshness to near real-time

5. **Build Data Quality Framework:**
   - Implement data validation rules in clean views
   - Add NULL handling and data type conversions
   - Create data quality monitoring dashboards

6. **Expand to Additional Modules:**
   - Materials Management (MM)
   - Sales and Distribution (SD)
   - Production Planning (PP)

### Long-Term

7. **Enterprise Data Warehouse Design:**
   - Implement dimensional modeling (star schema)
   - Create conformed dimensions across modules
   - Build aggregate tables for performance

8. **Advanced Analytics:**
   - Integrate with BI tools (Tableau, Power BI, Looker)
   - Develop predictive models using Snowflake ML functions
   - Implement real-time dashboards

9. **Data Governance:**
   - Implement role-based access control (RBAC)
   - Add data masking for sensitive fields
   - Establish data lineage tracking

---

## Technical Specifications

### Technology Stack

| Layer | Technology | Version/Details |
|-------|-----------|-----------------|
| **Source** | SAP BTP (HANA Cloud) | Cloud-hosted HANA database |
| **ETL** | Fivetran | SaaS ETL platform |
| **Target** | Snowflake | Cloud data warehouse |
| **Connection** | SSL/TLS | Secure encrypted connection |

### Data Characteristics

- **Volume**: POC with 3-5 tables (scalable to full schema)
- **Frequency**: Daily batch (configurable to hourly)
- **Mode**: Full historical load + incremental updates
- **Latency**: 24-hour freshness (POC), reducible to 1-hour

### Performance Metrics (Baseline)

- **Initial Sync**: Varies by volume (typically 10-60 minutes for POC)
- **Incremental Sync**: 5-15 minutes for daily batch
- **Warehouse**: SMALL size adequate for POC workloads

---

## Key Learnings and Observations

### Technical Insights

1. **Always-On Warehouse Strategy:**
   - Eliminates cold-start latency for batch jobs
   - Provides consistent performance for scheduled loads
   - Trade-off between cost and predictability

2. **Fivetran Schema Management:**
   - Automatic schema detection and mapping
   - Adds metadata columns (`_FIVETRAN_SYNCED`, `_FIVETRAN_DELETED`)
   - Handles schema drift automatically

3. **Read-Only Source Access:**
   - Minimizes risk of unintended data modifications
   - Simplifies security audit trail
   - Follows principle of least privilege

### Best Practices Identified

1. **Start Small, Scale Gradually:**
   - POC with 3-5 tables validates approach quickly
   - Incremental expansion reduces risk

2. **Preserve Raw Data:**
   - Keep Fivetran-loaded tables unchanged
   - Build transformation logic in separate views
   - Maintains audit trail and data lineage

3. **Monitor and Optimize:**
   - Track warehouse utilization from day one
   - Right-size resources based on actual metrics
   - Balance cost and performance requirements

---

## References and Resources

### SAP BTP Documentation
- SAP HANA Cloud Administration Guide
- SAP BTP Cockpit User Guide
- Network security and IP allowlisting

### Snowflake Documentation
- Warehouse management and sizing
- Security best practices
- Cost optimization strategies

### Fivetran Documentation
- SAP HANA connector documentation
- Sync frequency and scheduling options
- Troubleshooting guides

---

## Project Metadata

**Project Duration:** 3-4 weeks (POC phase)  
**Implementation Approach:** Iterative, starting with minimal viable scope  
**Validation Method:** Data quality checks, row counts, freshness verification  
**Success Criteria:** Successful daily batch loads with <1 hour latency from source to target  

---

## Appendix

### Glossary

- **BTP**: Business Technology Platform (SAP's cloud platform)
- **CDC**: Change Data Capture (real-time data synchronization)
- **ETL**: Extract, Transform, Load (data integration pattern)
- **ODP**: Operational Data Provisioning (SAP data extraction framework)
- **POC**: Proof of Concept (initial validation phase)
- **RBAC**: Role-Based Access Control (security pattern)

### Acronyms for SAP Tables

- **BKPF**: Accounting Document Header Table
- **BSEG**: Accounting Document Segment (Line Items)
- **SKA1**: G/L Account Master Data

---

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Author:** Research Student Implementation  
**Review Status:** POC Complete, Ready for Production Planning

