# CHAPTER 12 - TRANSACTIONS

- Application logic will use multiple SQL statements that need to execute together as a logical unit of work, rather than just individual, independent SQL statements

- Transactions are a mechanism used to group a set of SQL statements together such their either all or none of the statements succeed

- Database management systems allow a single user to query and modify data, but in today's world there may be thousands of people making changes to a database simultaneously. If every user is only executing queries, such as might be the case with a data warehouse during normal business hours, then there are very few issues for the database server to deal with. If some of the users are adding and / or modifying data, the server must handle quite a bit more bookkeeping
	- For example let's say that you are running a report that sums up the current week's film rental activity. At the same time you are running the report, however, the following activities are occuring:
		- A customer rents a film
		- A customer returns a film after the due date and pays a late fee
		- Five new films are added to inventory
	- While your report is running, multiple users are modifying the underlying data, so what numbers should appear on the report? The answer depends on how your server handles locking

- Locks are mechanism that the database server uses to control simultaneous use of data resources. When some portion of the database is locked, any other users wishing to modify (or possibly read) that data must wait until the lock has been released. Most database servers use one of two locking strategies:
	- Database writers must request and receive from the server a write lock to modify data and the database reader must request and receive from the server a read lock to query data. While multiple users can read data simulatenously, only one write lock is given out at a time for each table (or portion thereof) and read requests are blocked until the write lock is released
	- Database writers must request and receive from the server a write lock to modify data, but readers do not need any type of lock to query data. Instead, the server ensures that a reader sees a consistent view of data (the data seems the same even though other usres may be making modifications) from the time her query begings until her query has finished. This approach is called versioning
	- The first approach can lead to long wait times if there are many concurrent read and write requests, and the second approach may be problematic if there are long-running queries while data is being modified

- There are also a number of different strategies that you may employ when deciding how to lock a resource. The server may apply a lock at one of the three different levels, or granularities:
	- Table locks - Keep multiple users from modifying data in the same table simultaneously
	- Page locks - Keep multiple users from modifying data on the same page (a page is a segment of memory generally in the range of 2KB to 16KB) of a table simulatenously
	- Row locks - Keep multiple usrs from modifying the same row in a table simulatenously
	- It takes very little bookkeeping to lock entire tables, but this approach quickly yields unacceptable wait times as the number of users increase. Row locking takes quite a bit more bookkeeping, but it allows many users to modify the same table as long as they are interested in different rows

- To get back to your report, the data that appears on the pages of the report will mirror either the state of the database when your report started (if your server uses a versioning approach) or the state of the database when the server issues the reporting application a read lock (if your server uses both read and write locks)

- Transaction is a device for grouping together multiple SQL statements such that either all or none of the statements succeed (a property known as atomicity). If you attempt to transfer $500 from your savings account to your checking account, you would be a bit upset if the money were successfully withdrawn from your savings account but never made ti to your checking account. Whatever the reason for the failure you want your $500 back. To protect against this kind of error, the program that handles your transfer request would first begin a transaction, then issue the SQL statements needed to move the money from your savings to your checking account, and if everything succeeds, end the transaction by issuing the `commit` command. If something unexpected happens, however, the program would issue a `rollback` command which instructs the server to undo all changes made since the transaction began. By using transactions the program ensures that your $500 either stays in your savings acocunt or moves to your checking account, without the possibility of it failing into a crack. Regardless of whether the transaction was committed or was rolled back, all resources acquired (eg write locks) during the execution of the transaction are released when the trasaction completes
	- If the program manages to complete both the `update` statements but the server shuts down before a `commit` or `rollback` can be executed, then the transaction will be rolled back when the server comes back online (one of the tasks that a database server must complete before coming online is to find any incomplete transactions that were underway when the server shut down and roll them back). Additionally, if your program finishes a transaction and issues a `commit` but the server shuts down before the changes have been applied to permanent storage (that is, the modified data is sitting in memory but has not been flushed to disk) then the database server must reapply the changes from your transaction when the server is restarted (a property known as durability)

```sql
START TRANSACTION

/* withdraw money from first account, making sure balance is sufficient */
UPDATE account SET avail_balance = avail_balance - 500
	WHERE account_id = 9988
		AND avail_balance > 500;

IF <exactly one row was updated by the previous statment> THEN
/* deposit money into second account */
	UPDATE account SET avail_balance = avail_balance + 500
		WHERE account_id = 9989;

	IF <exactly one row was updated by the previous statment> THEN
	/* everything worked make the changes permanent */
		COMMIT;

	ELSE
	/* something went wrong, undo all changes in this transaction */
		ROLLBACK;

	END IF;
ELSE

	/* insufficient funds, or error encountered during update */
	ROLLBACK;
END IF;
```

- Database servers handle transaction creation in one of two ways:
	- An active transaction is always associated with a database session, so there is no need or method to explicitly begin a transaction. When the current transaction ends, the server automatically begins a new transaction for your session
	- Unless you explicitly begin a transaction, individual SQL statements are automatically committed independently of one another. To begin a transaction, you must first issue a command
	- The SQL:2003 standard includes a `start transaction` command to be used when you want to explicitly begin a transaction
	- With some DBMS until you explicitly begin a transaction you are in autocommit mode - which means that individual statements are automatically committed by the server. You can decide that you want to be in a transaction and issue a start/begin transaction command, or you can simply let the server commit individual statements
	- Shut off autocommit mode each time you login and get in the habit of running all your SQL statements within a transaction

- Once a transaction has begun (whether explicitly via the `start transaction` command or implicitly by the database server) you must explicitly end your transaction for your changes to become permanent. You do this by way of the `commit` command, which instructs the server to mark the changes as permanent and release any resources (that is page or row locks) used during the transaction. If you decide you want to undo all the changes make since starting the transaction, you must issue the `rollback` command, which instructs the server to return the data to its pre-transaction state. After the `rollback` has been completed any resourced used by your session are released
	- Along with issuing either the `commit` or `rollback` command there are several other scenarios by which your transaction can end, either as an indirect result of your actions or as a result of something outside your control:
		- The server shuts down, in which case your transaction will be rolled back automatically when the server is restarted
		- You issue an SQL schema statement, such as `alter table`, which will cause the current transaction to be committed and a new transaction to be started. Alterations to database (addition of a new table, or index, or the removal of a column from a table, cannot) cannot be rolled back, so commands that alter your schema must take place outside a transaction. If a transaction is currently underway, the server will commit your current transaction, execute the sQL schema statement commands, and then automatically start a new transaction for your session. The server will not inform you of what has happened, so you should be careful that the statements that comprise a unit of work are not inadvertently broken up into multiple transactions by the server
		- You issue another `start transaction` command which will cause the previous transaction to be committed
		- The server prematurely ends your transaction because the server detect a deadlock, and decides that your transaction is the culprit. in this case, the transaction will be rolled back and you will receive an error message. A deadlock occurs when two different transactions are waiting for resources that the other transaction currently holds. For example, transaction A might just have updated the `account` table and is waiting for a write lock on the `transaction` table, while transaction B has inserted a new row into the `transaction` table and is waiting for a write lock on the `account` table. If both transactions happen to be modifying the same page or row (depending on the lock granularity in use by the database server) then they will each wait forevr for the other transaction to finish up and free up the needed resource. Database servers must always be on the lookout for these situations so that throughput does not grind to a halt; when a deadlock is detected, one of the transactions is chosen (either arbitrarily or by some criteria) to be rolled back so that the other transaction may proceed. Most of the time the terminated transaction can be restarted and will succeed without encountering another deadlock situation. The database server will raise an error to inform you that your transaction has been rolled back due to deadlock situation. It is a reasonalbe practice to retry a transaction that has been rolled back due to a deadlock situation

	- You may encounter an issue within a transaction that requires a rollback, but you may not want to undo all of the work that has transpired. For these situations, you can establish one or more savepoints within a transaction and use them to roll back to a particular location within your transaction rather than rolling all the way back to the start of the transaction
		- All the savepoint must be given a name, which allows you to have multiple savepoints within a single transaction. To create a savepoint - `SAVEPOINT my_savepoint;`
		- To roll back to a particular savepoint - `ROLLBACK TO SAVEPOINT my_savepoint;`
	
```sql
START TRANSACTION;

UPDATE product
	SET date_retired = CURRENT_TIMESTAMP()
	WHERE product_cd = 'XYZ';

SAVEPOINT before_close_accounts;

UPDATE account
	SET status = 'CLOSED', close_date = CURRENT_TIMESTAMP(), last_activity_date = CURRENT_TIMESTAMP()
	WHERE product_cd = 'XYZ';

ROLLBACK TO SAVEPOINT before_close_accounts;
COMMIT;
```

- When using savepoints, remember that:
	- Despite the name, nothing is saved when you create a savepoint. You must eventually issue a `commit` if you want your transaction to be made permanent
	- If you issue a `rollback` without naming a savepoint all savepoints within the transaction will be ignored, and the entire transaction will be undone