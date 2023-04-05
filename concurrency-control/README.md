# Concurrency Control

## Transactions

### References

- [Transactions in PostgreSQL](https://www.postgresql.org/docs/current/tutorial-transactions.html)

A transaction is said to be atomic: from the point of view of other transactions, it either happens completely or not at all. A transactional database guarantees that all the updates made by a transaction are logged in permanent storage (i.e., on disk) before the transaction is reported complete.

When multiple transactions are running concurrently, each one should not be able to see theincomplete changes made by others. So transactions must be all-or-nothing not only in terms of their permanent effect on the database, but also in terms of their visibility as they happen. The updates made so far by an open transaction are invisible to other transactions until the transaction completes, whereupon all the updates become visible simultaneously.

```sql
BEGIN
    UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
COMMIT;
```

If, partway through the transaction, we decide we do not want to commit (perhaps we just noticed that Alice's balance went negative), we can issue the command `ROLLBACK` instead of `COMMIT`, and all our updates so far will be canceled.

PostgreSQL actually treats every SQL statement as being executed within a transaction. If you do not issue a `BEGIN` command, then each individual statement has an implicit `BEGIN` and (if successful) `COMMIT` wrapped around it. A group of statements surrounded by `BEGIN` and `COMMIT` is sometimes called a *transaction block*.

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

See [Locks](../locks/README.md)

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

See [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)

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

See [Locking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)

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

See [Advisory Locks](https://www.postgresql.org/docs/14/explicit-locking.html#ADVISORY-LOCKS)

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