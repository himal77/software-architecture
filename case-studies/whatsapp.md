# Case Study: WhatsApp
**Covered:** Day 8 | **Topic:** Messaging at 2B-user scale with minimal team

---

## Scale
- ~2B users
- ~100B messages/day
- ~50 engineers ran 900M users at acquisition

---

## Architectural Choices

### Sharded by chat_id from day one
The single most important decision in any messaging system. Co-locates all messages for a conversation, makes paginated chat reads efficient.

### Erlang BEAM VM
Designed for massive concurrency. Single server handles millions of WebSocket connections — fundamental fit for messaging where each user maintains a long-lived connection.

### Append-only message log
Messages never updated — only inserted. Perfect fit for SSD-optimized writes and easy to back up.

### Phone number as identity
No separate auth/account system needed. Routing is simple — phone number maps directly to user.

### Database evolution
- Started with Mnesia (Erlang's built-in DB)
- Moved to MySQL clusters for some data as scale grew
- Modern era: integrated into Meta infrastructure

### Media deduplication
1,000 users send the same meme → stored once. Saves enormous storage.

---

## What Made WhatsApp Possible

| Choice | Effect |
|---|---|
| chat_id sharding | Even distribution, fast paginated reads |
| Erlang concurrency | Millions of connections per server |
| Phone-as-identity | No auth complexity |
| Minimal feature creep | Engineering team stays small |

---

## The Lesson

**The right partitioning + the right concurrency model + discipline beats raw infrastructure.**

WhatsApp ran 900M users on 50 engineers. Most companies that try to copy this fail because they:
1. Pick the wrong shard key (often by user, not by chat)
2. Use a JVM-style threading model that can't handle massive WebSocket connections
3. Add features that complicate the storage model

---

## Universal Pattern

Every major messaging system uses the same fundamental design:

| System | Shard Key | Connection Model |
|---|---|---|
| WhatsApp | chat_id | Erlang/BEAM |
| Slack | channel_id | Custom WebSocket service |
| Discord | guild_id (server) | Elixir (Erlang VM) |
| Telegram | chat_id | Custom MTProto |
| Signal | conversation_id | Java/Netty WebSockets |

**The convergent evolution to the same architecture is itself the lesson:** for messaging, this is the right answer.
