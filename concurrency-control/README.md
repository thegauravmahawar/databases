# Concurrency Control

## References 

- [Transactions](../transactions/README.md)
- [Concurrency Control in Databases](https://www.gojek.io/blog/on-concurrency-control-in-databases)
- [Deep Dive into Database Concurrency Control](https://www.alibabacloud.com/blog/a-deep-dive-into-database-concurrency-control_596779)
- [Concurrency Control in PostgreSQL](https://www.postgresql.org/docs/current/mvcc.html)

## Locking Reads

See: [Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)

If you query data and then insert or update related data within the same transaction, the regular `SELECT` statement does not give enough protection. Other transactions can update or delete the same rows you just queried.

### SELECT ... FOR SHARE

Sets the shared mode lock on any rows that are read. Other sessions can read the rows, but cannot modify them until your transaction commits. If any of these rows are changed by another transaction that has not yet committed, your query waits until that transaction ends and then uses the latest values.

### SELECT ... FOR UPDATE

For index records the search encounters, locks the rows and any associated index entries, the same as if you issued an `UPDATE` statement for those rows. Other transactions are blocked from updating those rows, from doing `SELECT ... FOR SHARE`, or from reading the data in certain transaction isolation levels.

All locks set by `FOR SHARE` and `FOR UPDATE` queries are released when the transaction is committed or rolled back.

A locking read clause in an outer statement does not lock the rows of a table in a nested subquery unless a locking read clause is also specified in the subquery.

```sql
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2) FOR UPDATE;
```

## Optimistic & Pessimistic Locking

See: [Optimistic vs Pessimistic Locking](https://vladmihalcea.com/optimistic-vs-pessimistic-locking/)

**Optimistic Locking** - We could allow the conflict to occur, but then we need to detect it upon committing our transaction.

**Pessimistic Locking** - If the cost of retrying is high, we could try to avoid the conflict altogether via locking.

## Explicit Locking

See: [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

PostgreSQL provides various lock modes to control concurrent access to data in tables. These modes can be used for application-controlled locking in situations where MVCC does not give the desired behavior. Also, most PostgreSQL commands automatically acquire locks of appropriate modes to ensure that referenced tables are not dropped or modified in incompatible ways while the command executes. (For example, `TRUNCATE` cannot safely be executed concurrently with other operations on the same table, so it obtains an `ACCESS EXCLUSIVE` lock on the table to enforce that.)

### Table-Level Locks

Two transactions cannot hold locks of conflicting modes on the same table at the same time. (However, a transaction never conflicts with itself. For example, it might acquire `ACCESS EXCLUSIVE` lock and later acquire `ACCESS SHARE` lock on the same table.) Non-conflicting lock modes can be held concurrently by many transactions. Notice in particular that some lock modes are self-conflicting (for example, an `ACCESS EXCLUSIVE` lock cannot be held by more than one transaction at a time) while others are not self-conflicting (for example, an `ACCESS SHARE` lock can be held by multiple transactions).

### Row-Level Locks

Transaction can hold conflicting locks on the same row, even in different subtransactions; but other than that, two transactions can never hold conflicting locks on the same row. Row-level locks do not affect data querying; they block only writers and lockers to the same row. Row-level locks are released at transaction end or during savepoint rollback, just like table-level locks.

### Deadlocks

The use of explicit locking can increase the likelihood of *deadlocks*, wherein two (or more) transactions each hold locks that the other wants. For example, if transaction 1 acquires an exclusive lock on table A and then tries to acquire an exclusive lock on table B, while transaction 2 has already exclusive-locked table B and now wants an exclusive lock on table A, then neither one can proceed. PostgreSQL automatically detects deadlock situations and resolves them by aborting one of the transactions involved, allowing the other(s) to complete. (Exactly which transaction will be aborted is difficult to predict and should not be relied upon.)

Note that deadlocks can also occur as the result of row-level locks (and thus, they can occur even if explicit locking is not used).

So long as no deadlock situation is detected, a transaction seeking either a table-level or row-level lock will wait indefinitely for conflicting locks to be released. This means it is a bad idea for applications to hold transactions open for long periods of time (e.g., while waiting for user input).

### Advisory Locks

PostgreSQL provides a means for creating locks that have application-defined meanings. These are called advisory locks, because the system does not enforce their use — it is up to the application to use them correctly. Advisory locks can be useful for locking strategies that are an awkward fit for the MVCC model. For example, a common use of advisory locks is to emulate pessimistic locking strategies typical of so-called “flat file” data management systems. While a flag stored in a table could be used for the same purpose, advisory locks are faster, avoid table bloat, and are automatically cleaned up by the server at the end of the session.

## Handling Concurrenct Requests

### References

- [Ensuring data(base) consistency during concurrent requests](https://blog.frankdejonge.nl/ensuring-consistency-during-concurrent-requests/)

### Examples

- Two requests setting the same field to a different value.
- Two requests that alter different fields that are used together to enforce a domain constraint.
- Two requests that cause a collection limit to be breached.

### Issues with Transaction

Transactions can cause for exclusive access on a piece of data when certain types of operations are performed. Exclusive access is provided through locks, which limit access from competing processes.

There are some cases that are not automatically solved by using a transaction. 

Example:

```text
SQL: START TRANSACTION

SQL: SELECT balance FROM loyalty_points WHERE user_id = :user_id; 
    
APP: IF balance >= item_cost THEN spend_points();

SQL: UPDATE loyalty_points SET balance = :new_balance;
SQL: INSERT INTO purchases(...);

SQL: COMMIT;
```

If our user would have a balance **100** and sends two requests to spend 65, then:

- Both requests would pass
- There would be 2 purchases
- The resulting balance would be 35

This has to do with when the lock kicks in. In the application above, the lock would start at the `UPDATE` query. This means that both processes `SELECT` the same balance. Even when the `UPDATE` query has run, competing processes will still be able to read the old value up until the moment the transaction is committed. In the end, the `UPDATE` queries run one after the other, both updating the balance to **35**.

### Ways to handle concurrent requests

- Use a decrementing query
- Use a schema constraint
- Use an optimistic lock
- Use a mutex
- Use deliberate database locking
- Use named database locks

**Use a decrementing query**

The most impactful issue is the loss of data. In this case we're spending a total of **130** yet only **65** points are deducted from the balance.

```text
SQL: START TRANSACTION

SQL: SELECT balance FROM loyalty_points WHERE user_id = :user_id; 
    
APP: IF balance >= item_cost THEN spend_points();

SQL: UPDATE loyalty_points SET balance = balance - :item_cost;
SQL: INSERT INTO purchases(...);

SQL: COMMIT
```

**Use a schema constraint**

We could store the balance as an *unsigned integer*. Using this mitigation makes the database responsible for upholding a domain constraint.

**Use an optimistic lock**

See: [Optimistic Locking](https://vladmihalcea.com/optimistic-vs-pessimistic-locking/)

When using optimistic locking, the query is designed only to succeed if certain conditions are met. In this case, the balance should be more or equal to the spending amount.

```text
SQL: START TRANSACTION

SQL: SELECT balance FROM loyalty_points WHERE user_id = :user_id; 
    
APP: IF balance >= item_cost THEN spend_points();

SQL: UPDATE loyalty_points SET balance = balance - :item_cost WHERE balance >= :item_cost;
SQL: INSERT INTO purchases(...);

SQL: COMMIT
```

**Use a mutex**

See: [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

Mutexes provide a way to ensure exclusive access to a section of code. The use of mutex is visible in application code, this makes locking deliberate and explicit. 

```text
MTX: GET LOCK

SQL: START TRANSACTION

SQL: SELECT balance FROM loyalty_points WHERE user_id = :user_id; 
    
APP: IF balance >= item_cost THEN spend_points();

SQL: UPDATE loyalty_points SET balance = balance - :item_cost;
SQL: INSERT INTO purchases(...);

SQL: COMMIT

MTX: RELEASE LOCK
```

A mutex generally has a TTL, so it prevents locking only for a certain time-frame. This is good otherwise any failing requests could result in an external lock. The bad part about it is that the execution of the code inside a lock can exceed the locking time, something a mutex can not provide a protection against.

**Use deliberate database locking**

See: [Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)

By using `SELECT ... FOR UPDATE` we can turn our select statement into a locking read.

```text
SQL: START TRANSACTION

SQL: SELECT balance FROM loyalty_points WHERE user_id = :user_id FOR UPDATE; 
    
APP: IF balance >= item_cost THEN spend_points();

SQL: UPDATE loyalty_points SET balance = :new_balance;
SQL: INSERT INTO purchases(...);

SQL: COMMIT;
```

The locking read will block any other locking query. It is important to note `SELECT ... FOR UPDATE` will not block standard non-locking reads. Any other piece of code that requires a consistent view MUST also use a locking read, otherwise the original problem remains.

An up-side to this, over mutexes, is that the lock will remain for the duration of the transaction. If the transaction is completed (failed or succeeded) the lock is automatically released. A down-side is that you need a transaction. In some cases you want to commit multiple times during a certain business operation. Transaction locks only last for the duration of the transaction, having multiple transactions in one routine means other processes can run in the between the transactions, which can alter the database state.

**Use named database locks**

See: [Advisory Locks](https://www.postgresql.org/docs/14/explicit-locking.html#ADVISORY-LOCKS)

Named locks are application defined locks that are enforced by the database. These locks do not block any other queries on any table, so it's up to the developer to use them correctly.

```text
SQL: SELECT GET LOCK('lock_name', timeout)

APP: Abort or retry if lock was not acquired

SQL: SELECT balance FROM loyalty_points WHERE user_id = :user_id FOR UPDATE; 
    
APP: IF balance >= item_cost THEN spend_points();

SQL: START TRANSACTION
SQL: UPDATE loyalty_points SET balance = :new_balance;
SQL: INSERT INTO purchases(...);
SQL: COMMIT;

SQL: DO RELEASE_LOCK('lock_name')
```

The lock is acquired outside of the transaction. The inner transaction is only still to ensure the other SQL queries succeed or fail together. In the case of a single query the transaction could be entirely omitted.
