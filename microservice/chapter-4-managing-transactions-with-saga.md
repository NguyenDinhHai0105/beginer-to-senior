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
- this transaction has multiple participants (like an interaction with other services; each interaction is a participant)
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
