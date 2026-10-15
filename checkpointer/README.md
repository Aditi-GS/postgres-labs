| # | LAB | OBJECTIVE | LINK |
|---|---|---|---|
| 1 | Idle Server | Check the settings, log location, log format of an idle checkpoint | [Lab 1](./idle-server.md) |
| 2 | Manual Checkpoint with Dirty Pages | Observe :<br>- dirty buffers accumulate in shmem<br>- checkpoint flush them to disk<br>- REDO point advance in control file | [Lab 2](./manual-checkpoint-with-dirty-pages.md) |
| 3 | Timed Checkpoint | Compare an immediate checkpoint with a paced (timed) one. Read write, sync and total in the log | [Lab 3](./checkpoint_pacing.md)