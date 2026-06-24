# KijaniKiosk API Server — Triage Report

**Date:** 2024-01-15  
**Investigated by:** Hussein Abdallah  
**Server:** kijanikiosk-prod-01 (Ubuntu 22.04)  
**Incident start (approximate):** 04:07 on 2024-01-15 (first `ERROR` in application log)

---

## Summary

A Python process spawned on the server is consuming approximately 500 MB of RAM by allocating memory in a tight loop, placing the system under sustained memory pressure. This coincides with a progressive database connection pool exhaustion that began at 03:45 and reached total failure at 04:07, after which the application began timing out on every query. A separate, orphaned log file (`access.log.1`, ~267 MB) is consuming significant disk space and suggests log rotation is not functioning correctly. The combination of memory pressure and database unavailability is the most defensible explanation for the observed P95 latency spike from ~120 ms to ~480 ms.

---

## Process and Resource State

### Investigation commands run

```bash
ps aux --sort=-%mem | head -15
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|Cached|SwapUsed"
ps aux | awk '$8=="Z" {print "ZOMBIE:", $0}'
```

### Findings

**Top processes by memory:**

| Rank | PID   | User | %MEM  | %CPU | Command                          |
|------|-------|------|-------|------|----------------------------------|
| 1    | 2341  | root | 24.7% | 98.2 | python3 -c (memory loop)         |
| 2    | 1201  | www  | 4.2%  | 1.1  | node /opt/kijanikiosk/app/index  |
| 3    | 987   | root | 0.8%  | 0.1  | nginx: worker process            |

**Memory summary (`free -h`):**

```
              total        used        free      shared  buff/cache   available
Mem:           2.0G        1.7G        112M        24M       230M        180M
Swap:          1.0G        824M        200M
```

**Key observations:**
- The Python process at PID 2341 is consuming ~500 MB of heap by looping 500 iterations of `x.append(' ' * 1024 * 1024)`. It has no legitimate application role — it is an orphaned memory consumer.
- Available memory has dropped to ~180 MB. The kernel has begun using swap (~824 MB used), which causes latency in any process that must page memory in/out — this directly explains slower request handling.
- No zombie (`Z`) or uninterruptible-sleep (`D`) processes found at investigation time, though the connection pool exhaustion event at 04:07 would have left worker threads in blocked I/O states temporarily.

---

## Filesystem and Disk

### Investigation commands run

```bash
df -h
du -sh /var/log/* 2>/dev/null | sort -rh | head -10
find /var/log -name "*.log" -size +50M -ls 2>/dev/null
find /tmp -mmin -60 -type f 2>/dev/null
ls -lhtr /var/log/ | tail -10
```

### Findings

**Partition usage (`df -h`):**

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G   17G  1.9G  90% /
tmpfs           1.0G  1.2M  999M   1% /run
```

The root partition is at **90% capacity**. This is above the 80% threshold that warrants immediate attention. At this level, log writes can begin failing silently, and some applications will refuse to start or accept new connections.

**Largest log files (`du -sh /var/log/*`):**

```
267M  /var/log/kijanikiosk/access.log.1
4.2M  /var/log/nginx/access.log
1.1M  /var/log/syslog
12K   /var/log/kijanikiosk/app.log
```

**Critical finding:** `/var/log/kijanikiosk/access.log.1` is 267 MB. This is a rotated log file that was never compressed or deleted. Standard logrotate configuration compresses rotated files (producing `.gz` files of roughly 1–5% of the original size) and deletes files older than a configured retention period. The presence of an uncompressed 267 MB rotated file indicates that either:
- `logrotate` is not installed or not scheduled for this path, or
- The `postrotate` compress step failed silently.

This file alone accounts for roughly 1.3% of total disk capacity on a 20 GB volume. If the application is generating access logs at a similar rate, disk exhaustion is a near-term risk.

---

## Log Analysis

### Investigation commands run

```bash
grep -E "ERROR|WARN|CRITICAL" /var/log/kijanikiosk/app.log \
  | awk '{print $4}' | sort | uniq -c | sort -rn

grep "ERROR" /var/log/kijanikiosk/app.log \
  | awk '{print $1, $2}' | head -20

grep -E "oom|OOM|killed process|I/O error" /var/log/syslog | tail -20

grep -E "Accepted|Failed|Invalid" /var/log/auth.log \
  | awk '{print $1,$2,$3,$9,$11}' | sort | uniq -c | sort -rn | head -20
```

### Findings

**Error and warning frequency:**

```
6  ERROR
3  WARN
```

**Full annotated timeline from `/var/log/kijanikiosk/app.log`:**

```
2024-01-15 03:12:04  INFO   Worker process started — system healthy at this point
2024-01-15 03:14:22  INFO   Request processed /api/products 200 112ms — baseline latency normal
2024-01-15 03:45:10  WARN   Database connection pool at 85% capacity
2024-01-15 04:01:33  WARN   Database connection pool at 94% capacity
2024-01-15 04:07:55  ERROR  Connection pool exhausted — queuing requests
2024-01-15 04:08:01  ERROR  Query timeout after 30000ms: SELECT * FROM orders
2024-01-15 04:08:01  ERROR  Query timeout after 30000ms: SELECT * FROM products
2024-01-15 04:09:12  WARN   Memory usage at 87% — consider restarting workers
2024-01-15 06:22:18  ERROR  ECONNREFUSED database:5432 — retrying in 5s
2024-01-15 06:22:23  ERROR  ECONNREFUSED database:5432 — retrying in 5s
2024-01-15 06:22:28  ERROR  Retry limit reached — database connection failed
```

**Pattern analysis:**

The log tells a clear escalation story across three phases:

1. **03:45–04:01 (Warning phase):** Connection pool climbed from 85% to 94% over ~16 minutes. Either a spike in traffic or slow/stuck queries were holding connections open without releasing them.

2. **04:07–04:09 (Failure phase):** Pool exhausted. All new queries queue. Two different tables (`orders`, `products`) begin timing out at 30 s, suggesting this is a pool-wide failure, not a single slow query. Memory warning at 04:09 correlates with the Python memory-consumer process being active at the same time.

3. **06:22 (Hard failure):** `ECONNREFUSED` on `database:5432`. This means the PostgreSQL process either crashed or was killed (possibly by the OOM killer as memory pressure intensified). The application exhausted its retry budget and abandoned the database connection entirely. Any request arriving after 06:22 would fail at the database layer immediately.

**`/var/log/syslog` OOM check:** No OOM kill events found in the current syslog window, but given swap usage at 824 MB and available memory at ~180 MB, an OOM event is likely pending if the Python process is not terminated.

---

## Network and Service State

### Investigation commands run

```bash
ss -tlnp
ss -s
curl -o /dev/null -s -w "HTTP %{http_code} - %{time_total}s\n" http://localhost/
curl -o /dev/null -s -w "HTTP %{time_total}s\n" http://localhost/api/health
ss -tlnp | grep -E "80|443|3000|8080|5432"
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
```

### Findings

**Listening ports (`ss -tlnp`):**

```
State   Recv-Q  Send-Q  Local Address:Port  Process
LISTEN  0       511     0.0.0.0:80          nginx
LISTEN  0       511     0.0.0.0:443         nginx
LISTEN  0       128     0.0.0.0:3000        node
```

**PostgreSQL (port 5432) is not present in the listen table.** This confirms what the 06:22 log entries implied — the database process is not running at investigation time.

**HTTP response (curl to localhost):**

```
HTTP 502 - 0.023s
```

NGINX is responding immediately (0.023 s) but returning `502 Bad Gateway`. This means NGINX is healthy and accepting connections, but the upstream Node.js application is either not responding or returning errors — consistent with an application that cannot reach its database.

**TCP connection state distribution (`ss -tan`):**

```
  348  ESTABLISHED
  112  TIME-WAIT
   24  CLOSE-WAIT
    6  SYN-SENT
```

The 24 `CLOSE-WAIT` connections are a secondary concern: these are connections the remote side closed but the application has not yet acknowledged. This can indicate the Node.js process is hanging on operations (database calls) instead of completing connection teardown promptly.

---

## Assessment

The most defensible root cause hypothesis is a **database connection pool exhaustion followed by database process failure**, compounded by memory pressure from an orphaned Python process.

The timeline:
- Between 03:45 and 04:07, connection pool saturation indicates either a traffic surge or a slow-query leak. Once the pool exhausted at 04:07, all requests began queueing, directly causing the latency spike from ~120 ms to ~480 ms and beyond.
- An unrelated Python process consuming ~500 MB of memory forced the system into heavy swap usage. This degraded I/O performance across all processes, making the already-degraded request handling even slower.
- By 06:22, PostgreSQL was no longer accepting connections on port 5432, either due to a crash or an OOM kill. The application is now fully failing to serve any data-dependent request.
- The disk partition at 90% usage is a latent risk: if log rotation is not fixed and disk reaches 100%, NGINX and Node.js will stop writing access logs and may begin rejecting requests.

---

## Recommended Next Steps

**1. Kill the orphaned Python process and verify memory recovers (immediate, ~2 minutes)**

```bash
kill $(pgrep -f "python3 -c")
free -h   # verify available memory rises and swap usage drops
```

This removes the memory pressure immediately and prevents a likely imminent OOM kill of a legitimate process (Node.js or NGINX).

**2. Restart PostgreSQL and verify connectivity before restarting the application (immediate, ~5 minutes)**

```bash
systemctl status postgresql
systemctl start postgresql
# Wait for postgres to be ready, then verify:
psql -U kijanikiosk -h localhost -c "SELECT 1;"
# Then restart the Node.js app to re-establish connection pool:
systemctl restart kijanikiosk-app
curl -o /dev/null -s -w "HTTP %{http_code}\n" http://localhost/api/health
```

Investigate why PostgreSQL went down before declaring it resolved — check `/var/log/postgresql/postgresql-*.log` for the last entries before the crash.

**3. Fix log rotation and reclaim disk space (within the hour)**

```bash
# Compress or delete the orphaned rotated log:
gzip /var/log/kijanikiosk/access.log.1
# Then verify logrotate is configured:
cat /etc/logrotate.d/kijanikiosk 2>/dev/null || echo "No logrotate config found for kijanikiosk"
# If missing, create one targeting /var/log/kijanikiosk/*.log
```

Without a working logrotate config, this disk exhaustion will recur. At 90% usage, any further log growth risks hitting 100% and causing silent write failures.
