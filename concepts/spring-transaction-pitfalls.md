# Spring Transaction Pitfalls
**Introduced:** Day 23

---

Four failures that are invisible in tests and appear in production.

## 1. `@Transactional` is a proxy — self-calls bypass it

```java
public void outer() {
    this.credit(...);      // ← NO TRANSACTION. Never goes through the proxy.
}
@Transactional
public void credit(...) { }
```

Spring wraps the bean; an internal `this.` call skips the wrapper entirely. **No error, no
transaction.** Same for `private` and `final` methods — the proxy cannot override them.

Fix: move the transactional method to a different bean, or inject self.

## 2. Checked exceptions do NOT roll back

```java
@Transactional
public void transfer() throws IOException {
    accounts.save(account);
    throw new IOException();      // CHECKED → transaction COMMITS
}
```

Default rollback rule is `RuntimeException` and `Error` only.

```java
@Transactional(rollbackFor = Exception.class)   // explicit
```

This is why domain exceptions in the trading platform all extend `RuntimeException` — not a style
choice, it's what makes rollback actually happen.

## 3. Never hold a transaction across a network call

```java
@Transactional
public void process() {
    var data = repository.findAll();
    externalApi.call(data);        // ← 30s HTTP call, transaction open the whole time
    repository.saveAll(data);
}
```

VACUUM cannot remove any row version an open transaction might still need, so this bloats the
database for the full duration (see [[mvcc]]). A connection is also held from the pool.

Fix: read and commit, make the call, then open a new transaction to write.

## 4. `readOnly = true` is not merely documentation

```java
@Transactional(readOnly = true)
```

- Hibernate skips dirty-checking (no snapshot of every loaded entity)
- Postgres avoids assigning a transaction ID
- The connection can be routed to a read replica later without code change

Worth applying to every query method.

## Related

The READ COMMITTED trap — two reads in one transaction can legitimately differ, because each
STATEMENT gets a fresh snapshot:

```java
@Transactional
public void process(UUID id) {
    var a = accounts.findById(id);   // 100
    var b = accounts.findById(id);   // 50  ← another transaction committed in between
}
```
Logic assuming both reads agree is wrong, and will only fail under load.
