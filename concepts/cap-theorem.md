# CAP Theorem
**Introduced:** Day 4

---

## Definition
A distributed system can only guarantee **two of three** simultaneously:
- **C — Consistency:** every read returns the most recent write
- **A — Availability:** every request receives a response
- **P — Partition Tolerance:** system continues when nodes can't communicate

---

## The Real Choice
Network partitions are NOT optional — they will happen in any distributed system. So the real choice is always **CP vs AP**:

```
CP — Consistency + Partition Tolerance
     During partition: reject requests rather than return stale data
     System says "I don't know" → returns error
     Use when: data correctness is critical (banking, payments)

AP — Availability + Partition Tolerance
     During partition: serve requests with potentially stale data
     System says "here's what I have" → returns old value
     Use when: availability matters more than perfect consistency
```

CA = only possible on single-node systems (not distributed).

---

## Real Systems

| System | Choice | Reason |
|---|---|---|
| Zookeeper, etcd | CP | Distributed coordination |
| Cassandra, DynamoDB | AP | High availability |
| HBase | CP | Strong consistency |
| Kafka (default) | AP | Delivery over consistency |
| Kafka (acks=all) | CP | Consistency over throughput |
| DNS | AP | Always available, stale ok |
| Google Spanner | CP | Global distributed SQL |

---

## Bank Example
```
Partition between Frankfurt + Singapore nodes:

CP: Singapore user withdraws → REJECTED until partition heals
    No money lost. User frustrated.

AP: Singapore user withdraws → APPROVED with stale balance
    Risk of overdraft if Frankfurt also withdrew.
```

---

## Rule
Match CP vs AP to the cost of being wrong:
- Wrong answer causes data loss / money loss → CP
- Wrong answer causes temporary stale display → AP
