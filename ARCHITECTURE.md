# System Architecture Documentation
## SAP BTP to Snowflake Data Integration Pipeline

**Project Type:** Batch-based ETL Data Integration  
**Documentation Version:** 1.0  
**Author:** Research Student Implementation  
**Date:** October 2025

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Component Architecture](#component-architecture)
3. [Network Architecture](#network-architecture)
4. [Data Architecture](#data-architecture)
5. [Security Architecture](#security-architecture)
6. [Deployment Architecture](#deployment-architecture)

---

## Architecture Overview

### High-Level System Architecture (Mermaid)

```mermaid
flowchart TB
    subgraph SAP["SAP BTP Cloud Environment"]
        HANA[("SAP HANA Cloud<br/>Database")]
        SCHEMA["Finance Schema<br/>(BKPF, BSEG, SKA1)"]
        USER["Read-Only User<br/>FIVETRAN_HANA_USER"]
    end

    subgraph FIVETRAN["Fivetran Cloud Platform"]
        CONNECTOR["SAP HANA<br/>Connector"]
        SCHEDULER["Batch Scheduler<br/>(Daily/Hourly)"]
        MAPPER["Schema Mapper"]
        MONITOR["Sync Monitor"]
    end

    subgraph SNOWFLAKE["Snowflake Cloud Data Platform"]
        WH["Dedicated Warehouse<br/>FIVETRAN_WH_DEDICATED<br/>(Always-On)"]
        DB[("Database<br/>FIVETRAN_DB")]
        RAW["Raw Schema<br/>FINANCE_RAW"]
        CLEAN["Clean Views<br/>FINANCE_RAW_V"]
    end

    subgraph CONSUME["Data Consumers"]
        BI["BI Tools"]
        ANALYTICS["Analytics"]
        REPORTS["Reports"]
    end

    HANA --> SCHEMA
    SCHEMA --> USER
    USER -->|"SSL/TLS<br/>Port 443"| CONNECTOR
    CONNECTOR --> SCHEDULER
    SCHEDULER --> MAPPER
    MAPPER --> MONITOR
    MONITOR -->|"Bulk Load"| WH
    WH --> DB
    DB --> RAW
    RAW -.->|"Optional"| CLEAN
    RAW --> CONSUME
    CLEAN --> CONSUME

    style SAP fill:#0066cc,stroke:#003d7a,color:#fff
    style FIVETRAN fill:#6200ea,stroke:#4a0099,color:#fff
    style SNOWFLAKE fill:#29b5e8,stroke:#1a8cb8,color:#fff
    style CONSUME fill:#00c853,stroke:#009624,color:#fff
```

### Three-Tier Architecture

```mermaid
graph TB
    subgraph Tier1["TIER 1: SOURCE LAYER<br/>(SAP BTP - HANA Cloud)"]
        H1[("SAP HANA Cloud<br/>Database Instance")]
        F1["Finance Schema<br/>• BKPF (Document Headers) ~2M rows<br/>• BSEG (Line Items) ~15M rows<br/>• SKA1 (G/L Master) ~5K rows<br/>• Custom Views (Variable)"]
        U1["Access Control<br/>• User: FIVETRAN_HANA_USER<br/>• Privilege: SELECT only<br/>• Authentication: Password-based<br/>• Connection: SSL/TLS encrypted"]
    end
    
    subgraph Tier2["TIER 2: INTEGRATION LAYER<br/>(Fivetran Platform)"]
        C1["SAP HANA Connector<br/>• Connection Manager<br/>• Query Executor<br/>• Schema Reader"]
        W1["Snowflake Writer<br/>• Bulk Loader<br/>• Type Mapper<br/>• Error Handler"]
        O1["Orchestration Engine<br/>• Batch Scheduler (Daily sync)<br/>• State Manager (Track syncs)<br/>• Metadata Store"]
    end
    
    subgraph Tier3["TIER 3: TARGET LAYER<br/>(Snowflake Data Warehouse)"]
        WH1["Compute Layer<br/>FIVETRAN_WH_DEDICATED<br/>• Size: SMALL (1 credit/hour)<br/>• Auto Suspend: 0 (Always On)<br/>• Auto Resume: TRUE<br/>• Concurrency: 8 queries max"]
        ST1["Storage Layer<br/>FIVETRAN_DB Database"]
        S1["FINANCE_RAW Schema<br/>• BKPF (Business Columns + Metadata)<br/>• BSEG<br/>• SKA1<br/>• Additional Tables"]
        V1["FINANCE_RAW_V Schema<br/>• BKPF_CLEAN (deduplicated)<br/>• BSEG_CLEAN<br/>• Business Logic Views"]
    end
    
    H1 --> F1
    F1 --> U1
    U1 -->|"SSL/TLS (Port 443)<br/>Pull-based"| C1
    C1 --> O1
    O1 --> W1
    W1 -->|"Snowflake Protocol<br/>Push-based"| WH1
    WH1 --> ST1
    ST1 --> S1
    S1 --> V1
    
    style Tier1 fill:#0066cc,stroke:#003d7a,color:#fff
    style Tier2 fill:#6200ea,stroke:#4a0099,color:#fff
    style Tier3 fill:#29b5e8,stroke:#1a8cb8,color:#fff
```

---

## Component Architecture

### Component Interaction Diagram (Mermaid)

```mermaid
graph LR
    subgraph Source["Source Components"]
        HC[HANA Cloud DB]
        RU[Read-Only User]
        FS[Finance Schema]
    end

    subgraph Integration["Integration Components"]
        FC[Fivetran Connector]
        BS[Batch Scheduler]
        SM[Schema Mapper]
        MS[Metadata Store]
        MN[Monitoring]
    end

    subgraph Target["Target Components"]
        WH[Warehouse]
        LS[Landing Schema]
        VS[View Schema]
        MT[Metadata Tables]
    end

    HC -->|Contains| FS
    FS -->|Accessed by| RU
    RU -->|Authenticates| FC
    FC -->|Queries| FS
    FC -->|Reports to| MS
    BS -->|Triggers| FC
    FC -->|Uses| SM
    SM -->|Writes| LS
    MN -->|Monitors| FC
    WH -->|Executes| LS
    LS -->|Source for| VS
    FC -->|Updates| MT

    style Source fill:#e3f2fd,stroke:#1976d2
    style Integration fill:#f3e5f5,stroke:#7b1fa2
    style Target fill:#e0f2f1,stroke:#00796b
```

### Component Details

```mermaid
graph TD
    subgraph Source["SOURCE COMPONENTS (SAP BTP)"]
        A["[A] SAP HANA Cloud Database<br/>• In-memory columnar database<br/>• SAP BTP Cloud Infrastructure<br/>• HANA Cloud 2.x<br/>• Operational data store"]
        B["[B] Finance Schema<br/>• Tables: BKPF, BSEG, SKA1<br/>• Custom reporting views<br/>• Business logic organization"]
        C["[C] Database User<br/>• FIVETRAN_HANA_USER<br/>• SELECT only (read-only)<br/>• Password authentication<br/>• Secure data extraction"]
    end
    
    subgraph Integration["INTEGRATION COMPONENTS (Fivetran)"]
        D["[D] SAP HANA Connector<br/>• Pre-built connector<br/>• JDBC over SSL<br/>• Batch pull mode<br/>• Auto schema detection"]
        E["[E] Batch Scheduler<br/>• Daily frequency<br/>• Time-based triggers<br/>• Sync orchestration"]
        F["[F] Schema Mapper<br/>• Data type conversion<br/>• HANA → Snowflake mapping<br/>• Type standardization"]
        G["[G] Metadata Store<br/>• Sync history tracking<br/>• Schema versions<br/>• Incremental sync state"]
        H["[H] Monitoring & Alerting<br/>• Sync metrics<br/>• Error notifications<br/>• Web dashboard"]
    end
    
    subgraph Target["TARGET COMPONENTS (Snowflake)"]
        I["[I] Virtual Warehouse<br/>• FIVETRAN_WH_DEDICATED<br/>• SMALL size (4 servers)<br/>• Always-on (no suspend)<br/>• Dedicated compute"]
        J["[J] Database<br/>• FIVETRAN_DB<br/>• Logical container<br/>• Unlimited storage"]
        K["[K] Landing Schema<br/>• FINANCE_RAW<br/>• Raw data tables<br/>• Metadata columns"]
        L["[L] Clean Views Schema<br/>• FINANCE_RAW_V<br/>• Deduplication logic<br/>• Type casting"]
        M["[M] Access Control<br/>• FIVETRAN_USER<br/>• FIVETRAN_ROLE<br/>• Limited privileges"]
    end
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I --> J
    J --> K
    K --> L
    L --> M
    
    style Source fill:#e3f2fd,stroke:#1976d2
    style Integration fill:#f3e5f5,stroke:#7b1fa2
    style Target fill:#e0f2f1,stroke:#00796b
```

---

## Network Architecture

### Network Flow Diagram (Mermaid)

```mermaid
sequenceDiagram
    participant HANA as SAP HANA Cloud<br/>(Source)
    participant FW1 as SAP Firewall<br/>(IP Allowlist)
    participant FT as Fivetran Cloud<br/>(ETL Engine)
    participant FW2 as Snowflake Security<br/>(Auth Layer)
    participant SF as Snowflake<br/>(Target)

    Note over FT: Scheduled Sync Triggered<br/>(Daily at 02:00 UTC)
    
    FT->>FW1: Connection Request<br/>(Source IP from Fivetran Range)
    FW1->>FW1: Validate IP Allowlist
    FW1->>HANA: Forward Request
    
    HANA->>FT: TLS Handshake (Port 443)
    FT->>HANA: Authenticate (FIVETRAN_HANA_USER)
    HANA->>FT: Authentication Success
    
    FT->>HANA: SELECT * FROM FINANCE.BKPF<br/>WHERE MODIFIED > last_sync
    HANA->>FT: Result Set (Encrypted)
    
    Note over FT: Transform & Buffer Data
    
    FT->>FW2: Snowflake Authentication<br/>(FIVETRAN_USER credentials)
    FW2->>SF: Validate User & Role
    SF->>FT: Session Token
    
    FT->>SF: COPY INTO FINANCE_RAW.BKPF<br/>(Bulk Load via Snowpipe)
    SF->>SF: Load Data into Table
    SF->>FT: Load Confirmation (rows inserted)
    
    FT->>FT: Update Metadata Store<br/>(Sync timestamp, row count)
    
    Note over FT: Sync Complete<br/>Status: Success
```

### Network Topology (ASCII)

```
                    INTERNET / CLOUD BACKBONE
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌──────────────────┐   ┌──────────────┐
│  SAP BTP      │   │    FIVETRAN      │   │  SNOWFLAKE   │
│  (Source)     │   │  (Integration)   │   │   (Target)   │
├───────────────┤   ├──────────────────┤   ├──────────────┤
│               │   │                  │   │              │
│  ┌─────────┐  │   │  ┌────────────┐ │   │  ┌────────┐  │
│  │ HANA DB │  │   │  │ Connector  │ │   │  │ Account│  │
│  │         │  │   │  │ Services   │ │   │  │ Region │  │
│  └────┬────┘  │   │  └─────┬──────┘ │   │  └───┬────┘  │
│       │       │   │        │        │   │      │       │
│  ┌────▼────┐  │   │  ┌─────▼──────┐ │   │  ┌───▼────┐  │
│  │Firewall │  │   │  │   Router   │ │   │  │Security│  │
│  │IP Allow │  │   │  │            │ │   │  │ Layer  │  │
│  └────┬────┘  │   │  └─────┬──────┘ │   │  └───┬────┘  │
│       │       │   │        │        │   │      │       │
└───────┼───────┘   └────────┼────────┘   └──────┼───────┘
        │                    │                   │
        │ Port: 443          │ Port: 443         │
        │ Protocol: SSL/TLS  │ Protocol: HTTPS   │
        │ Direction: Inbound │ Direction:Outbound│
        │                    │                   │
        └──────────►┌────────▼────────┐◄─────────┘
                    │                 │
                    │  Fivetran       │
                    │  Orchestrator   │
                    │                 │
                    │  IP Ranges:     │
                    │  • 52.xx.xx.xx  │
                    │  • 34.xx.xx.xx  │
                    │  (Region-based) │
                    │                 │
                    └─────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  NETWORK SECURITY CONFIGURATION                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  SAP BTP Firewall Rules:                                     │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Source IP Range    │ Port │ Protocol │ Direction   │     │
│  ├────────────────────┼──────┼──────────┼─────────────┤     │
│  │ Fivetran IPs       │ 443  │ TLS 1.2+ │ Inbound     │     │
│  │ (Region-specific)  │      │          │ Allow       │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
│  Snowflake Security:                                         │
│  ┌────────────────────────────────────────────────────┐     │
│  │ Authentication: User/Password + Role                │     │
│  │ IP Restrictions: None (handled by user auth)        │     │
│  │ Network Policy: Default (all IPs allowed)           │     │
│  │ MFA: Optional (not required for service accounts)   │     │
│  └────────────────────────────────────────────────────┘     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Data Architecture

### Data Model Architecture (Mermaid)

```mermaid
erDiagram
    BKPF ||--o{ BSEG : contains
    SKA1 ||--o{ BSEG : "G/L account"
    
    BKPF {
        string BUKRS "Company Code"
        string BELNR "Document Number"
        int GJAHR "Fiscal Year"
        date BUDAT "Posting Date"
        string BLART "Document Type"
        decimal WRBTR "Amount"
        timestamp _FIVETRAN_SYNCED "Sync Timestamp"
        boolean _FIVETRAN_DELETED "Delete Flag"
    }
    
    BSEG {
        string BUKRS "Company Code"
        string BELNR "Document Number"
        int GJAHR "Fiscal Year"
        int BUZEI "Line Item"
        string HKONT "G/L Account"
        decimal DMBTR "Amount in Local Currency"
        string KOSTL "Cost Center"
        timestamp _FIVETRAN_SYNCED "Sync Timestamp"
        boolean _FIVETRAN_DELETED "Delete Flag"
    }
    
    SKA1 {
        string KTOPL "Chart of Accounts"
        string SAKNR "G/L Account Number"
        string KTOKS "G/L Account Group"
        string XBILK "Balance Sheet Account"
        timestamp _FIVETRAN_SYNCED "Sync Timestamp"
        boolean _FIVETRAN_DELETED "Delete Flag"
    }
```

### Data Flow Architecture (ASCII)

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DATA FLOW ARCHITECTURE                           │
└──────────────────────────────────────────────────────────────────────┘

STAGE 1: SOURCE DATA (SAP HANA)
═══════════════════════════════════════
┌─────────────────────────────────────┐
│  FINANCE.BKPF (Document Headers)    │
│  Columns: 50+                       │
│  Rows: ~2,000,000                   │
│  Size: ~500 MB                      │
│  Update Pattern: Append + Update    │
└──────────────┬──────────────────────┘
               │
               │ Extraction Query:
               │ SELECT * FROM FINANCE.BKPF
               │ WHERE MODIFIED_AT > :last_sync
               │
               ▼
STAGE 2: IN-FLIGHT TRANSFORMATION (Fivetran)
═══════════════════════════════════════════════
┌─────────────────────────────────────┐
│  Fivetran Processing Buffer         │
│  • Data Type Mapping                │
│  • UTF-8 Encoding                   │
│  • Null Handling                    │
│  • Add Metadata Columns:            │
│    - _FIVETRAN_SYNCED               │
│    - _FIVETRAN_DELETED              │
└──────────────┬──────────────────────┘
               │
               │ Bulk Load Protocol:
               │ COPY INTO ... FROM @STAGE
               │
               ▼
STAGE 3: LANDING ZONE (Snowflake Raw)
═══════════════════════════════════════
┌─────────────────────────────────────┐
│  FINANCE_RAW.BKPF (Raw Table)       │
│  Columns: 52 (50 + 2 metadata)      │
│  Rows: ~2,000,000 (cumulative)      │
│  Size: ~600 MB (with metadata)      │
│  Clustering: None (unoptimized)     │
│  Purpose: Immutable audit trail     │
└──────────────┬──────────────────────┘
               │
               │ View Definition:
               │ SELECT *, ROW_NUMBER() OVER (
               │   PARTITION BY BELNR 
               │   ORDER BY _FIVETRAN_SYNCED DESC
               │ ) as rn
               │
               ▼
STAGE 4: CLEAN LAYER (Snowflake Views)
═══════════════════════════════════════
┌─────────────────────────────────────┐
│  FINANCE_RAW_V.BKPF_CLEAN (View)    │
│  Transformations:                   │
│  • Deduplication by BELNR           │
│  • Filter: rn = 1                   │
│  • Latest version only              │
│  • Type casting (if needed)         │
│  Purpose: Consumption-ready         │
└──────────────┬──────────────────────┘
               │
               │ Query by BI Tools:
               │ SELECT * FROM 
               │ FINANCE_RAW_V.BKPF_CLEAN
               │ WHERE BUDAT >= '2025-01-01'
               │
               ▼
STAGE 5: CONSUMPTION (Analytics/BI)
════════════════════════════════════════
┌─────────────────────────────────────┐
│  Data Consumers                     │
│  • Tableau / Power BI               │
│  • Custom SQL queries               │
│  • Snowflake ML models              │
│  • Export jobs                      │
└─────────────────────────────────────┘
```

### Data Lifecycle Management (Mermaid)

```mermaid
stateDiagram-v2
    [*] --> SourceCreated: New record in SAP HANA
    
    SourceCreated --> Detected: Batch scheduler runs
    Detected --> Extracted: Fivetran queries HANA
    Extracted --> InTransit: Data in Fivetran buffer
    InTransit --> Loaded: COPY INTO Snowflake
    
    Loaded --> Active: Available in FINANCE_RAW
    Active --> Deduplicated: View layer filters latest
    Deduplicated --> Consumed: BI tools query
    
    Active --> Updated: Source record modified
    Updated --> Detected
    
    Active --> SoftDeleted: Source record deleted
    SoftDeleted --> Archived: _FIVETRAN_DELETED = TRUE
    
    Consumed --> [*]: End of lifecycle
    Archived --> [*]: Retained per policy
```

---

## Security Architecture

### Security Layers (ASCII)

```
┌──────────────────────────────────────────────────────────────────┐
│                    SECURITY ARCHITECTURE                         │
│                      (Defense in Depth)                          │
└──────────────────────────────────────────────────────────────────┘

LAYER 1: NETWORK SECURITY
══════════════════════════
┌────────────────────────────────────────────────────────┐
│  SAP BTP Network Controls                              │
│  ┌──────────────────────────────────────────────┐     │
│  │ • IP Allowlisting (Fivetran IP ranges)       │     │
│  │ • TLS 1.2+ encryption mandatory              │     │
│  │ • Port 443 only (no plain HTTP)              │     │
│  │ • Connection timeout: 30 seconds             │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  Snowflake Network Controls                            │
│  ┌──────────────────────────────────────────────┐     │
│  │ • Network policies (optional)                │     │
│  │ • VPC endpoints (not used in this POC)       │     │
│  │ • TLS encryption for all connections         │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘

LAYER 2: AUTHENTICATION
═══════════════════════════
┌────────────────────────────────────────────────────────┐
│  SAP HANA Authentication                               │
│  ┌──────────────────────────────────────────────┐     │
│  │ User: FIVETRAN_HANA_USER                     │     │
│  │ Method: Database password authentication     │     │
│  │ Password Policy:                             │     │
│  │   • Minimum 12 characters                    │     │
│  │   • Complexity required (upper/lower/digit)  │     │
│  │   • Rotation: 90 days (recommended)          │     │
│  │ Session timeout: 24 hours                    │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  Snowflake Authentication                              │
│  ┌──────────────────────────────────────────────┐     │
│  │ User: FIVETRAN_USER                          │     │
│  │ Method: Username + Password                  │     │
│  │ MFA: Not enabled (service account)           │     │
│  │ Key-pair auth: Not used (can be upgraded)    │     │
│  │ Session timeout: 4 hours (default)           │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘

LAYER 3: AUTHORIZATION
══════════════════════════
┌────────────────────────────────────────────────────────┐
│  SAP HANA Privileges                                   │
│  ┌──────────────────────────────────────────────┐     │
│  │ FIVETRAN_HANA_USER:                          │     │
│  │   GRANT SELECT ON SCHEMA FINANCE             │     │
│  │   (Read-only access)                         │     │
│  │                                              │     │
│  │ Denied Privileges:                           │     │
│  │   • INSERT, UPDATE, DELETE                   │     │
│  │   • CREATE, ALTER, DROP                      │     │
│  │   • EXECUTE procedures                       │     │
│  │   • System privileges                        │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  Snowflake RBAC Model                                  │
│  ┌──────────────────────────────────────────────┐     │
│  │ Role Hierarchy:                              │     │
│  │   ACCOUNTADMIN (system admin)                │     │
│  │        └─ FIVETRAN_ROLE (limited scope)      │     │
│  │                                              │     │
│  │ FIVETRAN_ROLE Grants:                        │     │
│  │   • USAGE on FIVETRAN_WH_DEDICATED           │     │
│  │   • USAGE on FIVETRAN_DB                     │     │
│  │   • CREATE TABLE on FINANCE_RAW              │     │
│  │   • CREATE VIEW on FINANCE_RAW               │     │
│  │                                              │     │
│  │ No access to:                                │     │
│  │   • Other databases                          │     │
│  │   • System functions                         │     │
│  │   • Account management                       │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘

LAYER 4: DATA ENCRYPTION
════════════════════════════
┌────────────────────────────────────────────────────────┐
│  In Transit                                            │
│  ┌──────────────────────────────────────────────┐     │
│  │ SAP HANA → Fivetran:                         │     │
│  │   • TLS 1.2+ (mandatory)                     │     │
│  │   • Certificate validation                   │     │
│  │                                              │     │
│  │ Fivetran → Snowflake:                        │     │
│  │   • TLS 1.2+ (mandatory)                     │     │
│  │   • Snowflake-issued certificates            │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  At Rest                                               │
│  ┌──────────────────────────────────────────────┐     │
│  │ SAP HANA:                                    │     │
│  │   • Data volumes encrypted                   │     │
│  │   • Encryption: AES-256                      │     │
│  │   • Key management: SAP-managed              │     │
│  │                                              │     │
│  │ Snowflake:                                   │     │
│  │   • All data encrypted by default            │     │
│  │   • Encryption: AES-256                      │     │
│  │   • Key management: Snowflake-managed        │     │
│  │   • Key rotation: Automatic (30 days)        │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘

LAYER 5: AUDIT & MONITORING
════════════════════════════════
┌────────────────────────────────────────────────────────┐
│  Audit Trails                                          │
│  ┌──────────────────────────────────────────────┐     │
│  │ Fivetran Logs:                               │     │
│  │   • All sync activities logged               │     │
│  │   • Failed connection attempts               │     │
│  │   • Row counts per sync                      │     │
│  │   • Retention: 30 days                       │     │
│  │                                              │     │
│  │ Snowflake Query History:                     │     │
│  │   • All queries logged                       │     │
│  │   • User/Role attribution                    │     │
│  │   • Data access tracking                     │     │
│  │   • Retention: 365 days                      │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  Monitoring Alerts                                     │
│  ┌──────────────────────────────────────────────┐     │
│  │ • Failed sync attempts                       │     │
│  │ • Unusual data volumes                       │     │
│  │ • Authentication failures                    │     │
│  │ • Schema changes detected                    │     │
│  │ Notification: Email/Slack                    │     │
│  └──────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘
```

### Security Threat Model (Mermaid)

```mermaid
graph TD
    A[Threat: Unauthorized Data Access] -->|Mitigation| B[Read-Only User]
    A -->|Mitigation| C[IP Allowlisting]
    A -->|Mitigation| D[TLS Encryption]
    
    E[Threat: Credential Compromise] -->|Mitigation| F[Password Complexity]
    E -->|Mitigation| G[Password Rotation]
    E -->|Mitigation| H[Least Privilege]
    
    I[Threat: Data in Transit Interception] -->|Mitigation| J[TLS 1.2+]
    I -->|Mitigation| K[Certificate Validation]
    
    L[Threat: Unauthorized Modifications] -->|Mitigation| M[SELECT Only Grants]
    L -->|Mitigation| N[Immutable Raw Tables]
    
    O[Threat: Data Exfiltration] -->|Detection| P[Audit Logs]
    O -->|Detection| Q[Monitoring Alerts]
    
    style A fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style E fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style I fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style L fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style O fill:#ff6b6b,stroke:#c92a2a,color:#fff
    
    style B fill:#51cf66,stroke:#2f9e44,color:#fff
    style C fill:#51cf66,stroke:#2f9e44,color:#fff
    style D fill:#51cf66,stroke:#2f9e44,color:#fff
    style F fill:#51cf66,stroke:#2f9e44,color:#fff
    style G fill:#51cf66,stroke:#2f9e44,color:#fff
    style H fill:#51cf66,stroke:#2f9e44,color:#fff
    style J fill:#51cf66,stroke:#2f9e44,color:#fff
    style K fill:#51cf66,stroke:#2f9e44,color:#fff
    style M fill:#51cf66,stroke:#2f9e44,color:#fff
    style N fill:#51cf66,stroke:#2f9e44,color:#fff
    style P fill:#51cf66,stroke:#2f9e44,color:#fff
    style Q fill:#51cf66,stroke:#2f9e44,color:#fff
```

---

## Deployment Architecture

### Deployment Model (Mermaid)

```mermaid
graph TB
    subgraph Cloud["Multi-Cloud Deployment"]
        subgraph SAP["SAP BTP (Cloud Provider Agnostic)"]
            H1[HANA Cloud Instance]
            H2[BTP Cockpit]
        end
        
        subgraph FTV["Fivetran SaaS (AWS-hosted)"]
            F1[Control Plane]
            F2[Data Plane]
            F3[Metadata Store]
        end
        
        subgraph SF["Snowflake (Multi-Region)"]
            S1[Compute Layer]
            S2[Storage Layer]
            S3[Metadata Layer]
        end
    end
    
    H1 -->|Secure Tunnel| F2
    F2 -->|Snowflake Protocol| S1
    F1 -->|Orchestrate| F2
    F3 -->|Track State| F2
    S1 -->|Read/Write| S2
    S1 -->|Query| S3
    
    style SAP fill:#0066cc,stroke:#003d7a,color:#fff
    style FTV fill:#6200ea,stroke:#4a0099,color:#fff
    style SF fill:#29b5e8,stroke:#1a8cb8,color:#fff
```

### Infrastructure Overview (ASCII)

```
┌──────────────────────────────────────────────────────────────────┐
│                  DEPLOYMENT INFRASTRUCTURE                       │
└──────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ COMPONENT 1: SAP BTP (Source Infrastructure)                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Deployment Model: Cloud-Native SaaS                               │
│  ├─ Cloud Provider: AWS/Azure/GCP (abstracted by SAP)             │
│  ├─ Region: Customer-selected (e.g., eu-central-1)                │
│  ├─ High Availability: Multi-AZ deployment                        │
│  └─ Backup: Automated daily backups                               │
│                                                                    │
│  HANA Cloud Instance:                                              │
│  ├─ Size: Production (configurable)                               │
│  ├─ Memory: 32GB - 1TB+ (scalable)                                │
│  ├─ Storage: 120GB - 6TB+ (scalable)                              │
│  ├─ Compute: 2-32+ vCPUs                                           │
│  └─ SLA: 99.5% uptime                                              │
│                                                                    │
│  Estimated Cost (HANA Cloud):                                      │
│  └─ $0.90 - $3.00/hour (depends on size)                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ COMPONENT 2: Fivetran (Integration Infrastructure)                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Deployment Model: Fully-Managed SaaS                              │
│  ├─ Cloud Provider: AWS (Fivetran-managed)                        │
│  ├─ Region: Multi-region (closest to source/target)               │
│  ├─ High Availability: Built-in redundancy                        │
│  └─ Maintenance: Zero-downtime updates                             │
│                                                                    │
│  Infrastructure Details:                                           │
│  ├─ Connector Instances: Auto-scaling                             │
│  ├─ Buffer Storage: Ephemeral (in-memory + temp S3)               │
│  ├─ Metadata Store: PostgreSQL (managed)                          │
│  └─ Monitoring: Datadog/CloudWatch                                │
│                                                                    │
│  Pricing Model:                                                    │
│  ├─ Billing: Based on Monthly Active Rows (MAR)                   │
│  ├─ MAR = Unique rows modified/inserted per month                 │
│  └─ Estimated: $0.001 - $0.003 per MAR                            │
│                                                                    │
│  Example Cost Calculation:                                         │
│  └─ 10M rows synced monthly = $10,000 - $30,000/month             │
│      (varies by plan tier)                                         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ COMPONENT 3: Snowflake (Target Infrastructure)                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Deployment Model: Cloud-Native Data Platform                      │
│  ├─ Cloud Provider: AWS/Azure/GCP (customer choice)               │
│  ├─ Region: Customer-selected (e.g., us-east-1)                   │
│  ├─ High Availability: Cross-AZ by default                        │
│  └─ Disaster Recovery: Cross-region replication (optional)        │
│                                                                    │
│  Compute (Virtual Warehouse):                                      │
│  ┌──────────┬──────────┬──────────────┬──────────────────┐       │
│  │ Size     │ Servers  │ Memory       │ Credits/Hour     │       │
│  ├──────────┼──────────┼──────────────┼──────────────────┤       │
│  │ X-SMALL  │ 1        │ 8 GB         │ 1                │       │
│  │ SMALL    │ 2        │ 16 GB        │ 2                │       │
│  │ MEDIUM   │ 4        │ 32 GB        │ 4                │       │
│  │ LARGE    │ 8        │ 64 GB        │ 8                │       │
│  └──────────┴──────────┴──────────────┴──────────────────┘       │
│                                                                    │
│  Current Configuration:                                            │
│  └─ SMALL warehouse × 24 hours/day = 48 credits/day               │
│     = 1,440 credits/month (always-on)                             │
│                                                                    │
│  Storage:                                                          │
│  ├─ Type: Columnar, compressed                                    │
│  ├─ Pricing: $23 - $40 per TB/month (region-dependent)            │
│  ├─ Estimated for POC: 10GB = $0.23 - $0.40/month                 │
│  └─ Time Travel: 1 day (default)                                  │
│                                                                    │
│  Estimated Monthly Cost:                                           │
│  ├─ Compute: 1,440 credits × $2-4/credit = $2,880 - $5,760       │
│  ├─ Storage: ~$0.40 (POC volume)                                  │
│  └─ Total: ~$2,900 - $5,800/month                                 │
│                                                                    │
│  Cost Optimization Options:                                        │
│  ├─ Auto-suspend during off-hours → 50% savings                   │
│  ├─ Scale down to X-SMALL → 50% savings                           │
│  └─ Combination → 75% savings ($725-1,450/month)                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Disaster Recovery Architecture (Mermaid)

```mermaid
graph LR
    subgraph Primary["Primary Region"]
        P1[SAP HANA Primary]
        P2[Fivetran Connector]
        P3[Snowflake Primary]
    end
    
    subgraph DR["DR/Backup Strategy"]
        B1[HANA Backups<br/>Daily]
        B2[Fivetran Metadata<br/>Auto-Backup]
        B3[Snowflake Time Travel<br/>1-90 days]
    end
    
    P1 -.->|Backup| B1
    P2 -.->|Replicate| B2
    P3 -.->|Retain| B3
    
    B1 -.->|Restore<br/>RTO: 4 hours| P1
    B2 -.->|Failover<br/>RTO: 1 hour| P2
    B3 -.->|Recover<br/>RTO: Minutes| P3
    
    style Primary fill:#e3f2fd,stroke:#1976d2
    style DR fill:#fff3e0,stroke:#f57c00
```

---

## Performance Characteristics

### Performance Metrics (ASCII)

```
┌──────────────────────────────────────────────────────────────────┐
│              PERFORMANCE BENCHMARKS (POC Environment)            │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ INITIAL SYNC PERFORMANCE                                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Table: BKPF (2M rows, 500MB)                               │
│  ┌───────────────────┬─────────────────────────────┐       │
│  │ Warehouse Size    │ Duration        │ Cost      │       │
│  ├───────────────────┼─────────────────┼───────────┤       │
│  │ X-SMALL          │ ~45 minutes     │ $1.50     │       │
│  │ SMALL            │ ~25 minutes     │ $1.67     │       │
│  │ MEDIUM           │ ~15 minutes     │ $2.00     │       │
│  └───────────────────┴─────────────────┴───────────┘       │
│                                                             │
│  Recommended: SMALL for balance of speed and cost           │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ INCREMENTAL SYNC PERFORMANCE (Daily Batch)                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Typical Daily Changes: 50K rows (~12MB)                    │
│  ┌───────────────────────────────────────────────────┐     │
│  │ Phase                │ Duration    │ Description   │     │
│  ├──────────────────────┼─────────────┼───────────────┤     │
│  │ Query Source         │ 2 min       │ HANA query    │     │
│  │ Transfer to Fivetran │ 3 min       │ Network I/O   │     │
│  │ Load to Snowflake    │ 5 min       │ COPY INTO     │     │
│  │ Total                │ ~10 min     │ End-to-end    │     │
│  └──────────────────────┴─────────────┴───────────────┘     │
│                                                             │
│  Data Freshness: < 15 minutes from source update            │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ QUERY PERFORMANCE (Snowflake)                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Query Type: Full table scan (2M rows)                      │
│  ┌───────────────────┬─────────────────────────────┐       │
│  │ Warehouse Size    │ Query Time   │ Credits Used │       │
│  ├───────────────────┼──────────────┼──────────────┤       │
│  │ X-SMALL          │ 8 seconds    │ 0.002        │       │
│  │ SMALL            │ 4 seconds    │ 0.002        │       │
│  │ MEDIUM           │ 2 seconds    │ 0.002        │       │
│  └───────────────────┴──────────────┴──────────────┘       │
│                                                             │
│  Query Type: Filtered query (10K result rows)               │
│  └─ All sizes: < 1 second (benefit of result cache)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ THROUGHPUT METRICS                                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Fivetran Connector:                                        │
│  ├─ Extraction Rate: ~50K rows/minute from HANA            │
│  ├─ Network Throughput: ~10 MB/second                      │
│  └─ Parallelism: 1 table at a time (sequential)            │
│                                                             │
│  Snowflake Loading:                                         │
│  ├─ Load Rate: ~100K rows/minute (SMALL warehouse)         │
│  ├─ Compression Ratio: ~3:1 (500MB → 170MB stored)         │
│  └─ Parallelism: Auto-scaling (up to 10 files parallel)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Scalability Considerations

### Scaling Strategy (Mermaid)

```mermaid
graph TD
    A[Current: POC Scale] -->|Scale Dimension| B[Data Volume]
    A -->|Scale Dimension| C[Sync Frequency]
    A -->|Scale Dimension| D[Table Count]
    
    B -->|10x rows| B1[Increase Warehouse Size<br/>SMALL → MEDIUM]
    B -->|100x rows| B2[Add Clustering Keys<br/>Partition Large Tables]
    
    C -->|Hourly| C1[Same Infrastructure<br/>More Frequent Syncs]
    C -->|Real-time CDC| C2[Enable Log-Based CDC<br/>Stream Processing]
    
    D -->|10-50 tables| D1[Same Setup<br/>Sequential Loading]
    D -->|50+ tables| D2[Parallel Connectors<br/>Table Groups]
    
    style A fill:#4CAF50,stroke:#2E7D32,color:#fff
    style B1 fill:#2196F3,stroke:#1565C0,color:#fff
    style B2 fill:#2196F3,stroke:#1565C0,color:#fff
    style C1 fill:#FF9800,stroke:#E65100,color:#fff
    style C2 fill:#FF9800,stroke:#E65100,color:#fff
    style D1 fill:#9C27B0,stroke:#6A1B9A,color:#fff
    style D2 fill:#9C27B0,stroke:#6A1B9A,color:#fff
```

---

## Appendix: Architecture Decisions

### Key Architecture Decision Records (ASCII)

```
┌──────────────────────────────────────────────────────────────────┐
│              ARCHITECTURE DECISION RECORDS (ADR)                 │
└──────────────────────────────────────────────────────────────────┘

ADR-001: Always-On Warehouse
════════════════════════════════
Decision: Configure Snowflake warehouse with Auto Suspend = 0

Rationale:
  • Eliminates cold-start delays (5-15 seconds per sync)
  • Predictable performance for scheduled batch jobs
  • Acceptable cost trade-off for POC phase

Consequences:
  • Increased monthly cost (~$2,900 vs ~$1,500 with auto-suspend)
  • Simplified operational model (no warm-up logic needed)
  • Can be revised based on cost analysis post-POC

───────────────────────────────────────────────────────────────────

ADR-002: Batch vs Real-Time CDC
═══════════════════════════════════
Decision: Start with daily batch sync, not real-time CDC

Rationale:
  • POC requirements allow 24-hour data freshness
  • Batch mode simpler to implement and troubleshoot
  • Lower cost for initial validation phase
  • Can upgrade to CDC without architecture changes

Consequences:
  • Data latency: 24 hours max
  • Simpler failure recovery (retry entire batch)
  • Lower network utilization

───────────────────────────────────────────────────────────────────

ADR-003: Raw + Views Pattern
═════════════════════════════════
Decision: Immutable raw tables + optional clean views

Rationale:
  • Preserves audit trail (Fivetran metadata intact)
  • Enables schema evolution without data loss
  • Separates concerns (load vs transform)
  • Industry best practice (ELT pattern)

Consequences:
  • Slightly higher storage cost (raw + views)
  • Clear data lineage for compliance
  • Easier troubleshooting (raw data always available)

───────────────────────────────────────────────────────────────────

ADR-004: Read-Only Source Access
════════════════════════════════════
Decision: CREATE dedicated read-only user for Fivetran

Rationale:
  • Principle of least privilege (security)
  • Prevents accidental data modifications
  • Easier audit trail (single-purpose user)

Consequences:
  • Additional user management overhead (password rotation)
  • No risk of write operations from integration
  • Clear separation of duties

───────────────────────────────────────────────────────────────────

ADR-005: IP Allowlisting (Optional)
═══════════════════════════════════════
Decision: Implement if organizationally required

Rationale:
  • Depends on corporate security policy
  • Adds defense-in-depth layer
  • Fivetran provides static IP ranges

Consequences:
  • Network team coordination required
  • Potential delays during setup
  • Increased operational complexity (IP changes require updates)

───────────────────────────────────────────────────────────────────
```

---

**Document Version:** 1.0  
**Last Updated:** October 2025  
**Review Cycle:** Quarterly  
**Owner:** Research Student Implementation Team  
**Classification:** Internal - Project Documentation

