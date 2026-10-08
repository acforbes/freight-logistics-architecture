# Appendix A: Architectural Trade-off Matrix

This reference guide defends our technical selection for the Freight Logistics System against common alternatives explored in industrial and academic systems engineering.

| Architectural Decision | Chosen Approach | Explored Alternatives | Trade-offs & Engineering Justification |
| :--- | :--- | :--- | :--- |
| **Identity Delegation Architecture** | **MSAL for Angular + Backend Azure Entra ID Validation** | Custom OAuth2 Server / Auth0 Third-Party Middleware | **Pros:** Pure zero-trust token exchange interface. Eliminates client-side security data persistence risks. Relies entirely on Microsoft’s cryptographically hardened token lifecycle. Integrates natively with institutional single sign-on (SSO).<br>**Cons:** Forces hard dependency on Azure Identity availability; token caching mechanics must be thoroughly handled via local interceptor overrides. |
| **Database Tuning Framework** | **Horizontal Partitioning + Time-Bounded Retention Caps** | Unbounded standard tables with indexing / Offloading to data lakes | **Pros:** Keeps the relational core running at high operational efficiencies indefinitely. Significantly lowers Azure SQL compute and storage tiers by continuously shrinking active indexing structures.<br>**Cons:** Demands administrative execution policies for partition functions; history beyond retention boundary is permanently pruned. |
| **APIM Gateway Positioning** | **Internal Subsystem Facade:**<br>Web API ➔ APIM ➔ Azure Functions ➔ Third-Party APIs | Exposing APIM globally as the public ingress point for the Angular SWA frontend. | **Pros:** Massively simplifies client-side frontend request piping. By reserving APIM strictly as an encapsulation barrier for external integrations and Azure Functions, we reduce latency for primary UI interactions while locking down expensive third-party webhooks behind Entra ID security scopes. Isolated verification is trivially achievable via Postman toolchains.<br>**Cons:** Direct UI-to-API requests bypass APIM policies (like global throttling metrics), which must instead be managed at the App Service tier. |
| **Frontend vs. Backend Delivery Split** | **Direct SWA Publishing for UI** + **Deployment Slots for API** | Running unified frontend/backend builds inside a shared App Service container instance. | **Pros:** Exceptional isolation boundaries. Frontend updates (UI adjustments, CSS updates, asset swapping) bypass backend slot recycling completely. Static web assets are aggressively cached across the global CDN edge immediately upon direct publish.<br>**Cons:** Requires explicit CORS configuration inside Azure App Services to securely allow decoupled UI interaction points across environments. |
| **Code Promotion Strategy** | **Azure App Service Deployment Slots (Staging ➔ Prod)** | Multi-instance VM redeployments / Container tag updates | **Pros:** Fully eliminates "cold start" latency spikes for logistics dispatchers because code binaries are entirely initialized before traffic switches. Instant rollbacks if production anomalies surface.<br>**Cons:** Demands backward-compatible code strategies during database schema shifts. |
| **Database Migration Model** | **Expand & Contract Parallel Migration** | Complete In-Place Overwrites (Down-time Migrations) | **Pros:** Guarantees structural zero-downtime execution. The production database is evolved to accept both old and new code layers seamlessly during a slot swap.<br>**Cons:** Requires writing temporary nullable fields and data copy scripts for column updates. |
| **Perimeter Access Control** | **Azure Native App Service CORS Policies** | Application-level middleware custom filtering (`app.UseCors()`) | **Pros:** Offloads preflight evaluation computations entirely from the Kestrel/C# runtime thread pool directly onto Azure's frontend proxy engine. Minimizes compute noise on worker nodes.<br>**Cons:** Requires maintaining environmental configuration arrays across deployment slots inside the Azure Infrastructure-as-Code layer. |

## Detailed Cache Performance Rationale

### 1. In-Memory Static Lookups (`ResponseCache`)
Data like country lists, carrier transport types, or fixed logistics status IDs (e.g., `Status 40 = Dispatched`) change rarely. Fetching this over the network from Redis on every single request is an anti-pattern. By hard-capping these in local `.NET ResponseCache`, we handle lookups in microseconds with zero network I/O.

### 2. Typeahead Distributed Dropdowns (`Redis`)
Typeahead dropdown arrays (e.g., looking up `Chicago` vs. `Chico` from thousands of available shipping depots) require sub-10ms response times while a user typing is actively triggering API requests. 
* We push these heavy string arrays into Redis using the `prod:typeahead:*` naming pattern. 
* This keeps the search blazingly fast, allows all scaled instances of the App Service to share the exact same autocompletion index, and protects the relational Azure SQL engine from being hammered by partial string queries like `LIKE 'Chi%'`.

## Isolated Testability Framework (APIM & Postman)

By segregating Azure API Management (APIM) away from raw UI traffic, the system gains a highly predictable, repeatable testing surface area.

### 1. Integration Isolation
When developers or automated scripts run test suites in **Postman**, they target the APIM endpoint directly. Because APIM mandates **Azure Entra ID security validation and token mapping**, Postman simulates realistic token payloads. This validates permissions, policy transformations, and serverless Azure Function endpoints without spinning up or interacting with the Angular user interface.

### 2. Safeguarding Serverless Compute Boundaries
Azure Functions execute complex translation algorithms on incoming freight carrier telemetry. Reserving APIM to act as the direct supervisor of these Functions means we can configure strict rate-limiting, IP-whitelisting, and contract checking at the gateway edge. If an external carrier API goes down or misbehaves during testing, it is safely isolated within the APIM/Function tier, keeping the core Web API container completely stable.
