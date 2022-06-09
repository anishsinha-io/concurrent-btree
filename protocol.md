### This file contains a description and proof for the locking protocol used in this project

Some necessary terminology is as follows:

- A Read-Write Lock (rwlock) refers to a synchronization primitive that has two modes: *shared read mode* and *exclusive
  write mode*. A rwlock may have any number of shared readers, but only one exclusive writer. When a write lock is
  acquired on the rwlock, all new readers and writers will block until the write operation finishes. When there are any
  readers, any write will block until the readers are all released. Read precedence favors concurrency because it
  potentially allows many threads to accomplish work simultaneously [1]. The implementation of rwlock that is used in
  this project is from the pthreads (POSIX threads) API. The implementation used supports the following operations:
  initialization, destruction, acquire_readlock, try_acquire_readlock, release_readlock, acquire_writelock,
  try_acquire_writelock, and release_writelock. The pthreads API uses slightly different names but the spirit remains
  the same. A rwlock is used to synchronize access to the database file.
- A lock table is a global data structure that contains a map of locks on the database at any given time. The lock table
  itself is protected with a mutual exclusion lock (mutex).
- A Mutual Exclusion Lock (mutex) is a synchronization primitive that can be used to synchronize access to a data
  structure shared across threads. Unlike a rwlock, a mutex does not have multiple modes. When locked, any thread that
  attempts to lock it will block until the lock is released.

### The protocol

There are two files that will be concurrently acted upon by threads in this project, the **database** file and the
**index** file. The database file contains all the data in the database. The database file is uncomplicated compared to
professional, production implementations of database engines, for example, PostgreSQL or Oracle, because the entirety of
the database file consists of exactly one table with a fixed schema. There are no foreign keys, foreign tables, other
databases, and the database only supports the operations: database_search, database_insert, and database_delete. No
complicated queries are supported.

The index file contains an implementation of a B-Link Tree [2], [3]. The index is divided up into pages. To read a page,
the calling thread does not need to acquire a lock since B-Link Trees don't need to acquire read locks. However, to
modify a page, the calling thread must acquire a mutex on it. To do this, it needs to hash the page's header and create
a new entry in the lock table, where it initializes the mutex associated with the page and initializes a queue with
which any other transactions that wish to modify this page will be serialized. The queue comes with its own thread,
mutex, and condition variable function that will process transactions serially. This is important because if
transactions were not processed serially, two transactions could result in two different database states after
completion. If the queue is empty then it will be destroyed immediately since it is not necessary anymore.

To search the database, the calling thread must perform a (lockless) search on the index file. Once it finds the index
page where the item to be searched is presumed to be found, it will either find the item or not find the item. If it
does not find the item, the thread should return a message to the user that states that no item that matched their
search could be found. If it does find the item, then it should acquire a read lock on the database file. If there are
no writers present on the database, then it will successfully acquire the lock, and then it should find the row that
matches the user's search and return it to the user. If there are writers on the database, then the thread will block
until there aren't any, and then do the same as before.

To insert an element into the database, the calling thread must perform the same lockless search as in the search
operation on the index file. However, this time, it must first initialize a stack, and keep track of every page it
encounters on the path to the leaf that possibly contains the element. If it arrives at a leaf and the leaf contains the
element, the thread will return with an error message to the user, since duplicates are not allowed in the database. If
no element is found, a write lock will be acquired on the index page but nothing will be written just yet. First, the
calling thread will acquire a write lock on the database so that it blocks all readers from reading an inconsistent
database. Once it receives it, the thread will modify the index, adding the element into the proper disk page. The
insert process will obtain and release locks on the disk pages in the index as necessary and all locks will be released
once the process terminates. Then, it will modify the database and add the row. Finally, the lock on the database will
be released. This ensures that at all times, the database state remains consistent. At no point can a thread view an
element in the index and not view it in the database. At no point can a thread observe an element in the database that
is not also in the index.

Bulleted Steps For Search:

- Search through index for item. If not found, return. Otherwise, continue to next step.
- Acquire read lock on database
- Find row and store it in memory.
- Release read lock
- Return row to calling thread

Bulleted Steps for Insert:

- Search through index for item. If found, return. Otherwise, acquire write lock on page and proceed to the next step.
- Acquire write lock on database
- Perform the insert of the key into the index
- Append row to database file
- Release the write lock on the database (any locks held on the database are released in the previous step)

Proof:

Consider a search operation (T1) and an insert operation (T2) which are happening concurrently on a database. T1 is an
operation that asks the database to search for a row (R) based on a primary key (K) and T2 is an operation that asks the
database to insert row with primary key (K). If T1 finishes before T2 starts or before T2 has acquired the write lock on
the database, the program will not be able to find the row (because duplicates are not allowed, so it is impossible that
a row with primary key (K) already exists in the database). If T2 finishes before T1 starts, then the search will find
the row indexed with primary key (K) with no issue.

If T1 starts when the T2 has just acquired a write lock on the database but done nothing else, then T1 will return
without finding the row because the row has neither been inserted in the index nor in the database. If T1 starts when
the insert thread has acquired a write lock on the database but not modified the index yet, then this means that T2 has
not written anything to either file yet. T1 will terminate and return to the user that the row was not found. If T1
starts when the key has been inserted into the index but no row has been appended to the database, then T1 will see that
the row exists, and will attempt to acquire a read lock on the database. It will block until it acquires a read lock
(until T2 releases its write lock). Once T2 releases its write lock, the changes will be reflected in the database, and
now T1 will acquire the read lock. When T1 acquires the read lock, it will be able to find R successfully and return it
to the user. If T1 begins when the row has been appended to the database and is present in the index, the result will be
the exact same as the previous case. Therefore, no matter when T1 occurs with respect to T2, the results will always be
consistent.

[1] Programming with POSIX Threads by Dave Butenhof
[2] Efficient Locking for Concurrent Operations on B-Trees by Philip L. Lehman and S. Bing Yao.
[3] Concurrency control and recovery for balanced B-Link trees by Ibrahim Jaluta, et al. 