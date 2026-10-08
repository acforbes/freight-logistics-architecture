# Appendix A: Architectural Trade-off Matrix

This reference guide defends our technical selection for the Freight Logistics System against common alternatives explored in industrial and academic systems engineering.

| Architectural Decision | Chosen Approach | Explored Alternatives | Trade-offs & Engineering Justification |
| :--- | :--- | :--- | :--- |
| **APIM Gateway Positioning** | **Internal Subsystem Facade:**<br>Web API ➔ APIM ➔ Azure Functions ➔ Third-Party APIs | Exposing APIM globally as the public ingress point for the Angular SWA frontend. | **Pros:** Massively simplifies client-side frontend request piping. By reserving APIM strictly as an encapsulation barrier for external integrations and Azure Functions, we reduce latency for primary UI interactions while locking down expensive third-party webhooks behind Entra ID security scopes. Isolated verification is trivially achievable via Postman toolchains.<br>**Cons:** Direct UI-to-API requests bypass APIM policies (like global throttling metrics), which must instead be managed at the App Service tier. |
| **Frontend vs. Backend Delivery Split** | **Direct SWA Publishing for UI** + **Deployment Slots for API** | Running unified frontend/backend builds inside a shared App Service container instance. | **Pros:** Exceptional isolation boundaries. Frontend updates (UI adjustments, CSS updates, asset swapping) bypass backend slot recycling completely. Static web assets are aggressively cached across the global CDN edge immediately upon direct publish.<br>**Cons:** Requires explicit CORS configuration inside Azure App Services to securely allow decoupled UI interaction points across environments. |
| **Database Migration Model** | **Expand & Contract Parallel Migration** | Complete In-Place Overwrites (Down-time Migrations) | **Pros:** Guarantees structural zero-downtime execution. The production database is evolved to accept both old and new code layers seamlessly during a slot swap.<br>**Cons:** Requires writing temporary nullable fields and data copy scripts for column updates. |

## Isolated Testability Framework (APIM & Postman)

By segregating Azure API Management (APIM) away from raw UI traffic, the system gains a highly predictable, repeatable testing surface area.

### 1. Integration Isolation
When developers or automated scripts run test suites in **Postman**, they target the APIM endpoint directly. Because APIM mandates **Azure Entra ID security validation and token mapping**, Postman simulates realistic token payloads. This validates permissions, policy transformations, and serverless Azure Function endpoints without spinning up or interacting with the Angular user interface.

### 2. Safeguarding Serverless Compute Boundaries
Azure Functions execute complex translation algorithms on incoming freight carrier telemetry. Reserving APIM to act as the direct supervisor of these Functions means we can configure strict rate-limiting, IP-whitelisting, and contract checking at the gateway edge. If an external carrier API goes down or misbehaves during testing, it is safely isolated within the APIM/Function tier, keeping the core Web API container completely stable.
