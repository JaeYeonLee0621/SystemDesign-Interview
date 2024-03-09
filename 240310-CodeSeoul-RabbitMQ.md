- Producer:Consumer = 1:1
- Publish:Subscribers = 1:N

Message Queue
1. Ordering (ex) financial comapny)
2. Routing : hold a bit with the consumer server is ready
3. Rebalancing

When?
1. Decoupling
2. Fast
3. Tunable scaling : horizontal scaling
4. Load balancing
- cluster : round robin (if one of server become CPU intensive, it is going to be hard)
5. No Redundancy (ex) financial company)
6. Limited consumer -> Easy to scale out

RabbitMQ?
- widely deployed
- open source / commercial version
- Broad company (wifi chips : harware company)
- Support for big language (support = have libraries)
- Erlang (= functional language, super fast)

Concepts
- exchanges = router (with certain criteria)
- Queues = buffer
- Routing : with routing key (ex) server.language.english)
- Aknowledgement
1) Automatically
2) Consume and ask : Delay > Duplicated execution (need to be careful)

Exchanges
- Fan out: like fan, one to everyone (e.g. publishers and suscribers)
- Direct: exact match with Routing key
- Topic : Publish/Subscribe models, wild cards

+) Message : only have one routing key
+) Durable : save the data in the persistance memory
