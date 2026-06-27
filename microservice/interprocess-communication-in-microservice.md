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

messaging and channel is a good way to design the asynchronous API, but need to choose messaging technology and determine how to implement the design.

## 3.2 Using message broker