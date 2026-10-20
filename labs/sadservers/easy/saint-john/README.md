# Saint John: What Is Writing to This Log File?

- **Scenario:** [Saint John on SadServers](https://sadservers.com/scenario/saint-john)
- **Difficulty:** Easy
- **Type:** Fix
- **Topics:** Processes, Open Files, Signals

## Problem

A testing program continuously writes to `/var/log/bad.log`, consuming disk
space. The program is no longer needed. Find and terminate it while keeping
the log file.

### Environment

- Debian 11
- SSH access and a user with `sudo` access
- Internet access disabled in the scenario VM

### Success Criteria

The log file's size must remain unchanged over an interval longer than its
write interval. Do not delete the log file.

The scenario's **Check My Solution** button runs
`/home/admin/agent/check.sh`. Although the problem description says the script
can be viewed and executed, its contents were unavailable during this run.
The investigation below uses direct observations of the file and its writer.

## Solution

### Investigation

#### 1. Inspect recent log entries

```bash
sudo tail -n 5 /var/log/bad.log
```

Recorded output:

```text
2026-10-20 13:46:01.843682 token: 318465741
2026-10-20 13:46:02.144164 token: 1364156999
2026-10-20 13:46:02.444712 token: 103160394
2026-10-20 13:46:02.745194 token: 1517042618
2026-10-20 13:46:03.045662 token: 1861253799
```

The timestamps suggest a write interval of approximately `0.3` seconds.
The entries show the symptom but do not identify the process responsible.

#### 2. Find the process with the file open

```bash
sudo lsof /var/log/bad.log
```

Recorded output:

```text
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
badlog.py 590 admin    3w   REG  259,1     5738 265802 /var/log/bad.log
```

`lsof` lists open files and the processes using them. This result identifies
`badlog.py`, running as `admin` with PID `590`, as a candidate writer.
The `FD` value `3w` means file descriptor `3` is open for writing; `REG`
identifies a regular file. Opening a file for writing does not by itself
prove that a process is actively writing, so inspect its command next and
verify the effect of stopping it afterward.

#### 3. Inspect the full command

```bash
ps -p 590 -o pid,ppid,user,args
```

Recorded output:

```text
    PID    PPID USER     COMMAND
    590       1 admin    /usr/bin/python3 /home/admin/badlog.py
```

The process is a Python interpreter running `/home/admin/badlog.py`.
Its parent PID is `1`; that value alone does not establish whether a service
manager will restart it.

### Root Cause

The testing script `/home/admin/badlog.py` was still running and had
`/var/log/bad.log` open for writing. Log entries appeared approximately every
`0.3` seconds. Stopping this process was followed by a stable file size and
no remaining process reported by `lsof` for the log, supporting its
identification as the writer.

### Resolution

Send `SIGTERM` to the identified process:

```bash
sudo kill -TERM 590
```

`SIGTERM` requests termination and gives the program an opportunity to handle
the signal and clean up. The command produced no output in this run; the
checks below establish whether the unwanted writing stopped.

PID `590` belongs to this recorded run. In a new VM session, use the PID from
the current `lsof` and `ps` output before sending a signal.

### Verification

Compare the file's size over five seconds, then check its open-file users:

```bash
before=$(stat -c %s /var/log/bad.log)
sleep 5
after=$(stat -c %s /var/log/bad.log)
printf 'before=%s bytes\nafter=%s bytes\n' "$before" "$after"

sudo lsof /var/log/bad.log
```

Recorded output:

```text
before=12631 bytes
after=12631 bytes
```

`stat -c %s` reports the file size in bytes. Both measurements succeeded and
returned `12631`, so the file remained present and its size was unchanged
across a five-second interval. This interval is substantially longer than
the observed `0.3`-second write interval. The size is larger than in the
earlier `lsof` output because the measurements were taken later.

The final `lsof` command returned no lines, indicating that it found no
process with the file open at that time. Together, these observations
support that the unwanted writing stopped without deleting the log.

Finally, use **Check My Solution** in the scenario UI. Its result was not
provided for this write-up, so the official acceptance check remains
unconfirmed. The direct verification above was observed in the VM.

## Key Takeaways

- Use log timestamps to estimate how frequently a program writes, then
  choose a verification interval longer than that rate.
- Use `lsof` to connect a file to the processes that have it open, and
  inspect the access mode to identify potential writers.
- Use `ps` to inspect a candidate process's full command before stopping it.
- Start with `SIGTERM` and verify the result; a successful signal command
  alone does not establish that the process stopped or stayed stopped.
- Stop the source of unwanted growth while preserving the existing log.
- Distinguish observed verification results from an unreported acceptance
  test result.
