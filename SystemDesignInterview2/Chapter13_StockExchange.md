# Chapter 13. Stock Exchange

# Step 1. Understand the Problem and Establish Design Scope

## Non-functional requirements

- `Availability` : At least 99.99%
- `Fault tolerance` : Limiting the impact of a production incident
- `Latency` : Focus on the 99th percentile latency
- `Security` : Should have an account management system (ex) KYC (Know Your Client), DDoS (Distributed Denial-Of-Service)

## Back-of-the-envelope estimation

- 100 Symbols
- 1 billion orders per day
- NYSE Stock exchange is open Monday~Friday (09:30~16:00)
- QPS = 1 billion / 6.5 / 3600 ~= 43,000
- Peak QPS = 5 x QPS = 215,000

# Step 2. Propose High-Level Design and Get Buy-In

# Business Knowledge 101

## Broker
- Most retail clients trade with an exchange via a broker
- These brokers provide a `friendly user interface` for retail users to place trades and view market data

## Institutional client
- Diferrent institutional clients operate with different requirements

ex) pension funds aim for a stable income
- Trade infrequently but the volume is large
- Therefore, they need features like order splitting to minimize the market impact of their sizable orders
