---
title: "Testing Transaction Isolation"
date: 2025-02-7
tags:
  - mysql
categories:
  - MySQL
hidden: true
---

An important part of transactions in databases is that they can be isolated from one another to prevent race conditions, or _read
phenomena_. These phenomena are commonly identified using the SQL 92 terminology: phantom reads, non-repeatable reads, and dirty reads. In
most databases, we decide on the balance between performance and isolation by specifying one of four transaction isolation levels: READ
UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE.

In this article, I want to show a strategy I've used in the past to test transaction isolation. Why would you want to test transaction
isolation? Well, because if you need to think about isolation, it's because you have data to protect against corruption. This seems like a
good cause for testing to me.

For example, consider the common requirement of executing a DML statement if some condition holds. For example:

* perform a balance transfer between two accounts if the corresponding entry in the `operations` table does not have its `processed`
  column set to `true`;
* save a concert ticket if the seat is not already taken in the venue;
* book an airplane ticket if the cargo is under the weight limit.

In these examples, we would want to properly isolate transactions so that a balance transfer isn't processed twice, or so that two
concert-goers don't have to sit on the same seat, or so that a passenger doesn't get rescheduled at the gate because their plane is
overweight.

## The test strategy

The strategy is based on this idea: if two transactions run concurrently, they'll either:

* produce undesirable read phenomena;
* deadlock; or
* produce none of the undesirable read phenomena.

So, to get reasonable confidence that a transaction is correctly isolated, we can race two or more threads repeatedly until one of the
following holds:

* we see an undesirable read phenomena; or
* we've seen N deadlocks, where N is an arbitrary number.

With this in mind, we'll build a test suite to check that we're using the right isolation level so that two threads running the following
transaction concurrently can't step on each other's toes:

```sql
-- insert 1 if it isn't already there
insert into t
select 1 where not exists (select * from t where num = 1)
```

In this case, the result we want to prevent is that `1` could end up twice in the table. This could be better addressed using unique
indexes, but as an example it'll do.

## Implementing the test

Let's start with an empty skeleton:

```java
@SpringBootTest
class MysqlTransactionTestApplicationTests implements WithAssertions {

  @Autowired
  private JdbcTemplate jdbcTemplate;

  @Autowired
  private PlatformTransactionManager platformTransactionManager;

  private TransactionTemplate transactionTemplate;

  @BeforeEach
  void beforeEach() {
    transactionTemplate = new TransactionTemplate(platformTransactionManager);
  }

  @Test
  @SneakyThrows
  void testIsolation() {
    //TODO
  }
}
```

Pretty straightforward. We could autowire a `TransactionTemplate` directly into the test suite, but creating a separate one from the global
template will let us customize its isolation level.

Our test will have an iterative structure, where we count the number of deadlocks, but for now let's start with a single iteration, where we
just race two threads:

```java
private void raceSqlThreads() throws InterruptedException, BrokenBarrierException {
  jdbcTemplate.execute("truncate t");

  String insertSql =
    "insert into t select 1 where not exists (select * from t where num = 1)";

  CyclicBarrier barrier = new CyclicBarrier(2);

  Thread otherThread = new Thread(() ->
    transactionTemplate.execute(throwing(_ -> {
      barrier.await();
      // I'll explain this later
      Thread.sleep(1);
      jdbcTemplate.execute(insertSql);
      return null;
    })));

  otherThread.start();
  barrier.await();
  jdbcTemplate.execute(insertSql);

  otherThread.join();
}
```

I used this little helper to circumvent the unpleasantness of combining checked exceptions and lambdas:

```java
private <T> TransactionCallback<T> throwing(ThrowingCallback<T> callback) {
  return callback;
}

private interface ThrowingCallback<T> extends TransactionCallback<T> {
  @Override
  @SneakyThrows
  default T doInTransaction(@NotNull TransactionStatus status) {
    return doThrowing(status);
  }

  T doThrowing(TransactionStatus status) throws Exception;
}
```

Now, let's build the loop and test harness:

```java
@Test
@SneakyThrows
void testIsolation() {
  for (int numDeadlocks = 0; numDeadlocks < NUM_EXPECTED_DEADLOCKS; ) {
    Exception deadlock = catchThrowableOfType(CannotAcquireLockException.class,
      this::raceSqlThreads);

    if (deadlock != null) {
      numDeadlocks++;
      continue;
    }

    assertNoPhantomRead();
  }
}

private void assertNoPhantomRead() {
  Integer count = jdbcTemplate.queryForObject(
    "select count(*) from t where num = 1", Integer.class);
  assertThat(count)
    .withFailMessage("found phantom read, transaction is not properly isolated")
    .isEqualTo(1);
}
```

We just keep a counter of the deadlocks we've seen, pass the test if we've seen enough, and fail as soon as we see something fishy. The
specific phenomenon we're looking for is actually called G2 (anti-dependency cycle)
in [the literature](https://pmg.csail.mit.edu/papers/icde00.pdf), but let's not open that can of worms.

To make this test fail, set the transaction isolation level to anything lower than `REPEATABLE_READ` on the `TransactionTemplate` (this is
why we didn't just use the global template):

```java
transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
```

On my computer (Apple M2 Max), this test fails in under a second, and with the correct isolation level and `NUM_EXPECTED_DEADLOCKS = 1`, it
mostly passes in under 5 seconds, and most often under 10 seconds, although I've had more unlucky runs of up to 50 seconds.

## Tweaking the test harness

In the past, for more complicated transactions, I've found that adding a short `Thread.sleep()` before one of them was a good way to
maximize the rate of deadlocks. Even with this simple `insert`, I found that I got far more deadlocks if I added a `Thread.sleep(1)`
before the async thread. I'm not quite sure why that is, maybe it's something to do with the connection pool, or the scheduler.

In order to simplify the code, I've deliberately ignored the issue that a test could run forever. Addressing this just requires watching the
number of iterations with an extra counter variable, and failing when it reaches an arbitrary limit.

## Conclusion

Transaction isolation can be the source of hard-to-track bugs and introduce corruption our your data. For these reasons, it's important to
know what can go wrong when our data is modified and accessed concurrently, and to be able to identify the correct isolation levels to
prevent anomalies. In fact, that's
what [my talk at ScALE 22x](https://www.socallinuxexpo.org/scale/22x/presentations/how-not-go-bankrupt-and-look-foolish-mastering-transactions-mysql)
is all about.

This test harness is database-agnostic, but transaction isolation levels [are not](https://github.com/ept/hermitage?tab=readme-ov-file).
Therefore, it's important to understand the safety guarantees provided by a specific database.

I've shown how to build a simple test harness that helps secure development, and catch transaction isolation regressions. In the past, such
tests have served me well, both as a way to secure development, and as a way to prevent regressions. I'm hoping this strategy can now serve
other people as well.

_The code from this article is available [here](https://github.com/LeMikaelF/testing-transaction-isolation)._
