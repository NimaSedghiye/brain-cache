# Tokyo: Can't Serve Web File

- **Scenario:** [Tokyo on SadServers](https://sadservers.com/scenario/tokyo)
- **Difficulty:** Medium
- **Type:** Fix
- **Topics:** HTTP, Apache, iptables, File Permissions
- **Points:** 5
- **Time Limit:** 15 minutes

## Problem

A web server should serve `/var/www/html/index.html`, containing
`hello sadserver`, but a local HTTP request to port `80` returns nothing.
The scenario requires general knowledge of web servers rather than detailed
knowledge of a particular server's configuration.

### Environment

- Debian 13 with Apache
- SSH access and a user with `sudo` access
- Internet access disabled in the scenario VM

### Success Criteria

```bash
curl 127.0.0.1:80
```

Expected response body:

```text
hello sadserver
```

## Solution

### Investigation

#### 1. Distinguish a connection failure from an HTTP error

Use verbose output and a timeout to see where the request stops:

```bash
curl --noproxy '*' -v --max-time 5 http://127.0.0.1:80/
sudo ss -ltnp 'sport = :80'
```

Recorded `curl` output:

```text
*   Trying 127.0.0.1:80...
* Connection timed out after 5000 milliseconds
* closing connection #0
curl: (28) Connection timed out after 5000 milliseconds
```

The listener check showed Apache listening on port `80`:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process
LISTEN 0      511    *:80               *:*               users:(("apache2",pid=860,fd=4),("apache2",pid=859,fd=4),("apache2",pid=855,fd=4))
```

`--noproxy '*'` bypasses proxy settings, `-v` shows connection details, and
`--max-time 5` bounds the request to five seconds. In `ss`, `-l` selects
listeners, `-t` selects TCP, `-n` uses numeric addresses and ports, and `-p`
shows the owning processes.

Apache has a listening socket, but this request never establishes a TCP
connection. Investigate the network path before checking file contents or
HTTP configuration.

#### 2. Inspect firewall rules and loopback addressing

```bash
sudo iptables -L -n -v --line-numbers
ip -br addr show lo
```

Recorded firewall output:

```text
Chain INPUT (policy ACCEPT 160 packets, 18081 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        5   300 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 164 packets, 193K bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

Recorded loopback output:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
```

The loopback address is present. The explicit `INPUT` rule drops all incoming
TCP packets whose destination port is `80`, including traffic arriving through
loopback. The chain's default `ACCEPT` policy applies only when no terminating
rule has already matched. The rule's nonzero packet counter shows that it has
matched traffic.

`DROP` silently discards matching packets, which explains why the connection
times out while Apache is listening.

#### 3. Remove the blocking rule and retry

Delete the specific rule found above:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
curl --noproxy '*' http://127.0.0.1:80/
```

The deletion completed without output. The next request returned an HTML
error page containing:

```html
<title>403 Forbidden</title>
```

The response also included:

```text
You don't have permission to access this resource.
```

Receiving an HTTP response demonstrates that the connection now reaches
Apache. The `403` exposes a second problem that was hidden by the firewall.

#### 4. Inspect file permissions and the Apache error log

```bash
sudo namei -l /var/www/html/index.html
sudo tail -n 15 /var/log/apache2/error.log
```

Recorded path permissions:

```text
f: /var/www/html/index.html
drwxr-xr-x root root /
drwxr-xr-x root root var
drwxr-xr-x root root www
drwxr-xr-x root root html
-rw------- root root index.html
```

Recorded error log (decisive excerpt):

```text
[Mon Oct 26 12:35:02.957781 2026] [core:error] [pid 859:tid 862] (13)Permission denied: [client 127.0.0.1:42146] AH00132: file permissions deny server access: /var/www/html/index.html
```

`namei -l` shows each component's permissions along the path. The directories
allow traversal, but `index.html` is owned by `root:root` with mode `600`
(`-rw-------`). Only its owner can read or write it. Apache's worker processes
serve requests without root privileges, so they cannot read this file.
The error log confirms that file permissions caused the rejection.

### Root Cause

Two independent access restrictions prevented the expected response:

1. An `iptables` rule in `INPUT` silently dropped TCP traffic to port `80`,
   preventing the local client from connecting to Apache.
2. The root-owned `index.html` had mode `600`, preventing Apache's workers
   from reading it after the network restriction was removed.

The symptoms changed from a connection timeout to `403 Forbidden` as each
layer became reachable.

### Resolution

The complete fix used in this run was:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
sudo chmod 644 /var/www/html/index.html
```

The firewall command was already executed during the investigation; do not
repeat it if the matching rule has been removed. It targets the observed
blocking rule while preserving unrelated firewall rules.

Mode `644` gives the owner read/write access and the group and others read
access. This allows Apache to read the public web page while retaining
`root:root` ownership. The directory permissions already permit traversal,
so only the file's mode needs changing. Apache does not need a restart for
either change.

The `iptables` command changes the running ruleset. Persistence across a
reboot was not checked in this run.

### Verification

Repeat the original local request, explicitly bypassing any proxy settings:

```bash
curl --noproxy '*' http://127.0.0.1:80/
```

Expected response body:

```text
hello sadserver
```

The user confirmed that the issue was resolved after applying `chmod 644`.
The final command output and the scenario's **Check My Solution** result
were not provided in the session.

## Key Takeaways

- A listening socket does not guarantee that clients can reach it; firewall
  rules also apply to local traffic.
- Distinguish a TCP connection timeout from an HTTP error response. They
  identify different stages of a request.
- Inspect explicit firewall rules as well as default chain policies, and
  remove the specific rule responsible for the observed failure.
- Retest after each change. One failure can hide another independent issue.
- Use `namei -l` to inspect both file permissions and directory traversal
  permissions, then connect those findings to the server's error log.
- A public static file needs to be readable by the serving process; it does
  not need executable permissions or a change of ownership.
- Verify the requested response body, rather than stopping when Apache
  becomes reachable or returns any HTTP response.
