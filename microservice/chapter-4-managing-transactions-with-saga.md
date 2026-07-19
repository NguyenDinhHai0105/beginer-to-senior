# Managing Transactions with SAGA

## Chapter Overview

This chapter covers:
- Why traditional distributed transactions are not a good fit for modern applications
- Using SAGA to maintain consistency in microservice pattern
- Coordinating SAGA using choreography and orchestration
- Using countermeasures to deal with the lack of isolation

---

## 4.1 Managing Transactions in Microservice Architecture

Almost every request handled by enterprise applications is executed within a database transaction. To be precise, transaction management is straightforward in a monolithic application that uses a single database.

However, it's more challenging in microservice architecture, because transactions span multiple services, requiring a more elaborate mechanism to manage transactions. Traditional approaches like 2PC (Two-Phase Commit) are not viable in modern applications. Instead, microservice applications must use **SAGAs**.

### 4.1.1 The need for distributed transactions in a microservice architecture

Imagine that we develop a create-order feature: the operations must verify the consumer can place the order, must verify the menu details, authorize the credit card, then create the order in the database.

If with a monolithic application, one can easily ensure data consistency with ACID, because a single database is readily accessible.

In contrast, implementing the same operations in a microservice architecture is more complicated because each service has its own data, and the operations need to access multiple services. Therefore, we need a mechanism to maintain data consistency.

---

### 4.1.2 The trouble with distributed transactions

In summary, the traditional distributed transactions that use 2PC are described as:

- only one transaction logic
- this transaction has multiple participants (like an interaction with other services, each interaction is a participant)
- has a transaction coordinator to coordinate
- the coordinator ensures that all participants are committed or rolled back (guarantee the atomicity)

This approach has some drawbacks:

- increases latency, because the participants need to wait for each other to commit or roll back
- reduces availability; if a participant has an issue, then the transaction cannot commit -> reduces the loose coupling of services
- single point of failure; if the coordinator has an issue, all participants may be left PENDING
- poor scalability; if adding a new participant, throughput is reduced
- not supported by many technologies, like Kafka, RabbitMQ, NoSQL databases

The worst of this approach is that it reduces the loose coupling of services, whereas a microservice application is built on the concept of loosely-coupled, asynchronous communication.

This is where SAGA comes in.

---

### 4.1.3 Using SAGA to maintain data consistency

SAGA is the mechanism to maintain data consistency between services without using distributed transactions (2PC).

A SAGA is a sequence of local transactions where:
- Each transaction updates data within a single service with familiar ACID transaction guarantees
- The completion of each local transaction triggers the execution of the next local transaction by using asynchronous messaging

#### Example: Create Order with SAGA

Let's say we have 4 services: Order Service, Consumer Service, Kitchen Service, and Accounting Service.

**This SAGA consists of the following local transactions:**

1. **Order Service** — Create an Order in an APPROVAL_PENDING state
2. **Consumer Service** — Verify that the consumer can place an order
3. **Kitchen Service** — Validate order details and create a Ticket in CREATE_PENDING state
4. **Accounting Service** — Authorize consumer's credit card
5. **Kitchen Service** — Change the state of the Ticket to AWAITING_ACCEPTANCE
6. **Order Service** — Change the state of the Order to APPROVED

**How it works:**

Each time a local transaction completes, it triggers the next step in the SAGA by publishing a message using asynchronous messaging.

With this approach:
- All steps of the SAGA are executed even if one or more participants are temporarily unavailable (because messages are buffered in the broker and consumed when the consumer comes back online)

**The challenge:**

On the surface, SAGAs seem straightforward, but one challenging aspect is the lack of isolation between SAGAs. This creates a need for a rollback mechanism when an error occurs.

This is where compensating transactions come in.

#### SAGA uses compensating transactions to roll back changes

A SAGA is a sequence of local transactions; if one local transaction fails, it is necessary to roll back commits in other services.

However, each local transaction typically executes in a different database, so it is not possible to simply roll back like a traditional ACID transaction where a single DB undoes every change.

Therefore, compensating transactions must be implemented to handle rollbacks in a SAGA.

For example, consider a SAGA with five local transactions: A -> B -> C -> D -> E.
If A, B, C, D are committed but E fails, then the corresponding compensating transactions D' -> C' -> B' -> A' must be executed to compensate for A, B, C, D. The compensating transactions can themselves execute as a SAGA, where D' triggers C', and so on.

One important thing is that not all steps need a compensating transaction. For example, step B might only check that menu details are available (a read-only transaction) and does not update anything, so it does not need a compensating transaction.

Example sequence:

1. Order Service — Create an Order in APPROVAL_PENDING state
2. Consumer Service — Verify that the consumer can place an order
3. Kitchen Service — Validate order details and create a Ticket in CREATE_PENDING state
4. Accounting Service — Authorize consumer's credit card, which fails
5. Kitchen Service — Change the state of the Ticket to CREATE_REJECTED (compensates step 3)
6. Order Service — Change the state of the Order to REJECTED (compensates step 1)

Step 2 does not need compensation because it is a verification-only transaction.

Therefore, a SAGA coordinator (coordination logic) is needed to execute forward and compensating transactions.

---

## 4.2 Coordinating SAGA

Again, a SAGA is a sequence of local transactions. It processes and manages forward and compensating local transactions until the SAGA completes all steps.

We need a structure to coordinate the SAGA logic. There are two different approaches:

- **Choreography:** participants exchange messages and decide the next step themselves.
- **Orchestration:** a centralized SAGA coordinator manages the flow. The orchestrator sends command messages to saga participants telling them which operation to perform.

### 4.2.1 Choreography-based sagas 

Implementing the Create Order SAGA using Choreography
Example (happy path):

1. Order Service creates an Order in APPROVAL_PENDING state and publishes an OrderCreated event.
2. Consumer Service consumes the OrderCreated event, verifies the consumer can place the order, and publishes a ConsumerVerified event.
3. Kitchen Service consumes the OrderCreated event, validates the Order, creates a Ticket in CREATE_PENDING state, and publishes the TicketCreated event.
4. Accounting Service consumes the OrderCreated event and creates a CreditCardAuthorization in PENDING state.
5. Accounting Service consumes the TicketCreated and ConsumerVerified events, charges the consumer's credit card, and publishes a CreditCardAuthorized event.
6. Kitchen Service consumes the CreditCardAuthorized event and changes the Ticket state to AWAITING_ACCEPTANCE.
7. Order Service receives the CreditCardAuthorized event, changes the Order state to APPROVED, and publishes an OrderApproved event.

The created order flow must handle cases where a participant publishes a reject event or fails.

#### Reliable event-based communication

There are a couple of issues to consider:

- Ensure that saving data to the DB and sending an event are coordinated — use transactional messaging (Outbox pattern).
- Ensure that each event is received and used to update the participant's data — use a correlationId (for example, use orderId directly).

#### Benefits and drawbacks of choreography-based SAGA

Choreography SAGA has several benefits:

- Simplicity: a service publishes an event when performing a business operation (e.g., CRUD).
- Loose coupling: saga participants publish events without knowing implementation details of other services.

But it also has drawbacks:

- Difficult to understand: choreography distributes the implementation of the SAGA among services; consequently, it can be hard for a developer to understand how a given SAGA works.
- Risk of hidden coupling: each saga participant may need to subscribe to multiple events that affect them, which can create implicit coupling.

Choreography can work for simple SAGAs, but given these drawbacks, orchestration is often preferable for more complex SAGAs.

### 4.2.2 Orchestration-based SAGAs

Orchestration is another way to implement SAGAs by defining an orchestration class. This class is responsible for telling saga participants what to do.

The orchestration communicates with participants using asynchronous messaging, instructing them what to execute. When the execution completes, participants reply to the orchestrator, which then processes the response and determines the next action.

#### Modeling SAGA orchestrators as state machines

A good way to model an orchestrator is as a state machine:

A state machine consists of states and transitions between states:
- **States:** e.g., PENDING_ORDER, CREATED_ORDER, REJECTED_ORDER
- **Transitions:** e.g., PENDING_ORDER → CREATED_ORDER, PENDING_ORDER → REJECTED_ORDER

Each transition can have an action, which for a SAGA is the invocation of a participant. Transitions between states are triggered by the completion of a local transaction in a participant.

Example:
- To transit from PENDING_ORDER → CREATED_ORDER, the SAGA calls the AuthorizeCard participant to perform payment. If payment succeeds, the participant replies to the orchestrator.

The current state and the specific outcome of the local transaction determine the next state transition and what action (if any) to perform.

#### SAGA orchestration and transactional messaging

Because each local transaction must ensure that data is updated and an event is delivered, transactional messaging must be implemented. See the Outbox pattern for details.

#### Benefits and drawbacks of orchestration

**Benefits:**

- **Simpler dependency:** the orchestrator invokes saga participants, but participants do not invoke the orchestrator. As a result, the orchestrator depends on the participants but not vice versa, and there are no cyclic dependencies.
- **Loose coupling:** each service is invoked by the orchestrator, so services do not need to know about each other.
- **Improved separation of concerns:** business logic is more clearly separated from sequencing logic.

**Drawbacks:**

- **Risk of centralizing too much business logic:** the orchestrator can become a "smart" orchestrator that tells "dumb" services what to do. However, you can avoid this problem by designing orchestrators that are solely responsible for sequencing and do not contain any other business logic.

**Recommendation:** Use orchestration for all but the simplest SAGAs. Another significant challenge is the lack of isolation.

---

## 4.3 Handling the lack of isolation

