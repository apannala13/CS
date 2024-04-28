# distributed message queue

Background:

- Let's say there are two web services, a producer and consumer. They need to communicate with each other:
    - Option 1: synchronous communication.
        - Producer makes a call to consumer and waits for a response.
        - Synchronous communication is easier and faster to implement, but harder to deal with consumer service failures.
    - Option 2:
        - Asynchronous communication: Producer sends data to a **queue**, and one consumer gets this data a short time after.
        - Queue vs Topic: in topics, published messages are sent to every subscriber. In a queue, messages are received by only one consumer.
- Need to sync how to retry failed requests, how to not overwhelm requests, and how to deal with slow consumer service host.

Functional requirements:

Core APIs:

- sendMessage(body)
- getMessage(body)
- Other:
    - createAndDelete(body)
    - system needs to avoid duplicate submissions
    - security requirements
    - ordering guarantees

Nonfunctional requirements:

- Scalable (handles load increases, more queues and messages)
- Highly available (survives hardware/network failures)
- Highly performant (single digit latency for main operations).
- Other:
    - Specific SLAs:
        - Minimum throughput
        - Minimum latency
        - System needs to support hardware costs

High Level Architecture:

[![Untitled](distributed%20message%20queue%20ca75678f1c7f48a68c9a6ada824ea815/Untitled.png)
](https://github.com/apannala13/images/blob/main/architecture.jpg)

- A l**oad balancer** is a reverse proxy. It presents a **virtual IP address** (VIP) representing the application to the client. The client connects to the VIP and the load balancer makes a determination through its algorithms to send the connection to a specific application instance on a server. Load balancers help achieve high throughput and availability:
- When domain name is hit, the request is transferred to one of the VIPs registered in the domain name system (DNS) for our domain name.
- VIP is resolved to a load balancer device which has a knowledge of front end hosts.
- What happens if a load balancer device goes down? Load balancers also have limits of number of requests and bytes they can transfer. What happens when load balancer limits are reached?
    - To address high availability, load balancers use a concept of primary and secondary nodes. The primary node accepts connections and serves requests while the secondary node monitors the primary. If primary node can't accept connections, secondary takes over.
    - For scalability concerns, multiple VIPs or VIP partitioning can be utilized. For example, in DNS, multiple A records can be assigned to the same DNS for the service. As a result, requests are partitioned across several load balancers, which can be spread across multiple data centers.

Front End:

- Front end is a lightweight web service, which consists of stateless machines deployed across several data centers.
- Responsible for:
    - Request validation:
        - Helps ensure all required params are present in request and values of params honor constraints and data falls within an acceptable range:
            - E.g. For our queue, every queue name should comes with every send msg request, or message size doesn't exceed specific threshold.
    - Authentication/Authorization:
        - Authentication is the process of validating the identity of a user or service, and authorization determines whether or not a specific actor is permitted to take a certain action.
            - E.g., For authentication, check that message sender is a registered customer of distributed queue service, for authorization, verify that sender is authorized to publish messages to the queue it claims.
    - TLS (SSL termination):
        - Protocol that aims to provide privacy and data integrity. Refers to the process of decrypting encrypted traffic (HTTPS) at a network endpoint, such as a load balancer or reverse proxy, and forwarding the decrypted traffic to the destination server/application.
        - SSL on load balancer is expensive
        - Termination is usually handled by not a FrontEnd service itself, but a separate TLS HTTP Proxy that runs as a process on the same host.
    - Server-Side Encryption:
        - Messages are stored in encrypted form as soon as Front End receives them.
        - Messages are stored in encrypted form and Front end decrypts messages only when they are sent back to a consumer.
    - Caching:
        - Stores copies of source data.
        - Helps reduce load to backend services, increases overall system throughput and availability, and decreases latency.
        - Stores metadata info about most actively used queues. Also stores user identity information to save on calls to auth services.
    - Rate Limiting/Throttling:
        - Throttling is the process of limiting number of requests you can submit to a given operation in a given amount of time.
        - Throttling protects web service from being overwhelmed with requests.
        - Leaky bucket algorithm is one of the most famous: [https://www.linkedin.com/advice/0/what-benefits-drawbacks-using-token-bucket-leaky](https://www.linkedin.com/advice/0/what-benefits-drawbacks-using-token-bucket-leaky)
    - Request dispatching:
        - Responsible for sending all activities associated with sending requests to backend services (client management, response handling, resource isolation, etc.)
        - Bulkhead pattern helps to isolate elements of an app into pools so that if one fails, the others will continue to function.
        - Circuit breaker pattern prevents an app from repeatedly trying to execute an operation thats likely to fail. [https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
    - Request deduplication:
        - May occur when a response from a successful sendMessage request failed to reach a client.
        - Lesser an issue for ‘at least once’ delivery semantics, a bigger issue for ‘exactly once’ and ‘at most once’ delivery semantics.
        - Caching is usually used to store previously seen request ids.
    - Usage data collection:
        - Gather real time information that can be used for audit and billing (invoices).

Metadata Service:

- The metadata service acts a caching layer between the front end and a persistent storage
- Many reads, little writes
- Strong consistency storage is preferred but not required
- Options for organizing cache clusters:
    1. Cache is relatively  small and we can store the whole dataset on every cluster node. 
        1. Front end host randomly calls a chosen metadata service host, because all the cache clusters contain the same data.
        2. Can introduce a load balancer between front end and metadata service, as all metadata service hosts are equal.
    2. Partition data into small chunks (shards). 
        1. Dataset is too big and cannot be placed in memory of a single host. Store each chunk of data on a separate node in a cluster. Front end knows which shard stores the data and calls the shard directly.
        2. Consistent hashing 
    3. Partition data into shards, but front end doesn't know on what shard data is stored. 
        1. Front end calls a random metadata service host, and host itself knows where to forward the request to.
        2. Consistent hashing

Back End:

- Example questions:
    - where and how do we store messages?
        - database is an option but since we are building a message queue with high throughput, all the throughput will be offloaded to the database. Could use a highly available and scalable db. Not recommended.
        - Answer: RAM and local disk of a backend host.
    - How do we replicate data?
        - Send copies of messages to other hosts
    - How does front end select a backend host to send data to? To retrieve data from?
        - Leverage metadata service
- Summary:
    - Message comes to front end
    - front end consults metadata service to find what backend host to send data to
    - message is sent to a selected backend host and data is replicated
    - when Get message call comes, front end talks to metadata service to identify a backend host that stores the data.
