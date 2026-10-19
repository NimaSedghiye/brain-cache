# Manhattan: Can't Write Data into Database

- **Scenario:** [Manhattan on SadServers](https://sadservers.com/scenario/manhattan)
- **Difficulty:** Medium
- **Type:** Fix
- **Topics:** Disk Volumes, PostgreSQL, systemd

## Problem

The official scenario introduces the objective as follows:

> Your objective is to be able to insert a row in an existing Postgres database.

Source: [SadServers — Manhattan](https://sadservers.com/scenario/manhattan).

The remaining scenario details are summarized below. The failure is not
PostgreSQL-specific, so detailed knowledge of the database is not required to
solve it.

### Environment

- PostgreSQL listens on port `5432` and stores its data on disk.
- Its configuration file is `/etc/postgresql/14/main/postgresql.conf`.
- The `data_directory` setting specifies where PostgreSQL stores its data.
- systemd manages the service under the name `postgresql`.
- The scenario provides `sudo` access.

### Success Criteria

From the default `admin` account, the following command must insert a row into
the `persons` table in the `dt` database:

```bash
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt
```

Expected output:

```text
INSERT 0 1
```

## Solution

### Investigation

#### 1. Check the service, listener, and connection

Start by checking the PostgreSQL service:

```bash
sudo systemctl status postgresql
```

Recorded output (excerpt):

```text
● postgresql.service - PostgreSQL RDBMS
   Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
   Active: active (exited) since Mon 2026-10-19 11:14:12 UTC; 2min 48s ago
  Process: 677 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
 Main PID: 677 (code=exited, status=0/SUCCESS)
```

Here, `postgresql.service` is an aggregate service. Its successful
`ExecStart=/bin/true` and `active (exited)` state do not establish that a database
server is running. Check the listener separately:

```bash
sudo ss -ltnp | grep ':5432'
```

No lines were returned: there was no matching TCP listener on port `5432`.
Next, try querying the data directory through a local connection:

```bash
sudo -u postgres psql -tAc "SHOW data_directory;"
```

Recorded output:

```text
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: No such file or directory
        Is the server running locally and accepting connections on that socket?
```

Unlike the TCP listener check, this command attempts to use a Unix-domain
socket. The missing socket confirms that this local connection is unavailable;
it does not yet explain why. Since SQL queries cannot supply the configuration
while the server is unavailable, inspect the cluster information next.

#### 2. Inspect the actual PostgreSQL cluster

A PostgreSQL cluster is a collection of databases managed by one server
instance. On Debian-based installations, `pg_lsclusters` lists the configured
clusters, including their versions, ports, status, data directories, and logs.

```bash
sudo pg_lsclusters
```

Recorded output:

```text
Ver Cluster Port Status Owner    Data directory   Log file
14  main    5432 down   postgres /opt/pgdata/main /var/log/postgresql/postgresql-14-main.log
```

The `14/main` cluster is **down**, even though the aggregate service appeared
active. Its data directory is `/opt/pgdata/main`, and the output also identifies
the log to inspect.

`pg_lsclusters` reports cluster information; it does not start a server.
For example, `sudo pg_lsclusters 14 main start` is not a startup command.
Use `pg_ctlcluster` to control a specific cluster instead.

#### 3. Attempt startup and inspect the failure

```bash
sudo pg_ctlcluster 14 main start
```

This targets PostgreSQL version `14`, cluster `main`, with the action `start`.
When run as root under systemd, the tool normally delegates to the cluster's
systemd unit.

Recorded output:

```text
Job for postgresql@14-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@14-main.service" and "journalctl -xe" for details.
```

This reports a startup failure without identifying its cause. Read the
PostgreSQL-specific log found in the previous step:

```bash
sudo tail -n 100 /var/log/postgresql/postgresql-14-main.log
```

Recorded output (decisive excerpt):

```text
pg_ctl: could not start server
Examine the log output.
2026-10-19 11:19:23.857 UTC [909] FATAL:  could not create lock file "postmaster.pid": No space left on device
pg_ctl: could not start server
Examine the log output.
```

PostgreSQL cannot create the `postmaster.pid` lock file needed for startup in
its data directory. The next question is whether that filesystem has run out
of storage blocks or inodes.

#### 4. Identify the full filesystem

```bash
df -h
```

Recorded output (relevant rows):

```text
Filesystem       Size  Used Avail Use% Mounted on
/dev/nvme1n1p1   7.7G  1.2G  6.1G  17% /
/dev/nvme0n1     8.0G  8.0G   28K 100% /opt/pgdata
```

The root filesystem has free space, but `/opt/pgdata` is on a separate
filesystem that is **100% full**. Free space on `/` cannot satisfy writes to
this other filesystem.

Check inode usage as well. Inodes hold filesystem metadata for files and
directories; exhausting them can also prevent file creation.

```bash
df -i
```

Recorded output (relevant row):

```text
Filesystem      Inodes IUsed IFree IUse% Mounted on
/dev/nvme0n1      1464  1281   183   88% /opt/pgdata
```

There are still `183` free inodes. Together with the startup error and block
usage, this points to exhausted storage capacity rather than inode exhaustion.

#### 5. Locate the space consumers on the affected filesystem

Compare the cluster's size with its logs, then inspect the entire data mount:

```bash
sudo du -sh /opt/pgdata/main
sudo du -sh /var/log/postgresql
sudo du -ahx /opt/pgdata | sort -h | tail -n 30
```

Recorded output from the first two commands:

```text
50M     /opt/pgdata/main
28K     /var/log/postgresql
```

Recorded output from the final command (excerpt):

```text
16M     /opt/pgdata/main/pg_wal
16M     /opt/pgdata/main/pg_wal/000000010000000000000001
33M     /opt/pgdata/main/base
50M     /opt/pgdata/main
923M    /opt/pgdata/file2.bk
7.0G    /opt/pgdata/file1.bk
8.0G    /opt/pgdata
```

The database cluster occupies only about `50M`. Two files outside `main`
account for almost all the filesystem's usage.

In the final command, `-a` includes individual files, `-h` uses readable sizes,
and `-x` stays on the filesystem where the scan starts. `sort -h` orders those
sizes, and `tail` selects the largest entries.

The starting path matters: `sudo du -xhd1 / 2>/dev/null | sort -h` does not
descend into the separately mounted `/opt/pgdata` filesystem. Starting the
scan at `/opt/pgdata` exposes the files responsible for this failure.

#### 6. Inspect the large files before choosing a recovery action

The `.bk` suffix alone does not establish a file's purpose or whether it is
safe to delete. Inspect the file types, sizes, and timestamps:

```bash
sudo file /opt/pgdata/file1.bk /opt/pgdata/file2.bk /opt/pgdata/file3.bk
sudo ls -lh --time-style=long-iso /opt/pgdata/*.bk
```

Recorded output:

```text
/opt/pgdata/file1.bk: data
/opt/pgdata/file2.bk: ASCII text
/opt/pgdata/file3.bk: ASCII text
```

```text
-rw-r--r-- 1 root root 7.0G 2022-05-21 22:06 /opt/pgdata/file1.bk
-rw-r--r-- 1 root root 923M 2022-05-21 22:17 /opt/pgdata/file2.bk
-rw-r--r-- 1 root root 488K 2022-05-21 22:23 /opt/pgdata/file3.bk
```

Examine a small sample of the text files:

```bash
sudo head -n 20 /opt/pgdata/file2.bk
sudo head -n 20 /opt/pgdata/file3.bk
```

Recorded output (one line from each result, respectively):

```text
abcdefghijklmnopqrstuvwxyz0123456789
```

```text
a
```

The repetitive content suggests generated data, but does not prove that the
files are disposable. Relocating `file2.bk` to another filesystem preserves
its contents while making room for PostgreSQL to start.

### Root Cause

The filesystem mounted at `/opt/pgdata` had exhausted its available storage
capacity. Most of its space was occupied by `file1.bk` and `file2.bk`, while
the live PostgreSQL cluster used about `50M`.

As a result, PostgreSQL could not create `postmaster.pid` and the `14/main`
cluster could not start. With the server down, both the TCP listener and the
local socket were unavailable, preventing clients from reaching the database
to insert a row. The aggregate service's state did not reveal this failure.

### Resolution

#### 1. Relocate file2.bk to a filesystem with sufficient free space

The following recovery procedure uses `/var/tmp/manhattan-recovery` as an
example destination. First check which filesystems contain the source and
destination, and how much space is available:

```bash
df -h /opt/pgdata /var/tmp
```

Proceed only if `/var/tmp` is on a different filesystem with enough space for
`file2.bk` (about `923M`) plus working headroom. In the recorded output, `/`
had `6.1G` free, so it could accommodate `file2.bk` if `/var/tmp` resides there.
It could not accommodate the `7.0G` `file1.bk`. If `/var/tmp` is on the full
filesystem or lacks capacity, choose a different destination and check it
before moving the file.

```bash
sudo mkdir -p /var/tmp/manhattan-recovery
sudo mv -i /opt/pgdata/file2.bk /var/tmp/manhattan-recovery/
```

The `-i` option asks before replacing an existing destination file. A
cross-filesystem `mv` copies the file and removes the source after the copy
succeeds, releasing its space on the original filesystem. Renaming the file
or moving it to another directory inside `/opt/pgdata` would not free that
filesystem's storage.

Confirm that the move completed and space is now available:

```bash
sudo ls -lh /var/tmp/manhattan-recovery/file2.bk
df -h /opt/pgdata
```

The destination should contain the file, and `/opt/pgdata` should show
substantially more available space.

This procedure preserves both the relocated file's contents and the existing
cluster. It does not require deleting `file1.bk` or modifying files inside
`/opt/pgdata/main`.

#### 2. Start the cluster

Once space is available, retry startup and check the cluster:

```bash
sudo pg_ctlcluster 14 main start
sudo pg_lsclusters
```

The expected state for `14/main` is `online` on port `5432`. If startup still
fails, read the latest entries in the same PostgreSQL log to identify the
remaining error before making another change.

### Verification

An `online` cluster is an intermediate check. The scenario's objective is a
successful write to the existing database, so run its acceptance test:

```bash
sudo -u postgres psql -c "insert into persons(name) values ('jane smith');" -d dt
```

Expected output:

```text
INSERT 0 1
```

This indicates that one row was inserted into `dt.persons`. Each successful
execution inserts another row, so this is a write test rather than a read-only
health check.

## Key Takeaways

- Check the actual database cluster, listener, and connection rather than
  relying only on an aggregate service's status.
- Follow the startup error into the application's own log. In this case, the
  log connected a database availability problem to filesystem capacity.
- Check the filesystem containing the data, not just `/`. Separate mounts
  have separate capacity limits.
- Use both `df -h` and `df -i` to distinguish storage capacity from inode
  exhaustion.
- Start a `du -x` scan on the affected filesystem so that the mount boundary
  does not hide the relevant files.
- Preserve uncertain files by relocating them to a filesystem with capacity;
  a filename extension or a short content sample is not proof of disposability.
- Verify the original user-facing operation after restoring service health.
- For sustained operation, address storage growth and capacity monitoring as
  well. Relocating one file creates immediate headroom but does not explain
  or prevent future growth of the remaining large files.

## References

- [SadServers: Manhattan](https://sadservers.com/scenario/manhattan)
- [Debian: pg_lsclusters](https://manpages.debian.org/bookworm/postgresql-common/pg_lsclusters.1.en.html)
- [Debian: pg_ctlcluster](https://manpages.debian.org/bookworm/postgresql-common/pg_ctlcluster.1.en.html)
- [GNU Coreutils: du](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [GNU Coreutils: mv](https://www.gnu.org/software/coreutils/manual/html_node/mv-invocation.html)
