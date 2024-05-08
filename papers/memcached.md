# Memcached Notes

- A social network’s infrastructure needs to:
    - allow near realtime communication,
    - aggregate content on-the-fly from multiple sources,
    - be able to access and update very popular shared content,
    - scale to process millions of user requests per second.
- Users consume an order of magnitude more than they create; results in a workload dominated by fetching data and suggests that caching can have significant advantages. Facebook’s read operations fetch data from a variety of sources such as MySQL dbs, HDFS installations and backend services.
- Memcached provides a simple set of operations (get, set, delete). As, memcached provides no server to server coordination; it is an in memory hash table running on a single server. It was scaled by facebook.sy
- Memcached is used to lighten read load on databases. Memcached is used as a demand-filled look-aside cache. When a webserver needs data, it first requests the value from memcached by providing a single string key.
    - If the item addressed by that key is not cached, the web sever retrieves the data from the database or other backend service and populates the cache with the key val pair.
    - For write requests, the web server issues SQL statements to the database and then sends a delete request to memcache that invalidates any stale data. Chose to delete delete cached data instead of updating it because deletes are idempotent.
    - Separating caching layer from persistence layer also allows to adjust each layer independently as workload changes.
- Facebook also leverages memcache as a more
general key-value store. For example, engineers use memcache to store pre-computed results from machine learning algorithms which can then be used by a variety of other applications.
-
