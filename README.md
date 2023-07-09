## Introduction

### Design Problem Breakup
- Functional requirement: Dictate the functionality/feature of the product. Unique aspect of our system like ride sharing company, video-on-demand company, etc
- Non-functional requirement (quality attribute): Performance, scalability, reliability, etc.

Software Architecture Patterns are solutions to common problems and non-functional requirements.

### Cloud Computing Benefits

Access to (virtually) infinite:
- Computation
- Storage
- Network
- Also referred as IaaS (Infrastructure as a Service)

Cloud's pricing model charges only for:
- Services we use
- Hardware we reserve

- Removes entry barrier for new companies that would have to make large investments upfront.
- So essentially, just like any software architectural pattern, cloud computing also solves a general problem, and provides an on-demand infrastructure to any company regardless of its product.
- Limitations of cloud computing - (1) Hardware we get on-demand can be unreliable i.e. the hardware won't be newest and can freeze/die/lost connection at any given moment, (2) Evaluate costing for cloud infra, bill increases proportionately.

## Scalability Pattern

### Load Balancing Pattern

- LB pattern places a dispatcher b/w requests and the workers that process that request.
- Load balancer is used as a proxy b/w frontend and backend.
- Each service would have a dedicated load balancer/dispatcher, which can be scaled independently. In K8s, service object is LB.
- Implementation
    - Using Cloud LB Service, this technology is proprietary to cloud provider.
        - LB Algorithms - RR, Sticky session and Least connections.
            - Sticky Session / Session Affinity LB algorithm - LB tries to send traffic to same server for a given client, this can be achieved by placing cookie on client's device which will be sent with every request to LB.
            - Least Connections LB algorithm - For tasks associated with long-term connections like SQL, LDAP, etc.
        - LB can also tie it with the autoscaling metrics of the backend servers and scale those up or down depending upon that metric.
    - Using Message Broker, the main purpose of message broker is not load balancing, nevertheless in the right usecase it can do a good job at load balancing of high number of message for a particular service and that right usecase is when the communication between the publisher and the consumer service is one-directional and asynchronous meaning publisher doesn't need to get any response from the consumer after message has been published.
    
### Pipes and Filters Pattern

Terminologies -
- Data source - The origin of incoming data.
- Data sink - The final destination of the data.

Points -
- Data source can be backend service/FaaS which receives requests from client.
- Data sink can be S3 (distributed file system) or DB or any external service which listens to events from the service.
- The pipes that connect filters in this pattern are typically distributed queues or message brokers.
- Filters are isolated software components which process the data at different stages.

Problems solved by pipes and filters -
- Tight coupling - Can't use different programming languages for different tasks.
- Hardware restrictions - Each task may require a different hardware.
- Low scalability - Each task may require a different number of instances.

Example -
This pattern is used by Video sharing platform. Steps are below -

1. Splitting large video file into smaller chunks.
    - This allows downloading video chunk-by-chunk when viewing it.
2. We create thumbnail for each frame.
3. We resize each chunk to different resolutions & bitrates.
    - This allows us to use a technique called adaptive streaming.
    - With this technique we can change the video resolution dynamically while streaming the video to user depending on the network conditions.
    - For ex. when user have good connection, we can send them high quality & resolution chunks of the video & if their connection speed drops, we can switch to lower quality chunks to avoid buffering delays.
4. Chunk encoding
    - Encode each chunk in each of the diff resolutions to diff formats so we can support multiple types of devices & video players.

Note:
- We can pass audio parallely while this process is going on to get text for speech and generate captions.
- We can have another pipeline parallely to scan for copyright or inapt content.

### Scatter Gather Pattern

- We have client/requestor who sends request to system, and we have a group of workers which respond to these sender requests and in the middle, we have the dispatcher which dispatches the request from sender to worker and aggregates their results.
- Compared to LB pattern, where there are same/identical type of workers behind the LB, in scatter gather pattern, the workers are different, the workers can either be instances of the same app but have access to different data, they can be different services within our company that provide different functionality.
- The core principle of scatter gather is that each request is sent to a different worker is entirely independent of other requests sent to other worker, and because of this principle, we can perform a large number of requests completely in parallel, which has the potential to provide a large amount of info in a constant time that is completely independent of the number of workers.

UseCases
- Search service - The request is dispatched to multiple internal workers, each worker instance does some processing like performing some search on its own subset of data, the worker sends a response for aggregation on the backend/dispatcher service, the backend component might do some more processing on data and sent it to client.
- Hospitality service - We will interact with external services which has Hotel prices, we aggregate response from all of these and return to client.

Important considerations -
- Workers can become unreachable/unavailable at any moment.
    - We can ignore those workers and continue performing.
- Decoupling dispatcher from workers.
    - We can make communication b/w dispatcher and workers as asynchronous.
    
### Orchestrator Pattern

- Extension on scatter gather pattern.
- Instead of 1 operation to perform in parallel, we have a sequence of operations, some stages can be performed in parallel and others are performed sequentially.
- This pattern is frequently used in micro-services architecture like Vulcan where each service is limited to its domain and is unaware of other microservices.
- The orchestrator is responsible for handling the exceptions and retries, and maintaining the flow until it gets the final result.
- Useful for tracing and debugging issues in the flow.

Important consideration -
- Orchestrator service should keep track of progress by maintaining its own database, inside this db the state of the process can be stored for each transaction.
    - This way a new instance of orchestration service can always pick up the task at any stage even if some earlier instance of orchestration service was crashed.

### Choreography Pattern

- The orchestrator service is tightly coupled with all services, so any service making change should coordinate with other service teams when making change to make sure that flow executed by orchestrator service is not broken. This is called as distributed monolith anti-pattern. The solution to above is choreography software architecture pattern.
- In this pattern, we replace orchestrator service with distributed message queue / broker that stores all incoming events and the microservices subscribe to relevant events from the message broker where they need to get involved, once the microservice processes an incoming event, it emits a result as another event to a different channel or topic which in turn triggers another service to take action and produce another event. This chain of events continue completion of transaction.
- The analogy this pattern is using is a sequence of steps in a dance.

Advantages -
- Communication is asynchronous.
- Easily make changes to system.
- Add/remove services.
- Scale the org easily with many team with very little friction.
  
Downsides -
- No centralised orchestrator means a lot harder to troubleshoot and trace the flow of events.
    - Hard to troubleshoot.
- More complex integration tests need to be added to catch issues earlier i.e. before they go on production env.
    - The more services we have, the more challenging it becomes.

This pattern is thus more suitable for simple flows that have fewer services.

Note:
- The entire flow happens through async communication without any central entity.
- Choreography is an event driven and asynchronous communication pattern. However, we need to provide the user with an immediate response, synchronously so choose orchestrator pattern.

## Performance Patterns for Data Intensive Systems

### MapReduce

- Map/Reduce is a batch processing model i.e. on demand/on schedule execution. We pay only for the resources we use during the processing.
- The programming model of MapReduce requires us to describe each computation that we perform into 2 simple operations map and reduce.
- If we can describe the desired computation in one set of map and reduce functions, we can break it into multiple MapReduce executions and chain them together.

Steps -
- First thing we need to do as users of MapReduce is to describe our input data as a set of key value pairs. Then the MapReduce framework passes all the input key value pairs through the map function we define which produces a new set of intermediate key/value pairs.
- After that MapReduce framework shuffles and groups all those intermediate key value pairs by key and passes them to the Reduce function that we provide.
- The Reduce function then merges all the values for each such intermediate key and produces a typically smaller result of either 0 or 1 value.

Dominant factors (even for most straightforward large data computation) -
- Parallelising
- Distribution of data
- Scheduling execution
- Result aggregation
- Handling and recovering from errors


Source of overhead -
- Infrastructure to run all tasks parallely

Components -
- The general software architecture of MapReduce in 2 components - Master and Workers (Reduce and Map Workers)
- This pattern helps to process massive amounts of data in a relatively short period of time.
- Master: Computer that schedules & orchestrates the entire MapReduce computation.
- Worker Machines (Map Workers / Reduce Workers): This large set of worker machines can be in the hundreds or thousands.
- How do these components work together?
    - The master then breaks the entire set of input key value pairs into chunks.
    - This way entire data set is split across M different machines & runs the developer supplied map function on those M machines, completely in parallel.
    - Each map worker runs the map function on every key value pair that was assigned to it and periodically stores the results into some intermediate storage.
    - When the master notifies the Reduce Workers that the data is ready, the Reduce Worker takes the data from the intermediate storage, groups all same key value pairs and sorts it by key for consistency, this is called shuffling. 
    - Finally the Reduce Workers invoke the developer supplied function on each key and outputs the result to output storage.

### Saga Pattern

- Setting some context, in microservices pattern, we should have only one DB per service, otherwise if a DB is shared among more than 1 microservices then that would create dependence of one service over another and any DB schema changes (its related downtime) or DB version upgrade due to one service owning team can affect other team as well and it would be distributed monolith anti-pattern instead, therefore one DB per micro-service.
- This brings us to problem statement of Saga pattern, with one DB per micro-service, we lose the ACID Transaction properties.
    - Incase of a transaction, involving updating multiple records (now) spread across multiple DBs, we will lose the ability to perform those operations as part of a single transaction as our service owns one DB.
    - Saga pattern solves how to ensure consistency across multiple DBs for a distributed transaction?

Implementations, Saga pattern can be implemented using -
- Orchestrator pattern
    - If one of the services in the sequence fails then the orchestrator service will initiate compensating/rollback operations by calling the previous service.
- Choreography pattern
    - In this pattern, we have a message broker in middle and the services are subscribed to relevant events.
    - But in choreography pattern, we don't have a centralised service to manage the transaction, so each service is responsible for either triggering the next event in the transaction sequence or triggering a compensating event for the previous service in the sequence.
- In either of these 2 implementations, we can execute transactions:
    - That span multiple services.
    - Without having a centralised DB.

Saga pattern helps with data consistency management in microservices architecture i.e. allows performing distributed transactions across multiple DBs.

### Transactional Outbox Pattern

Problem Statement -
- Lets say we are updating the DB and publishing event to message broker sequentially. Now if lets say after updating the DB, the service crashed and we couldn't publish event to the message broker then in this case, we would miss processing for some record. To tackle this we use this transactional outbox pattern.

Solution -
- Using this pattern, we can guarantee that each DB operation is followed by an event published to a message broker (and avoid what is highlighted in problem statement)
- How?
    - We would have an additional "outbox" table in the DB.
    - Each time an update goes to the main table, we update the "outbox" table as well, as a part of transaction.
        - Similar to audit events, which are managed by DB, when some operation happen in DB.
    - There is a separate service which monitors/listens to this outbox table.

Potential issues with the pattern -
- Duplicate events
    - Can be solved if other services are idempotent, idempotency can be based on eventId field, which is associated with every event.
- Lack of support of DB transactions
    - Only possible for SQL DBs and not NoSQL DBs.
    - FYI Transaction would be to update the main table (1) and the additional "outbox" table (2).
    - For NoSQL DBs, we can add an "outbox" field in the document.
- Reordering of events
    - The service listening events should sort the events it reads (coming in batches as shouldn't be an issue if reads immediately).

### Materialised View Pattern

- Idea of this pattern is to pre-compute and pre-populate a separate table with the results of particular queries.
- This significantly improves the performance of those queries.
- It saves money on frequently repeated data operations on the cloud.

Where to store these materialised view tables?
- Option 1: In the same DB, as the original data.
    - Some DBs support the feature of materialised views out-of-the-box.
    - This way our view will get updated every time there is update in original data in base table changes, without the need to be do it manually or programmatically. We can control the frequency of updation of this materialised view.
- Option 2: Store in a separate, read-optimised DB like cache.
    - This would have to be done programatically.

### CQRS Pattern

- CQRS - Command and Query Responsibility Segregation

CQRS Terminology (We can group the DB operations into following) -
- Command - An action we perform that results in data mutation. Examples: Add a user, Update a review, Delete a comment, etc.
- Query - Only reads data and returns to the caller. Examples: Show all products in a category.

CQRS pattern has 2 main purposes -
- The first is related to materialised view pattern.
- Can be used in event sourcing as well.

How?
- The problem that CQRS solves w.r.t materialised view is generic. CQRS pattern takes the same concept of materialised view but a step further, with CQRS we have separate DB and services for Command (Business logic, validation, permission, etc.) and Query service (Query logic), each DB can be evolved independently. This way we can choose the write optimised DB for Command operations and read optimised DB for Query operations.
- This way we can optimise our system for both type of operations, this is particularly important when we have frequent writes and reads.
- To keep the command and query DB in sync, every time a data mutation request is received by the command part, we publish an event which Query side can consume and act accordingly.

CQRS Drawbacks -
- We can only guarantee eventual consistency b/w the command and query DB.
    - If the requirement is for strong consistency then this pattern is not right.
- It adds overhead and complexity to the system.
    - Now we have 2 DBs and 2 separate services.
    - We have message broker/cloud function for synchronising command and query DB.
    - So its important that the performance benefits we get from using CQRS are well are worth this overhead otherwise we should keep things simple and have 1 DB and 1 service.

#### CQRS + Materialised View Pattern

Example: Courses service, reviews service, and course search service (query service). 

- The course search service would maintain the materialised view table for both course and its reviews.
- To achieve this, whenever an update is done to courses or reviews service, events are emitted to the message broker, which are from there, consumed by the course search service and it updates the materialised view table it maintains.

Here,
- Materialised view: We place the joined data in a separate DB.
- CQRS: We store the materialised view in separate DBs, each behind its own microservice.
- Event driven architecture: We keep the external materialised view up-to-date with the original data.

### Event Sourcing Pattern
    
- In this pattern, instead of storing the current state, we store the events.
- Events are immutable. This way complete history would be available for reporting and auditing. This is important for financial purposes / banking systems.

Where to store events?
- DB - Separate record in DB.
- For example - In online store use-case, we can store every new order in the orders table and every time the order status changes, we append a new row with same order id, its new state and timestamp.

Benefit?
- An added benefit to using event sourcing is write performance.
- Without event sourcing, if we have a write intensive workload, we get high contention over the same entities in a DB which typically result in poor performance.
- As an example, we may have an inventory table that keeps track of current inventory of each product. 
    - If we have many concurrent updates to the same table or the same product, then all the updates with contend with each other and slow down each other as well as all the readers of that table. 
    - On the other hand, if we move to event sourcing, then each purchase or return becomes an append only event, which doesn’t require any DB locking and is a lot more efficient.
    
## Software Extensibility Architecture Patterns

### Sidecar and Ambassador Pattern

- In this pattern, we take the additional functionality required by the application and run it as a separate container on the same server/host machine as the main application container.
- So we get benefit of isolation between the main service instance and the sidecar process but at the same time, they are sharing the same host, so the communication b/w them is very fast and reliable.
- Since sidecar and core app are running together, they both have access to the same resources like the file system, CPU or memory. This way the sidecar can do things like monitor the host CPU or memory and report it on main application's behalf. The sidecar can also read the application log files and update its configuration files without the need of any network communication.

Ambassador sidecar -
- It is a special type of sidecar which is responsible for sending all the network requests on behalf of the service.
- It is like a proxy that runs on the same host as the core application.
- It handles the retries, disconnections, authentication, routing, etc like istio-proxy.

### Anti-Corruption Adapter Pattern

- To ensure new system not corrupted by old system.
- In anti-corruption pattern, we deploy a new service between the old and the new system that acts as an adapter between them, the new part of our system talks to the anti-corruption service using only new api, tech and data models, the anti-corruption service performs all translations and between the old and new systems.

Use-cases -
- During migration.
    - Migration - Anti-corruption layer is temporary.
    - Two part system - Anti-corruption layer is permanent.

Anti-corruption service -
- Performance overhead (latency)
- Additional cost on cloud environment
- Need to ensure its maintenance, development, testing, deployment and scaling

### Backends for Frontends Pattern

When there is a common service catering to different type of frontend then this pattern/ideology can be used to split the backend logically into multiple services each to cater to a specific type of frontend so as to give a great UX to each type of frontend users rather than sub-optimal experience for every frontend.

Challenges -
- Share common logic.
    - There must be some common logic being used across these frontend specific backends which can be extracted into a shared library, since this library affects every service where it is used, therefore changes to this library needs to be coordinated.
- Granularity of backends i.e. till what level should segregation be done.

## Reliability, Error Handling and Recovery Software Architecture Patterns

### Throttling and Rate Limiting Pattern

Why?
- Overconsumption. We can end up in 2 scenarios.
- Our system becomes unable to handle this high rate of incoming requests or data because of running out of CPU or memory or both which results in slowness in entire system. We need to protect our systems against traffic spikes.
- Our system is able to scale out properly using auto-scaling policies but scale out results in costing us way more money than we anticipated. Overconsumption of external resources/APIs may end up costing us a lot of money.
- So regardless if the client had malicious intentions or a legitimate need to call our API at such a high rate, we need to protect our system against traffic spike.
- We can use throttling and rate-limiting pattern in such scenarios.

What?
- In throttling and rate-limiting pattern, we set a limit on the number of requests that can be made during a period of time, this limit could also be set on data/bandwidth consumed as well. This limit can be set for a period of one second, one minute, one day and so on.
- Scenario where we are service providers and need to protect our system from overconsumption is referred as server side throttling.
- Scenario where in we are the clients calling external services and we want to protect ourselves from overspending on the services, that scenario is called as client-side throttling.

What should we do if some client of ours exceeds the limit we set for them?
- There are few strategies we can use depending on the use-case.
- Dropping requests
    - Server would sent back a response with an error code that indicates the reason why this request was rejected.
    - If we use HTTP for communication then the status code for 429 could be used.
- Queue the requests and process them later
- Reduce bandwidth/quality of stream incase request queueing is not possible

Important considerations -
- Customer based throttling vs Global (API) level throttling
- External throttling (server-side throttling) vs service based throttling (client-side throttling)

The best approach depends on -
- Usecase
- Requirements

### Retry Pattern

Software/Hardware/Network errors introduce -
- Delays
- Timeouts
- Failures

Introduction -
- As part of retry process, we need to classify whether error is a user error or a system error. 
    - Example - 403 http error is user error i.e. user is not authorised to perform or access a particular resource. In user errors, we should send error back to the user with as much info as possible and would help user in correcting it.
- For internal system error, we need to try our best to hide the error from the user and attempt to recover from it if possible. 
- One way to recover from it is using retry pattern.

Retry considerations -
- Which errors to retry? Like 503 http status, which is temporary and recoverable.
- What delay/backoff strategy to use?
    - Delay needs to added b/w subsequent retries so that faulty service is recovered.
    - Strategies
        - Fixed delay 
        - Incremental delay
        - Exponential backoff 
        - Adding random jitter b/w retries
- How many times to retry? And how much time?
- Is the operation idempotent?
    - Retrying idempotent operation is safe but not non-idempotent operation.
- Where to imply retry logic?
    - Implement as library which different service can use or moving logic entirely to sidecar.

Note: The approach you choose depends on your system.

### Circuit Breaker

Problem Statement -
- The last n requests to the image service in the last minute failed.
    - Should we keep retrying and holding the user?
    - Save resources and NOT retry?

Introduction -
- Retry pattern is an optimistic approach, assuming if the first request failed then next will succeed.
- Circuit breaker pattern is a pessimistic approach, since the errors it handles are more severe and long-lasting, and assumes that if first few request then next request would likely fail as well, so there is no point in retrying.
- This pattern follows electric circuit breaker analogy where a closed circuit means that electricity is flowing normally and the system in healthy. And when there is a power surge, the circuit breaker opens the circuit and stops the electricity from flowing b/w different system components.

What ?
- The circuit breaker keeps track of the number of successful and failed requests for any given period of time. As long as the failure rate stays low, the circuit remains closed, and every request from our service to the external service is allowed to go through.
- However if the failure rate exceeds a certain threshold then the circuit breaker trips and goes into an open state. In the open state, it stops any request to go through and returns an error or throws an exception immediately to the caller. This way we can save the time waiting for the external service to respond and also the cpu and n/w for making calls to the faulty service.
- If we stop sending requests to faulty external service how will we know it is healthy? For this, the circuit breaker has a half-open state i.e. after being in open state for sometime, the circuit breaker automatically transitions to the half-open state where it allows a small % of requests to go through and be sent to the faulty remote service.
- This small % of requests act as a sample to probe the state of faulty external service, if the success rate of those requests that do go through is high enough then CB assumes the external service has recovered and transitions back to the closed state. Otherwise, if the success rate of those sample requests is still low then the CB assumes that the remote service is still unhealthy and transitions back to the open state. This process will be repeated until the success rate is high enough for the circuit to go into closed state.

Circuit breaker important considerations -
- What to do with the outgoing request to the remote service when the CB is open?
    - Drop it (with proper logging)
    - Log and replay
- What response should we provide to the caller?
    - Fail silently
        - Provide a placeholder/fallback response
    - Best effort
        - Check in cache if the response for similar request was cached earlier
- Separate circuit breaker for each remote service, so that if CB is open for one remote service, it doesn't affect other remote services.
- Replace the half-open state with async pings/health checks to the external service. This has a few benefits - we won't be sending real requests when the CB is in half-open state, these health-check requests are very small and do not contain any payload which isn't the case with most real requests, so by sending pings instead of real requests, we save on n/w bandwidth, cpu and memory resources.
- Where to imply CB logic?
    - Implement as library which different service can use or move logic entirely to sidecar.

### Dead Letter Queue (DLQ)

Event driven architectures -
- Decoupling producers from consumers.
- Greater scalability.
- Async communication.

DLQ -
- Special queue in a message broker for messages that cannot be delivered to their destination.
- 2 ways to send message to DLQ:
    - Programmatic publishing
        - Can be published by either producer or by consumer if unable to process.
    - Automatic transfer from original queue
        - If there are delivery errors for a message then it would be sent automatically to DLQ by the message broker or if a message stays in message broker for too long then also.
        - Requires this support from message broker.
- The messages in DLQ can be processed to fix and republish.

## Deployment and Production Testing Patterns

### Rolling Deployment Pattern

What?
- We use the LB to stop sending traffic to the app servers one at a time.
- Once no more traffic is going to a particular server, we can stop our app instance on it and deploy new version of app.
- We can then add the server back to the load balancers group of backend servers and then  we repeat the same process on all the servers until all of them are on the latest version.

Benefit -
- No downtime
- No additional cost of hardware
- We can rollback quickly if something goes wrong

### Blue-Green Deployment Pattern

What?
- In this deployment pattern, we keep the old version of our app instance running throughout the entire duration of release. The old version is referred to as the blue environment.
- At the same time, we add a new set of servers which is called the Green environment and deploy the new version of our app on those servers.
- After we verified that the new instances started up just fine and maybe even run a few tests on them, we use the LB to shift traffic from the blue environment to the green environment. If during this transition, we start seeing issues in the logs or in our monitoring dashboards, we can easily stop the release and shift the traffic back to the blue environment which is running the old version of app.
- Otherwise, we fully transfer the traffic to the new version, wait for a bit to make sure everything is okay and then we can shutdown the old/blue environment or keep it available for the next release cycle.

Advantage
- We have an equal number of servers for both blue and green environment, so if all of sudden server start failing in green environment, we can easily switch back to the blue/old environment which can take the load right away.
- We can have a single version of software at any given moment, which gives all users the same experience, except for maybe a very short period of transition that should be generally unnoticeable.

Downside
- During deployment of new release, we have to run twice as many servers.
    - If we use those additional servers only during our release then the cost may not be that high.

### Canary Release and A/B Testing Deployment Patterns

Canary deployment
- Instead of deploying the new app version to a new of servers, we dedicate a small subset of the existing group of servers and update them with the new version of software right way.
- We monitor the performance of the canary version and compare it in real time to the performance of rest of the servers which run the old version.
- Once we gain confidence with the new version and we see no degradation in functionality or performance, we can proceed with the release and update rest of the servers with the new version.

Downside
- Setting clear success criteria for the release so we can automate the monitoring, otherwise the engineer will have to look at lot of graphs to decide whether we should roll out the release globally or roll back.

A/B deployment
- Purpose is different compared to canary release deployment, when we do the canary release our goal is to safely release a new version of our app eventually to entire group of servers.
- On the other hand, the purpose of AB testing is to test a new feature on a portion of our users in production, this info that we gather from this comparison can inform our product or business team towards future work or features.
- And unlike canary release, after A/B testing is finished, the experimental version of our software is removed and replaced with previous version.
- The metrics gathered during the experiment can be assessed to see if that change should be released as part of future release.

In both canary and A/B deployment we dedicate a small portion of servers for a different version of software.

### Chaos Engineering Pattern

- Herein we inject failures deliberately in the system in a controlled way, we can find single point of failures, scalability issues, performance bottlenecks in our system.
- This helps to ensure that real failures happen then logic is there in place to handle the situation gracefully.

Important consideration
- Minimise blast radius of failures we create i.e. we stay well within our error budget when injecting failures.
