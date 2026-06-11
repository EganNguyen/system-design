# The Gold Standard for Cloud-Native Microservices

> A reference architecture distilled from Google's **Online Boutique** (`microservices-demo`):
> an eleven-service, polyglot e-commerce system designed to demonstrate production-grade
> cloud-native patterns on Kubernetes.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [API Contract Design: Protobuf-First](#2-api-contract-design-protobuf-first)
3. [Container Discipline](#3-container-discipline)
4. [Service Boundaries and Responsibilities](#4-service-boundaries-and-responsibilities)
5. [Kubernetes Deployment Patterns](#5-kubernetes-deployment-patterns)
6. [Observability as a First-Class Citizen](#6-observability-as-a-first-class-citizen)
7. [Security Hardening](#7-security-hardening)
8. [Infrastructure as Code and the Developer Loop](#8-infrastructure-as-code-and-the-developer-loop)
9. [Resilience and Scalability Patterns](#9-resilience-and-scalability-patterns)

---

## 1. Architecture Overview

Online Boutique is a **polyglot microservices system** composed of eleven application services plus a
co-deployed load generator. Traffic enters through a single HTTP edge, fans out over gRPC internally,
and converges at a checkout orchestrator.

```
Browser / Locust
      │  HTTP :8080
      ▼
  frontend (Go)
  ├──► productcatalogservice (Go)    gRPC :3550
  ├──► currencyservice (Node.js)     gRPC :7000
  ├──► cartservice (C#)              gRPC :7070  ──► Redis / Spanner / AlloyDB
  ├──► recommendationservice (Py)    gRPC :8080
  ├──► adservice (Java)              gRPC :9555
  └──► checkoutservice (Go)          gRPC :5050
       ├──► cartservice
       ├──► productcatalogservice
       ├──► currencyservice
       ├──► shippingservice (Go)     gRPC :50051
       ├──► paymentservice (Node.js) gRPC :50051
       └──► emailservice (Python)    gRPC
```

**Key structural decisions:**

| Decision | Rationale |
|---|---|
| HTTP only at the edge | Simplifies TLS termination and load balancer integration to one surface |
| gRPC for all internal calls | Strong typing, efficient binary framing, bidirectional streaming, built-in health protocol |
| Polyglot by design | Each service uses the language best suited to it — Go for I/O-bound orchestration, Java for complex business logic, Node.js for high-QPS stateless work |
| Load generator co-deployed | Realistic traffic in every environment from day one; never surprises you in production |

---

## 2. API Contract Design: Protobuf-First

All service contracts live in a **single canonical file**: [`protos/demo.proto`](../protos/demo.proto).
There is no ambiguity about the source of truth, no version skew between service teams, and no
hand-written JSON schemas that drift from implementation.

### The single-proto discipline

```protobuf
syntax = "proto3";
package hipstershop;

service CartService {
    rpc AddItem(AddItemRequest) returns (Empty) {}
    rpc GetCart(GetCartRequest) returns (Cart) {}
    rpc EmptyCart(EmptyCartRequest) returns (Empty) {}
}
```

Every service — Cart, Recommendation, ProductCatalog, Shipping, Currency, Payment, Email,
Checkout, Ad — is defined here. Generated stubs are checked in under each service's `genproto/`
directory, so CI never needs a protoc step to compile the service.

### The `Money` type: precision over convenience

The most instructive type in the file is `Money`:

```protobuf
message Money {
    string currency_code = 1;   // ISO 4217
    int64  units = 2;            // whole units
    int32  nanos = 3;            // fractional part × 10⁻⁹
}
```

This mirrors Google's own `google.type.Money`. By separating whole units from nano-fractional parts
(both signed integers), the type avoids all floating-point rounding errors across currency
conversions and arithmetic — a subtle invariant that would be catastrophic to get wrong in a
payment flow. `checkoutservice` ships a dedicated `money/` package that implements safe
addition and multiplication against this invariant.

### Shared domain types

Rather than duplicating primitives, the proto defines reusable domain objects — `Address`,
`CartItem`, `OrderItem`, `OrderResult` — that flow across service boundaries unchanged. A
`PlaceOrderRequest` composes `Address`, `CreditCardInfo`, and `Money` from the same namespace,
ensuring a consistent data model from the browser form to the shipping label.

### gRPC health checks as a first-class protocol

Every backend service exposes the standard `grpc.health.v1.Health` service. Kubernetes native
gRPC liveness and readiness probes (`grpc:` probe type) use this directly — no HTTP `/healthz`
shim required.

---

## 3. Container Discipline

Every service follows the same two invariants: **multi-stage builds** and **minimal runtime
images**. The result is the smallest possible attack surface with the fastest possible pull time.

### Multi-stage builds across every language

**Go** ([`src/frontend/Dockerfile`](../src/frontend/Dockerfile)):

```dockerfile
ARG BUILDPLATFORM=linux/amd64
FROM --platform=$BUILDPLATFORM golang:1.26.2-alpine@sha256:f85330... AS builder
ARG TARGETOS=linux
ARG TARGETARCH=amd64
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} CGO_ENABLED=0 \
    go build -ldflags="-s -w" -o /go/bin/frontend .

FROM gcr.io/distroless/static
COPY --from=builder /go/bin/frontend /src/server
ENTRYPOINT ["/src/server"]
```

The runtime stage is `gcr.io/distroless/static` — no shell, no package manager, no libc. The
`-ldflags="-s -w"` flag strips the symbol table and DWARF debug information, further shrinking
the binary.

**C# (.NET 10)** ([`src/cartservice/src/Dockerfile`](../src/cartservice/src/Dockerfile)):

```dockerfile
FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:10.0.100-noble@sha256:c7445f... AS builder
RUN dotnet publish cartservice.csproj \
    -p:PublishSingleFile=true \
    -p:PublishTrimmed=true \
    -p:TrimMode=full \
    --self-contained true \
    -c release

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0.0-noble-chiseled@sha256:b857c8...
USER 1000
```

The runtime stage uses **Ubuntu Chiseled** images — a minimal Ubuntu variant that ships only the
files actually needed by the .NET runtime. `PublishTrimmed=true` with `TrimMode=full` performs
IL tree-shaking at publish time, removing unreachable code from the self-contained binary.

**Java** ([`src/adservice/Dockerfile`](../src/adservice/Dockerfile)):

```dockerfile
FROM --platform=$BUILDPLATFORM eclipse-temurin:24-jdk-noble@sha256:dacac8... AS builder
RUN ./gradlew installDist

FROM eclipse-temurin:25-jre-alpine@sha256:5fcc27...
```

Builder uses the full JDK; the runtime drops to the JRE on Alpine — roughly one-third of the
image size.

### Image pinning by SHA256 digest

Every `FROM` instruction pins to a SHA256 digest, not a mutable tag. This guarantees that a build
from six months ago produces an identical image today. Tags like `latest` and even `1.26-alpine`
are mutable; digest pins are not.

### Multi-architecture builds

[`skaffold.yaml`](../skaffold.yaml) declares:

```yaml
build:
  platforms: ["linux/amd64", "linux/arm64"]
  local:
    useDockerCLI: true
    useBuildkit: true
```

BuildKit's `--platform` cross-compilation support means the same `Dockerfile` — using the
`BUILDPLATFORM` / `TARGETOS` / `TARGETARCH` ARG pattern — produces both `amd64` and `arm64`
images from a single build context. No separate Dockerfiles per architecture.

### Tag policy: git commit as the source of truth

```yaml
tagPolicy:
  gitCommit: {}
```

Images are tagged with the exact git commit SHA. There is no `latest`, no manually incremented
version. Every deployed image is traceable to a specific commit, making rollback and audit
straightforward.

---

## 4. Service Boundaries and Responsibilities

Each service owns exactly one domain capability. The decomposition follows the **single
responsibility principle** at the service level:

| Service | Owns | Does NOT own |
|---|---|---|
| `productcatalogservice` | Product data, search | Pricing, inventory |
| `currencyservice` | Exchange rate conversion | Payment processing |
| `cartservice` | Cart state per user | Checkout logic |
| `checkoutservice` | Order orchestration | Persistence of any kind |
| `paymentservice` | Charge authorization | Order state |
| `shippingservice` | Cost quotes, tracking IDs | Address validation |
| `emailservice` | Confirmation delivery | Order data storage |
| `recommendationservice` | Product scoring | Product data |
| `adservice` | Ad selection by context | User tracking |
| `frontend` | HTTP presentation | Business logic |

### No shared libraries

There are no internal packages shared across service boundaries. Each service independently
imports from `protos/` generated stubs, implements its own gRPC server, and manages its own
dependencies. This hard seam means services can be upgraded, replaced, or rewritten in a different
language without any ripple effect.

### Language chosen for fit, not uniformity

| Language | Services | Why |
|---|---|---|
| Go | frontend, checkout, shipping, productcatalog | Low overhead, fast startup, excellent gRPC client support |
| C# (.NET) | cartservice | Stateful service with strong typing needs; .NET's async I/O suits Redis access |
| Node.js | currencyservice, paymentservice | High-QPS stateless work; non-blocking event loop ideal |
| Python | emailservice, recommendationservice | Fast prototyping; ML/data ecosystem for recommendations |
| Java | adservice | Mature gRPC ecosystem; heavy business logic benefits from JVM's JIT |

---

## 5. Kubernetes Deployment Patterns

The project provides three layers of Kubernetes configuration, each adding abstraction and
flexibility while the underlying manifests stay consistent.

### Layer 1: Raw manifests as the ground truth

[`kubernetes-manifests/`](../kubernetes-manifests/) contains one YAML file per service. Each file
defines exactly three objects in this order: `Deployment`, `Service`, `ServiceAccount`. This
predictable structure makes manifests scannable and diffable.

**Every deployment sets both requests and limits:**

```yaml
resources:
  requests:
    cpu: 200m
    memory: 64Mi
  limits:
    cpu: 300m
    memory: 128Mi
```

Without requests, the scheduler cannot bin-pack correctly. Without limits, a misbehaving pod
can starve its neighbors. Both are mandatory.

**gRPC health probes at the Kubernetes layer:**

```yaml
readinessProbe:
  initialDelaySeconds: 15
  grpc:
    port: 7070
livenessProbe:
  initialDelaySeconds: 15
  periodSeconds: 10
  grpc:
    port: 7070
```

Kubernetes 1.24+ supports native gRPC probes via the `grpc:` probe type. This eliminates the
need for a `/healthz` HTTP endpoint or a bundled `grpc_health_probe` binary in every image.

**Fast pod shutdown:**

```yaml
terminationGracePeriodSeconds: 5
```

A 5-second grace period matches the expected connection drain time for these short-lived gRPC
calls. Setting it unnecessarily high delays rolling deployments; too low risks dropped in-flight
requests.

### Layer 2: Kustomize components for opt-in capabilities

[`kustomize/components/`](../kustomize/components/) contains self-contained overlays that can be
composed independently:

```
network-policies/     — per-service NetworkPolicy objects
service-mesh-istio/   — Istio sidecar injection labels
spanner/              — swap Redis for Cloud Spanner
alloydb/              — swap Redis for AlloyDB
shopping-assistant/   — add the AI assistant service
cymbal-branding/      — Google Cymbal visual theme
```

This component model means a team can activate network policies for a security audit, then remove
them, without touching the base manifests. Kustomize patches compose cleanly; there is no template
inheritance to debug.

### Layer 3: Helm chart for parameterized production deployment

The [`helm-chart/`](../helm-chart/) wraps everything in a versioned, values-driven release. Key
design choices:

- Each service can be disabled via `create: false` — useful when testing subsets of the stack
- `serviceAccounts.annotationsOnlyForCartservice: false` enables scoped Workload Identity
  annotation only where external cloud access is required
- `opentelemetryCollector.create` / `googleCloudOperations.*` are all off by default — observability
  is opt-in, so the chart ships without any cloud-provider dependency unless explicitly enabled
- `cartDatabase.type` accepts `redis`, `spanner`, or `alloydb` — the storage backend is a
  configuration choice, not a code change

---

## 6. Observability as a First-Class Citizen

Observability is instrumented at every layer: structured logs from process start, distributed
traces across every gRPC hop, and metrics exported to the cloud operations backend of choice.

### Structured JSON logging — consistent across languages

Every service emits logs as JSON to stdout with a consistent schema: `timestamp`, `severity`,
`message`. This is not a convention enforced by a shared library — it is independently implemented
in each language to match the target collector's expected field names.

**Go** (logrus in `checkoutservice`):

```go
log.Formatter = &logrus.JSONFormatter{
    FieldMap: logrus.FieldMap{
        logrus.FieldKeyTime:  "timestamp",
        logrus.FieldKeyLevel: "severity",
        logrus.FieldKeyMsg:   "message",
    },
    TimestampFormat: time.RFC3339Nano,
}
log.Out = os.Stdout
```

**Node.js** (pino in `currencyservice`):

```js
const logger = pino({
  name: 'currencyservice-server',
  messageKey: 'message',
  formatters: {
    level(logLevelString) { return { severity: logLevelString } }
  }
});
```

The field name `severity` is not arbitrary — it is the field Google Cloud Logging uses to map
log entries to their severity level. Conforming to this contract means logs are automatically
parsed and filterable in Cloud Logging without any additional pipeline configuration.

### OpenTelemetry: vendor-neutral instrumentation across all languages

Every backend service registers OTel gRPC instrumentation unconditionally at startup — trace
context is always propagated, even when export is disabled. Exporting is gated behind an
environment variable:

```js
// Always register instrumentation for context propagation
const { GrpcInstrumentation } = require('@opentelemetry/instrumentation-grpc');
registerInstrumentations({ instrumentations: [new GrpcInstrumentation()] });

// Only export traces when explicitly enabled
if (process.env.ENABLE_TRACING == "1") {
    // configure OTLPTraceExporter → collector
}
```

This separation is deliberate: context propagation must work for distributed traces to be complete
regardless of whether this specific service is exporting. A service that skips instrumentation
entirely creates a gap in the trace that cannot be filled later.

### The collector as a centralized gateway

Rather than having each service export directly to a cloud backend, all services send to a
shared **OpenTelemetry Collector** (`otel/opentelemetry-collector-contrib`) running as a Deployment
in the same namespace. The collector translates to Google Cloud Trace, Prometheus metrics, and
other backends via its pipeline configuration.

The collector's Helm template includes an `initContainer` that auto-discovers the GCP project ID
from the instance metadata server at startup:

```sh
sed "s/PROJECT_ID/$(curl -H 'Metadata-Flavor: Google' \
  http://metadata.google.internal/computeMetadata/v1/project/project-id)" \
  /template/collector-gateway-config-template.yaml >> /conf/collector-gateway-config.yaml
```

This avoids hardcoding the project ID in the Helm values while remaining fully automated in GKE.

---

## 7. Security Hardening

Security is layered: the container, the pod, and the network each enforce independent restrictions.
A compromise at one layer does not automatically grant access at the next.

### Container-level: drop everything, grant nothing

Every container in every manifest carries the same security context:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  privileged: false
  readOnlyRootFilesystem: true
```

- `allowPrivilegeEscalation: false` prevents `sudo` or `setuid` escalation inside the container
- `capabilities.drop: [ALL]` removes every Linux capability, including `NET_RAW` (no raw socket
  access), `SYS_ADMIN`, `CHOWN`, and others that are default-granted by the container runtime
- `readOnlyRootFilesystem: true` means even if an attacker achieves code execution, they cannot
  write malicious files to the container filesystem

### Pod-level: non-root execution

```yaml
securityContext:
  fsGroup: 1000
  runAsGroup: 1000
  runAsNonRoot: true
  runAsUser: 1000
```

No process in any pod runs as root (UID 0). The C# cartservice Dockerfile enforces this at build
time with `USER 1000`, making the container refuse to start as root even if the Kubernetes
security context were absent.

The optional `seccompProfile.type: RuntimeDefault` (available in the Helm chart) adds a seccomp
filter that blocks syscalls not in the OCI default allowlist, providing a fourth independent
restriction layer.

### Network-level: deny-all default with explicit allow rules

The network policy component starts from a **default deny**:

```yaml
# network-policy-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  podSelector: {}       # matches ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
```

Then each service gets its own policy that opens only the exact ingress sources and ports it
requires. For `cartservice`:

```yaml
spec:
  podSelector:
    matchLabels:
      app: cartservice
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - podSelector:
        matchLabels:
          app: checkoutservice
    ports:
    - port: 7070
      protocol: TCP
```

The policy encodes the service dependency graph directly: only `frontend` and `checkoutservice`
can reach `cartservice`, and only on its gRPC port. Any other pod — even in the same namespace —
is silently dropped.

### Identity: one ServiceAccount per service

Every service runs under its own `ServiceAccount`. This enables fine-grained Workload Identity
bindings (only `cartservice` needs cloud storage IAM permissions), and ensures RBAC policies can
be scoped to individual services rather than sharing a default account across the namespace.

---

## 8. Infrastructure as Code and the Developer Loop

### Terraform: GKE Autopilot + dependent APIs

[`terraform/main.tf`](../terraform/main.tf) provisions the entire platform layer:

```hcl
module "enable_google_apis" {
  source  = "terraform-google-modules/project-factory/google//modules/project_services"
  activate_apis = concat(local.base_apis, var.memorystore ? local.memorystore_apis : [])
}

resource "google_container_cluster" "my_cluster" {
  enable_autopilot = true
  ip_allocation_policy {}
}
```

**GKE Autopilot** is the deliberate choice here. Autopilot manages node provisioning, scaling,
and patching automatically. The operator declares workload resource requirements; the control
plane decides how many nodes to run. This eliminates the node-pool tuning and OS patching burden
that consumes significant ops time in Standard mode clusters.

API enablement — `container`, `monitoring`, `cloudtrace`, `cloudprofiler`, `redis` — is
expressed as code. Enabling an API is a reviewable, auditable change, not an invisible click in
the console. The `memorystore` APIs are conditionally enabled only when the user opts into
managed Redis, preventing unnecessary API surface from being activated.

The Terraform also provisions a `null_resource` that waits for all pods to reach `Ready` before
the apply completes — infrastructure provisioning and application deployment are coupled in a
single atomic operation.

### Skaffold: the inner development loop

[`skaffold.yaml`](../skaffold.yaml) defines a fast local development cycle:

```yaml
tagPolicy:
  gitCommit: {}
build:
  platforms: ["linux/amd64", "linux/arm64"]
  local:
    useDockerCLI: true
    useBuildkit: true
manifests:
  kustomize:
    paths:
    - kubernetes-manifests
```

The `gitCommit` tag policy means that `skaffold dev` detects source changes, rebuilds only
affected service images, re-tags them with the new commit SHA, and re-deploys — all in the time
it takes to build a single Docker layer.

**Skaffold profiles** extend the base config without duplicating it:

| Profile | Purpose |
|---|---|
| `debug` | Replaces cartservice's Dockerfile with `Dockerfile.debug`, enabling remote debugging via Skaffold's debug protocol |
| `gcb` | Routes image builds to Google Cloud Build (32-vCPU N1 machine), eliminating the need for Docker locally |
| `network-policies` | Adds the Kustomize network-policy component to the deploy path, testing production security posture locally |

### Cloud Build CI

[`cloudbuild.yaml`](../cloudbuild.yaml) drives CI on Google Cloud Build, using the same `skaffold`
tooling developers use locally. There is no divergence between the CI build and the local build
— same Dockerfiles, same tag policy, same Kustomize paths. What passes locally will pass in CI.

---

## 9. Resilience and Scalability Patterns

### Pluggable storage: backend as a configuration choice

`cartservice` is the only stateful service in the system. Its backing store is deliberately
abstracted through Kustomize components and Helm values:

| Backend | When to use |
|---|---|
| **In-cluster Redis** (default) | Local development, demos, cost-sensitive deployments |
| **Cloud Memorystore** (managed Redis) | Production: automatic failover, patching, TLS |
| **Cloud Spanner** | Global scale, strong consistency, multi-region writes |
| **AlloyDB** | PostgreSQL-compatible, high throughput OLTP |

Switching backends requires no code changes — only a Kustomize component swap or a Helm value
change. The service contract (`CartService` proto) is identical regardless of what sits behind it.

### Istio service mesh as an optional overlay

[`istio-manifests/`](../istio-manifests/) and the `service-mesh-istio/` Kustomize component add
mTLS between all services, traffic management policies, and circuit breaking — without touching
any service code. The mesh is an infrastructure concern applied as a Kubernetes overlay, not an
application library. Services that do not need the mesh run identically without it.

### Load generator as a first-class service

[`src/loadgenerator/`](../src/loadgenerator/) is a Locust-based load generator deployed in the
same cluster as the application. It is not an external tool or an afterthought. By co-deploying
it, every environment — development, staging, production — has realistic traffic from the moment
the cluster is up. This matters for:

- **Autoscaler calibration**: HPA and Autopilot node provisioning train on real traffic patterns
- **Observability validation**: traces and metrics are populated immediately, not on first real user
- **Regression detection**: a performance change is visible in minutes, not after a production push

The Helm chart includes an `initContainer` on the load generator that waits for the frontend to
be responsive before Locust begins, preventing false failures during cluster startup.

### Resource isolation prevents noisy neighbors

Every container specifies both CPU and memory requests and limits. In a shared cluster, a service
that suddenly consumes excess CPU (a hot cache miss, a GC pause) is capped by its limit. Its
neighbors continue to receive their requested shares. This is not a performance optimization —
it is a correctness property of multi-tenant scheduling.

---

## Summary: The Patterns That Matter

The eleven design decisions that distinguish this architecture from an average microservices system:

| # | Pattern | Implementation |
|---|---|---|
| 1 | Single proto as API source of truth | `protos/demo.proto` — all services, one file |
| 2 | Money as integers, not floats | `units` + `nanos` in the `Money` message |
| 3 | Distroless / chiseled runtime images | `gcr.io/distroless/static`, `noble-chiseled` |
| 4 | SHA256-pinned base images | Every `FROM` includes `@sha256:...` |
| 5 | gRPC health protocol in Kubernetes probes | Native `grpc:` probe type, no shim |
| 6 | Structured logs to stdout, always | Consistent `severity`/`timestamp`/`message` fields |
| 7 | OTel context propagation unconditional, export optional | `ENABLE_TRACING` gates the exporter, not the instrumentation |
| 8 | Default-deny network policy + per-service allow | `deny-all` + explicit ingress per pod selector |
| 9 | Drop ALL capabilities, read-only filesystem | Container `securityContext` on every workload |
| 10 | Storage backend as config, not code | Kustomize components / Helm values — no service changes |
| 11 | Developer tooling matches CI | Same Skaffold + Kustomize path in both environments |

These are not aspirational guidelines. Each one is a concrete, verifiable property of the running
system — checkable in the manifests, Dockerfiles, and source code of this repository.
