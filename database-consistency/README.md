# Database Consistency

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

### PostgreSQL Examples

```sql
SELECT * FROM users;
```