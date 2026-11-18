---
title: "Oracle 11g RAC alert 日志每 9 秒刷一轮 LogMiner 的溯源排查"
slug: oracle-11g-rac-logminer-trace
date: 2026-11-18
description: "一套 11g RAC 下线节点1后，alert 日志持续出现 LogMiner 信息。本文记录如何定位日志来源，以及废弃节点1的清理。"
categories: [database]
tags: [oracle, rac, logminer, troubleshooting, cdc]
---
## 环境

| 项目 | 内容 |
|---|---|
| 版本 | Oracle 11g RAC |
| 存储 | ASM，redo 在独立磁盘组 |
| 状态 | 节点1已下线，仅 orcl3 运行 |
| 角色 | PRIMARY，READ WRITE |

## 一、现象

下面是 alert 日志中连续 4 轮的节选，已手动换行，路径已脱敏。第 1 轮保留全部行，第 2 到 4 轮省略固定行：

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

整理成表（SCN 省略前 8 位 15628771）：

| 轮次 | 时间行 | StartScn | EndScn |
|---|---|---|---|
| 1 | 20:41:23 | ...104082 | ...104143 |
| 2 | 无 | ...104143 | ...104448 |
| 3 | 20:41:42 | ...104448 | ...104699 |
| 4 | 无 | ...104698 | ...104742 |

从这段日志可以看到三点：

1. 上一轮的 EndScn 就是下一轮的 StartScn（第 3、4 轮之间有 1 个 SCN 的重叠），说明有一个带位点的持续消费者在读取日志，人工执行一次性挖掘不会出现这种连续推进。
2. 4 轮读取的都是 thread 3、sequence 651065 这个在线日志，每轮跨度分别为 61、305、251、44 个 SCN，读取位置在日志尾部附近。
3. 第 1 轮（20:41:23）与第 3 轮（20:41:42）相隔 19 秒，中间隔着第 2 轮，折合约 9 到 10 秒一轮。

`Memory LWM` 一行是 LogMiner 的内存水位提示，与内存不足无关。

## 二、排查思路

持续使用 LogMiner 的来源有以下几类，逐一排除：

| 候选 | 排除方法 |
|---|---|
| OGG / Streams Capture | 查 dba_capture |
| Logical Standby | 查库角色 |
| 第三方 CDC 工具 | 查外部会话和 SQL |
| 人工 DBMS_LOGMNR | 日志持续推进，基本排除 |

## 三、排查过程

### 1. 确认 thread 3 对应的实例

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

thread 3 对应 orcl3，是当前唯一运行的实例，读取它的在线日志属于正常行为。线程号由实例的 thread 参数指定，不要求连续。

### 2. 排除 Streams、OGG 和逻辑备库

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

- dba_capture 为空，排除 Streams 和 OGG Integrated Extract
- 库角色是 PRIMARY，排除逻辑备库
- 最小补充日志已开启，这是 LogMiner 型 CDC 工具的前置条件

dba_logstdby_parameters 在非逻辑备库上也会返回 MAX_SERVERS、APPLY_SERVERS 等默认值，判断是否逻辑备库以库角色为准。

### 3. LogMiner 会话生命周期很短

```sql
select session_id, session_name,
       session_state
  from v$logmnr_session;
```

```
SESSION_ID  SESSION_STATE
2147494915  UNKNOWN
```

隔一会再查，session_id 变成 2147495171，v$logmnr_logs 为空。每一轮都经历"建会话、挖掘、结束"，很难在某一时刻抓到完整现场。

### 4. 按 SQL 文本过滤会话

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

命中的是执行这条查询的 sqlplus 会话自身，因为查询文本里含有 DBMS_LOGMNR。v$session.sql_id 只反映会话当前正在执行的 SQL，外部工具每次调用耗时很短，难以在这里抓到。

### 5. 查看全部外部会话，按 PGA 排序

全库共 111 个用户会话，PGA 最高的几个：

| SID | 来源 | PGA(MB) |
|---|---|---|
| 4101 | sidecar-x（k8s pod） | 9.83 |
| 4183 | app-web | 3.96 |
| 2297 | app-server-a | 3.72 |

- 数十个 JDBC 会话来自同一个报表服务，PGA 均为 1.04MB，是空闲连接池
- 8 个会话来自 sidecar-x，其中只有 SID 4101 为 ACTIVE，PGA 为全库最高
- sqlplus、rman、oraagent 属于本机运维和集群代理

sidecar-x 的主机名符合 k8s StatefulSet 的 pod 域名格式：

```
<pod>.<service>.<namespace>
.svc.cluster.local
```

### 6. 抓取 SID 4101 的 SQL

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

两个 sql_id 对应的文本（已换行，省略号为省略内容）：

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

这两条 SQL 是调用 LogMiner 之前的准备工作：先取当前 SCN，再合并未归档的在线日志和位点之后的归档日志，得到待挖掘的日志清单。位点 15628771107301 与 alert 日志中的 SCN 处于同一区间，说明消费者的读取位置紧跟日志尾部。等待事件 control file sequential read 对应查询 V$LOG、V$ARCHIVED_LOG 时读控制文件。

### 7. 来源 IP：spid + netstat

v$session 没有客户端 IP 字段，machine 是客户端自报的主机名。客户端 IP 通过服务器进程 spid 和操作系统连接信息获取：

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

输出仅保留对端地址和进程两列：

```
172.x.x.242:4912   170810/oracleorcl3
172.x.x.242:4907   151700/oracleorcl3
172.x.x.242:10989  170804/oracleorcl3
...（共 8 条）
```

8 个连接源 IP 相同、源端口各不相同，属于同一个客户端的连接池。pod 访问外部库时，数据库侧看到的可能是 pod IP，也可能是经 SNAT 后的节点或网关 IP，需结合 k8s 侧确认。

### 8. 游标缓存中的 START_LOGMNR

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

返回 15936 行。节选如下（原文为单行，此处换行，并省略 SYS.DBMS_LOGMNR. 前缀）：

```
BEGIN SYS.DBMS_LOGMNR.START_LOGMNR(
 startScn => '15628771244415',
 endScn   => '15628771244467',
 OPTIONS  =>
  DICT_FROM_ONLINE_CATALOG +
  NO_ROWID_IN_STMT + ...
```

从中可以看到：

- 外部程序在循环调用 LogMiner，每次只覆盖几十到几百个 SCN，与 alert 日志一致
- OPTIONS 为 DICT_FROM_ONLINE_CATALOG，直接使用在线字典，只开最小补充日志即可满足
- startScn、endScn 以字面量拼入 SQL，每一轮都是新的 SQL 文本，产生一次硬解析和一个新游标，每个游标只执行 1 次

按解析用户聚合：

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

游标的解析用户与步骤 7 中的会话账号一致。账号、会话 SQL、来源 IP、游标缓存互相吻合，在 CDC 工具控制台核对，该账号对应的是一个增量同步任务。

按小时统计调用次数（节选）：

| 时段 | 次数 |
|---|---|
| 11-16 21 时 | 405 |
| 11-17 00 时 | 392 |
| 11-17 12 时 | 382 |
| 11-18 04 时 | 408 |
| 11-18 10 时 | 364 |

41 个小时中，除首尾不足一小时的时段外，每小时在 350 到 408 次之间，平均约每 9 秒一次，全天没有明显峰谷，与第一节日志中的间隔一致。

再看实例启动时间：

```sql
select to_char(startup_time,
       'yyyy-mm-dd hh24:mi:ss')
  from v$instance;
```

```
2026-11-16 19:58:28
```

实例 19:58:28 启动，缓存中最早的 START_LOGMNR 游标出现在 20:01:37，相隔约 3 分钟。同步任务在库重启后自动重连并恢复读取日志。

### 9. 对共享池的影响

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

对比整体 library cache 和共享池空闲内存：

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

| 指标 | 数值 |
|---|---|
| LogMiner 游标占用 | 307MB |
| library cache 总量 | 12434MB |
| 占比 | 约 2.5% |
| 共享池空闲 | 33347MB |
| 当前 alert 中 ORA-04031 | 0 次 |

单个游标约 19.7KB，每天新增约 9400 个，约 180MB。实例启动约 40 小时，这些游标没有被老化淘汰。每秒约 0.1 次硬解析，CPU 开销可以忽略。

LogMiner 游标持续堆积，占 library cache 约 2.5%，共享池空闲 33GB，目前没有可观察的影响。

## 四、结论与处理

alert 日志中的这些内容，是外部 CDC 工具通过 LogMiner 近实时读取当前节点在线日志产生的正常记录。

| 情况 | 处理 |
|---|---|
| 同步任务仍在使用 | 保留，通过日志轮转控制 alert 体积 |
| 同步任务已废弃 | 在 CDC 工具控制台停止并删除任务 |

处理要点：

- 同步任务的停止和删除在 CDC 工具控制台完成
- 文本版 `alert_<SID>.log` 使用操作系统的 logrotate 轮转，`adrci purge` 处理的是 XML 版日志
- 字面量拼接 SCN 产生的游标堆积，可反馈给工具方，评估绑定变量或调整拉取策略

## 五、下线节点1的清理

**清理前核对**

- thread 1 的 8 个 redo 组（1、2、3、4、11、12、13、14）均为 INACTIVE
- thread 1 的归档已按备份策略备份
- CDC 位点所在的日志不属于 thread 1
- 当前实例 undo_tablespace 为 UNDOTBS3，节点1对应的 undo 表空间为 UNDOTBS1

**在节点3上执行**

```sql
alter database disable thread 1;

alter database drop logfile
  group 1;
-- 对 2、3、4、11、12、13、14
-- 重复执行
```

若出现 ORA-01624，说明该线程上次并非正常关闭，先执行：

```sql
alter system checkpoint global;
```

仍报错时，拉起节点1正常关闭一次后再删除。

**清理后验证**

```sql
select thread#, status, enabled
  from v$thread;
```

```
THREAD# STATUS ENABLED
      3 OPEN   PRIVATE
```

只剩 thread 3，v$log 中也只剩它的 8 个组（30 到 33、54 到 57，其中 56 为 CURRENT）。

**删除 undo 表空间**

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

该操作不可逆，执行前核对两项：表空间不属于运行中的实例；没有未提交事务引用。

将整个节点从集群中摘除（`srvctl remove instance`、`crsctl delete node`、更新 inventory 等）属于集群层面的操作，单独按删除节点流程执行。

## 六、小结

1. 先判断日志性质，再定位来源
2. 按 SQL 文本过滤会话时，用前缀匹配或排除自身 sid
3. 外部工具的会话生命周期短，从 machine、program、PGA 入手，再抓取会话 SQL 特征
4. 客户端 IP 通过 spid 加 netstat 获取
5. gv$sql 的游标缓存可回溯短命调用，配合 parsing_schema_name 对应账号
6. 用数字评估影响：LogMiner 游标占 library cache 约 2.5%，共享池空闲 33GB
7. 同步任务在 CDC 工具控制台停止
8. 清理废弃节点的顺序：核对依赖、disable thread、drop logfile、处理 undo 表空间
