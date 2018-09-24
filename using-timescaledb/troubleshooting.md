# Troubleshooting

If you run into problems when using TimescaleDB, there are a few things that you
can do.  There are some solutions to common errors below as well as ways to output
diagnostic information about your setup.  If you need more guidance, you can join
the support [Slack group][slack] or post an issue on the TimescaleDB [Github][].

## Performance Best Practices

### Inserting multiple metrics with the same timestamp

Recommended pgtune. Turn of default indexes when setting up hypertable because higher cardinality. Device_id then time index. Matching timestamps. 

### Queries that filter by id

A common querying pattern involves selecting data across a time range and filtering by
an identifier. We typically recommend users with this querying pattern to build an
index on (id, timestamp) so that the query planner can filter by id efficiently.

For users querying a lot of data over a large time range, disk I/O is an important
factor that impacts query performance. Depending on how your data is written to disk
during ingest, queries that involve filtering by id may inefficiently pull many
pages from disk, ultimately slowing down performance.

To speed up this type of query, we recommend either utilizing a covering index or taking
advantage of PostgreSQL's built-in `CLUSTER` command [(PostgreSQL docs)][cluster-docs].
A covering index is formatted as (id, timestamp, value), and thus allows fast queries
filtered by id. However, it does require building another index that takes up disk space.
The `CLUSTER` command, on the other hand, re-orders data written to disk by an index, thus
making disk access more efficient for queries filtering by id. The `CLUSTER` command, however,
takes a lock on a table. We recommend running the `CLUSTER` command one chunk at a time to reduce
the amount of locking that occurs.

### Storing and querying waveforms

Waveforms are a common data type associated with IoT use cases. This typically involves raw data
that is collected multiple times a second, but more often queried at an aggregated level.

## Common Errors
### Error updating TimescaleDB when using a third-party PostgreSQL admin tool.

The update command `ALTER EXTENSION timescaledb UPDATE` must be the first command
executed upon connection to a database.  Some admin tools execute command before
this, which can disrupt the process.  It may be necessary for you to manually update
the database with `psql`.  See our [update docs][update-db] for details.

###  Log error: could not access file "timescaledb" [](access-timescaledb)

If your PostgreSQL logs have this error preventing it from starting up,
you should double check that the TimescaleDB files have been installed
to the correct location. Our installation methods use `pg_config` to
get PostgreSQL's location. However if you have multiple versions of
PostgreSQL installed on the same machine, the location `pg_config`
points to may not be for the version you expect. To check which
version TimescaleDB used:
```bash
$ pg_config --version
PostgreSQL 9.6.6
```

If that is the correct version, double check that the installation path is
the one you'd expect. For example, for PostgreSQL 10.1 installed via
Homebrew on macOS it should be `/usr/local/Cellar/postgresql/10.1/bin`:
```bash
$ pg_config --bindir
/usr/local/Cellar/postgresql/10.1/bin
```

If either of those steps is not the version you are expecting, you need
to either (a) uninstall the incorrect version of PostgreSQL if you can or
(b) update your `PATH` environmental variable to have the correct
path of `pg_config` listed first, i.e., by prepending the full path:
```bash
$ export PATH = /usr/local/Cellar/postgresql/10.1/bin:$PATH
```
Then, reinstall TimescaleDB and it should find the correct installation
path.

### ERROR: could not access file "timescaledb-\<version\>": No such file or directory [](alter-issue)

If the error occurs immediately after updating your version of TimescaleDB and
the file mentioned is from the previous version, it is probably due to an incomplete
update process. Within the greater PostgreSQL server instance, each
database that has TimescaleDB installed needs to be updated with the SQL command
`ALTER EXTENSION timescaledb UPDATE;` while connected to that database.  Otherwise,
the database will be looking for the previous version of the timescaledb files.

See [our update docs][update-db] for more info.

---

## Getting more information

###  EXPLAINing query performance [](explain)

PostgreSQL's EXPLAIN feature allows users to understand the underlying query
plan that PostgreSQL uses to execute a query. There are multiple ways that
PostgreSQL can execute a query: for example, a query might be fulfilled using a
slow sequence scan or a much more efficient index scan. The choice of plan
depends on what indexes are created on the table, the statistics that PostgreSQL
has about your data, and various planner settings. The EXPLAIN output let's you
know which plan PostgreSQL is choosing for a particular query. PostgreSQL has a
[in-depth explanation][using explain] of this feature.

To understand the query performance on a hypertable, we suggest first
making sure that the planner statistics and table maintenance is up-to-date on the hypertable
by running `VACUUM ANALYZE <your-hypertable>;`. Then, we suggest running the
following version of EXPLAIN:

```
EXPLAIN (ANALYZE on, BUFFERS on) &lt;original query&gt;;
```

If you suspect that your performance issues are due to slow IOs from disk, you
can get even more information by enabling the
[track\_io\_timing][track_io_timing] variable with `SET track_io_timing = 'on';`
before running the above EXPLAIN.

---

## Dump TimescaleDB meta data [](dump-meta-data)

To help when asking for support and reporting bugs,
TimescaleDB includes a SQL script that outputs metadata
from the internal TimescaleDB tables as well as version information.
The script is available in the source distribution in `scripts/`
but can also be [downloaded separately][].
To use it, run:

```bash
psql [your connect flags] -d your_timescale_db < dump_meta_data.sql > dumpfile.txt
```

and then inspect `dump_file.txt` before sending it together with a bug report or support question.

[slack]: https://slack.timescale.com/
[github]: https://github.com/timescale/timescaledb/issues
[update-db]: /using-timescaledb/update-db
[using explain]: https://www.postgresql.org/docs/current/static/using-explain.html
[track_io_timing]: https://www.postgresql.org/docs/current/static/runtime-config-statistics.html#GUC-TRACK-IO-TIMING
[downloaded separately]: https://raw.githubusercontent.com/timescale/timescaledb/master/scripts/dump_meta_data.sql
[cluster-docs]: https://www.postgresql.org/docs/current/static/sql-cluster.html
