# Senior Guide: System Evaluation (Backend → Frontend) — Performance and Quality Criteria

## Purpose
This document guides a senior engineer on how to perform a comprehensive system evaluation: from backend and infrastructure to frontend. It includes criteria, performance indicators, a checklist, and reporting guidance.

## Evaluation overview
Goals of the evaluation:
- Correctness: the system works as intended
- Performance: fast and stable under realistic load
- Scalability: can scale to meet demand
- Reliability: low error rate and fast recovery
- Maintainability: tests and clear structure for ongoing work
- Security: follows security best practices

Desired outcome: list of strengths/weaknesses and prioritized improvement recommendations.

---

## Backend
1. Architecture & design
- Clear separation of concerns
- Microservices vs monolith: trade-offs and clear service boundaries
- API design: versioning, pagination, idempotency, and contract stability

2. Performance & optimization
- Latency targets: p50/p95/p99 per endpoint
- Throughput (RPS) and concurrent handling capacity
- DB: sensible schema, correct indexes, avoid N+1 queries
- Query analysis: use EXPLAIN, avoid full table scans
- Caching: caching layers (Redis, in-memory) and invalidation strategies
- Connection pooling, throttling, backpressure handling

3. Batching & background jobs
- Offload heavy tasks to background queues, with retry and dead-letter handling
- Ensure idempotence for job consumers

4. Observability
- Metrics: latency, error rate, saturation, throughput
- Distributed tracing (e.g., OpenTelemetry) for root-cause analysis
- Structured logs with correlation IDs
- Dashboards and SLO/SLI tracking

5. Testing
- Unit, integration, and contract tests (consumer-driven contracts if needed)
- Performance and load tests
- Chaos testing for failure scenarios

6. Security
- Authentication, authorization, input validation
- Rate limiting and audit logging
- Secret management and TLS

---

## Infrastructure & DevOps
- Autoscaling, health checks, graceful shutdown
- Load balancer behavior and sticky sessions if required
- CI/CD: reproducible builds, canary/blue-green deployments
- IaC (Terraform/CloudFormation) and versioning
- Backups and disaster recovery plans

---

## Frontend
1. User-facing performance metrics
- FCP, LCP, TTFB, TTI, TBT, CLS
- Perceived performance: skeleton screens, progressive rendering

2. Bundling & assets
- Bundle size budgets, code-splitting, tree-shaking
- HTTP/2, HTTP/3, compression, preconnect/prefetch
- CDN usage, cache headers, cache-busting strategies

3. Runtime & rendering
- SSR / hybrid SSR when needed (SEO, TTFB)
- Hydration performance and avoiding heavy reflows
- Use Web Workers for heavy computations

4. Network
- Test throttling scenarios (low bandwidth, high latency)
- Provide graceful fallbacks when APIs are slow or failing

5. Observability & testing
- Real User Monitoring (RUM) and synthetic tests
- E2E tests, visual regression, performance budgets

6. Accessibility & UX
- ARIA, keyboard navigation, color contrast
- Progressive enhancement

---

## Suggested performance thresholds
Note: thresholds vary by domain; these are common references:
- API p95 latency: < 300 ms
- API p99 latency: < 1 s
- Error rate: < 0.1% (under realistic load tests)
- Throughput: meet RPS target with 2x headroom
- LCP (real users): < 2.5 s
- CLS: < 0.1
- Initial JS bundle (gzipped): < 200–300 KB (depends on app)
- FCP: < 1 s

---

## Quick Go/No-Go checklist
- [ ] Metrics exist for all important services (latency, errors, throughput)
- [ ] Tracing across the full request flow
- [ ] DB queries profiled and indexed appropriately
- [ ] Caching in place with a clear invalidation strategy
- [ ] Load tests using realistic scenarios
- [ ] CI/CD includes smoke tests and rollback procedures
- [ ] Frontend has performance budgets and RUM
- [ ] Runbooks and alerting are documented and clear

---

## Reporting the evaluation
1. Executive summary (a few lines): overall status and top 3 risks
2. Data: metrics, latency/error/throughput charts
3. Root causes: endpoints/queries/assets causing bottlenecks
4. Prioritized recommendations: quick wins vs infra/architectural work
5. Action plan: owner, estimate, impact

---

## One-page summary template
- Area: Backend / DB / Infra / Frontend
- Status: OK / Needs Attention / Critical
- Main issue: ...
- Suggested solution & priority

---

## Final note
Numbers matter more than impressions; provide measurable, reproducible tests and clear thresholds. A senior should focus on root causes, trade-offs, and a measurable improvement roadmap.
