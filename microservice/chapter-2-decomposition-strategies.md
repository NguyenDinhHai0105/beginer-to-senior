# Service Decomposition Strategies

## Strategies for Decomposing a Service

- **Decompose by business capability:** each service is responsible for a specific business capability (e.g., order management, inventory, payments)
- **Decompose by subdomain:** based on Domain-Driven Design (DDD), each service maps to a bounded context within the application's domain model

---

## Microservice Pattern Language

Patterns are grouped into three layers depending on where the problem lives.

### Application Patterns
_Solve problems faced by developers._

| Pattern | Description |
|---------|-------------|
| **Service decomposition** | How to split the application into services (by business capability, by subdomain) |
| **Data management** | How to manage data across services (database per service, shared database, event sourcing) |
| **Querying** | How clients query data distributed across multiple services |
| **Data consistency** | How to keep data consistent across service boundaries (sagas, outbox pattern) |
| **Testing** | Strategies for testing distributed services (contract testing, consumer-driven contracts) |

### Infrastructure Patterns
_Solve problems that are mostly infrastructure concerns outside of development._

| Pattern | Description |
|---------|-------------|
| **Cross-cutting concerns** | Logging, monitoring, tracing, and security applied uniformly across services |
| **Communication style** | How services talk to each other (REST, gRPC, message queues) |
| **Reliability** | Circuit breakers, retries, and timeouts to handle partial failures |

### Application Infrastructure Patterns
_Solve infrastructure issues that also directly impact development._

| Pattern | Description |
|---------|-------------|
| **Deployment** | Containerization, orchestration (Kubernetes), and CI/CD pipelines |
| **External APIs** | API gateways and BFF (Backend for Frontend) patterns |
| **Discovery** | Service registry and client-side / server-side service discovery |
