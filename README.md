# pglockanalyze: analyze postgres locking behavior

pglockanalyze examines queries from the Postgresql's "pg\_locks" system catalog
and produces some summary output about which backend processes are blocked on
which others.

This was really a one-off written to deal with a specific problem, but may be
helpful for others in the future.

Specifically, the Postgres wiki suggests using [this
query](https://wiki.postgresql.org/wiki/Lock_dependency_information) to get a
"flat view of blocking [processes]":

```sql
SELECT 
    waiting.locktype           AS waiting_locktype,
    waiting.relation::regclass AS waiting_table,
    waiting_stm.query          AS waiting_query,
    waiting.mode               AS waiting_mode,
    waiting.pid                AS waiting_pid,
    other.locktype             AS other_locktype,
    other.relation::regclass   AS other_table,
    other_stm.query            AS other_query,
    other.mode                 AS other_mode,
    other.pid                  AS other_pid,
    other.GRANTED              AS other_granted
FROM
    pg_catalog.pg_locks AS waiting
JOIN
    pg_catalog.pg_stat_activity AS waiting_stm
    ON (
        waiting_stm.pid = waiting.pid
    )
JOIN
    pg_catalog.pg_locks AS other
    ON (
        (
            waiting."database" = other."database"
        AND waiting.relation  = other.relation
        )
        OR waiting.transactionid = other.transactionid
    )
JOIN
    pg_catalog.pg_stat_activity AS other_stm
    ON (
        other_stm.pid = other.pid
    )
WHERE
    NOT waiting.GRANTED
AND
    waiting.pid <> other.pid
```

[Each row in the pg_locks
view](http://www.postgresql.org/docs/9.2/static/view-pg-locks.html) represents a
Postgresql backend process either holding or attempting to take a lock.
Processes holding or attempting to take multiple locks may appear multiple times
in the view.

The query above essentially selects all pairs of rows from "pg\_locks" that are
operating on the same lock, and then joins that information with the
"pg\_activity" information for each backend.  In principle, this can show you
when one process is blocking another, but interpretation is complicated by the
fact that this query reports rows where neither process holds the lock
(indicating they're both contending for the lock, but not necessarily blocking
each other).  It also reports rows for lock types that don't conflict.  For
example, the pids in a given row may both be attempting to take a shared
(non-exclusive) lock on a table, in which case it's not a problem that they're
both contending for it.

pganalyze takes on stdin the textual output you'd get by running this query
using psql(1), and produces on stdout a report like this:

    $ psql -c "..." > query.out
    $ pglockanalyze < query.out
    ...
    pid 93744: query: SELECT *, '488f0950-bf92-405a-8b77-ae700accca48' AS req_id FROM manta WHERE _key=$1
        wants AccessShareLock: may contend with 1 pids wanting AccessExclusiveLock (e.g., 42367)
        wants AccessShareLock: may contend with 983 pids wanting AccessShareLock (e.g., 93618)
        wants AccessShareLock: may contend with 9 pids wanting RowShareLock (e.g., 93737)
    pid 93759: query: SELECT *, '8dce62c2-4f42-4c70-87be-d62b1c1f84f4' AS req_id FROM manta WHERE _key=$1 FOR UPDATE
        wants RowShareLock: may contend with 1 pids wanting AccessExclusiveLock (e.g., 42367)
        wants RowShareLock: may contend with 984 pids wanting AccessShareLock (e.g., 93618)
        wants RowShareLock: may contend with 8 pids wanting RowShareLock (e.g., 93737)
    pid 91544: query: autovacuum: VACUUM ANALYZE public.manta (to prevent wraparound)
        holds ShareUpdateExclusiveLock: may contend with 1 pids wanting AccessExclusiveLock (e.g., 42367)
        holds ShareUpdateExclusiveLock: may contend with 984 pids wanting AccessShareLock (e.g., 93618)
        holds ShareUpdateExclusiveLock: may contend with 9 pids wanting RowShareLock (e.g., 93737)
    pid 42367: query: DROP TRIGGER IF EXISTS count_directories ON manta
        holds AccessShareLock: may contend with 984 pids wanting AccessShareLock (e.g., 93618)
        holds AccessShareLock: may contend with 9 pids wanting RowShareLock (e.g., 93737)
        wants AccessExclusiveLock: may contend with 984 pids wanting AccessShareLock (e.g., 93618)
        wants AccessExclusiveLock: may contend with 9 pids wanting RowShareLock (e.g., 93737)

In order to make sense of this, you're really going to have to read and
understand the Postgresql documentation on [explicit
locking](http://www.postgresql.org/docs/9.2/static/explicit-locking.html).

There's also a "json" mode that converts the rows to json.  You can then query
them with tools like json(1):

    $ pglockanalyze json < query.out > locks.json


# Wish list

* It would be easier (both for users and for the implementation of this program)
  if the input were actually just the contents of the "pg\_locks" and
  "pg\_activity" views, rather than pairs of pids that _might_ be related.
* Relatedly, instead of looking at pairs of processes blocking each other, it
  would be useful to pool together all information about a single lock.
