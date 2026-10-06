# Idle Server
1. [Settings](#1-check-settings)
2. [Log Location](#2-find-the-log-location)
3. [Manual Checkpoint](#3-manual-checkpoint)
4. [References](#references)

After starting postgres `systemclt start postgresql`, the process ids of auxiliary processes is listed when `systemctl status postgresql` is run.

### 1. Check Settings
> The view `pg_settings` provides access to run-time parameters of the server. It is essentially an alternative interface to the SHOW and SET commands. It also provides access to some facts about each parameter that are not directly available from SHOW, such as minimum and maximum values.

```sql
SELECT name, setting, unit, source
FROM pg_settings
WHERE name IN (
  'shared_buffers','checkpoint_timeout','checkpoint_completion_target',
  'max_wal_size','min_wal_size','checkpoint_warning','checkpoint_flush_after',
  'bgwriter_delay','bgwriter_lru_maxpages','bgwriter_lru_multiplier','bgwriter_flush_after',
  'full_page_writes','wal_compression','log_checkpoints',
  'logging_collector','log_directory','log_filename','data_checksums'
)
ORDER BY name;
```

| name | setting | unit | source |
|---|---|---|---|
| bgwriter_delay | 200 | ms | default | 
| bgwriter_flush_after | 64 | 8kB | default | 
| bgwriter_lru_maxpages | 100 | 		| default | 
| bgwriter_lru_multiplier | 2 | 		| default | 
| checkpoint_completion_target | 0.9 |   		| default | 
| checkpoint_flush_after | 32 | 8kB | default | 
| checkpoint_timeout | 300 | s | default | 
| checkpoint_warning | 30 | s | default | 
| data_checksums | on | 		| default | 
| full_page_writes | on | 		| default | 
| log_checkpoints | on | 		| default | 
| log_directory | log | 		| default | 
| log_filename | postgresql-%a.log | 		| configuration file | 
| logging_collector | on | 		| configuration file | 
| max_wal_size | 1024 | MB | configuration file | 
| min_wal_size | 80 | MB | configuration file | 
| shared_buffers | 16384 | 8kB | configuration file | 
| wal_compression | off | 		| default | 

### 2. Find the log location

```console
sudo ls -lt /var/lib/pgsql/data/log/ | head
```
Last few lines of the latest log file
```console
sudo tail -f /var/lib/pgsql/data/log/<latest-file>.log
```

### 3. Manual Checkpoint

```sql
CHECKPOINT;
```
Check Logs:

```
LOG:  checkpoint starting: immediate force wait
LOG:  checkpoint complete: wrote 0 buffers (0.0%), wrote 0 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.005 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=0 kB; lsn=0/1D5A260, redo lsn=0/1D5A208
```
=>  wrote 0 buffers : no dirty pages (idle server)
=>  0 WAL file(s) added, 0 removed, 0 recycled : WAL directory untouched
=>  sync files=0 : no data files fsynced
=>  distance=0 kB : WAL produced since previous checkpoint's REDO point
=>  estimate=0 kB : moving average used to size the recycle pool
=>  lsn=0/1D5A260 : lsn of the checkpoint record
=>  redo lsn=0/1D5A208 : where replay or recovery would start from 
=>  lsn 88 bytes away from redo lsn : very minimal

> The `pg_stat_checkpointer` view will always have a single row, containing data about the checkpointer process of the cluster - cumulative till present date. Will be resetting before labs for standalone lab comparable results.

```sql
SELECT num_timed, num_requested, write_time, sync_time, buffers_written, stats_reset
FROM pg_stat_checkpointer;
```

---

## References 

[Monitor Database Activity](https://www.postgresql.org/docs/18/monitoring-stats.html)

[System Views](https://www.postgresql.org/docs/18/views.html)
