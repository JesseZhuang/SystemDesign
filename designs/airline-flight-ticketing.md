# Airline Flight Ticketing

## Companies

Circle, Microsoft

## Requirements

Functional

1. Browse available flights, book flight, select seat while booking
2. out of scope (logistics). only handle single source single destination

Iterations

1. no payment gateway
2. add payment gateway
3. orchestration approaches: websocket, polling and email notification
4. 1k TPS for write scale up

Non-Functional

1. 1 k TPS for read, 10 TPS for write.

## High Level Design

Flights DB: leader and followers, 1000 TPS per node, n+1 (failover).

## Data Schema

```shell
Flight {
  flightId:1637UA
  planeTypeId: boeing787
  from:SFO (gate)
  to:SEA
  price: '$200.00'
  depart: timestamp(167_000_5000)
  arrival: timestamp
  available: true/false
}
# maybe both string and double for price
Ticket {
  string name;
  string flightId;
  string seat; # 37A/unassigned
  enum status; # requested,authorized
  bool payment_status;
}
SeatMapping {
  string planeTypeId;  # boeing787 80 seat options
  [seatId]: [37A, 37B];
}
```

## Database

1. DynamoDB: trigger, no 2D index support, 25 GSI
2. PostgreSQL: trigger, r-trees, support secondary index
3. redis: geohash, no secondary index support
4. ElasticSearch: no triggers, quad-trees, no limit on secondary index

## References

1. Microsoft https://www.youtube.com/watch?v=ckkbxPE-5wc