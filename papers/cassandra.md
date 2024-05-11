# Cassandra Notes

- Cassandra is a distributed storage system for managing very large amounts of structured data spread out across many commodity servers, while providing highly available service with no single point of failure.
- Aims to run on top of an infrastructure of hundreds of nodes (possibly spread out across different data centers). At this scale, small and large components fail continuously.
- Cassandra does not support a full relational data model
- Facebook runs the largest social networking platform that serves hundreds of millions users at peak times using tens of thousands of servers located in many data centers around the world.
    - There are strict operational requirements on Facebook’s platform in terms of performance, reliability and efficiency, and to support continuous growth the platform needs to be highly scalable.
    - there are always a small but significant number of server and network components that are failing at any given time
- Cassandra was designed to fulfill the storage needs of the Inbox Search problem. Inbox Search is a feature that enables users to search through their Facebook Inbox. At Facebook this meant the system was required to handle a very high write throughput, billions of writes per day, and also scale with the number of users.
    - Since users are served from data centers that are geographically distributed, being able to replicate data across data centers was key to keep search latencies down.
-
