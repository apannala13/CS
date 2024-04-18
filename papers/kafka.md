# Kafka Notes:

- Kafka is a distributed messaging system that was developed for collecting and delivering high volumes of log data with low latency.
- There is a large amount of “log” data generated at any sizable internet company.
    - user activity events corresponding to logins, pageviews, clicks, “likes”, sharing, comments, and search queries;
    - operational metrics such as service call stack, call latency, errors, and system metrics such as CPU, memory, network, or disk utilization on each machine.
    - Ex use cases for log data for site features: search relevance, item recommendations, ad targeting and reporting, security applications for spam/data scraping, and newsfeed features.
- Many early systems for processing this kind of data relied on physically scraping log files off production servers for analysis
- Kafka is a distributed, scalable and offers high throughput. Provides api to consume messages in real-time.
- Many traditional messaging systems are not a good fit for log processing:
    - they focus more on delivery guarantees, which are overkill for collecting log data.
    - also do not focus as strongly on throughput.
    - weak in distributed support.  no easy way to partition and store messages across machines.
    - assume near immediate consumption of messages, so the queue of unconsumed messages is always fairly small. performance degrades significantly if messages are allowed to accumulate, e.g. data warehouses that consume large loads of data in batches.
- Kafka uses a “pull” model - more suitable for applications since each consumer can retrieve the messages at the maximum rate it can sustain and avoid being flooded by messages pushed faster than it can handle.
- Architecture:
    - A stream of messages of a particular type is defined by a topic. A producer can publish messages to a topic, where they’re split into partitions.
    - The topics are then stored at a set of servers called brokers.
    - A consumer can subscribe to one or more topics from the brokers, and consume the subscribed messages by pulling data from the brokers.
    - To subscribe to a topic, a consumer first creates one or more message streams for the topic. The messages published to that topic will be evenly distributed into these sub-streams.
    - Each message stream provides an iterator interface over the continual stream of messages being produced. The consumer then iterates over every message in the stream and processes the payload of the message. Unlike traditional iterators, the message stream iterator never terminates.
    - If there are currently no more messages to consume, the iterator blocks until new messages are published to the topic.
    - Kafka supports both the point-to-point delivery model in which multiple consumers jointly consume a single copy of all messages in a topic, as well as the publish/subscribe model in which multiple consumers each retrieve its own copy of a topic.
- Since Kafka is distributed in nature, an Kafka cluster typically consists of multiple brokers. To balance load, a topic is divided into multiple partitions and each broker stores one or more of those partitions.
    - Multiple producers and consumers can publish and retrieve messages at the same time.
- Each partition of a topic corresponds to a logical log. Physically, a log is implemented as a set of segment files of approximately the same size (e.g., 1GB).
    - Every time a producer publishes a message to a partition, the broker appends the message to the last segment file.
    - Segment files are flushed to disk after a configurable number of messages have been published or a certain amount of time has elapsed. A message is exposed to consumer after it has been flushed.
- Kafka messages dont have message id’s. Instead, a message is addressed by its logical offset in the log.
    - Avoids overhead of maintaining auxiliary, seek-intensive random-access index structures that map the message ids to the actual message locations.
- A consumer always consumes messages from a particular partition sequentially.
    - If the consumer acknowledges a particular message offset, it implies that the consumer has received all messages prior to that offset in the partition.
    - Under the hood, the consumer issues asynchronous pull requests to the broker to have a buffer of data ready for the app to consume.
        - Each pull request contains the offset of the message from which the
        consumption begins and an acceptable number of bytes to fetch.
        - Each broker keeps in memory a sorted list of offsets, including the
        offset of the first message in every segment file.
        - The broker locates the segment file where the requested message resides by
        searching the offset list, and sends the data back to the consumer.
        After a consumer receives a message, it computes the offset of the
        next message to consume and uses it in the next pull request.
- Although the producer can submit a set of messages in a single send request, under the covers, each pull request from a consumer also retrieves multiple messages up to a certain size.
- Kafka does not cache messages in memory. Instead, it relies on the underlying file system page cache:
    - The page cache is ***the main disk cache used by the Linux kernel***. In most cases, the kernel refers to the page cache when reading from or writing to disk.
    - The benefit is that double buffering is avoided (temporary ***storage*** area in the main ***memory*** that allows ***data*** to be stored while it is being transferred).
    - This retains warm cache even when a broker process is restarted. (Warm cache is when the cache has started receiving requests and has begun retrieving objects and filling itself up)
    - Since Kafka doesn’t cache messages in process at all, it has very little overhead
    in garbage collecting its memory, making efficient implementation in a VM-based language feasible
    - Since the producer and consumer both access the segment files sequentially, with the consumer often lagging the producer by a small amount, write-through caching and read-ahead caching are effective.
        - Write through: the system's processor writes data to the cache first and then immediately copies that new cache data to the corresponding memory or disk.
        - Read ahead: system call of the Linux kernel that loads a file's contents into the page cache
- Kafka is a multi-subscriber system and a single message may be consumed multiple times by different consumer applications.  A typical approach to sending bytes from a local file to a remote socket involves the following steps:
    - read data from the storage media to the page cache in an OS
    - copy data in the page cache to an application buffer
    - copy application buffer to another kernel buffer
    - send the kernel buffer to the socket
    - This includes 4 data copying and 2 system calls.
    - Kafka exploits the Linux Sendfile API to efficiently deliver bytes in a log segment file from a broker to a consumer.
- The kafka broker is stateless. The information about how much each consumer has consumed is maintained by the consumer itself, not the broker.
    - However, this makes it tricky to delete a message, since a broker doesn’t know whether all subscribers have consumed the message.
    - Kafka uses a simple time-based SLA for this; A message is automatically deleted if it has been retained in the broker longer than a certain period, typically 7 days.
    - A consumer can deliberately *rewind* back to an old offset and re-consume data. For example, when there is an error in application logic in the consumer, the application can re-play certain messages after the error is fixed. Easier to support in pull model > push.
- Each producer can publish a message to either a randomly selected partition or a partition semantically determined by a partitioning key and a partitioning function.
    - Consumer groups consists of one or more consumers that consume a set of subscribed topics, i.e., each message is delivered to only one of the consumers within the group.
    - Different consumer groups each independently consume the full set of subscribed messages and no coordination is needed across consumer groups. It is best to evenly divide the messages in the brokers to the consumers without too much coordination overhead.
    - A partition within a topic is the smallest unit of parallelism. A any given time, all messages from one partition are consumed only by a single consumer within each consumer group.
        - processes only need coordinate when the consumers rebalance the load, an infrequent event.
    - If  multiple consumers were to simultaneously consume a single partition, they would have to coordinate who consumes what messages, which necessitates locking and state maintenance overhead.
- Kafka does not have a central master node, but rather the nodes coordinate amongst themselves in a decentralized fashion, using ZooKeeper, which has a simple API:
    - One can create a path, set the value of a path, read the value of a path, delete a path, and list the children of a path.
    - one can register a watcher on a path and get notified when the children
    of a path or the value of a path has changed;
    - a path can be created as ephemeral (as oppose to persistent), which means that
    if the creating client is gone, the path is automatically removed by
    the Zookeeper server;
    - zookeeper replicates its data to multiple servers, emphasizing reliability & availability
- Kafka uses ZooKeeper for multiple tasks:
    - detecting the addition and the removal of brokers and consumers
    - triggering a rebalance process in each consumer when the above events
    happen
    - maintaining the consumption relationship and keeping track of the consumed offset of each partition
- When each broker or consumer starts up, it stores its information in a broker or consumer registry in ZooKeeper.
    - The broker registry contains the brokers host and port name and set of topics and partitions stored on it
    - The consumer registry includes the consumer group to which a consumer belongs and the
    set of topics that it subscribes to.
    - Each consumer group is associated with an ownership registry and offset registry in zookeeper. The ownership registry has one path for every subscribed partition and the path value is the id of the consumer currently consuming from this partition.
    - The paths created in Zookeeper are ephemeral for the broker registry, the consumer registry and the ownership registry, and persistent for the offset registry. If a broker fails, all partitions on it are automatically removed from the broker registry.
    - The failure of a consumer causes it to lose its entry in the consumer registry and all partitions that it owns in the ownership registry. Each consumer registers a Zookeeper watcher on both the broker registry and the consumer registry, and will be notified whenever a change in the broker set or the consumer group occurs.
    - During the initial startup of a consumer or when the consumer is notified about a broker/consumer change through the watcher, the consumer initiates a rebalance process to determine the new subset of partitions that it should consume from.
    - When there are multiple consumers within a group, each of them will be notified of a broker or a consumer change.
        - However, the notification may come at slightly different times at the consumers.
        - it is possible that one consumer tries to take ownership of a partition still owned by another consumer.
        - When this happens, the first consumer simply releases all the partitions that it currently owns, waits a bit and retries the rebalance process. Stabilizes after a few retries.
    - When a new consumer group is created, no offsets are available in the offset registry. In this case, the consumers will begin with either the smallest or the largest offset (depending on a configuration) available on each subscribed partition, using an API on the brokers.
- Kafka only guarantees at-least-once delivery.
