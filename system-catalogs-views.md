# Used system catalogs, views, extensions and applications
- [pg_buffercache](#pg_buffercache)
- [pg_controldata](#pg_controldata)
- [pg_stat_recovery_prefetch](#pg_stat_recovery_prefetch)
- [pg_stat_checkpointer](#pg_stat_checkpointer)

[Functions Used](#functions-used)

[References](#references)


## `pg_buffercache` 

The module provides a means for examining what's happening in the shared buffer cache in real time. It also offers a low-level way to evict data from it, for testing purposes.

## `pg_controldata` 

The server application displays control information of a PostgreSQL database cluster. It prints information initialized during initdb, such as the catalog version. It also shows information about write-ahead logging and checkpoint processing. This information is cluster-wide, and not specific to any one database.

## `pg_stat_recovery_prefetch`

The single row view shows statistics about WAL referenced data blocks prefetched by a standby during recovery. The columns `wal_distance`, `block_distance` and `io_depth` show current values, and the other columns show cumulative cluster-wide counters that can be reset with the `pg_stat_reset_shared` function.

| Column | Description |
| --- | --- |
| stats_reset | Time these stats were last reset |
| prefetch | Blocks prefetched (not in buffer pool) |
| hit | Blocks already in buffer pool |
| skip_init | Blocks skipped (would be zero-initialized) |
| skip_new | Blocks skipped (didn't exist yet) |
| skip_fpw | Blocks skipped (full page write) |
| skip_rep | Blocks skipped (repeated) |
| wal_distance | Current WAL prefetch distance from the replay lsn (not reset) |
| block_distance | Current block prefetch distance (not reset) |
| io_depth | Current I/O requests issued/hinted to the OS (not reset) |


## `pg_stat_checkpointer`

The single-row view contains data about the checkpointer process of the cluster - cumulative till present date.

| Column | Description |
| --- | --- | 
| num_timed	| Checkpoints triggered by `checkpoint_timeout` (includes skipped ones when idle) |
| num_requested | Checkpoints triggered externally (e.g., CHECKPOINT command, `max_wal_size` reached, shutdown) |
| write_time | Total ms spent writing dirty pages to disk during checkpoints/restartpoints |
| sync_time | Total ms spent fsync'ing those pages to disk |
| buffers_written | Total number of shared buffers (8 KB pages) written during checkpoints/restartpoints |
| stats_reset | Timestamp of last reset |

restartpoints_timed, restartpoints_donea are columns revelant for standby.

## Functions Used

pg_size_pretty()
pg_walfile_name()
pg_relation_size()
pg_current_wal_lsn()
pg_stat_reset_shared()
pg_relation_filenode()

---

## References

[pg_buffercache](https://www.postgresql.org/docs/current/pgbuffercache.html)

[pg_controldata](https://www.postgresql.org/docs/current/app-pgcontroldata.html)