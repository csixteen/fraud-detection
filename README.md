# Fraud Detection

What follows is the proposal for a fully distributed and fault-tolerant fraud detection system.

# Architecture Overview

The following diagram should give a birds-eye view of the proposed architecture.

![alt text](architecture.png "Architectural diagram")

## Geo DNS

Before any request can be made from the Payment Providers to FraudIO's API, a name resolution needs to take place. For the sake of simplicity, let's assume that hostname for the API is `api.fraudio.com`. Name resolvers can return A or AAAA records based on pre-defined policies, such as the geographic location of the querier. This will ensure that the Payment Provider speaks with the Score service in its own region, hence keeping the latency as low as possible.

## Redis for the aggregates

Once the Payment Provider knows which endopint it can speak to, it can make an HTTP request with information about the transaction.The Score service then fetches the last 100 transactions made globally with that particular Credit Card. This information will live in Redis, since its retrieval must be as fast as possible.

Redis sorted sets allow us to keep a set of IDs (strings) ordered by scores. In Redis, a score is stored as a **double 64-bit floating point** number, giving us enough room for UTC timestamps (representing the time when the transaction took place). In a nutshell, each credit card will have its own sorted set where the ID is a transaction ID and the score is the UTC time when the transaction took place.

## Event sourcing with Kafka

Once a transaction is sent to the Score service, three things will happen:

1. The aggregated data (last 100 global transactions) for that particular Credit Card will be retrieved from Redis.
2. An event (the transaction) is recorded.
3. The aggregated data and the transaction features are fed into the model to retrieve a score, which is immediately returned as a response to the HTTP request.

Step number 2 is critical because it is responsible for recording that a transaction took place without affecting the latency of the overall response. The best way to achieve this is in a "fire and forget" fashion. In short, the transaction data will be published to a particular topic in Kafka, which will then be consumed by a process that:

1. Records the transaction to the **Main Data Store**.
2. Updates the aggregated information in Redis (more on this later).

## Keeping track of global transactions with Kafka Mirror

Each region will have its own Redis and Kafka clusters. However, every region must have the same "view of the world" (eventually). The Kafka cluster will ensure that this happens, within a reasonable amount of time, by replicating certain topics. Effectively, the cluster in each region will have (at least) two topics: one to register what happens in that region and another that registers what happens in every other region. Let's call this second topic the **Mirror topic**. Once a Payment Provider makes an HTTP request, the transaction is recorded in Kafka, on the topic with events for that particular region. However, the transactions recorded in this topic will also be *mirrored* to the **Mirror topic** of all the other regions. The Kafka cluster on each region will also have a process consuming new transactions that arrive at the **Mirror topic** and store them in Redis.
