---
title: "Testing Transaction Isolation"
date: 2025-02-7
tags:
  - mysql
categories:
  - MySQL
---

An important part of transactions in databases in that they can be isolated from one another to prevent race conditions (or _read
phenomena_). These phenomena are commonly identified using the SQL 92 terminology: phantom reads, non-repeatable reads, dirty reads. In most
databases, we decide on the balance between performance and isolation by specifying one of four transaction isolation levels: READ
UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE.

In this article, I want to show a strategy I've used to test transaction isolation in the past. Why would you want to test transaction
isolation? Well, because if you need to think about isolation, it's because you have data to protect against corruption. This seems like a
good cause for testing to me.

For example, consider the common requirement of executing a DML statement if some condition holds. For example:

* perform a balance transfer between two accounts if the corresponding entry in the `operations` table does not have its `processed`
  column set to `true`;
* save a concert ticket if the seat is not taken in the venue;
* book an airplane ticket if the cargo is under the weight limit.

In these examples, we would want to properly isolate transactions so that a balance transfer isn't processed twice, or so that two
concert-goers don't have to sit on the same seat, or so that a passenger doesn't get rescheduled at the gate because their plane is
overweight.

## The test strategy

The strategy is based on this idea: if two transactions run concurrently, they'll either:

* produce undesirable read phenomena;
* deadlock; or
* produce none of the undesirable read phenomena.

So to get a reasonable satisfaction that a transaction is correctly isolated, we can race two or more threads repeatedly until one of the
following holds:

* we see an undesirable read phenomena; or
* we have N deadlocks, where N is an arbitrary number.

With this in mind, we'll build a test suite to check that we're using the right isolation level so that two threads running the following
transaction concurrently can't step on each other's toes:

```sql
-- insert 1 if it isn't already there
insert into t
select (select 1
where not exists (select * from t where num = 1))
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

  // language=SQL
  String insertSql = "insert into t select (select 1 where not exists (select * from t where num = 1))";

  Thread otherThread = new Thread(() ->
    transactionTemplate.execute(_ -> {
      jdbcTemplate.execute(insertSql);
      return null;
    }));

  otherThread.start();
  jdbcTemplate.execute(insertSql);

  otherThread.join();
}
```

Now, let's build the loop and test harness:

```java

@Test
@SneakyThrows
void testIsolation() {
  for (int numDeadlocks = 0; numDeadlocks < NUM_EXPECTED_DEADLOCKS; ) {
    Exception deadlock = catchThrowableOfType(CannotAcquireLockException.class, this::raceSqlThreads);

    if (deadlock != null) {
      numDeadlocks++;
      continue;
    }

    assertNoPhantomRead();
  }
}

private void assertNoPhantomRead() {
  Integer count = jdbcTemplate.queryForObject("select count(*) from t where num = 1", Integer.class);
  assertThat(count)
    .withFailMessage("found phantom read, transaction is not properly isolated")
    .isEqualTo(1);
}
```

We just keep a counter of the deadlocks we've seen, pass the test if we've seen enough, and fail as soon as we see something fishy. This 
specific is actually called G2 (anti-dependency cycle) in [the literature](https://jepsen.io/consistency/phenomena/g2) in the literature,
but let's not open that can of worms.

To make this test fail, set the transaction isolation level to anything lower than `REPEATABLE_READ` on the `TransactionTemplate` (this is
why we didn't just use the global template):

```java
transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
```

On my computer (M2 Pro Max), this test fails in under a second, and with the correct isolation level and `NUM_EXPECTED_DEADLOCKS = 1`, if
mostly finishes in under 10 seconds.

## Tweaking the test harness

You might have noticed the lack of synchronization before starting the two queries. I tried having both thread wait on a
`CyclicBarrier` before running their SQL statements, but it reduced drastically the number of deadlocks I was able to generate. I'm not sure
why.

For more complicated transactions, I've had to synchronize both transactions using a `CyclicBarrier` and in one case where the two
transactions I wanted to race weren't identical, I've had to add a short `Thread.sleep()` before one of them in order to maximize the rate
of deadlocks.

In order to simplify the code, I've deliberately ignored the issue that a test could run forever. Fixing this just requires watching the
number of iterations with an extra counter variable, and failing when it reaches an arbitrary limit.

## Conclusion

Transaction isolation can be the source of hard-to-track bugs and introduce corruption in your data. For these reasons, it's important to
know what can go wrong when our data is modified and accessed concurrently, and to be able to identify the correct isolation levels to
prevent anomalies. In fact, that's
what [my talk at ScALE 22x](https://www.socallinuxexpo.org/scale/22x/presentations/how-not-go-bankrupt-and-look-foolish-mastering-transactions-mysql)
is all about.

I've shown how to build a simple test harness that help secure development, and catch transaction isolation regressions. In the past, I used
this strategy at Ticketmaster to guard against race conditions in high-traffic services.

_The code from this article is available [here](https://github.com/LeMikaelF/testing-transaction-isolation)._

[//]: # (TODO say that this is completely database-agnostic, but the isolation levels are not, so it's important to know the specifics of your database)
