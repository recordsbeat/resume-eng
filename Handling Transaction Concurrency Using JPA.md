### Introduction

I read the article over the weekend.

https://lion-king.tistory.com/entry/SpringJPATransaction-Write-skew-Phantom-1

An article that a transaction concurrency issue occurred between using JPA.
Not the usual update, but
select - isRowExist ? It was the situation of update : insert.
There was a table with the same structure in the real world, and I carefully read the issue.


### grasp the contents

There are difficult words from the very beginning.

> "Using an object-relational mapping framework makes it easy to write code that accidentally executes an insecure read-modify-write cycle instead of using the atomic operations provided by the database."

When using JPA (hibiernate - ORM), there are things to consider about transactions, and
the reason is as described below.

> Isolation Level <br>
Commonly used databases often have an isolation level corresponding to READ COMMITTED.
However, when using JPA, if an entity loaded into the persistence context is retrieved again, the entity is retrieved from the persistence context without looking up the database, so it behaves the same as the REPEATABLE READ isolation level.<br>
https://reiphiel.tistory.com/entry/understanding-jpa-lock

Then how is the isolation level of JPA determined? It is said that
**isolation of JPA (hibernate) is decided by Database Vendor**,
and **it can be changed by @Transactional of Spring** .
However, it is always necessary to pay attention to the persistence caching mentioned above.

> *default isolation level of Hibernate <br>
https://allaroundjava.com/transaction-management-hibernate/

One example is 'Write Skew'. It can be understood to the extent that a certain value is written and the content is different from the original one. Here's a closer look:

Only the values ​​that have changed according to the dirty checking mechanism of the persistent object are updated. However, if a value that is not the subject of update is changed at the time of update through dirty checking, it indicates a situation in which the relevant part cannot be recognized. ex) Balance withdrawal, point deduction
(this can be seen as a lost update issue. instead of a link.)

> A beginner’s guide to Read and Write Skew phenomena <br>
https://vladmihalcea.com/a-beginners-guide-to-read-and-write-skew-phenomena/




### Solutions


**Optimistic lock**

It is called an optimistic lock because it is from the perspective that locks do not occur between transactions.
@Version provided by JPA is used, and the version column is specified in the table, and the corresponding column is updated by +1 between updates.

If the version column data of the update target between transactions is different from the version data of the persistent object, an exception is raised and a compensation transaction is performed.

**PESSIMISTIC lock**

It is called a pessimistic lock because it is a view that locks will occur between transactions.
In mysql, the lock is requested with the same query as ~~ for update.



### Tests and results

As mentioned above, there is a similar logic in practice.
After selecting a specific Id, if there is no value in the associated table, insert if there is update.

Referring to the link above, I wrote the test code as follows.

```
	@Test
    void updateConcurrencyTest() throws InterruptedException
    {
		Long Id = 100L;
        int numberOfThreads = 10;
        ExecutorService service = Executors.newFixedThreadPool(10);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        boolean result;
        for(int i = 0; i < numberOfThreads; i++) {
            int finalI = i;
            service.execute(() -> {
                try {
                    service.updateById(Id, UseYn.Y);
                    System.out.println("Tid : " + finalI);
                } catch(Exception e) {
                    e.printStackTrace();
                }
                latch.countDown();
            });
        }

        latch.await();

        Entity entity = entityRepository.findById(Id)
            .orElseThrow(() -> new RuntimeExeception("Not Found"));


        Assertions.assertEquals(UseYn.Y, entity.getUseYn());
    }
```

![](https://images.velog.io/images/recordsbeat/post/4b86c04d-e79c-43b0-9076-529e9a669da9/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-02-03%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2010.53.29.png)

Also, the same error occurred in the insert logic.
At the time of insert, a unique value already exists in the target.



**Trial**

**1. PESSIMISTIC (pessimistic) lock - PESSIMISTIC_WRITE - success**
```
@Lock(LockModeType.PESSIMISTIC_WRITE)
List<Entity> findAllByEntityIdIn(@Param("Id") List<Long> Id);
```
(Annotation has been added to the repository method.)

```
select ..
where Id in (1234) for update
```

x lock is used for the for update query.
Only one insert and the rest are all updated.
In the example, it was said that a dead lock was caught (why...?) I was fortunately successful.






**2. PESSIMISTIC (pessimistic) lock - PESSIMISTIC_READ - fail**

```
select ..
where id in (1234) lock in share mode 
```
As above, the query was operated in lock in share mode.
(Note: From MySQL 8.0, you can simply write FOR SHARE instead of the existing LOCK IN SHARE MODE. (For backward compatibility, the existing syntax is executed without any problem.)
The following deadlock occurred due to share lock.
![](https://images.velog.io/images/recordsbeat/post/2e5c47b0-7e41-42d4-aec5-fc48fda742c1/image.png)


**3. Optimistic lock - OptimisticLockException and reward transaction application-failure **


OptimisticLockException is an exception that literally occurs during a transaction using an optimistic lock.
![](https://images.velog.io/images/recordsbeat/post/fd2b03c3-c94d-4a44-9d0f-3aa84f73769f/image.png)


I thought it would be the easiest way to succeed, but I failed.
The compensation transaction for transaction failure could not be executed because DeadLock occurred before the OptimisticLockException occurred.
The reason is that I was obsessed with the duplicate key above and focused only on the 'insert if it doesn't exist' logic. Basically, if you make a version column and notice a row update,
it seems appropriate to use it as a preventative measure for lost update, not insert.

> Hibernate Optimistic Locking for Concurrent Insert-possible? <br>
https://stackoverflow.com/questions/15761661/hibernate-optimistic-locking-for-concurrent-insert-possible


**what's best?**

While conducting the above test, I found a solution using pessimistic and optimistic locks
. Judging from the special case in which insert and update are performed in one method, it is also important to properly use the locking technique appropriate for each situation of insert and update.
It is also necessary to correctly understand the characteristics and strengths and weaknesses of each lock method.

> Concurrency Problem - Bussiness Application <br>
One of the business application concurrency control techniques, optimistic locking , can be used to structure the architecture to solve the unreliable inventory problem.
However , **if there is a process such as “interlocking external systems” that prevents the atomicity of a transaction from being guaranteed during the entire process, optimistic locking is difficult to use.** With optimistic locking, the failure of the entire process is known at the time of the last save attempt, because if the process is atomically difficult to rollback, the integrity of the entire system is broken.
**In this case, you can use “pessimistic locking” which can increase accuracy at the cost of a little bit of system activity.**
Pessimistic locking is less active than optimistic locking, so customers will experience slower orders at peak times, but less likely to experience inaccurate systems such as payment only and cancellations.
http://jaynewho.com/post/44



-------------------------------------------------------------------


### additional notes

> **Database Transactions: Difference between 'write skew' and 'lost update'**
https://stackoverflow.com/questions/27826714/database-transactions-difference-between-write-skew-and-lost-update

> **Isolation Level of Transaction understood as Lock**
https://suhwan.dev/2019/06/09/transaction-isolation-level-and-lock/

> **Mysql innoDB Lock, isolation level and Lock competition**
https://taes-k.github.io/2020/05/17/mysql-transaction-lock/

> **[JPA] Understanding persistence context and flushing**
https://ict-nroo.tistory.com/130


> **Transaction isolation in Hibernate**
https://www.waitingforcode.com/hibernate/transaction-isolation-in-hibernate/read

> **Optimistic Lock and Pessimistic Lock**
https://effectivesquid.tistory.com/entry/Optimistic-Lock%EA%B3%BC-Pessimistic-Lock




---

### Addtional Thoughts after discuss with colleague

As a result of sharing opinions with team members,
it is said that using for update may cause side effects such as bottlenecks. (due to lock..)

If it is not a batch program, it may be better to make a failure case by implementing it as a rollbackable method as an unchecked exception for a transaction. The view is that it is better for the user to request one more time after the request fails.

Also, if not, there is a way to retry using a reward transaction, but to limit the timeout for retry and the number of retry possible.



One more thing...
A parallel processing phantom read phenomenon occurred between about 50 million batch jobs. To handle the isolation level mentioned above, it is appropriate to use try-catch and unique key exceptions
because the weight of the task to be executed by the batch is large . However, when using JPA, a phenomenon of rollback may occur depending on the caller.


Woowa Brothers - Why is this a rollback..?
https://woowabros.github.io/experience/2019/01/29/exception-in-transaction.html

Therefore, even if a separate retry logic through an exception is used, careful coding is required so that an unexpected rollback does not occur.
