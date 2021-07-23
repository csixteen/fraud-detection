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
