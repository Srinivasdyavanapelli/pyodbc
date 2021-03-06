Those of you who come from a database background will be familiar with the idea of database transactions, where a series of SQL statements are committed together in one operation. This is crucial if you need to make multiple updates to a database, where each update would leave the database in an inconsistent state (albeit temporarily) if committed separately. The classic example of this is processing a check, where money is transferred from one bank account to another, i.e. a debit and a credit. It is important that both the debit and credit are committed together on both accounts, otherwise there will be an inconsistency within the database.

Note, this section is only relevant where `autocommit` is set to False on the pyodbc connection. False is the default. When `autocommit` is set to True, a commit is automatically executed after every SQL statement on that connection. Note, those automatic commits are done by the database itself, not pyodbc. To put it another way, every SQL statement is essentially run within its own database transaction.

When using pyodbc with `autocommit=False`, it is important to understand that you never explicitly open a database transaction in your Python code. Instead, a database transaction is implicitly opened when a Connection object is created with `pyodbc.connect()`. That database transaction is then either committed or rolled-back by explicitly calling `commit()` or `rollback()` on the connection, at which point a new database transaction is implicitly opened. Each database transaction may include just one SQL statement or multiple SQL statements, where each of those SQL statements is executed using the `Cursor.execute()` function.  Hence, the equivalent of the following SQL:
```sql
BEGIN TRANSACTION
  UPDATE T1 SET ...
  DELETE FROM T1 WHERE ...
  INSERT INTO T1 VALUES ...
COMMIT TRANSACTION
BEGIN TRANSACTION
  INSERT INTO T2 VALUES ...
  INSERT INTO T3 VALUES ...
COMMIT TRANSACTION
```
in Python would be:
```python
cnxn = pyodbc.connect('mydsn', autocommit=False)
crsr = cnxn.cursor()
crsr.execute("UPDATE T1 SET ...")
crsr.execute("DELETE FROM T1 WHERE ...")
crsr.execute("INSERT INTO T1 VALUES ...")
cnxn.commit()
crsr.execute("INSERT INTO T2 VALUES ...")
crsr.execute("INSERT INTO T3 VALUES ...")
cnxn.commit()
cnxn.close()
```
As you can see, no database transaction is ever explicitly opened in Python but they are explicitly committed. Also, it's important to remember that transactions are managed only at the Connection level, not the Cursor level. Cursors are merely vehicles to execute SQL statements and manage their results, nothing more.  (Yes, there is a convenience function `commit()` on the Cursor object but that simply calls `commit()` on the cursor's parent Connection object.)  Bear in mind too that when `commit()` is called on a connection, ALL the SQL statements from ALL the cursors on that connection are committed together (ditto for `rollback()`).

When a connection is closed with the `close()` function, a rollback is always issued on the connection just in case. In the event that a Connection object goes out of scope before it is closed (e.g. because an exception occurs), the Connection object is automatically deleted by Python, and a rollback is issued as part of the deletion process.

#### Specifying a Transaction Isolation level

Database management systems that support transactions often support several levels of transaction isolation to control the effects of multiple processes performing simultaneous operations within their own transactions. ODBC supports four (4) levels of transaction isolation:

- SQL_TXN_READ_UNCOMMITTED
- SQL_TXN_READ_COMMITTED
- SQL_TXN_REPEATABLE_READ
- SQL_TXN_SERIALIZABLE

You can specify one of these in your Python code using the `Connection#set_attr` method, e.g.,

```python
conn = pyodbc.connect(conn_str)
conn.set_attr(pyodbc.SQL_ATTR_TXN_ISOLATION, pyodbc.SQL_TXN_SERIALIZABLE)
```
