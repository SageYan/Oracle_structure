[TOC]



# Oracle RAC TAF

## 理论背景

```shell
TAF（ Transparent Application Failover ） allows oracle clients to reconnect to a surviving instance in the event of a failure of the instance to which it is connected. There are two types of TAF available, SESSION and SELECT.

TAF允许oracle客户端重新连接到一个可持续的实例，当客户端连接的实例出现失败时。
有2种有效的TAF类型:session and select 。
有2种模式在TAF建立的故障转移连接，basic 和preconnect 。

session: 
使用session方式，所有select查询相关的结果在重新建立新的连接后将全部丢失，需要重新发布select命令。

select:
使用select方式，Oracle net会跟踪事务期间的所有select语句，并跟踪每一个与当前select相关的游标已返回多少行给客户端。此时，假定select查询已返回500行，客户端当前连接的节点出现故障，Oracle Net自动建立连接到幸存的实例上并继续返回剩余的行数给客户端。假定总行数为1500，行，则1000行从剩余节点返回。

BASIC： 
客户端通过地址列表成功建立连接后，即仅当客户端感知到节点故障时才创建到其他实例的连接
PRECONNECT: 
预连接模式，是在最初建立连接时就同时建立到所有实例的连接，当发生故障时，立刻就可以切换到其他链路上


说明：
上述两种方式适用于不同的情形，
对于select方式，通常使用与OLAP数据库，
而对于session方式则使用与OLTP数据库。
因为 select 方式，Oracle 必须为每个session保存更多的内容，包括游标，用户上下文等，需要更多的资源。

其次，两种方式期间所有未提交的DML事务将自动回滚且必须重启启动。alter session语句不会failover。临时对象不会failover也不能被重新启动。 
```



## 配置Service-Side TAF 示例 （oracle用户操作）

### 1 创建TAF Service

```sql 
node1-> pwd
/u01/app/11.2.0/grid/bin
node1-> ./srvctl add service -d devdb -s server_taf -r "devdb1" -P BASIC

说明：
Srvctl add service -d -s -r "preferred-instance-list" -a "available-instance-list" -P 
srvctl add service中，只有perferred才会创建服务。 即在OCR中注册一个ora.raw.dmm.rac1.Srv的服务。
```



### 2 启动 server_taf服务

```sql
node1-> ./srvctl start service -d devdb -s server_taf
```

### 3 检查service 运行情况

```sql
node1-> ./srvctl config service -d devdb
Service name: server_taf
Service is enabled
Server pool: devdb_server_taf
Cardinality: 1
Disconnect: false
Service role: PRIMARY
Management policy: AUTOMATIC
DTP transaction: false
AQ HA notifications: false
Failover type: NONE
Failover method: NONE
TAF failover retries: 0
TAF failover delay: 0
Connection Load Balancing Goal: LONG
Runtime Load Balancing Goal: NONE
TAF policy specification: BASIC
Edition: 
Preferred instances: devdb1
Available instances:
```



### 4 确认service ID

```sql
SQL> select name,service_id from dba_services where name = 'server_taf';


NAMESERVICE_ID
---------------------------------------------------------------- ----------
server_taf 3
```



### 5 给service 添加参数

```sql
SQL> execute dbms_service.modify_service (service_name => 'server_taf' - 
, aq_ha_notifications => true - 
, failover_method => dbms_service.failover_method_basic - 
, failover_type => dbms_service.failover_type_select - 
, failover_retries => 180 - 
, failover_delay => 5 - 
, clb_goal => dbms_service.clb_goal_long); 
```



### 6 确认参数修改

```sql
col name format a15 
col failover_method format a11 heading 'METHOD' 
col failover_type format a10 heading 'TYPE' 
col failover_retries format 9999999 heading 'RETRIES' 
col goal format a10 
col clb_goal format a8 
col AQ_HA_NOTIFICATIONS format a5 heading 'AQNOT' 
SQL> select name, failover_method, failover_type, failover_retries,goal, clb_goal,aq_ha_notifications from dba_services where service_id = 3; 
NAMEMETHOD TYPERETRIES GOAL CLB_GOAL AQNOT
--------------- ----------- ---------- -------- ---------- -------- -----
server_tafBASIC SELECT 180 NONE LONG YES
```



### 7 检查service 注册情况

```sql
node1-> lsnrctl services

LSNRCTL for Linux: Version 11.2.0.4.0 - Production on 26-FEB-2017 04:43:55
Copyright (c) 1991, 2013, Oracle. All rights reserved.
Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=LISTENER)))
Services Summary...
Service "+ASM" has 1 instance(s).
Instance "+ASM1", status READY, has 1 handler(s) for this service...
Handler(s):
"DEDICATED" established:2 refused:0 state:ready
LOCAL SERVER
Service "devdb" has 1 instance(s).
Instance "devdb1", status READY, has 1 handler(s) for this service...
Handler(s):
"DEDICATED" established:0 refused:0 state:ready
LOCAL SERVER
Service "devdbXDB" has 1 instance(s).
Instance "devdb1", status READY, has 1 handler(s) for this service...
Handler(s):
"D000" established:0 refused:0 current:0 max:1022 state:ready
DISPATCHER 
(ADDRESS=(PROTOCOL=tcp)(HOST=node1.localdomain)(PORT=26677))
Service "server_taf" has 1 instance(s).
Instance "devdb1", status READY, has 1 handler(s) for this service...
Handler(s):
"DEDICATED" established:0 refused:0 state:ready
LOCAL SERVER
The command completed successfully
```



8 在客户端 使用Service-Side TAF

```perl
在客户端TNS 配置：

server_taf =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = ittestdb.uaes.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = server_taf)
    )
  )
  
连接测试
C:\Users\andy>sqlplus sys/oracle@10.100.25.8:1521/server_taf as sysdba
SQL*Plus: Release 12.1.0.2.0 Production on Sat Feb 25 20:59:41 2017
Copyright (c) 1982, 2014, Oracle. All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Data Mining and Real Application Testing options
```



### 9 停止service

```shell
node1-> ./srvctl stop service -d devdb -s server_taf -i devdb1
```



### 10 删除service

```shell
node1-> ./srvctl remove service -d devdb -s server_taf -i devdb1
```



