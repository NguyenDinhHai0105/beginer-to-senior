# 1. Overview of interprocess communication (IPC) in microservices

## 1.1 Interaction style

It's useful to think about the interaction style first, as it can impact the availability of application.

They can be categorized into two dimensions: one-to-one vs one-to-many, and synchronous vs asynchronous.

| | one-to-one | one-to-many |
|---|---|---|
| synchronous | request/response | -------- |
| asynchronous | async request/response | Publish/subscribe |
| | one-way notification | Publish/async responses |

- **request/response**: service A sends a request to service B and waits for a response immediately. might be get block while waiting.
- **async request/response**: service A sends a request to service B and continues processing without waiting for a response. service B will send a response back to service A when it's ready.
- **one-way notification**: service A sends a notification to service B without expecting a response.
- **publish/subscribe**: service A publishes a message to a topic, which consumed by 0 or more services
- **publish/async responses**: A client publishes a request message and then waits for a certain amount of time for responses from interested services

Each service can use a combination of synchronous and asynchronous APIs depending on the use case.

## 1.2 Involving APIs

### 1.2.1 APIs are updated over time

- In monolithic, update API is easier to all caller, or use a static type language, with help of compiler, IDE give use a list of compiler errors
- In microservice, its more difficult. the client's service can be developed by other team, or outside the organization, You usually can't force all clients to upgrade in lockstep with the service. then should have a strategy to deal with those challenges.

### 1.2.2 Use semantic versioning for APIs

- Use semantic versioning for APIs, and follow the rules of semantic versioning. The Semantic Versioning specification (Semvers) requires a version number to consist of three parts: MAJOR.MINOR.PATCH.
- If using REST, include the version number in the URL (e.g., /api/v1/resource) or in the request header (e.g., Accept: application/vnd.myapp.v1+json).

### 1.2.3 Message format

- Essence of IPC is to exchange messages, which contain data. then choose message format can impact to IPC's efficiency.
- 2 option: text format (Json, XML) and binary format (Protocol Buffers, Apache Avro).
  - Text-base format is human-readable, easy to debug, but less efficient in terms of size and parsing speed.
  - Binary format is more compact and faster to parse, but less human-readable. examples include

# 2. Communication using the synchronous Remote procedure invocation pattern (request/response)

## 2.1 Using REST

- **Benefit** of using REST for synchronous:
  - simple and widely adopted, easy to understand and use
  - directly support request/response pattern, with clear semantics
  - can leverage HTTP features (status codes, headers, caching)
- **Drawback** of using REST for synchronous:
  - can lead to tight coupling between services, as clients need to know the API contract. (reduced availability)
  - service must know the resource location (URL)
  - challenge to fetch from multiple resources
  - only supports request/response

## 2.2 Using gRPC

- gRPC is a binary message-based protocol
- **Benefit** of using gRPC for synchronous:
  - more efficient than REST (smaller message size, faster parsing)
  - supports multiple communication patterns (request/response, streaming)
- **Drawback** of using gRPC for synchronous:
  - less human-readable, harder to debug
  - requires code generation from .proto files, which adds complexity
  - not as widely adopted as REST, may have less community support
  - more work for JavaScript clients, as gRPC-Web is needed
  - older firewalls may block gRPC traffic, as it uses HTTP/2

## 2.3 Handle partial failures

- In distributed systems, when a service make a synchronous request to another service, there are some partial failures can happen, such as network issues, service unavailability, or timeouts
- This can block the other requests, make poor user experience, and even cause cascading failures.
- For solution, there are 2 part:
  - design service proxies to handle the unresponsive service
  - decide how to recover from the failure

### Developing robust proxies

- Whenever a service makes a synchronous request to another service, it should protect itself from the unresponsive service. by using some aprroaches described by Netflix
  - **network timeouts**: set a timeout for the request, if time for waiting response > threshold, then abort the request and return an error to the caller
  - **Limit Outstanding Requests**: limit the number of concurrent requests to a service, if the limit is reached, reject new requests with an error
  - **circuit breaker**: set a threshold for the number of failed requests, if the threshold is reached, open the circuit and reject new requests with an error. after a certain time, allow a limited number of requests to test if the service is back online. if successful, close the circuit and allow normal traffic again.
- There 3 approaches above usually combined to help the microservice system increase its availability and fault tolerance.

## 2.4 Using service discovery

- When do the IPC, the service need to know the location of the other service. In the traditional application run on physical hardware, the location like IP addresses usually static, but in modern cloud-base application, it is not simple
- Because of scaling, upgrade, failures, the service instances changes dymicly, consequently, application must use service discovery

There are 2 main way to implement service discovery:
- The service and client interact with service discovery itself (apply in application level)
- Use the service discovery on deployment flatform

### 2.4.1 Apply service discovery pattern at application level

- This pattern is combination of 2 patterns:
  - **self registration**: service instance invoke registry the to register their network location and maybe populate health-check url to check the availability of instances
  - **client-side discovery**: when client invoke service, client queries the service registry to obtain the available service'instances, client can cache the service instance and use the load balancing algorithm to select to service instance
- One benefit of this pattern is it can handle the scenario when services are deployed on multi deployment platform. for example, some service deployed in K8s, but rest of other service deployed in legacy platform. if using Netflix Eureka library, it can work on both environment
- One drawback is that must use different library for different language, then it's better to use discovery mechanism provided by deployment platfrom

### 2.4.2 apply service discovery provide by deployment platform
- modern deployment platform like Docker or K8s has a built-in service discovery mechanism, they give each service virtual IP address and a DNS name that resolves the vIP.
- a service make a request to DNS/vIP, the deployment platform will route the request to an available service instance
- as a result, service registry, service discovery, and routing entirely handled by deployment platform

- this approach is combination of 2 pattern
  - 3th party registration pattern: instead of services register itself, a 3th party provide by deployment platform handle the registration
  - server-side discovery pattern: instead of client queries service-registry, it make a request to a DNS name, which resolves to a request-router, that queries the service registry and loadbalancer request
- the key benefit of this approach that all aspects of service discovery entirely handled by deployment platform, no discovery service code need implement in service, then service discovery mechanism is available for all services, without caring what language or frameworks they used
- the drawback is that the service discovery mechanism is only available inside platform 

# 3. Communication using asynchronous messaging pattern
## 3.1 Implementing the interaction styles using messaging
- a message consists of a header and body. services exchange messages over message-channels
- there is 2 type of channel: 
  - point-to-point channel: a message is delivered to exactly one of consumers that reading message from channel
  - publish/subcrive channel: a message is delivered to all attached consumers

### 3.1.1 implementing request/response and asynchronous request/respone
- core concept is client send request and service send back a reply. the different with synchronous is client doesn't expect to receive reply immediately.
- messaging is only provide asynchronous request/response. but a client can block until the reply is received. 
- when interacting by exchanging message, client attaches a correlationId and reply-channel in the message, to ensure that service can reply message to the reply-channel and the message contain correlationId same as correlationId from client to match the client's request.

### 3.1.2 implementing one-way notification
- client send a message to point-to-point channel and doesn't expect for a reply

### 3.1.3 implementing publish/subcribe
- a service publish a message to a publish/subcribe channel, and multiple consumer service can read that message

messaging and channel is a good way to design the asynchronous API, but need to choose messaging technology and determine how to implement the design. let see message broker

## 3.2 Using message broker

- A message broker is an intermediary through which all messages flow
- a sender send message to broker and broker delivers message to the consumer
- benefit : 
  - important benefit of using broker is sender does not need to know the location of consumer
  - broker can buffer the message until consumer can process them
- there are many message broker to choose:
  - some opensource broker like
    - apache kafka 
    - rabbitMQ
    - ActiveMQ
  - there are also cloud base
    - AWS kinesis
    - AWS SQS
- when select a message broker, can consider to some factor:
  - message ordering
  - supported language
  - delivery guarantees 
  - latency
  - ....
- each broker make different trade-off, a broker has very-slow latency but not preserve ordering and no guarantees the delivers message. a broker that guarantee delivery and preserve ordering can have higher latency.

### 3.2.1 implement message channel using message broker

let look at benefit 
- loose coupling: sender only need to send message to broker and doesn't care about consumer location.
- message buffering: message can be buffer in broker until consumers are available to consume the message
- flexible communication: support all interaction style

there are some downside of using message
- potential bottle neck: There is a risk that the message broker could be a performance bottleneck. Fortunately, many modern message brokers are designed to be highly scalable
- potential single point failer: the message broker should be highly available, otherwise the application reliability is impact. fortunately, modern broker can handle this
- configuration a message broker is complex

### 3.2.2 Competing receivers and message ordering

- it's common requirement that to have multiple of instances of consumer to process message concurrently, or single instance of consumer use multiple threads to process message concurrently
- challenge is that ensure the message is processed once in order
- imagine that 3 instances of consumer reading message from a channel, and sender publishes order-created, order-update, order-cancelled sequentially, but somehow a instance of consumer read order-update before order-created, then the consumer will process the message in wrong order. this is a problem
- a common solution used by modern message broker like Kafka, rabbitMQ is to use the concept of partition/sharding. there are 3 part of this solution
  - a shard channel consists of more than one partition/shard, which behave like a channel
  - sender specifies a partition/shard key in message header, broker uses the shard/partition key assign the message to a particular partition/shard (maybe hash the shard key to determine the partition/shard)
  - broker groups together instances of consumer into a consumer group, and treats them as the same logical consumer, each shard/partition is assigned to single instance of consumer.
- by that solution, for example a sender produce 3 events with same shard key, the broker will assign all 3 events to the same partition/shard, and that partition/shard is assigned to a single instance of consumer, then the consumer will process the 3 events in order. if there are more than one instance of consumer in the consumer group, the broker will assign other partition/shard to other instance of consumer, then all instances of consumer can process message concurrently, but each partition/shard is processed by a single instance of consumer.

### 3.2.3 handling duplication messages

- another challenge is duplication message when dealing with messaging, ideally broker should process messages once but its too costly, instead, most broker provide at-least-once delivery guarantee.

- let say application working normally, then messages was delivered to consumer only once, but somehow consumer process message and update db, but application crash before consumer send acknowledgement (mean message was processed) to broker, then broker will redeliver the message to consumer, and consumer will process the message again, then the db will be updated twice. this is a problem

- there are 2 solution to handle duplication message
  - **idempotent message handler**
  - **tracking message and discarding duplicates**

- idempotent message handler
  - if application logic is idempotent, then it duplication message is harmless. for example, cancel a already canceled-order is an idempotent operation. but normally, application logic is not idempotent.

- tracking message and discarding duplicates
  - a simple solution is to track the message that has been processed then discard the duplication.
  - it records the messagesId in table as a part of transaction, if messageId already existed, the insert will fail and consumer discard the message.

### 3.2.4 Transactional messaging

- a sender normally produce messages as a part of a transaction. then both database update and sending message must happen in a transaction, otherwise, if the database update is successful but the message sending fails, then the consumer will never know about the change.

- traditionally, the solution is to use a distributed transaction, which is complex and not scalable, that normally use 2-phase commit protocol, but this is not good in modern application, instead, a common solution is to use the **outbox pattern**, will check it in other chapter.
## 3.3 Using asynchronous messaging to improve availability

By choosing IPC communications, there are different trade-offs. One particular trade-off is availability. Let's see how synchronous and asynchronous communication impact availability.

---

### 3.3.1 Using synchronous communication reduces availability

The nature of synchronous communication is request/response, which means a service needs to wait for a response from another service.

**Example:**
- If we have A → B → C, and B is down, then the transaction cannot complete. A will wait for B until timeout
- When adding a new service D, response time will increase
- REST is extremely popular for synchronous communication because HTTP protocol requires the request to wait for a response
- This problem exists even when using request/response over asynchronous messaging, because it still needs to wait for a response from the message broker

---

### 3.3.2 Eliminating synchronous interaction

#### Approach 1: Use Asynchronous Interaction Style

Ideally, using asynchronous interactions for all communications allows services to:
- Asynchronously exchange messages with other services
- Get responses through message-channels
- Ensure no participant is blocked

However, services often have external APIs that use synchronous interactions (e.g., REST), which require immediate responses. In such cases, one way to improve availability is **data replication**.

#### Approach 2: Replicate Data

A service can maintain a replica of data from other services. This replica stays up-to-date by subscribing to events from other services that own the original data.

**Benefits:**
- A service can process without interacting with other services

**Drawbacks:**
- Can require replication of large amounts of data
- Does not handle how a service updates data owned by other services

One way to solve this is by delaying interaction with other services until after responding to the client.

#### Approach 3: Finishing Processing After Returning Response

A service can return a response to the client first, then interact with other services to update data in the background.

**Implementation Steps:**
1. Service handles request using local data
2. Update the database and save message to outbox table
3. Return response to client

**Example Scenario: A → B → C**

Instead of waiting for all validations:
1. Service A gets the order and updates it to `PENDING`, then responds to client immediately
2. Service A sends a message to B to validate if the order can be placed; B sends a validation response back to A
3. Service A sends a message to C to validate if the order can be placed; C sends a validation response back to A
4. Service A updates the order to `CONFIRMED` if both B and C validated the order

**Benefits:**
- Even if B or C is down, when they recover, the system can still process the order

**Drawbacks:**
- Response is returned before the order is validated
- Must have a mechanism to let the client know if the order is confirmed or not
- This adds complexity but can be considered in many situations

---

## Summary

- **Microservice architecture is a distributed system**, so IPC plays a key role
- **API evolution is essential** — A backward-compatible change is easiest to avoid impacting clients. When updating APIs, typically need to support both old and new versions until clients upgrade
- **Many IPC technologies exist, each with different trade-offs** — Choosing between synchronous (REST) or asynchronous (messaging) is a key design decision. Services should ideally use asynchronous messaging patterns to increase availability
- **When using synchronous communication**, services must have mechanisms to handle failures:
  - Timeouts
  - Limit the number of outstanding requests
  - Circuit breaker pattern to avoid calling failing services
- **Service discovery is essential** in a synchronous architecture to know the location of services in the network
- **Message and channels model** is a good way to design a messaging-based architecture
- **Atomic database updates with message sending** is a key challenge that must be handled properly