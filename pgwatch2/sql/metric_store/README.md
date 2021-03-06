# Rollout sequence

First rollout the below files and then the chosen schema type's folder contents or use the according "roll_out_*" files.2
* "00_schema_base.sql" - schema type and listing of all known "dbname"-s are stored here
* "01_old_metrics_cleanup_procedures.sql" - used to list all unique dbnames and to delete/drop old metrics by the application (can also be used for manual cleanup).

# Schema types

## metric

A single / separate table for each distinct metric in the "public" schema. No partitioning. Works on all PG versions. Suitable for up to ~25 monitored DBs.

## metric-time

A single top-level table for each distinct metric in the "public" schema + weekly partitions in the "subpartitions" schema.
Works on PG 11+ versions. Suitable for up to ~50 monitored DBs. Reduced IO compared to "metric" as old data partitions will be dropped, not deleted.

Default storage schema for the "pgwatch2-postgres" Docker image.

## metric-dbname-time

A single top level table for each distinct metric in the "public" schema + 2 levels of subpartitions ("dbname" + weekly time based) in the "subpartitions" schema.
Works on PG 11+ versions. Provides the fastest query runtimes when having long retention intervals / lots of metrics data or slow disks and accessing mostly only a single DB's metrics at a time.
Best used for 50+ monitored DBs.

Also note that when having extremely many hosts under monitoring it might be necessary to increase the "max_locks_per_transaction"
postgresql.conf parameter on the metrics DB for automatic old partition dropping to work. One could of course also drop old
data partitions with some custom script / Cron when increasing "max_locks_per_transaction" is not wanted, and actually this
kind of approach is also working behind the scenes for versions above v1.8.1.

## custom

For cases where the available presets are not satisfactory / applicable. All data inserted into "public.metrics" table and the user is responsible for re-routing with a trigger and possible partition management. In that case all table creations and data cleanup must be performed by the user.

## timescale

Assumes TimescaleDB (v1.7+) extension and "outsources" partition management for normal metrics to the extensions. Realtime
metrics still use the "metric-time" schema as sadly Timescale doesn't support unlogged tables. Additionally one can also
tune the chunking and historic data compression intervals - by default it's 2 days and 1 day. To change use the
admin.timescale_change_chunk_interval() and admin.timescale_change_compress_interval() functions.

Most suitable storage schema when using long retention periods due to built-in extra compression.

# Data size considerations

When you're planning to monitor lots of databases or with very low intervals, i.e. generating a lot of data, but not selecting
all of it actively (alerting / Grafana) then it would make sense to consider BRIN indexes to save a lot on storage space. See
the according commented out line in the table template definition file.

# Notice on "realtime" metrics

Metrics that have the string 'realtime' in them are handled differently on storage level to draw less resources:

 * They're not normal persistent tables but UNLOGGED tables, meaning they're not WAL-logged and cleared on crash
 * Such subpartitions are dropped after 1d

# Notice on Grafana access to metric data and GRANT-s

For more security sensitive environments where a lot of people have access to metrics you'd want to secure things a bit by
creating a separate read-only user for queries generated by Grafana. And to make sure that this user, here "pgwatch2_grafana",
has access to all current and future tables in the metric DB you'd probably want to execute something like that:
```sql
ALTER DEFAULT PRIVILEGES GRANT SELECT ON TABLES TO pgwatch2_grafana;

GRANT USAGE ON SCHEMA public TO pgwatch2_grafana;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO pgwatch2_grafana;

GRANT USAGE ON SCHEMA admin TO pgwatch2_grafana;
GRANT SELECT ON ALL TABLES IN SCHEMA admin TO pgwatch2_grafana;

GRANT USAGE ON SCHEMA subpartitions TO pgwatch2_grafana;
GRANT SELECT ON ALL TABLES IN SCHEMA subpartitions TO pgwatch2_grafana;
```