docker-compose up -d
docker exec -it -u barman backup /bin/bash

# Running Backups With Barman

With [the Barman server configured in our last step](step02-backup-setup), we can run a full backup (or "base backup").

To run a full backup, we use [Barman's `backup` command](http://docs.pgbarman.org/release/2.12/#backup):

```shell
barman backup pg --wait
__OUTPUT__
Starting backup using postgres method for server pg in /var/lib/barman/pg/base/20231018T051643
Backup start at LSN: 0/3000000 (000000010000000000000002, 00000000)
Starting backup copy via pg_basebackup for 20231018T051643
Copy done (time: 2 seconds)
Finalising the backup.
This is the first backup for server pg
WAL segments preceding the current backup have been found:
        000000010000000000000002 from server pg has been removed
Backup size: 37.5 MiB
Backup end at LSN: 0/4000060 (000000010000000000000004, 00000060)
Backup completed (start time: 2023-10-18 05:16:43.435290, elapsed time: 3 seconds)
Waiting for the WAL file 000000010000000000000004 from server 'pg'
Processing xlog segments from streaming for pg
        000000010000000000000003
        000000010000000000000004
```

The `--wait` option causes Barman to wait for arrival and processing of the WAL file corresponding to the end of the backup, to ensure the backup is complete before the command returns. 

!!! Note: 
    Since the cron job is doing this periodically in the background, the WAL segments you see listed may be different than the output listed above!

Verify that it completed by listing backups for the server:

```shell
barman list-backup pg
__OUTPUT__
pg 20231018T051643 - Wed Oct 18 05:16:46 2023 - Size: 69.5 MiB - WAL Size: 0 B
```
(the timestamps will be different for you)

Of course, we've configured streaming backups - so we shouldn't need to depend on having a base backup for every change made to the database. Let's make a small modification to the data and verify that it arrives. 

First, log into the database (we'll use our barman user for convenience in this demonstration):

```shell
psql -h pg -d pagila -U barman
__OUTPUT__
psql (16.0 (Ubuntu 16.0-1.pgdg20.04+1), server 15.4 (Debian 15.4-2.pgdg120+1))
Type "help" for help.
```

Then make a modifications, and view the results:

```sql
Update actor Set first_name='ALOYSIUS' where actor_id=23;
Select * From actor Where last_name='KILMER';
\q
__OUTPUT__
UPDATE 1
 actor_id | first_name | last_name |          last_update          
----------+------------+-----------+-------------------------------
       45 | REESE      | KILMER    | 2020-02-15 09:34:33+00
       55 | FAY        | KILMER    | 2020-02-15 09:34:33+00
      153 | MINNIE     | KILMER    | 2020-02-15 09:34:33+00
      162 | OPRAH      | KILMER    | 2020-02-15 09:34:33+00
       23 | ALOYSIUS   | KILMER    | 2021-03-29 23:53:22.302271+00
(5 rows)
```

Ok, let's see if it showed up:

```shell
grep 'ALOYSIUS' ~/pg/streaming/*
__OUTPUT__
Binary file /var/lib/barman/pg/streaming/000000010000000000000005.partial matches
```

There it is! The current WAL segment hasn't been rotated yet, but we have the most recent data in the *partial* WAL streamed to Barman. So in theory, nothing would be lost of something *terrible* happened to the database right now... 

Now, the crucial question with backups is always the same: "can you get the data *back?*"

We'll answer this in [Step #4: Restoring a Backup](step04-restore).
