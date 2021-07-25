# Fraud Detection


What follows is a proposal for a fully distributed and fault-tolerant fraud detection system. I will start by providing a high-level explanation of the proposed architecture. Then, I'll do a deep dive into what I consider to be the crucial aspects that guarantee low latency responses while honoring the commitment of providing accurate fraud scores. I will also contrast the proposed architecture against other potential solutions and explain the trade-offs. Finally, I'll dedicate a section to propose some memory optimizations for Redis.

# High-level overview

In a nutshell, each region will be self-sufficient in order to provide fraud scores to transactions, as requested by Payment Providers. Each region will have its own Redis cluster, used to provide and store aggregates per Credit Card, as well its own Kafka cluster. The purpose of the Kafka cluster is to make sure that each region has a global view of all the transactions for all the Credit Cards in near-realtime.

![alt text](img/architecture2.svg "Architectural diagram")

Finally, there is a main Data Store that stores all the transactions made globally by all the Credit Cars. This data can be used to warm up Redis cache (e.g. we we spin up a new region) as well as serve other purposes (e.g. some batch processes). The main Data Store has cross-region replication. This means that each region reads data from the regional read-replica, but writes data to a single master node.

# Architecture deep-dive

## Geo DNS

Before the Payment Providers can speak to Fraudio's API, they need to know its location (IP address). Instead of having the name resolution return A or AAAA records for any of the load-balancers, we can use GeoDNS instead.

![alt text](img/GeoDNS.svg "Geo DNS")

GeoDNS will ensure that we can keep the latency to a minimum, by encouraging clients to speak with the nearest load-balancer (I mean encouraging because on the event of a full regional failure, the name resolution should still return records for the nearst non-regional load-balancer). As shown in the picture above, if the DNS resolve query originates in France, then the query response will contain A or AAAA records for `eu*.api.fraudio.com`. This is preferable to having the Payment Providers speak directly to `<region>.api.fraudio.com`, as it would force them to have knowledge of name schemas. It also gives us more control, as we can decide what routing policies to apply.

## Redis for the aggregates

![alt text](img/Redis.svg "Redis aggregates")

Once the Payment Provider knows which endopint it can speak to, it can make an HTTP request with information about the transaction.The Score service then fetches the last 100 transactions made globally with that particular Credit Card. This information will live in Redis, since its retrieval must be as fast as possible.

Redis sorted sets allow us to keep a set of IDs (strings) ordered by scores. In Redis, a score is stored as a **double 64-bit floating point** number, giving us enough room for UTC timestamps (representing the time when the transaction took place). In a nutshell, each credit card will have its own sorted set where the ID is a transaction ID and the score is the UTC time when the transaction took place.

## Event sourcing with Kafka

![alt text](img/EventSourcing.svg "Event sourcing")

Once a transaction is sent to the Score service, three things will happen:

1. The aggregated data (last 100 global transactions) for that particular Credit Card will be retrieved from Redis.
2. An event (the transaction) is recorded.
3. The aggregated data and the transaction features are fed into the model to retrieve a score, which is immediately returned as a response to the HTTP request.

Step number 2 is critical because it is responsible for recording that a transaction took place without affecting the latency of the overall response. The best way to achieve this is in a "fire and forget" fashion. In short, the transaction data will be published to a particular topic in Kafka, which will then be consumed by a process that:

1. Records the transaction to the **Main Data Store**.
2. Updates the aggregated information in Redis (more on this later).

## Keeping track of global transactions with Kafka Mirror

![alt text](img/KafkaMirror.svg "Kafka mirror")

Each region will have its own Redis and Kafka clusters. However, every region must have the same "view of the world" (eventually). The Kafka cluster will ensure that this happens, within a reasonable amount of time, by replicating certain topics. Effectively, the cluster in each region will have (at least) two topics: one to register what happens in that region and another that registers what happens in every other region. Let's call this second topic the **Mirror topic**. Once a Payment Provider makes an HTTP request, the transaction is recorded in Kafka, on the topic with events for that particular region. However, the transactions recorded in this topic will also be *mirrored* to the **Mirror topic** of all the other regions. The Kafka cluster on each region will also have a process consuming new transactions that arrive at the **Mirror topic** and store them in Redis.

The fact that we're using Redis sorted sets to keep track of the 100 latest global transactions for a Credit Card, we don't need to merge and sort the transactions recorded on the two topics in each Kafka cluster: their UTC value will ensure that they are stored in the correct order in Redis.

# Scalability and High-Availability

The nature of the Score service implies that the ratio of Redis read operations to write operations is 1:1, since every time we read the aggregates of a Credit Card we will also register the new transaction. The reads must happen faster than the writes, though, because we want to provide score responses within 100ms. Also, a high volume of requests is unlikely to be associated to the same Credit Card. In summary, the Redis cluster will be made of a master server (for write operations from the Kafka consumer) and read replicas, used by the Score service to fetch the aggregates.

The main Data Store is used to retrieve aggregate information for a Credit Card in case it doesn't exist in Redis yet (e.g. when we spin up a new region). For the sake of data locality, the main Data Store will have cross-region replication, where each region will fetch data from the corresponding read-replica.

# Fault-tolerance

On the event of service disruption in a particular region, the initial step of name resolution can still return the IP address of the Load-Balancer of the closest available region. This will, however, have an impact on the latency (the Payment Provider can decide to proceed if it doesn't get a response from us within 100ms).
