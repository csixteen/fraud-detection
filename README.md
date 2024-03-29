# Fraud Detection


What follows is a proposal for a fully distributed and fault-tolerant fraud detection system. I will start by providing a high-level explanation of the proposed architecture. Then, I'll do a deep dive into what I consider to be the crucial aspects that guarantee low latency responses while honoring the commitment of providing accurate fraud scores. I will also contrast the proposed architecture against other potential solutions and explain the trade-offs. Finally, I'll dedicate a section to propose some memory optimizations for Redis.

# Table of Contents
1. [High-level overview](#high-level-overview)
2. [Architecture deep-dive](#architecture-deep-dive)
   1. [Geo DNS](#geo-dns)
   2. [Redis for the aggregates](#redis-for-the-aggregates)
   3. [Event sourcing with Kafka](#event-sourcing-with-kafka)
   4. [Keeping global state](#keeping-global-state)
3. [Decisions and trade-offs](#design-decisions-and-trade-offs)
4. [Redis optimizations](#redis-optimizations)
5. [References](#references)

# High-level overview

In a nutshell, each region will be self-sufficient in order to provide fraud scores to transactions, as requested by Payment Providers. Each region will have its own Redis cluster, used to provide and store aggregates per Credit Card, as well its own Kafka cluster. The purpose of the Kafka cluster is to make sure that each region has a global view of all the transactions for all the Credit Cards in near-realtime.

![alt text](img/architecture2.svg "Architectural diagram")

Finally, there is a main Data Store that stores all the transactions made globally by all the Credit Cards. This data can be used to warm up Redis cache (e.g. we we spin up a new region) as well as serve other purposes (e.g. some batch processes). The main Data Store has cross-region replication. This means that each region reads data from the regional read-replica, but writes data to a single master node.

# Architecture deep-dive

## Geo DNS

Before the Payment Providers can speak to Fraudio's API, they need to know its location (IP address). Instead of having the name resolution return A or AAAA records for any of the load-balancers, we can use GeoDNS instead.

![alt text](img/GeoDNS.svg "Geo DNS")

GeoDNS will ensure that we can keep the latency to a minimum, by encouraging clients to speak with the nearest load-balancer (I mean encouraging because on the event of a full regional failure, the name resolution should still return records for the nearst non-regional load-balancer). As shown in the picture above, if the DNS resolve query originates in France, then the query response will contain A or AAAA records for `eu*.api.fraudio.com`. This is preferable to having the Payment Providers speak directly to `<region>.api.fraudio.com`, as it would force them to have knowledge of name schemas. It also gives us more control, as we can decide what routing policies to apply.

## Redis for the aggregates

Once the Payment Provider knows which endpoint it can speak to, it can make an HTTP request with the transaction. This HTTP request will be load-balanced to any of the Score services in that region, as they are stateless (hence horizontally scalable). The Score service, once receiving the transaction, asks Redis for the last 100 transactions made globally with that particular Credit Card. This information will be spreaded across a few different objects in Redis, but can be fetch in a single request. The read-replica is used for this purpose. The transaction must also be recorded, so the Score service writes the event (representing the transaction) to Kafka (more about this on the next section) and returns the response to the Payment Provider.

![alt text](img/Redis.svg "Redis aggregates")

Redis allows us to store information using different data structures, in particular Sorted Sets. A sorted set is a dictionary that maps keys (strings) to scores (64-bit floating point numbers), keeping the keys ordered by score. For our use case, each key is the transaction ID and the score is the UTC timestamp that represents the exact moment the transaction took place. On the other hand, each transaction needs to be stored as an individual object (e.g. a Hash). Thus, fetching the last 100 transactions for a Credit Card involves identifying the right sorted (the key that identifies the sorted set can be the Credit Card number), returning the last 100 IDs ordered by score in reverse order (from most to least recent) and returning the objects associated to each of those IDs. This can be achieve either with 2 requests (if we use Redis *pipelines* for the second part) or one single request (if we use Redis *eval* with a Lua script).

## Event sourcing with Kafka

Once the Payment Provider sends a transaction to the Score service, a few things will happen:

1. The aggregated data for that particular Credit Card is fetched from Redis (as explained in the previous section).
2. An event (the transaction) is recorded.
3. The aggregated data and the transaction features are fed into the ML model to retrieve the score, which is then returned as an HTTP response.

![alt text](img/EventSourcing.svg "Event sourcing")

Step number 2 is critical because it is responsible for recording that a transaction took place, without affecting the overall latency of the response. The best way to achieve this is in a "fire and forget" fashion. In short, the transaction data will be published to a particular topic in Kafka, which will then be consumed by a process that:

1. Records the transaction to the **Main Data Store**.
2. Updates the aggregated information in Redis (and also stores that particular transaction as an individual object).

The aggregated information for Credit Cards needs to be updated every time a transaction for that CC occurs. This means that the same transaction for which we want to get a score also needs to be stored. Instead of writing immediately to Redis and then to Kafka, we can simply write **only** to Kafka, to the regional topic (a topic that registers events that happen on that particular region). The consumers of that topic will then write the transaction to the Redis master as well as to the main Data Store. This introduces a bit of latency when registering the new transaction, but ensures that the score response is sent as quickly as possible to the Payment Provider.

## Keeping global state

As mentioned previously, each region should be self-sufficient to provide fraud scores to the Payment Providers. This means that each region must know what is happening in the other regions, and in the end all the regions must share the same global view. This is where Kafka mirroring comes in. Effectively, the Kafka Cluster in each region will have (at least) two topics: one topic that registers events that take place on that region (the *regional topic*) and another topic that mirrors (or replicates) what is happening in the other regions. The consumer processes on each topic will then write to the Redis master of the regional cluster, but **only** the consumer of the regional topic will write to the main Data Store. The reason for this is that whatever comes from the other regions has already been recorded by the consumer in that particular region (thus we can avoid unnecessary writes against the data store).

![alt text](img/KafkaMirror.svg "Kafka mirror")

The advantage of using Redis Sorted Sets for aggregated information is that we won't have to merge and sort the events from both topics on each region before inserting: the score (UTC of each transaction) does the sorting for us automatically implicitly (when inserting a key / score pair in a Sorted Set, the insert is sorted and takes `O(log(N))`).

# Design decisions and trade-offs

Any solution to this problem will suffer from (at least) one limitation that impacts the speed at which information is transmitted: the speed of light. A round-trip between San Francisco and Amsterdam takes roughly 150ms, which means that once a transaction takes place it's physically impossible for all the regions to register it immediately.

The proposed solution introduces potentially more complexity when compared to a fully centralized solution, since each region has its own Redis and Kafka cluster, and all the Kafka clusters mirror to each other. However, this also means that on the event of network partitioning, the service can still operate and provide low latency responses and fairly accurate scores, which keeping the number of cross-region requests to a minimum.

Besides extra complexity, the proposed solution can be potentially more expensive, since we replicate the same amount of data in every region. In this case, the main potential source of extra costs would be the Redis clusters, but there are several things that can be done in order to keep the expenses to an acceptable minimum (see next section).

# Redis optimizations

Redis plays a crucial role, since it allows us to quickly fetch aggregate data to be used by the ML model. However, since we need to have global data in order to detect fraudulent transactions, this can become quite expensive if we're not careful.

Here are some of the things that can be done in order to accomodate for an increase in transactions while keeping latencies low:

## Use Lists to store transactions instead of Hashes

When using Hashes to store a single transaction object, both keys and values consume storage space. If the transactions have a schema that barely changes, then we are repeating information unnecessarily with the field names. Take the following JSON representation of a transaction as an example:

```json
{
  "timestamp": 1627224482,
  "credit_card": "XXXXYYYYZZZZWWWW",
  "currency": "GBP",
  "amount": 100.5
}
```

A ballpark estimate of space for storing such transaction in Redis is ~90 bytes. If we want to store 1 billion of such transactions, it takes roughly 83GB of storage space, where almost half of it is dedicated to the repeated keys. Instead, we can store the transaction as a list of values and the field names become indices in the list.

## Compress field names in Hashes

If storing transactions as a list is not an option, we can still opt for smaller key names. For example:

```
credit_card -> cc  # saves 9 bytes
currency -> cur    # saves 5 bytes
amount -> total    # saves 1 byte
```

By saving 15 bytes on a single Hash, we save 13 GB when we store 1 billion of Hashes.

## Use eval + Lua scripting

In order to retrieve the aggregated data for a Credit Card, we must first identity the correct Sorted Set, then use Redis `ZRANGE` command to retrieve the transaction IDs ordered by score and then we retrieve each single transaction object identified by those IDs. Retrieving the IDs requires one call to Redis and retrieving all the individual objects can be done using Redis pipeline, making it a total of 2 calls to the Redis server. However, if all the keys related to a single Credit Card are stored on the same server (if we use Redis cluster, for example), then we can script the above logic in Lua, send a string to the server and call it with the `EVAL` command. This logic will be executed in the server, is much faster and only requires one single request to Redis.

# References

- [Amazon Route53 routing policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Kafka Geo-Replication](https://kafka.apache.org/documentation.html#georeplication)
- [Kafka MirrorMaker best practices](https://community.cloudera.com/t5/Community-Articles/Kafka-Mirror-Maker-Best-Practices/ta-p/249269)
- [Using Lua to implement multi-get on Redis hashes](https://beforeitwasround.com/2014/07/using-lua-to-implement-multi-get-on-redis-hashes.html)
- [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know)
- [Memory Optimization for Redis](https://docs.redislabs.com/latest/ri/memory-optimizations/)
- [AWS Latency Monitoring](https://www.cloudping.co/grid)
