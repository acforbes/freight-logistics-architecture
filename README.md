# Freight Logistics System Architecture

An enterprise cloud-native architecture for a mission-critical freight logistics web platform. This system decouples static client delivery from stateful compute layers, utilizes a strict 4-tier caching model to minimize database load, secures its application boundaries using MSAL with Microsoft login, and encapsulates external serverless microservices behind an internal API Management facade.

## 🚀 Architectural Pillars

* **Zero-Downtime Infrastructure:** Blue/Green style code promotions via Azure App Service Deployment Slots.
* **Enterprise Identity Federation:** Front-end access secured via MSAL (Microsoft Authentication Library) bound to Azure Entra ID RBAC profiles.
* **Strict State Isolation:** Unified physical environment boundaries for Test/Dev, with logical key-space isolation for shared caching tiers.
* **Cost-Controlled Compute:** A 3-tier hybrid route resolution pipeline designed to shield external computing budgets (Azure Maps API).
* **Decoupled Delivery Lifecycles:** Direct CDN asset publishing for front-end layers, fully isolated from backend binary modifications.
* **Optimized Data Lifecycles:** Horizontal date-range partitioning and rolling data retention to enforce strict budget caps on Azure SQL.

---

## 🔄 Architectural Evolution: System Modernization Blueprint

To fully understand the current architecture, this section outlines the technical transformations implemented to resolve critical architectural debt from the legacy deployment.

### System Comparison Profile

| Architectural Vector | Legacy Topology ("Before") | Modernized Architecture ("After") | Engineering & Financial Impact |
| :--- | :--- | :--- | :--- |
| **Release Confidence** | Direct Slot Swap from Test ➔ Prod. Zero safety staging buffer. | **Isolated Test bed** + **Staging-to-Prod Slot Swap** lifecycle. | Code is smoke-tested against live environment strings prior to routing production traffic. |
| **UI/API Lifecycle** | Monolithic combined UI/API container deployment package. | **Decoupled:** Direct CDN SWA publishing + isolated API App Service. | UI design polishes bypass backend restarts; assets scale globally on the CDN edge instantly. |
| **Perimeter Traffic** | Shared domain space (Zero CORS overhead required). | Cross-Origin boundaries isolated via target App Service policies. | Employs explicit browser-level preflight headers and origin restrictions (`Allow-Credentials`). |
| **Integration Boundary** | 3rd-party APIs invoked straight from Web API threads, causing UI thread stalls. | **Internal APIM Facade** + Serverless **Azure Functions** proxy layer. | Fragile external dependencies are sandboxed; failures or API lag never impact core thread loops. |
| **Search Performance** | Massive, resource-heavy city list array preloaded in server API RAM memory. | Dedicated distributed **Azure Cache for Redis** index using `prod:typeahead:*` tokens. | Drastically reduces Web API RAM footprints while providing sub-10ms autocompletion. |
| **Route Performance** | Unbounded, heavy relational SQL tables tracking multi-city GPS paths. | **3-Tier Fallback Loop:** `.NET ResponseCache` ➔ **Azure Blob Storage JSON** ➔ Azure Maps. | Shrinks database engine size; reduces Azure Maps compute expenses by caching static assets as immutable objects. |
| **Database Resiliency** | No data partitioning or archival rules; query lookups on `AuditLogs` timed out. | Horizontal **Date-Range Partitioning** (monthly) + rolling 90-day diagnostic retention. | Query engines use **partition pruning** for sub-second audit returns; hard-caps database file size growth. |

---

## 🗺️ System Topology

```mermaid
graph TD
    subgraph Backend App Service Plan [Sequential Swap Chain]
        SWA_Test -->|Direct HTTPS / CORS| Slot_Test[[Test Slot]]
        SWA_Stag -->|Direct HTTPS / CORS| Slot_Stag[[Staging Slot]]
        SWA_Prod -->|Direct HTTPS / CORS| Slot_Prod[[Production Slot]]
        
        Slot_Test <-->|1. Test to Staging Swap| Slot_Stag
        Slot_Stag <-->|2. Staging to Prod Swap| Slot_Prod
    end

    subgraph Data & Storage Isolation Tier
        Slot_Test --> DB_Test[(Azure SQL: Test)]
        Slot_Test --> Blob_Test[(Blob Storage: Test)]
        Slot_Stag -->|Sticky Setting| DB_Prod[(Azure SQL: Production)]
        Slot_Prod -->|Sticky Setting| DB_Prod
        Slot_Stag -->|Sticky Setting| Blob_Prod[(Blob Storage: Production)]
        Slot_Prod -->|Sticky Setting| Blob_Prod
        Slot_Prod -->|Cache Aside| Redis[(Azure Cache for Redis)]
    end

    subgraph Test Microservice Facade [Isolated Testing Perimeter]
        Slot_Test -->|Outbound Webhooks / MTLS| APIM_Test[APIM Gateway: Test]
        PM_Test[Postman Client: Test Suites] -->|Direct Request Validation| APIM_Test
        APIM_Test <-->|Claims Validation| Entra_Test[Azure Entra ID: Test]
        APIM_Test -->|Serverless Trigger| AzFunc_Test[Azure Functions: Test]
        AzFunc_Test -->|Simulated Data Loop| External_Test[3rd Party Carrier APIs: Test Endpoints]
    end

    subgraph Production Microservice Facade [Live Integration Perimeter]
        Slot_Stag -->|Outbound Webhooks / MTLS - Sticky| APIM_Prod[APIM Gateway: Production]
        Slot_Prod -->|Outbound Webhooks / MTLS| APIM_Prod
        PM_Prod[Postman Client: Live Sanity Verifications] -->|Direct Request Validation| APIM_Prod
        APIM_Prod <-->|Claims Validation| Entra_Prod[Azure Entra ID: Production]
        APIM_Prod -->|Serverless Trigger| AzFunc_Prod[Azure Functions: Production]
        AzFunc_Prod -->|Live Logistics Telemetry| External_Prod[3rd Party Carrier APIs: Production Core]
    end
```

---

## 🔒 Enterprise Identity & Secure Token Flow (MSAL + Entra ID)

The application implements a zero-trust identity architecture mapping frontend presentation directly to backend compute capabilities:

```mermaid
graph TD
    User[User / Dispatcher] -->|1. Interactive Login| MSAL[Angular SWA: MSAL Layer]
    MSAL -->|2. Redirect Auth| Microsoft[Microsoft Identity Platform]
    Microsoft -->|3. Issue JWT Access Token| MSAL
    
    MSAL -->|4. Bearer Token in Request Header| WebAPI[Web API: Azure App Service]
    WebAPI <-->|5. Cryptographic Claim Verification| Entra[Azure Entra ID]
```

### Authentication & Authorization Details
1. **Client-Side Guarding (Angular):** Routes within the Angular app are protected using native MSAL Guards. Unauthenticated users are automatically redirected to the organizational Microsoft sign-in page.
2. **The Interceptor Pattern:** An MSAL Interceptor maps your target backend API endpoints. It handles silent token acquisition and background token renewal, ensuring users are never interrupted during prolonged logistics dispatch sessions.
3. **API Perimeter Validation:** The .NET Web API extracts the claims array from the decrypted token payload to verify organizational tenant parameters and user-specific roles before allowing operations on core database or storage entities.

---

## 🛠️ Environmental Framework & Release Strategy

### 1. Frontend Promotion (Azure Static Web Apps)
The Angular application compiles against environment-specific profile variables (`environment.prod.ts`). Compiled assets are published **directly** to their respective environment containers. Front-end visual deployments are fully insulated from backend container recycling.

### ### 🌐 Cross-Origin Resource Sharing (CORS) Enforcement
Because the UI assets and API binaries exist on fundamentally decoupled domain endpoints, a strict CORS matrix is enforced at the App Service tier:
* **Preflight (OPTIONS) Resolution:** The App Service is configured to intercept and validate cross-origin preflight requests before execution.
* **Domain Restrictions:** Whitelists are configured natively in Azure on a per-slot basis, mapping the Test, Staging, and Production SWA domain locations respectively. Wildcards (`*`) are strictly prohibited.
* **Credentials Support:** `Access-Control-Allow-Credentials` is toggled on to allow safe cryptographic transmissions of bearer tokens derived from the MSAL pipeline.

### 2. Backend Promotion (Sequential App Service Slot Chain)
The .NET Web API utilizes a multi-step execution swap strategy across the **Test**, **Staging**, and **Production** slots inside a unified App Service Plan to enforce an immutable chain of custody:

* **Step 1: The Staging Promotion:** The latest code is manually published and validated in the `Test Slot`. To promote it, a swap is executed between `Test` and `Staging`. The binaries shift, and the code is smoke-tested against live production infrastructure dependencies in the isolated `Staging Slot`.
* **Step 2: The Production Deployment:** A second swap is executed between `Staging` and `Production`. This brings the newly verified code live to users instantly with zero downtime.
* **The Instant Rollback Safety Net:** Following the final swap, the `Staging Slot` dynamically retains the *previous* live production binary. If an anomaly surfaces in production, a simple reverse swap instantly restores the stable environment state, guaranteeing a near-zero Recovery Time Objective (RTO).

### 3. Database Schema Continuity (Azure SQL)
To prevent runtime exceptions during slot swaps, this repository mandates the **Expand and Contract (Parallel Change) Pattern**:
1. **Expand:** Schema migrations (adding nullable fields or new lookup tables) are executed manually on the Production Database *prior to a swap*.
2. **Swap:** The slot swap is executed. Both old and new API binaries simultaneously interact with the database safely.
3. **Contract:** Legacy fields/columns are removed after the environment completely stabilizes.

---

## 💾 Database Optimization & Lifecycle Management (Azure SQL)

To prevent unbounded database size growth and maintain rapid query performance as historical freight data accumulates, Azure SQL employs two lifecycle management strategies:

### 1. Horizontal Partitioning for Audit Trails
* **The Problem:** The `AuditLogs` table logs every structural change (e.g., dispatch modifications, user overrides, financial updates), causing it to grow by millions of rows rapidly.
* **The Solution:** The table is **partitioned by date ranges** (e.g., monthly chunks) utilizing Azure SQL Partition Functions and Schemes. 
* **The Benefit:** When a dispatcher requests an audit trail for a specific date range, the SQL engine executes **partition pruning**—completely ignoring irrelevant months. This keeps index sizes small, optimizes RAM usage, and maintains sub-second query execution times.

### 2. Time-Bounded Log Retention
* **The Problem:** System telemetry, application errors, and third-party API usage logs (`ApplicationLogs`) grow aggressively but lose their clinical value after a few months.
* **The Solution:** A **date-range retention period** (e.g., a rolling 90-day window) is strictly enforced. 
* **The Benefit:** An automated, low-priority cleanup process continuously purges records older than the retention boundary. This hard-caps the database file footprint, prevents data storage costs from spiraling, and ensures database backups and restoration times remain lean.

---

## ⚡ Data Volatility & Caching Lifecycle

The system enforces a **4-tier caching strategy** optimized around data longevity:

| Tier | Engine | Target Data Payload | Invalidation / Structural Strategy |
| :--- | :--- | :--- | :--- |
| **Tier 1** | `.NET ResponseCache` | Static System Lookups | Hard-capped local memory allocation; zero network I/O overhead. |
| **Tier 2** | `Azure Cache for Redis` | Dropdown Typeaheads | Shared instance isolated logically via key prefixing (`prod:typeahead:*` vs. `test:typeahead:*`). |
| **Tier 3** | Hybrid Cache-Aside / Blob | Multi-City Route GPS Coordinates | 24-hour `.NET ResponseCache` expiration backed by persistent JSON file writes on Azure Blob Storage. |
| **Tier 4** | `Azure Blob Storage` | Invoices / Bills of Lading | Cold, append-only document container storage. |

### Multi-City Route Resolution Fallback Flow
1. **Read Tier 1:** Check local `.NET ResponseCache` (Valid for 24 hours).
2. **Read Tier 2:** If missed, fetch the standardized object string from **Azure Blob Storage** based on a deterministic route waypoint hash naming convention (`route_[hash].json`). Re-populate Tier 1.
3. **Compute Tier 3:** If the file is not found, invoke **Azure Maps API**. Stream the resulting data payload back to the client, serialize it immediately to Blob Storage for subsequent runs, and warm up the cache layers.

---

## 🔒 Facade Security & Testability Blueprint

### Azure API Management (APIM) Positioning
Instead of facing the public internet, APIM is restricted to serving as an internal integration wall. 
* **Ingress Restriction:** Only authenticated requests from the primary Web API or authenticated testing infrastructure can cross the gateway boundary.
* **Token Hardening:** Entra ID issues strict Role-Based Access Control (RBAC) claims. APIM cryptographically validates these JWT signatures at the edge to shelter downstream Azure Functions from bad requests.

### Isolated Testing Engine (Postman)
Because APIM completely wraps all external third-party interactions, developers can validate microservice behaviors without firing up the Angular UI. Targeting APIM directly via **Postman** allows for the isolated testing of rate-limiting, error handling, webhook response mocking, and Entra ID scope parameters.
