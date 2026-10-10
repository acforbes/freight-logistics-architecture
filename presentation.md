---
marp: true
theme: gaia
_class: lead
paginate: true
backgroundColor: #f4f6f9
color: #2c3e50
style: |
  section {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    padding: 40px;
  }
  h1 { color: #0078d4; }
  h2 { color: #107c41; }
  footer { font-size: 0.5em; color: #555; }
  .columns {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 20px;
  }
---

# Freight Logistics Architecture
### Integrated Cloud Infrastructure & Secure Microservice Gateways
**Technical Presentation**
Architectural Optimization & Systems Modernization

---

## 1. System Overview & Core Requirements

* **Domain:** Mission-critical freight logistics web application.
    * Decoupled frontend static asset delivery from stateful compute layers.
    * Zero-downtime backend execution via warm App Service deployment slots.
    * Isolated microservice gateway via APIM for third-party integrations and testing.
* **Hosting Model:** Fully managed, cloud-native Azure ecosystem.

---

## 2. Legacy Topology & Production Pain Points

* **High-Risk Promotions:** Had only a Test and Production environment. Slot swaps were executed *between* Test and Production directly, with no Staging layer for validation.
* **Monolithic Complications:** Combined UI and API deployment package. 3rd-party API calls were fired straight from the application runtime, stalling user interfaces when external systems lagged.
* **Severe Memory & Database Pressure:**
    * **No Caching Layer:** Typeaheads relied on a massive, preloaded city-list object held entirely in API server memory.
    * **Storage Exhaustion:** Multi-city GPS coordinate paths sat inside a bloated database table. No partitioning on `AuditLogs` resulted in query timeouts.
    * **No Storage Offloading:** No document management engine or file storage was utilized.

---

## 3. Decoupled Multi-Environment Topology

 ```mermaid
graph TD
    subgraph Frontend Layer [Direct CDN Publishing]
        SWA_Test[Angular Test SWA]
        SWA_Stag[Angular Staging SWA]
        SWA_Prod[Angular Prod SWA]
    end

    subgraph Backend App Service Plan [Sequential Swap Chain]
        SWA_Test -->|Direct UI-to-API| Slot_Test[[Test Slot]]
        SWA_Stag -->|Direct UI-to-API| Slot_Stag[[Staging Slot]]
        SWA_Prod -->|Direct UI-to-API| Slot_Prod[[Production Slot]]
        
        Slot_Test <-->|1. PROMOTION SWAP| Slot_Stag
        Slot_Stag <-->|2. DEPLOYMENT SWAP| Slot_Prod
    end

    subgraph Data & Storage Isolation Tier
        Slot_Test --> DB_Test[(Azure SQL: Test)]
        Slot_Test --> Blob_Test[(Blob Storage: Test Container)]
        
        Slot_Stag -->|Sticky Setting| DB_Prod_Main[(Azure SQL: Prod)]
        Slot_Prod -->|Sticky Setting| DB_Prod_Main
        
        Slot_Stag -->|Sticky Setting| Blob_Prod[(Blob Storage: Prod Container)]
        Slot_Prod -->|Sticky Setting| Blob_Prod
    end
```

---

## 4. End-to-End Enterprise Identity & Security

* **Frontend Authentication:** The Angular UI integrates **MSAL (Microsoft Authentication Library)** to handle secure user authentication directly via Microsoft login.
* **The Token Lifecycle:**
    * MSAL acquires and automatically caches cryptographic JWT Access Tokens in the browser.
    * An internal HTTP interceptor attaches the token to all direct backend API requests.
* **Backend Authorization:** The `.NET Web API` validates the incoming Bearer token claims against **Azure Entra ID** to enforce granular Role-Based Access Control (RBAC).

---

## 5. Component Communication & Dual-Facade Architecture

```mermaid
graph TD
    Client[Angular Frontend / Azure SWA] -->|Direct HTTPS API Calls with MSAL Token| AppService[Web API: Azure App Services / Slots]
    
    subgraph Storage & Caching Layer
        SQL[(Azure SQL Database: Test / Prod)]
        Blob[(Azure Blob Storage: Test / Prod)]
        Redis[(Shared Azure Cache for Redis)]
        
        Slot_Test[[Test Slot]] --> DB_Test
        Slot_Test --> Blob_Test
        Slot_Test -->|Logical Naming Prefix: test:*| Redis
        
        Slot_Stag[[Staging Slot]] --> DB_Prod
        Slot_Stag --> Blob_Prod
        Slot_Stag -->|Logical Naming Prefix: prod:* - Sticky| Redis
        
        Slot_Prod[[Production Slot]] --> DB_Prod
        Slot_Prod --> Blob_Prod
        Slot_Prod -->|Logical Naming Prefix: prod:*| Redis
    end

    subgraph Test Gateway Facade
        Slot_Test[[Test Slot]] --> APIM_Test[APIM: Test]
        PM_Test[Postman: Test Suites] --> APIM_Test
        APIM_Test --> AzFunc_Test[Azure Functions: Test] --> Ext_Test[3rd Party APIs: Test]
    end

    subgraph Production Gateway Facade
        Slot_Stag[[Staging Slot]] -->|Sticky Connection| APIM_Prod[APIM: Prod]
        Slot_Prod[[Production Slot]] --> APIM_Prod
        PM_Prod[Postman: Live Checks] --> APIM_Prod
        APIM_Prod --> AzFunc_Prod[Azure Functions: Prod] --> Ext_Prod[3rd Party APIs: Prod]
    end
```

---

## 6. Compute State & Schema Continuity

* **Frontend Delivery:** Angular static web assets are directly published to their respective Azure SWA instances. No slot switches are performed on the CDN edge.
* **Backend Zero-Downtime Swaps:** API code is published to the **Test Slot** and fully warmed up via the root runtime path, followed by sequential slot swaps to **Staging** and then to **Production** with zero traffic interruption.
* **Database Drift Logic:** Employing an **Expand and Contract pattern**. Schema migrations are pushed *before* the slot swap occurs using backward-compatible, non-breaking mutations.

---

## 7. Database Tuning & Cloud Cost Controls

To prevent unbounded data growth and optimize compute costs within Azure SQL, the database tuning strategy deploys two lifecycle architectures:

* **Horizontal Table Partitioning:**
    * High-volume `AuditLogs` are partitioned by **date ranges**.
    * Query engines leverage **partition pruning** to completely bypass irrelevant data blocks.
    * Result: Sub-second user access to historic compliance trails.
* **Time-Bounded Log Retention:**
    * System telemetry and API `UsageLogs` are capped on a rolling retention window.
    * An automated background process continuously purges aged records.
    * Result: Hard-caps database file size growth and flattens Azure storage expenses.

---

## 8. Comprehensive 4-Tier Caching Topology

To aggressively maximize performance, caching layers are partitioned by data change frequency:

| Tier | Caching Engine | Targeted Data Payload | Lifespan / Isolation Strategy |
| :--- | :--- | :--- | :--- |
| **Tier 1** | `.NET ResponseCache` | Long-term Static Lookups | Completely local in-memory; zero network hops. |
| **Tier 2** | `Azure Cache for Redis` | High-frequency Typeahead Search | Prefixed key spaces (`prod:typeahead:*`). |
| **Tier 3** | Hybrid Response/Blob | Multi-City Route GPS Data | 24-Hour cache fallback to JSON objects. |
| **Tier 4** | Azure Blob Storage | Historical Document Archive | Persisted behind authenticated API proxies. |

---

## 9. Tiered Route Resolution Pipeline

When a user requests multi-city route GPS coordinates, the Web API processes the request through a strict **3-tier failover lifecycle**:

```mermaid
graph TD
    Req[Incoming Route Request] --> T1{.NET ResponseCache}
    T1 -->|Hit: Valid < 24 Hours| Return[Return JSON Payload]
    
    T1 -->|Miss / Expired| T2{Azure Blob Storage}
    T2 -->|Hit: File Found| Warm[.NET Re-caches + Return]
    
    T2 -->|Miss: File Not Found| T3[Azure Maps API Compute]
    T3 --> Save[Write JSON to Blob Storage]
    Save --> Warm
```

---

## 10. Q&A and Engineering Appendix

* Open for peer review regarding schema management, token validation lifetimes, and caching boundaries.
* **Deep dives available in repository:**
    * Appendix A: System Component Trade-off Matrix
    * Appendix B: Dynamic Slot Swap & State Boundary Rules
    * Appendix C: Cache Key & Storage Lifecycle Strategy

---
