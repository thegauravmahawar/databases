# Transactions

## References 

- [Transactions in PostgreSQL](https://www.postgresql.org/docs/current/tutorial-transactions.html)

A transaction is said to be atomic: from the point of view of other transactions, it either happens completely or not at all. A transactional database guarantees that all the updates made by a transaction are logged in permanent storage (i.e., on disk) before the transaction is reported complete.

When multiple transactions are running concurrently, each one should not be able to see the incomplete changes made by others. So transactions must be all-or-nothing not only in terms of their permanent effect on the database, but also in terms of their visibility as they happen. The updates made so far by an open transaction are invisible to other transactions until the transaction completes, whereupon all the updates become visible simultaneously.

```sql
BEGIN
    UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
COMMIT;
```

If, partway through the transaction, we decide we do not want to commit (perhaps we just noticed that Alice's balance went negative), we can issue the command `ROLLBACK` instead of `COMMIT`, and all our updates so far will be canceled.

PostgreSQL actually treats every SQL statement as being executed within a transaction. If you do not issue a `BEGIN` command, then each individual statement has an implicit `BEGIN` and (if successful) `COMMIT` wrapped around it. A group of statements surrounded by `BEGIN` and `COMMIT` is sometimes called a *transaction block*.