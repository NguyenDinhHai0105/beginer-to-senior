# Monolith vs Microservices

## Benefits of Monoliths

- **Easier to develop and test:** developers focus only on building a single application
- **Easier to change:** modifications are localized within one codebase
- **Straightforward to test:** no need to mock external services; the whole application can be tested in one go
- **Easier to scale:** can run multiple instances behind a load balancer

## Disadvantages of Monoliths

- **Slow development:** as the codebase grows, the IDE slows down, making the edit-run-test cycle take a long time
- **Harder to maintain:** large codebases become increasingly difficult to understand and modify
- **Long build and deploy cycles:** the large codebase means every change takes significant time to build and ship
- **Difficult to scale independently:** different modules may need different resources (e.g., module A is CPU-intensive, module B is memory-intensive), but they share one process — so resources must be over-provisioned for all
- **Locked into a single technology stack:** switching languages or frameworks is extremely costly once the monolith is established

---

## Scaling Cube

The **Scaling Cube** describes three independent dimensions of scaling:

| Axis | Strategy | Description |
|------|----------|-------------|
| **X** | Horizontal scaling | Clone the application and run multiple instances behind a load balancer |
| **Y** | Functional decomposition | Split the application into smaller, independently deployable services |
| **Z** | Data partitioning (sharding) | Route a subset of requests to a dedicated instance that holds a subset of data |

---

## Benefits of Microservices

- **Continuous delivery at scale:** enables large, complex applications to be developed and shipped continuously
- **Small, maintainable services:** each service is easy to understand; code stays fast in the IDE
- **Independent deployment:** services can be released on their own schedule
- **Independent scaling:** each service scales according to its own resource needs
- **Technology flexibility:** different services can use different languages, frameworks, or databases
- **Better fault isolation:** a failure in one service does not cascade to the entire system
- **Team autonomy:** small teams can own and operate services independently

## Drawbacks of Microservices

- **No universal decomposition algorithm:** deciding how to split an application into services is hard and requires domain expertise
- **Distributed systems complexity:** services need inter-process communication, requiring unfamiliar technologies (e.g., message queues, service discovery, load balancing)
- **Cross-service feature coordination:** a feature that spans multiple services requires careful planning and synchronized deployment
