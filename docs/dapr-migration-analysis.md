# Dapr Migration Analysis — Microservices Demo (Google Online Boutique)

## Executive Summary

This document assesses the architectural compatibility between the Google Online Boutique microservices-demo and [Dapr](https://dapr.io) (Distributed Application Runtime). The overall finding is that migration is **moderately easy** (7/10 compatibility) — the sidecar model fits naturally, and most Dapr building blocks align well with existing patterns. The main effort lies in rerouting gRPC client calls through the Dapr sidecar, concentrated in the Frontend and Checkout services.

---

## Current Architecture at a Glance

| Aspect | Current State |
|--------|--------------|
| **Services** | 11 application services + 1 load generator, polyglot (Go, C#, Node.js, Python, Java) |
| **Communication** | gRPC (10 services), HTTP (frontend & shopping assistant) |
| **Service Discovery** | Kubernetes DNS + hardcoded env vars (`*_SERVICE_ADDR`) |
| **State Store** | Redis (cart data) |
| **Service Contracts** | Single `protos/demo.proto` defining all gRPC services |
| **Observability** | OpenTelemetry (OTLP/gRPC exporters) + Google Cloud Profiler |
| **Service Mesh** | Optional Istio integration (sidecars, gateway, virtual services) |
| **Secrets** | Kubernetes env vars + Google Secret Manager |
| **Async Messaging** | None — all communication is synchronous request/response |
| **Deployment** | Kubernetes (Skaffold + Kustomize + Helm), GKE-oriented |

### Service Dependency Graph

```
LoadGenerator / User Browser
    ↓ HTTP
Frontend (Go, 8080)
    ├─ gRPC → ProductCatalogService (Go, 3550)
    ├─ gRPC → CartService (C#, 7070)
    ├─ gRPC → CheckoutService (Go, 5050)
    ├─ gRPC → ShippingService (Go, 50051)
    ├─ gRPC → CurrencyService (Node.js, 7000)
    ├─ gRPC → RecommendationService (Python, 8080)
    ├─ gRPC → AdService (Java, 9555)
    └─ HTTP → ShoppingAssistantService (Python, 80)

CheckoutService (Go, 5050)
    ├─ gRPC → ProductCatalogService
    ├─ gRPC → CartService
    ├─ gRPC → ShippingService
    ├─ gRPC → PaymentService (Node.js, 50051)
    ├─ gRPC → CurrencyService
    └─ gRPC → EmailService (Python, 5000)

RecommendationService (Python)
    └─ gRPC → ProductCatalogService

CartService (C#)
    └─ TCP → Redis (6379)
```

---

## Dapr Building Blocks — Mapping

### 1. Service Invocation — HIGH alignment

| | Details |
|---|--------|
| **Current** | Direct gRPC calls between services using K8s DNS names via env vars |
| **Dapr** | gRPC/HTTP via Dapr sidecar (`localhost:3500`) or gRPC proxy on port 50001 |
| **Effort** | **MODERATE** — Proto contracts stay the same, but client connection setup changes. Dapr supports gRPC proxying, so services keep their gRPC servers; callers target the Dapr sidecar instead. |
| **Key concern** | Checkout calls 6 services, Frontend calls 8+ — these are the bulk of the work. |

### 2. State Management — HIGH alignment

| | Details |
|---|--------|
| **Current** | CartService uses Redis directly via .NET StackExchange.Redis |
| **Dapr** | State Store API (`/v1.0/state/<store-name>`) with Redis component |
| **Effort** | **LOW-MODERATE** — Replace Redis client calls with Dapr state API. Cart model (key=userId, value=items) maps cleanly. |
| **Bonus** | Swap Redis for any Dapr-supported store (Cosmos DB, DynamoDB, PostgreSQL) without code changes. |

### 3. Pub/Sub Messaging — NOT currently used

| | Details |
|---|--------|
| **Current** | No async messaging — everything is synchronous |
| **Dapr** | Pub/Sub building block |
| **Effort** | **N/A** unless redesigned to event-driven |
| **Opportunity** | Email notifications and ad serving are natural candidates for async pub/sub. The checkout workflow (charge → ship → email) could benefit from event-driven decoupling. |

### 4. Observability — HIGH alignment

| | Details |
|---|--------|
| **Current** | OpenTelemetry with OTLP/gRPC exporters configured per-service |
| **Dapr** | Sidecar auto-generates distributed traces and metrics via OpenTelemetry |
| **Effort** | **LOW** — Dapr sidecars inject trace context automatically. Per-service OTel boilerplate can be simplified. Custom business spans would remain. |

### 5. Secrets Management — MODERATE alignment

| | Details |
|---|--------|
| **Current** | Kubernetes env vars + Google Secret Manager SDK |
| **Dapr** | Secrets API with K8s secrets or GCP Secret Manager component |
| **Effort** | **LOW-MODERATE** — Replace direct SDK usage with Dapr Secrets API. Env vars continue to work as-is. |

### 6. Configuration — LOW alignment (not needed)

| | Details |
|---|--------|
| **Current** | Environment variables in K8s manifests |
| **Dapr** | Configuration API |
| **Effort** | **LOW** — Optional adoption. Current approach works fine. |

### 7. Bindings — PARTIAL match

| | Details |
|---|--------|
| **Current** | Email service is a mock; no real external integrations |
| **Dapr** | Output bindings (SendGrid, SMTP), input bindings |
| **Effort** | **LOW** — Only relevant when adding real external integrations. |

---

## Migration Complexity by Service

| Service | Language | Effort | Reason |
|---------|----------|--------|--------|
| **Frontend** | Go | **HIGH** | Calls 8+ backend services via gRPC; all must route through Dapr |
| **Checkout** | Go | **HIGH** | Orchestrates 6 services; most complex call graph |
| **Cart** | C# | **MODERATE** | Replace direct Redis with Dapr state API; gRPC server unchanged |
| **Product Catalog** | Go | **LOW** | gRPC server only, no outbound service calls |
| **Currency** | Node.js | **LOW** | Pure function, no outbound calls |
| **Payment** | Node.js | **LOW** | Pure function, no outbound calls |
| **Shipping** | Go | **LOW** | Pure function, no outbound calls |
| **Email** | Python | **LOW** | Pure function, no outbound calls |
| **Recommendation** | Python | **LOW-MOD** | One outbound call to ProductCatalog to reroute |
| **Ad** | Java | **LOW** | In-memory data, no outbound calls |
| **Shopping Assistant** | Python | **LOW-MOD** | Already HTTP-based; Dapr HTTP invocation straightforward |
| **Load Generator** | Python | **NONE** | External HTTP client; unaffected |

---

## Advantages of Migration

1. **Sidecar model already familiar** — The project already supports optional Istio sidecars; Dapr's model is conceptually identical
2. **Polyglot-friendly** — Dapr is language-agnostic; the 5-language mix benefits from a uniform runtime API instead of per-language SDK patterns
3. **State store abstraction** — CartService gains pluggable backend portability for free
4. **Simplified observability** — Dapr auto-instruments inter-service calls, reducing duplicated OTel boilerplate across all services
5. **Incremental adoption** — Services can be migrated one at a time; Dapr and non-Dapr services coexist
6. **Service discovery for free** — Eliminates hardcoded `PRODUCT_CATALOG_SERVICE_ADDR`, `CURRENCY_SERVICE_ADDR`, etc.

---

## Challenges & Risks

1. **gRPC client refactoring** — All 10 gRPC services need client code updated to route through the Dapr sidecar. Proto definitions stay the same, but `grpc.Dial()` targets change from `service:port` to `localhost:dapr-port` with app-id metadata.

2. **No async messaging today** — Dapr's pub/sub is a major selling point, but provides no immediate benefit without an architectural redesign. The fully synchronous architecture won't leverage Dapr's event-driven capabilities out of the box.

3. **Double-sidecar overhead** — If Istio is retained alongside Dapr, each pod runs two sidecars (Envoy + daprd), increasing memory/CPU overhead and operational complexity. **Recommendation: choose one or the other.**

4. **Checkout orchestration** — The checkout service's synchronous multi-step flow (prepare → charge → ship → email → clear cart) doesn't naturally map to Dapr Workflows without significant refactoring.

5. **Local development complexity** — Each service now depends on a Dapr sidecar to run. Mitigated by `dapr run` CLI, but adds friction compared to the current `go run` / `npm start` workflow.

6. **Deployment manifest changes** — All Kubernetes manifests (13 YAML files), Helm chart templates, and 16 Kustomize components need Dapr annotations (`dapr.io/enabled`, `dapr.io/app-id`, `dapr.io/app-port`, `dapr.io/app-protocol`). New Dapr component YAMLs (state store, observability config) must be created.

---

## Overall Assessment

| Dimension | Rating | Notes |
|-----------|--------|-------|
| **Architectural compatibility** | **7/10** | Sidecar model fits naturally; gRPC is supported but requires client changes |
| **Migration effort** | **Moderate** | Bulk of work in Frontend + Checkout; leaf services are trivial |
| **Risk level** | **Low-Moderate** | No fundamental conflicts; mainly plumbing/wiring changes |
| **Value gained** | **Moderate** | Wins: state abstraction, observability, service discovery. Limited pub/sub value without redesign |
| **Incremental adoption** | **Fully supported** | Migrate one service at a time; non-Dapr services unaffected |

---

## Recommended Migration Order

### Phase 1 — Leaf Services (Low risk, high learning)
**Services:** Payment, Shipping, Currency, Email, Ad, Product Catalog

Add Dapr sidecar annotations to deployments. These services are gRPC servers with no outbound service calls — they work immediately with Dapr since the sidecar forwards inbound traffic to the app port. No code changes required in this phase.

### Phase 2 — State Migration
**Service:** CartService

Replace the direct StackExchange.Redis client with Dapr State API calls. Create a Dapr `statestore` component YAML pointing to the existing Redis instance. This validates the state management building block with minimal risk.

### Phase 3 — Simple Callers
**Services:** RecommendationService, ShoppingAssistantService

Update outbound gRPC/HTTP calls to route through the Dapr sidecar. These services have only 1-2 outbound dependencies, making them ideal for validating the service invocation pattern.

### Phase 4 — Orchestrators
**Services:** Frontend, CheckoutService

Reroute all outbound calls through Dapr. This is the highest-effort phase due to the number of dependencies (8+ and 6 respectively). Consider using Dapr's gRPC proxy mode to minimize code changes.

### Phase 5 — Infrastructure Cleanup
- Remove hardcoded `*_SERVICE_ADDR` environment variables from manifests
- Consolidate observability configuration into Dapr's telemetry config
- Evaluate removing Istio if Dapr provides sufficient traffic management
- Add Dapr component YAMLs to Kustomize/Helm for all environments
