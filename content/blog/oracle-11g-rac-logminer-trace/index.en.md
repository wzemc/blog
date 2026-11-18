---
title: "Tracing a LogMiner Round Every 9 Seconds in an Oracle 11g RAC Alert Log"
slug: oracle-11g-rac-logminer-trace
date: 2026-11-18
description: "After node 1 of an 11g RAC was decommissioned, the alert log on the remaining node kept showing LogMiner messages. This post traces the source of the activity and cleans up the retired node."
categories: [database]
tags: [oracle, rac, logminer, troubleshooting, cdc]
---
## Environment

| Item | Detail |
|---|---|
| Version | Oracle 11g RAC |
| Storage | ASM, redo on a separate disk group |
| State | Node 1 decommissioned, only orcl3 running |
| Role | PRIMARY, READ WRITE |

## 1. Symptoms

Below are four consecutive rounds from the alert log, wrapped by hand, with paths sanitized. Round 1 keeps every line; rounds 2 to 4 omit the fixed lines:

```
LOGMINER: Begin mining logfile
 for session -2147472893
 thread 3 sequence 651065
 +REDODG/db/onlinelog/group_31...
Mon Nov 16 20:41:23 2026
LOGMINER: summary for
 session# = 2147494659
LOGMINER: StartScn: 15628771104082
LOGMINER: EndScn:   15628771104143
LOGMINER: HighConsumedScn: 0
LOGMINER: session_flag: 0x0
LOGMINER: Read buffers: 8
LOGMINER: Memory LWM: limit 10M,
 LWM 8M, 80%
LOGMINER: Memory Release Limit: 0M
```

```
LOGMINER: Begin mining logfile
 for session -2147472637
 thread 3 sequence 651065 ...
LOGMINER: StartScn: 15628771104143
LOGMINER: EndScn:   15628771104448
```

```
LOGMINER: Begin mining logfile
 for session -2147472637
 thread 3 sequence 651065 ...
Mon Nov 16 20:41:42 2026
LOGMINER: StartScn: 15628771104448
LOGMINER: EndScn:   15628771104699
```

```
LOGMINER: Begin mining logfile
 for session -2147472637
 thread 3 sequence 651065 ...
LOGMINER: StartScn: 15628771104698
LOGMINER: EndScn:   15628771104742
```

The same data as a table (SCNs shown without the leading digits 15628771):

| Round | Timestamp line | StartScn | EndScn |
|---|---|---|---|
| 1 | 20:41:23 | ...104082 | ...104143 |
| 2 | none | ...104143 | ...104448 |
| 3 | 20:41:42 | ...104448 | ...104699 |
| 4 | none | ...104698 | ...104742 |

Three things stand out:

1. The EndScn of one round is the StartScn of the next (rounds 3 and 4 overlap by one SCN). That means a long-running consumer with a saved position is reading the log. A one-off manual mining session would not advance continuously like this.
2. All four rounds read thread 3, sequence 651065, which is an online log. The rounds span 61, 305, 251, and 44 SCNs, so the reader sits near the tail of the log.
3. Round 1 (20:41:23) and round 3 (20:41:42) are 19 seconds apart with round 2 in between, which works out to one round every 9 to 10 seconds.

The `Memory LWM` line is a LogMiner memory watermark message. It does not indicate a memory shortage.

## 2. Approach

Several kinds of sources can use LogMiner continuously. I ruled them out one by one:

| Candidate | How to rule it out |
|---|---|
| OGG / Streams Capture | Query dba_capture |
| Logical Standby | Check the database role |
| Third-party CDC tool | Inspect external sessions and their SQL |
| Manual DBMS_LOGMNR | Unlikely, given the continuous progress |

## 3. Investigation

### 3.1 Confirm which instance thread 3 belongs to

```sql
select thread#, status,
       enabled, instance
  from v$thread;

select inst_id, instance_name,
       thread#
  from gv$instance;
```

```
THREAD# STATUS ENABLED INSTANCE
      1 CLOSED PUBLIC  orcl1
      3 OPEN   PRIVATE orcl3

INST_ID INSTANCE_NAME THREAD#
      3 orcl3               3
```

Thread 3 belongs to orcl3, the only instance still running, so reading its online log is expected. The thread number is set by the instance's thread parameter and does not have to be consecutive.

### 3.2 Rule out Streams, OGG, and Logical Standby

```sql
select capture_name, status
  from dba_capture;

select name, database_role,
       open_mode,
       supplemental_log_data_min
  from v$database;
```

```
no rows selected

NAME DATABASE_ROLE OPEN_MODE
ORCL PRIMARY       READ WRITE

SUPPLEMENTAL_LOG_DATA_MIN
YES
```

- dba_capture is empty, which rules out Streams and OGG Integrated Extract.
- The database role is PRIMARY, which rules out a logical standby.
- Minimal supplemental logging is enabled, which is a prerequisite for LogMiner-based CDC tools.

On a database that is not a logical standby, dba_logstdby_parameters still returns defaults such as MAX_SERVERS and APPLY_SERVERS, so use the database role to decide whether it is a logical standby.

### 3.3 The LogMiner session is short-lived

```sql
select session_id, session_name,
       session_state
  from v$logmnr_session;
```

```
SESSION_ID  SESSION_STATE
2147494915  UNKNOWN
```

A moment later the session_id had changed to 2147495171, and v$logmnr_logs was empty. Every round goes through "create session, mine, end", so it is hard to catch a complete picture at any single moment.

### 3.4 Filter sessions by SQL text

```sql
select s.sid, s.username,
       s.program, q.sql_text
  from v$session s, v$sql q
 where s.sql_id = q.sql_id
   and (upper(q.sql_text)
        like '%DBMS_LOGMNR%'
    or upper(q.sql_text)
        like '%V$LOGMNR_CONTENTS%');
```

```
SID  USERNAME PROGRAM
1887 SYS      sqlplus@racnode3
```

The only hit is the sqlplus session running this very query, because the query text itself contains DBMS_LOGMNR. v$session.sql_id only reflects the statement a session is executing right now, and each call from the external tool is too short to catch there.

### 3.5 List all external sessions, sorted by PGA

The database had 111 user sessions in total. The ones with the highest PGA:

| SID | Source | PGA (MB) |
|---|---|---|
| 4101 | sidecar-x (k8s pod) | 9.83 |
| 4183 | app-web | 3.96 |
| 2297 | app-server-a | 3.72 |

- Dozens of JDBC sessions come from one reporting service, each with 1.04 MB of PGA. That is an idle connection pool.
- Eight sessions come from sidecar-x. Only SID 4101 is ACTIVE, and its PGA is the highest in the database.
- sqlplus, rman, and oraagent are local operations and cluster agents.

The sidecar-x hostname matches the pod domain format of a Kubernetes StatefulSet:

```
<pod>.<service>.<namespace>
.svc.cluster.local
```

### 3.6 Capture the SQL of SID 4101

```sql
select sid, serial#, status,
       sql_id, prev_sql_id,
       event, last_call_et
  from v$session
 where sid = 4101;
```

```
SID  STATUS SQL_ID
4101 ACTIVE 6aun97xkkkb8p

PREV_SQL_ID   4g6qya99gt1jm
EVENT         control file
              sequential read
```

The text of the two sql_ids (wrapped by hand; ellipses mark omitted content):

```sql
SELECT CURRENT_SCN
  FROM V$DATABASE
```

```sql
SELECT MIN(F.MEMBER) FILE_NAME,
  L.FIRST_CHANGE#, L.NEXT_CHANGE#,
  ... 'ONLINE' AS TYPE ...
FROM V$LOGFILE F, V$LOG L
LEFT JOIN V$ARCHIVED_LOG A ON ...
UNION
SELECT A.NAME, A.FIRST_CHANGE#,
  A.NEXT_CHANGE#, 'YES', ...
FROM V$ARCHIVED_LOG A
WHERE A.STATUS = 'A'
  AND A.NEXT_CHANGE#
      > 15628771107301
  AND A.DEST_ID IN (...)
ORDER BY 7
```

These two statements are the preparation before calling LogMiner: fetch the current SCN, then merge the not-yet-archived online logs with the archived logs after the saved position to get the list of logs to mine. The position 15628771107301 falls in the same range as the SCNs in the alert log, which shows that the consumer reads right at the tail of the log. The wait event control file sequential read corresponds to reading the control file while querying V$LOG and V$ARCHIVED_LOG.

### 3.7 Source IP: spid + netstat

v$session has no client IP column, and machine is a hostname reported by the client itself. To get the client IP, use the server process spid together with operating system connection information:

```sql
select s.sid, p.spid
  from v$session s, v$process p
 where s.paddr = p.addr
   and s.username = 'SYNC_READER'
   and s.machine like 'sidecar%';
```

```
SID   SPID
2625  170804
3650  170672
4101  170287
412   170953
1027  151700
1068  170810
1191  170291
1970  177304
```

```bash
netstat -antp | grep -E \
 "170804|170672|170287|170953|\
151700|170810|170291|177304"
```

Only the remote address and process columns are kept:

```
172.x.x.242:4912   170810/oracleorcl3
172.x.x.242:4907   151700/oracleorcl3
172.x.x.242:10989  170804/oracleorcl3
...(8 lines in total)
```

All 8 connections share the same source IP with different source ports, so they belong to one client's connection pool. When a pod connects to an external database, the database side may see either the pod IP or a node or gateway IP after SNAT, so this needs to be confirmed on the Kubernetes side.

### 3.8 START_LOGMNR in the cursor cache

```sql
select sql_id, executions,
       first_load_time,
       substr(sql_text,1,200)
  from gv$sql
 where upper(sql_text) like
       '%DBMS_LOGMNR.START_LOGMNR%'
   and upper(sql_text) not like
       '%GV$SQL%';
```

This returned 15,936 rows. An excerpt follows (the original is a single line; it is wrapped here, and the SYS.DBMS_LOGMNR. prefix is omitted):

```
BEGIN SYS.DBMS_LOGMNR.START_LOGMNR(
 startScn => '15628771244415',
 endScn   => '15628771244467',
 OPTIONS  =>
  DICT_FROM_ONLINE_CATALOG +
  NO_ROWID_IN_STMT + ...
```

What this shows:

- An external program is calling LogMiner in a loop, each call covering only a few dozen to a few hundred SCNs, which matches the alert log.
- OPTIONS is DICT_FROM_ONLINE_CATALOG, so the online dictionary is used directly and minimal supplemental logging is enough.
- startScn and endScn are concatenated into the SQL as literals, so every round produces new SQL text, a hard parse, and a new cursor, and each cursor is executed only once.

Aggregated by parsing user:

```sql
select parsing_schema_name,
       count(*) cursors,
       min(first_load_time) first_load,
       max(last_active_time) last_act
  from gv$sql
 where sql_text like
  'BEGIN SYS.DBMS_LOGMNR.START_LOGMNR%'
 group by parsing_schema_name;
```

```
PARSING_SCHEMA  CURSORS
SYNC_READER       15960

FIRST_LOAD  2026-11-16/20:01:37
```

The parsing user of the cursors matches the session account from step 3.7. The account, the session SQL, the source IP, and the cursor cache all agree. Checking the CDC tool's console confirmed that this account belongs to an incremental sync task.

Calls per hour (excerpt):

| Period | Calls |
|---|---|
| 11-16 21:00 | 405 |
| 11-17 00:00 | 392 |
| 11-17 12:00 | 382 |
| 11-18 04:00 | 408 |
| 11-18 10:00 | 364 |

Across 41 hours, apart from the partial hours at each end, every hour falls between 350 and 408 calls, about one call every 9 seconds on average, with no obvious peaks or troughs during the day. This matches the interval seen in the alert log in section 1.

Now the instance startup time:

```sql
select to_char(startup_time,
       'yyyy-mm-dd hh24:mi:ss')
  from v$instance;
```

```
2026-11-16 19:58:28
```

The instance started at 19:58:28, and the earliest START_LOGMNR cursor in the cache appeared at 20:01:37, about 3 minutes later. The sync task reconnected automatically after the database restarted and resumed reading the log.

### 3.9 Impact on the shared pool

```sql
select count(*) cursors,
  round(sum(sharable_mem)
        /1024/1024,1) mb
  from gv$sql
 where sql_text like
  'BEGIN SYS.DBMS_LOGMNR.START_LOGMNR%';
```

```
CURSORS  MB
  15962  306.9
```

Compare with the whole library cache and the free memory in the shared pool:

```sql
select
 round(sum(sharable_mem)
       /1024/1024) total_mb,
 round(sum(case when sql_text like
  'BEGIN SYS.DBMS_LOGMNR.START_LOGMNR%'
  then sharable_mem end)
       /1024/1024) logmnr_mb
  from gv$sql;

select pool, name,
       round(bytes/1024/1024) mb
  from v$sgastat
 where pool = 'shared pool'
   and name = 'free memory';
```

```
TOTAL_MB  LOGMNR_MB
   12434        307

POOL         NAME         MB
shared pool  free memory  33347
```

```bash
grep -c "ORA-04031" alert_orcl3.log
0
```

| Metric | Value |
|---|---|
| Memory used by LogMiner cursors | 307 MB |
| Total library cache | 12,434 MB |
| Share | about 2.5% |
| Free shared pool | 33,347 MB |
| ORA-04031 in the current alert log | 0 |

A single cursor takes about 19.7 KB, and about 9,400 new ones appear each day, roughly 180 MB. The instance had been up for about 40 hours, and these cursors had not been aged out. That is about 0.1 hard parses per second, so the CPU cost is negligible.

The LogMiner cursors keep accumulating and take about 2.5% of the library cache, with 33 GB of free shared pool. There is no observable impact at present.

## 4. Conclusion and handling

These entries in the alert log are normal records produced by an external CDC tool that reads the current node's online log in near real time through LogMiner.

| Situation | Action |
|---|---|
| The sync task is still in use | Keep it, and control the size of the alert log with log rotation |
| The sync task is obsolete | Stop and delete the task in the CDC tool's console |

Key points:

- Stopping and deleting the sync task is done in the CDC tool's console.
- Rotate the text-format `alert_<SID>.log` with the operating system's logrotate. `adrci purge` only handles the XML version of the log.
- The cursor buildup caused by SCNs concatenated as literals can be reported to the tool vendor, to evaluate bind variables or a different fetch strategy.

## 5. Cleaning up the decommissioned node 1

**Checks before cleanup**

- All 8 redo groups of thread 1 (1, 2, 3, 4, 11, 12, 13, 14) are INACTIVE.
- The archived logs of thread 1 are backed up according to the backup policy.
- The log holding the CDC position does not belong to thread 1.
- The current instance uses undo_tablespace UNDOTBS3, and the undo tablespace that belonged to node 1 is UNDOTBS1.

**Run on node 3**

```sql
alter database disable thread 1;

alter database drop logfile
  group 1;
-- repeat for groups 2, 3, 4,
-- 11, 12, 13, 14
```

If ORA-01624 appears, the thread was not shut down cleanly the last time. Run this first:

```sql
alter system checkpoint global;
```

If the error persists, start node 1, shut it down cleanly once, and then drop the log groups.

**Verification after cleanup**

```sql
select thread#, status, enabled
  from v$thread;
```

```
THREAD# STATUS ENABLED
      3 OPEN   PRIVATE
```

Only thread 3 is left, and v$log now shows only its 8 groups (30 to 33 and 54 to 57, with 56 as CURRENT).

**Drop the undo tablespace**

```sql
show parameter undo_tablespace
```

```
undo_tablespace  string  UNDOTBS3
```

```sql
drop tablespace UNDOTBS1
  including contents
  and datafiles;
```

```
Tablespace dropped.
```

This operation cannot be undone. Before running it, check two things: the tablespace does not belong to a running instance, and no uncommitted transaction references it.

Removing the whole node from the cluster (`srvctl remove instance`, `crsctl delete node`, updating the inventory, and so on) is a cluster-level operation and should follow the delete-node procedure separately.

## 6. Summary

1. Work out what kind of log entry it is first, then find where it comes from.
2. When filtering sessions by SQL text, use a prefix match or exclude your own sid.
3. External tools hold very short sessions, so start from machine, program, and PGA, then capture the session's SQL pattern.
4. Get the client IP from spid plus netstat.
5. The cursor cache in gv$sql lets you look back at short-lived calls, and parsing_schema_name maps them to an account.
6. Quantify the impact: LogMiner cursors take about 2.5% of the library cache, with 33 GB of free shared pool.
7. Stop the sync task in the CDC tool's console.
8. Order for cleaning up a retired node: check dependencies, disable the thread, drop the log files, then handle the undo tablespace.
