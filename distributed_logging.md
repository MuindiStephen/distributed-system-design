# Designing Distributed Logging System

One of the most challenging aspects of debugging distributed systems is understanding system behavior in the period leading up to a bug.
As we all know by now, a distributed system is made up of microservices calling each other to complete an operation.
Multiple services can talk to each other to complete a single business requirement.

In this architecture, logs are accumulated in each machine running the microservice. A single microservice can also be deployed to hundreds of nodes. In an archirectural setup where multiple microservices are interdependent, and failure of one service can result in failures of other services. If we do not have well organized logging, we might not be able to determine the root cause of failure.

## Understanding the system
### Restrain Log Size
At any given time, the distributed system logs hundreds of concurrent messages. 
The number of logs increases over time. But, not all logs are important enough to be logged.
To solve this, logs have to be structured. We need to decide what to log into the system on the application or logging level.

### Log sampling
Storage and processing resources is a constraint. We must determine which messages we should log into the system so as to control volume of logs generated.

High-throughput systems will emit lots of messages from the same set of events. Instead of logging all the messages, we can use a sampler service that only logs a smaller set of messages from a larger chunk. The sampler service can use various sampling algorithms such as adaptive and priority sampling to log events. For large systems with thousands of microservices and billions of events per seconds, an appropriate 

### Structured logging
The first benefit of structured logs is better interoperability between log readers and writers.
Use structured logging to make the job of log processing system easier. 

### Categorization
The following severity levels are commonly used in logging:
- `DEBUG`
- `INFO`
- `WARNING`
- `ERROR`
- `CRITICAL`

## Requirements
### Functional requirements
- Writing logs: the microservices should be able to write into the logging system.
- Query-based logs: It should be effortless to search for logs.
- The logs should reside in distributed storage for easy access.
- The logging mechanism should be secure and not vulnerable. Access to logs should be for authenticated users and necessary read-only permissions granted to everyone.
- The system should avoid logging sensitive information like credit cards numbers, passwords, and so on.
- Since logging is a I/O-heavy operation, the system should avoid logging excessive information. Logging all information is unnecessary. It only takes up more space and impacts performance.
- Avoid logging personally identifiable information (PII) such as names, addresses, emails, etc.


### Non-functional requirements
- **Low latency:** logging is a resource-intensive operation that's significantly slower than CPU operations. To ensure low latency, the logging system should be designed so that logging does not block or delay a service's main application process.
- **Scalability:** Our logging system must be scalable. It should efficiently handle increasing log volumes over time and support a growing number of concurrent users.
- **Availability:** The logging system should be highly available to ensure data is consistently logged without interruption.

## Components to use
We will use the following components:
- **Pub-Sub system:** we will use a publish-subscribe system to efficiently handle the large volume of logs.
- **Distributed Search:** we will employ distributed search to query logs efficiently.

>A distributed search system is a search architecture that efficiently handles large dataset and high query loads by spreading search operations across multiple servers or nodes. It has the following components:
>1. **Crawler:** This component fetches the content and creates documents.
>2. **Indexer:** Builds a searchable index from the fetched documents.
>3. **Searcher:** Responds to user queries by running searches on the index created by the indexer.

- **Logging Accumulator:** This component will collect logs from each node and store them in a central location, allowing for easy retrieval of logs related to specific events without needing to query each individual node.
- **Blob Storage:** The blob storage provides a scalable and reliable storage for large volumes of data.
- **Log Indexer:** Due to the increasing number of log files, efficient searching is crucial. The log indexer utilizes distributed search techniques to index and make logs searchable, ensuring fast retrieval of relevant information.
- **Visualizer:** The visualizer component provides a unified view of all logs. It enables users to analyze and monitor system behavior and performance through visual representation and analytics.


## API Design
We design for reads and writes


Read
```python
searching(keyword)
```
This API call returns a list of logs that contain the keyword.

Write
```python
write_logs(unique_ID, message)
```
This API call writes the log message against against a unique key.


## High Level System Design

![](images/distributed_logging_design.png)

## Component Design 

### Logging at Various Levels in a Server
In a server environment, logging occurs across various services and application, each producing logs crucial for monitoring and troubleshooting.

#### Server Level
- **Multiple Applications:** A server hosts multiple apps, such as App1, App2, etc. Each running various microservices with user authentication, fetching the cart, storage etc in an e-commerce context.
- **Logging Structure:** each service within the application generates logs identified by an ID conprising application ID, service ID, and timestamp, ensuring unique identification and event causality determination.

#### Logging Process
Each service will push logs into the log accumulator service.
The service will store the logs logically and push the logs to a pub-sub system.

We use the pub-sub system to handle scalability challenge by efficiently managing and distributing a large volume of logs across the system.

#### Ensuring Low Latency and Performance
- **Asynchronous Logging:** Logs are sent asynchronously via low-priority threads to avoid impacting the performance of critical processes. This also ensure continuous availability of services without any disruptions caused by logging activities.
- **Data Loss Awareness:** Logging large volumes of messsages can lead to potential data loss. To balance user-perceived latency with data peristence, log services often use RAM and save data asynchronously. To minimize data loss, we will add more log accumulators to handle increasing concurrent users.

#### Log Retention
Logs also have an expiration date. We can delete regular logs after a few days or months. Comliance logs are usually stored for up to five years. If depends on the requirements of the application.

Another crucial component therefore is to have an expiration checker. It will verity the logs that have to be deleted

### Data Center Level
All servers in the data center transmit logs to a publish-subscribe architecture.
By utilizing a horizontally-scalable pub-sub framework, we can effectively manager large log volumes. 

Implementing multiple pub-sub instance within each data center enhances scalability and prevents throughput limitations and bottlenecks. Subsequently, the pub-sub system routes the log data to blob storage.

![](images/distributed_logging_datacenter_level.png)

Now, data in the pub-sub system is temporary and get deleted after a few days before being moved to archival storage. 
However, while the data is still present in the pub-sub system, we can utilize it using the following services:
- **Alerts Service:** This service identifies alerts and errors and notifies the appropriate stakeholders if a critical error is detected, or sends a message to a monitoring tool, ensuring timely awareness of important alerts. The service will also monitor logs for suspicious activities or security incidents, triggering alerts or automated responses to mitigate threats.
- **Analytics service:** This service analyzes trends and patterns in the logged data to provide insights into system perf, user behavior, or operational metrics.
