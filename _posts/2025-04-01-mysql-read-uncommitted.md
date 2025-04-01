---
title: "Pitfalls of MySQL's READ UNCOMMITTED"
date: 2025-04-01
tags:
  - mysql
categories:
  - MySQL
---

In this article, I discuss a quirk of `READ UNCOMMITTED` in InnoDB (MySQL's default storage engine) that can lead to surprising behaviour,
and discuss the implementation characteristics causing it.

Consider the following table:

[//]: # (@formatter:off)
```sql
create table t (
  n int primary key
);

insert into t (n)
values (1), (2), (3), (4), (5), (6), (7), (8), (9), (10);
```
[//]: # (@formatter:on)

and the following sequence of statements run by two concurrent transactions, `A` and `B`:

[//]: # (@formatter:off)
```sql
-- session A
set session
  transaction isolation level
  read uncommitted;

select count(*) from t; -- 10

                                     -- session B, concurrently
                                     -- with A's last statement
                                     update t
                                     set n = n + 10;
-- session A, concurrently
-- with the B's UPDATE
select count(*)
from t; -- 14 (!)
```
[//]: # (@formatter:on)

From the point of view of session `A`, session `B` inserted 4 rows into table `t`, even though all it did was update existing rows!

To reproduce this, I built a [test harness](https://github.com/LeMikaelF/mysql-non-consistent-read-test) that races two transactions until
it finds an issue. For this specific example, it takes anywhere between 500 and a few thousand runs on my computer (Mac M2 Max) to produce
an invalid count.

Similar effects can be obtained with variations of this transaction. For example, issuing a `select * from t` as the last statement from
session `A` will show that it reads both old and new versions of the rows updated by session `B`, hence the 4 extra rows (more on this
later).

## A variation

It's easy to reproduce a similar issue manually as long as the table has
an [auto-generated primary key](https://dev.mysql.com/doc/refman/8.4/en/innodb-index-types.html). If the primary key is not auto-generated,
meaning either it is specified explicitly (like above), or the table contains a `UNIQUE NOT NULL` index, this issue is still
reproducible using the test harness linked above, but I wasn't able to reproduce it manually, since the timing constraints seem to be a lot
stricter.

[//]: # (@formatter:off)
```sql
create table t (
  n int
)

-- session A
set session
  transaction isolation level
  read uncommitted;

-- slow down session A's table scan
select *, sleep(1)
from t;
                                             -- session B, while session A is working
                                             update t
                                             set n = 1000;
```
[//]: # (@formatter:on)

This will return something like this:

| n    | sleep(1) |
|------|----------|
| 1    | 1        |
| 2    | 1        |
| 3    | 1        |
| 1000 | 1        |
| 1000 | 1        |

In this case, the first 3 rows were scanned before the `UPDATE`, and the last 2 were scanned after.

## The culprit: MVCC and non-consistent reads

This quirk is due to how InnoDB (MySQL's default storage engine) deals with concurrency using a strategy known
as [Multi-Version Concurrenty Control](https://web.archive.org/web/20150621224732/http://dev.mysql.com/doc/refman/5.5/en/innodb-multi-versioning.html)
(MVCC), and how `READ UNCOMMITTED` ignores MVCC.

MVCC is used by InnoDB to give the illusion that all rows in a `SELECT` statement were scanned at the exact same time. For such reads, the
storage engine uses [undo logs](https://dev.mysql.com/doc/refman/8.4/en/innodb-undo-logs.html) to perform computation analogous to a Ctrl-Z
in a text editor to "undo" concurrent changes. MySQL is therefore able to present the viewer with a "concurrent" read, where each row
belongs to the same snapshot.[^1]

During a transaction, InnoDB optimistically writes (or overwrites) data directly in
the [clustered index](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_clustered_index), even before it commits, in addition
to leaving an undo log record that can be used in case of a rollback or during crash recovery. This means that commits are basically free,
whereas rollbacks require reconstructing the old rows and writing them back in the clustered index. This makes sense under the assumption
that most transactions end up commiting, and that rollbacks are relatively rare.

Because of this, a transaction that does not use MVCC will see uncommitted data, but its reads will also be non-consistent. Here is a
representation of the steps that could lead to the anomalous `count(*)` discussed above, with a 3-row table for simplicity:

```text
1 => 1 2 3
     ^- SELECT reads this
     
2 => 1 2 3
       ^- SELECT reads this
      
3 => 1 2 3
         ^- SELECT reads this

4 => UPDATE t SET n = n + 3
      
5 => 1̶-2̶-3̶ 4 5 6
     ^^^^^- UPDATE removes 1-3 and adds 4-6

6 => 1̶-2̶-3̶ 4 5 6
           ^- SELECT keeps going
```

The primary key affects where rows are stored in the clustered index (a B+tree), so updating primary keys requires moving records to
different parts of the index. I haven't been able to prove (or disprove) that this is in fact what happens, but based on what I know about
how InnoDB does MVCC, it seems reasonable.

## Some confusion in the documentation

The fact that reads in `READ UNCOMMITTED` are not consistent is
indeed [documented](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html#isolevel_read-uncommitted), but the
relevant paragraph is somewhat unclear:

> SELECT statements are performed in a nonlocking fashion, but a possible earlier version of a row might be used. Thus, using this isolation
> level, such reads are not consistent. This is also called a dirty read. Otherwise, this isolation level works like READ COMMITTED.

First, a "possible earlier version of a row" might definitely not be used, contrary to what the documentation says. That's exactly the point
of not using MVCC, that undo logs cannot be used to provide consistent snapshots by reconstructing previous versions of a row (see also
[this comment](https://github.com/mysql/mysql-server/blob/trunk/storage/innobase/include/trx0trx.h#L678) in the source code).

Second, there seems to be some confusion in the documentation between dirty reads and consistent reads. Dirty reads are not non-consistent
reads, unlike what the paragraph above and [MySQL's glossary](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_dirty_read)
suggest. Read phenomena (dirty read, non-repeatable
read, and phantom read) are famously ill-defined in SQL-92, but both in the SQL standard and
in [Adya's formalization of read phenomena](https://pmg.csail.mit.edu/papers/icde00.pdf) (or _read anomalies_), dirty reads are not related
to consistent reads. Dirty reads are when a transaction reads uncommitted data from another transaction.

This confusion might be due to how MVCC interacts with dirty reads. In MVCC, the engine effectively looks at each row and asks "can I see
this?" based on some information in the row. In InnoDB, this information is the hidden column `DB_TRX_ID`, which stores the identifier of
the latest transaction that modified the row. The algorithm InnoDB uses to determine of a transaction can see a given record is located in
the [`changes_visible(trx_id_t, table_name_t)`](https://github.com/mysql/mysql-server/blob/trunk/storage/innobase/include/read0types.h#L163)
function, and consists of the following checks:

1. is `DB_TRX_ID` low enough that we can safely say that the row is visible?
2. is `DB_TRX_ID` our own id?
3. is `DB_TRX_ID` high enough that we can safely say that the row is not visible?
4. if there were other ongoing transactions when the snapshot was taken, does `DB_TRX_ID` belong to one of them?

If we want to allow dirty reads, then we have to skip these checks. But the problem is that then, when you ask "can I see this?", the
database does now have enough information to tell you if the row belongs to a completed statement or if it belongs to an ongoing statement,
and therefore chooses to allow them both.

An MVCC architecture that distinguishes between completed and ongoing statements of an ongoing transaction would have to keep track of a
more granular identifier than just transaction ids. For instance, the `DB_TRX_ID` column would have to also include a per-transaction
statement counter. This way, a variation of the algorithm above would be able to answer the question "does this row belong to a completed
statement?" regardless of the transaction status. But the performance cost of such a strategy would likely be non-negligible.

This could be (although I'm just guessing) why Postgres chose to not implement `READ UNCOMMITTED`, and in indeed,
the [documentation](https://www.postgresql.org/docs/current/transaction-iso.html) hints at this:

> PostgreSQL's Read Uncommitted mode behaves like Read Committed. This is because it is the only sensible way to map the standard isolation
> levels to PostgreSQL's multiversion concurrency control architecture.

## A good use case for READ UNCOMMITTED

Because this lack of read consistency in `READ UNCOMMITTED` can lead to surprising results, including things like an `UPDATE` statement
appearing to insert or delete rows, my recommendation is to only use this isolation level if the explicit goal of a query is to view
uncommitted data.

I've seen one such use case. When working with [ORMs](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping), sometimes it can be
unclear what's happening at the database level. If you stop the application or the test mid-transaction, either with a breakpoint or with a
long sleep instruction, and you log into the database, you won't be able to see what the application is doing with the data, because MySQL
usually prevents dirty reads. But you can disable this behaviour with:

```sql
set session transaction isolation level read uncommitted;
```

and then you'll be able to see what your application is doing.

## Conclusion

MySQL's `READ UNCOMMITTED` can lead to all sorts of surprises beyond dirty reads, making it unreliable. Because InnoDB uses MVCC, it can't
distinguish between dirty reads and non-consistent reads. For this reason, apart from troubleshooting applications or integration tests,
you're better off staying away from `READ UNCOMMITTED` and sticking with `READ COMMITTED` and up.

## Footnotes

[//]: # (@formatter:off)
[^1]: Many databases use MVCC, see this [list](https://en.wikipedia.org/wiki/List_of_databases_using_MVCC) on Wikipedia. Some file systems, such as ZFS, also use similar strategies to provide snapshots ([more info](https://www.open-e.com/blog/how-do-zfs-snapshots-really-work/)). 
[//]: # (@formatter:on)
