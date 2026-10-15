# Checkpoint Pacing
1. [Setup](#1-setup)
2. [Immediate Checkpoint](#2-immediate-checkpoint)

    a. [Inference](#inference-)
3. [Pace Checkpoint](#3-paced-checkpoint)

    a. [Inference](#inference--1)

## 1. Setup

Get the posgres data directory - <u>***$PGDATA***</u>
```sql
SHOW data_directory
```

> What we know from the previous lab: 
- Table size:
```sql
pg_relation_size('lab_dirty') -- 26 MB
pg_relation_size('lab_dirty') / 8192 -- 3334 pages
```  
```sql
SELECT count(*) FROM lab_dirty;
-- 400000
```
```sql
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'lab_dirty';
-- 400000    0
```
```sql
SELECT count(*) FILTER (WHERE isdirty) AS dirty_total FROM pg_buffercache;
-- 0
```
```sql
SELECT pg_stat_reset_shared('checkpointer');
CHECKPOINT;
```
```console
sudo tail -n 0 -f <$PGDATA>/log/postgresql-$(date +%a).log
```
```console
LOG:  checkpoint starting: immediate force wait

LOG:  checkpoint complete: wrote 0 buffers (0.0%), wrote 0 SLRU buffers; 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.005 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=12450 kB; lsn=0/5663E70, redo lsn=0/5663E18
```

=> idle server

## 2. Immediate Checkpoint 

Update all 400_000 rows
```sql
UPDATE lab_dirty SET pad = md5(pad);
```
Table size & # of pages
```sql
SELECT pg_size_pretty(pg_relation_size('lab_dirty')) AS size,
       pg_relation_size('lab_dirty') / 8192 AS pages;
-- "52 MB"    6667
```
Dirty pages count
```sql
SELECT count(*) FILTER (WHERE isdirty) AS dirty_total FROM pg_buffercache;
-- 6677
``` 

```sql
CHECKPOINT;
```
Timed, requested checkpoints; # of dirty buffers written; write and sync time
```sql
SELECT num_timed, num_requested, buffers_written, write_time, sync_time FROM pg_stat_checkpointer;
-- 0    2    6677    347    10
```
```console
sudo tail -n 0 -f <$PGDATA>/log/postgresql-$(date +%a).log
```
```console
LOG:  checkpoint starting: immediate force wait

LOG:  checkpoint complete: wrote 6677 buffers (40.8%), wrote 1 SLRU buffers; 0 WAL file(s) added, 0 removed, 6 recycled; write=0.346 s, sync=0.009 s, total=0.366 s; sync files=8, longest=0.004 s, average=0.001 s; distance=94032 kB, estimate=94032 kB; lsn=0/B237F20, redo lsn=0/B237EC8
``` 
```
5663E18 (first redo)
   |	88 bytes
5663E70 (first checkpoint record)
   |	96288856 bytes ~ 96 MB
B237EC8 (second redo)
   |	88 bytes
B237F20 (second checkpoint record)
  |	    176 bytes
B237FD0 (current WAL LSN) 
```
---

### Inference :

- Originally the table was `26 MB` of size and contained `3334 data pages`. Post `UPDATE`, the table size doubled to `52 MB` of size and `6667 data pages`. 

- ***Around 40.8% of the buffer pool were dirty : 6677 dirty pages out of 16,384 buffers.***

- The number of checkpoints requested were 2 and no timed checkpoints were carried out in between these manual immediate checkpoints requested. 

- The time taken to write the dirty buffers to OS page cache is `347 ms` and time taken to fsync them to the disk is `10 ms`. Total time taken is `366 ms` to complete the checkpointing operation.

- `8 fsync calls` were made to complete flushing all the `6677 dirty buffers` written to OS page cache onto stable storage. Each fsync took `at most 4 ms` (longest) and `an average of 1ms` to complete.

- After the first checkpoint, when the update is performed - nearly `96 MB of WAL` is generated  -> the distance WAL moved from the previous checkpoint's REDO to the current one. 

- The estimate is a moving average the server maintains to decide how many WAL segments to keep (recycle) for the next cycle -> the server's current guess of how much WAL it should be ready to keep. The previous estimate was 0 since the checkpoint was idle and the distance was 0. When distance > estimate, estimate = distance -> it jumps up to accomodate the peak - cover up the bursts. When distance < estimate then it decays gradually ->  estimate = (distance + estimate)/2. Hence, distance = 94032 KB = estimate.

- This WAL retention estimate controls the recycle pool size: the server keeps `estimate / segment_size` segments pre-allocated. So in this case : `94032 KB / 16 MB = 5.74 ~ 6`. That's why `6 segments` were `recycled`. This avoids the cost of creating and zero-filling new 16 MB files when the WAL reaches them. The remaining old segments (before the new REDO point) are deleted. In this case, no new files were created hence deleted so `0 WAL file(s) added, 0 removed, 6 recycled`.

- After the first REDO point, 88 bytes generated are the `XLOG_CHECKPOINT_REDO` record (32 bytes) and 1 `XLOG_RUNNING_XACTS` record (56 bytes). Same with the 88 bytes generated after the second REDO point. 

- The 176 bytes between the second checkpoint record and the current WAL LSN are the `XLOG_CHECKPOINT_ONLINE` record (120 bytes) and 1 `XLOG_RUNNING_XACTS` record (56 bytes).

---

> 1 - Initially the table was 26 MB and contained 3334 pages. Then it nearly doubled to 52 MB and 6667 pages. Why ?

PostgreSQL uses MVCC - an `UPDATE` does not overwrite in place - it writes a new tuple version and marks the old one dead (sets `xmax` to the updating xact ID). Since md5(pad) produces the same length output - each new tuple is the same size as the old.

The original pages were already full (no free space to fit new tuples alongside dead ones), so all 400_000 rows land on new pages - nearly 2x the page count. Now the old 3334 pages hold only dead tuples but still occupy disk space until VACUUM reclaims them.

> How many rows per page ?

Total size of page = 8 KB = 8192 bytes

PageHeaderData (fixed) = 24 bytes 

ItemData line pointer array = 4 x N rows

<u>Tuple data section :</u>

HeapTupleHeaderData = 24 bytes (23 max aligned)

varlena header = 1 byte (indicates length)

=> since the tuple data is text which is of variable length type (md5(g::text))

md5 hex data = 32 bytes

Total = 24 + 1 + 32 = 57 = max aligned = 64 bytes

So each tuple is of same size due to same data (md5()) and each are of size 64 bytes.

Total available space = Total size - Page header = 8192 - 24 = 8168

Number of rows (N) => total available space = 64N (tuple data section) + 4N (line pointer array)

N = 8168 / 68 `~ 120 rows per page`

68 x 120 = 8160

Remaining space = available space - 68N = 8168 - 8160 = 8 bytes

=> There is free space of 8 bytes are wasted as another row cannot be added here.


> 2 - Why 6677 dirty buffers written instead 6667 pages (table size) ?

The extra 10 pages `might` be the auxiliary structures that got dirtied during the `UPDATE`.
They `might` be -> FSM (free space map - tracks free space in each page), VM (visibility map - tracks all-visible pages) and catalog pages.
These are all 8 KB pages that live in shared_buffers just like table pages, so they're counted in the checkpoint's buffers_written stat.

> 3 - When the same update step was rerun, the expected result was triple the original size of the table (26 MB and 3334 pages) but it remained the same instead (53 MB and 6677 pages). Why ?                   

Might either be because of `autovacuum` or `page pruning`.

> 4 - When the table size is 52 MB what contributes to 96 MB of WAL generated during `UPDATE` operation ?

After a checkpoint, the first modification to each page logs the entire 8 KB page image (FPI) into WAL, not just the row-level change. Since the UPDATE ran immediately after the first CHECKPOINT, every page touched gets an FPI. An FPI is logged for every original page (modification - rows marked dead) out 3334 pages and also for every new page (modification - new tuple version is added) 3333 pages. 
 
> 5 - What is SLRU buffer - how is it different from shared buffers to which pages are written to and read from ? WHat does it mean the logs provided ?

SLRU - Simple Least Recently Used. It's a lightweight buffer cache for PostgreSQL's internal metadata structures that are too small or specialized to use the shared_buffers. Caches CLOG, FSM, VM, etc. 
During the checkpoint, the checkpointer flushed one dirty SLRU page to disk. This `might` be a CLOG page - the commit status for The transaction's XID was written to CLOG, dirtying one of its cached pages. The checkpoint ensures all dirty metadata (both shared_buffers pages and SLRU pages) reaches stable storage, so that after a crash, the server knows which transactions committed.

---

## 3. Paced Checkpoint

1. Terminal 
```console
sudo tail -n 0 -f /var/lib/pgsql/data/log/postgresql-$(date +%a).log
```

2. PSQL Tool
```sql
SELECT now()::time(0) AS t,
       count(*) FILTER (WHERE isdirty) AS dirty,
       (SELECT wait_event FROM pg_stat_activity WHERE backend_type = 'checkpointer') AS ckpt_wait
FROM pg_buffercache \watch 5
```

3. Lower `checkpoint_timeout` (default 5 mins) to 60s.
A manual CHECKPOINT wrote 6,677 buffers in 0.35 s. A timed checkpoint is throttled to finish its write phase at `checkpoint_completion_target × checkpoint_timeout`.

Disable automvacuum - since it would dirty and write pages on it's own.

Run the SQL commands as a script so that there is no time gaps between statements.
```sql
ALTER TABLE lab_dirty SET (autovacuum_enabled = false);
SELECT pg_stat_reset_shared('checkpointer');
CHECKPOINT;
ALTER SYSTEM SET checkpoint_timeout = '60s';
SELECT pg_reload_conf();
UPDATE lab_dirty SET pad = md5(pad);
```

4. Table size + pages
```sql
SELECT pg_size_pretty(pg_relation_size('lab_dirty')) AS size,
       pg_relation_size('lab_dirty') / 8192 AS pages;
--        "78 MB"    10000
```

5. Checkpointer stats
```sql
SELECT num_timed, num_requested, buffers_written, write_time, sync_time FROM pg_stat_checkpointer;
-- 1    1    10005    53946    7
```

6. 
Output of log: 
```console
LOG:  checkpoint starting: time
LOG:  checkpoint complete: wrote 10004 buffers (61.1%), wrote 1 SLRU buffers; 0 WAL file(s) added, 0 removed, 8 recycled; write=53.945 s, sync=0.005 s, total=53.979 s; sync files=4, longest=0.003 s, average=0.002 s; distance=120610 kB, estimate=120610 kB; lsn=0/29593B10, redo lsn=0/29593A80
```

Output of pSQL: 
```
Saturday 10 October 2026 08:38:39 PM (every 5s)

    t     | dirty |    ckpt_wait     
----------+-------+------------------
 20:38:39 | 10004 | CheckpointerMain
(1 row)

Saturday 10 October 2026 08:38:44 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:38:44 |  9222 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:38:49 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:38:49 |  8303 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:38:54 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:38:54 |  7366 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:38:59 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:38:59 |  6448 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:04 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:39:04 |  5510 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:09 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:39:09 |  4589 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:14 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:39:14 |  3667 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:19 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:39:19 |  2746 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:24 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:39:24 |  1805 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:29 PM (every 5s)

    t     | dirty |      ckpt_wait       
----------+-------+----------------------
 20:39:29 |   883 | CheckpointWriteDelay
(1 row)

Saturday 10 October 2026 08:39:34 PM (every 5s)

    t     | dirty |    ckpt_wait     
----------+-------+------------------
 20:39:34 |     0 | CheckpointerMain

```


7. Reset the settings

```sql
ALTER SYSTEM RESET checkpoint_timeout;
SELECT pg_reload_conf();
ALTER TABLE lab_dirty RESET (autovacuum_enabled);
```

### Inference : 
- In the previous lab, the table was `52 MB` of size and contained `6667 data pages`. Post `UPDATE`, the table size has `1.5x` to `78 MB` of size and `10000 data pages`. 

- ***Around 61.1% of the buffer pool were dirty : 10004 dirty pages out of 16,384 buffers.***

- 1 timed checkpoint was carried out with a `checkpoint_timeout` of 60s.

- The time taken to write the dirty buffers to OS page cache is `53.945 s` and time taken to fsync them to the disk is `5 ms`. Total time taken is `53.979 s` to complete the checkpointing operation. This matches the math behind spreading the checkpointing over `checkpoint_completion_target`, that is, `total time = checkpoint_timeout x checkpoint_completion_target` => `60 x 0.9 = 54s`. The checkpointer slept between batches to finish at the target (`CheckpointWriteDelay`).

- `4 fsync calls` were made to complete flushing all the `10004 dirty buffers` written to OS page cache onto stable storage. Each fsync took `at most 3 ms` (longest) and `an average of 2 ms` to complete.

-  Nearly `118 MB of WAL` is generated.  

- This WAL retention estimate controls the recycle pool size: the server keeps `estimate / segment_size` segments pre-allocated. So in this case : `120610 KB / 16 MB = 7.36 ~ 8`. That's why `8 segments` were `recycled`.

> Why did the buffers written increase by 1.5x times ?

Going back to earlier runs including previouse labs: 

- The first insertion resulted in 400_000 rows and 3334 pages (live tuples)
- The second run was updating the all the existing rows which resulted in 6667 pages where 3333 pages contained live tuples and the tuples in 3334 old pages were marked dead. 
- When the same update was run again, the result remained the same which `might` be because of autovacuum or page pruning cleaning up the pages with dead tuples and reclaiming the space of old 3334 pages and updating the new tuple version there instead.
- The third run was updating all the existing rows again but this time with autovacuum disabled. Another 3333 pages were added resulting in 400_000 rows but 10_000 pages. 
- This is because the autovacuum couldn't cleanup the 3334 dead pages from second run and reclaim it's space, the 3333 live pages now marked dead from the rerun and 3333 new pages from the current update run. 
- So there are 2 tuple versions that are dead and 1 live tuple version spread across ~3334 + ~3333 + ~3333 pages.

The checkpointer flushes every dirty buffer, and the `UPDATE` dirtied every page it touched (old versions getting xmax set, new versions being written). Since the table grew from 6,667 to 10,000 pages (1.5x), buffers_written grew from 6,677 to 10,005 (also 1.5x).

> Why fsync wasn’t slower with more data ?

In the immediate checkpoint run, 52 MB of data was handed to the OS cache in 0.35 s, so the kernel had almost no time to write it out before the fsyncs began.
But in the paced checkpoint run, 78 MB was spread over 54 s. The kernel’s background flushing (it writes out old dirty pages every few seconds) and the `checkpoint_flush_after` (a sync_file_range request after every 32 pages) had time to push nearly everything to the device. When the fsyncs arrived, there was almost nothing left to flush.