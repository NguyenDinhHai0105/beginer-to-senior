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

### 4.1.1 Why Transaction Management is Challenging in Microservice Architecture