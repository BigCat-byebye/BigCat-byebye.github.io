---
title: Tchouse学习记录
date: 2024-05-10 15:26:37
tags:
---
本文档用于记录Tchouse的学习总结

## Tchouse结构-OSS

- Etcd 关键键值对,比如oss master的地址和端口等
- Confdb 整个oss系统的所有配置, 主备版的pg, 采用stolon做成高可用自动选举
- OssCenterMaster 调度器
- OssCenterSlave 执行器
- Alarm 告警
- Tstudio 界面版的pg客户端
- Dbbridge 数据同步工具

### Etcd

#### 操作

查看Etcd中保存的系统关键数据

```shell
[root@tbase01 lwl]# export ETCDCTL_API=3; etcdctl get --prefix /
/tbase_arb/tbase_oss_center_master/7ca28d258dc12521
172.16.16.13
/tbase_arb/tbase_oss_center_master/7ca28f71b46dc8f1
172.16.16.4
/tbase_arb/tbase_oss_center_master_old
172.16.16.4
/tbase_oss/start_time+172.16.16.13
1706520012
/tbase_oss/start_time+172.16.16.4
```

#### 管理

采用systemd管理

```shell
systemctl status etcd
```

### Confdb

#### 查询主备状态

查看stolon数据

```shell
[root@tbase01 lwl]#  stolonctl status --cluster-name cluster01 --store-backend etcdv3
=== Active sentinels ===

ID              LEADER
b1666289        true
c5de8bce        false

=== Active proxies ===

ID
6ae4487f
8c4da45d

=== Keepers ===

UID             HEALTHY PG LISTENADDRESS        PG HEALTHY      PG WANTEDGENERATION     PG CURRENTGENERATION
71efa6da        true    172.16.16.13:5432       true            4                       4
e0134c03        true    172.16.16.4:5432        true            2                       2

=== Cluster Info ===

Master: 71efa6da

===== Keepers/DB tree =====

71efa6da (master)
└─e0134c03

```

#### 管理

confdb采用stolon的高可用方案, 本质上stolon-keeper, 直接停掉stolon即可

##### Stolon结构

- stolon-keeper
- - 每个keeper都是1个pg实例, 即confdb
- stolon-proxy

  - 统一端口为54321, 该端口高可用, 其后端对应keeper的5432端口
- stolon-sentinel

  - sentinel和proxy拥有各自不同的随机id
- backend: etcd

  - 通过raft协议保证主节点高可用

##### Stolon管理

采用systemd管理

```shell
systemctl status stolon-sentinel && systemctl status stolon-keeper && systemctl status stolon-proxy
```

##### 查看Stolon注册信息

通过命令可查看stolon在etcd中注册的集群数据情况

```shell
[root@tbase01 lwl]#  stolonctl clusterdata --cluster-name cluster01  --store-backend etcdv3 | jq
{
  "formatVersion": 1,
  "changeTime": "2024-05-13T23:54:59.329463908+08:00",
  "cluster": {
    "uid": "11459097",
    "generation": 1,
    "changeTime": "2024-05-13T19:24:23.606149892+08:00",
    "spec": {
      "sleepInterval": "5s",
      "requestTimeout": "10s",
      "maxStandbysPerSender": 5,
      "synchronousReplication": false,
      "additionalWalSenders": null,
      "additionalMasterReplicationSlots": null,
      "usePgrewind": true,
      "initMode": "new",

## 内容为json, 可用jq提取指定字段, 例如 从.cluster.status.master可以看到当前主节点的id

[root@tbase01 lwl]#  stolonctl clusterdata --cluster-name cluster01  --store-backend etcdv3 | jq .cluster.status
{
  "phase": "normal",
  "master": "49f98d79"
}
```

### 运维管理操作

采用shell脚本

### Pg实例

pgxz架构, 据说是基于pgxc架构的增强, 具体差异未知

#### 结构

- Gtm 全局事务管理
  - 全局只能有1个Master, 可以有多个standy节点
- CN 协调节点,
  - 常用1主1从
  - 前端Client统一接入点
  - 存储全局元数据
  - 对实际执行的sql进行优化
- DN 实际数据存储节点
  - 分布式核心, 进行数据存储
  - 生产环境必须至少1主2备

#### 基于pgxc的pg集群搭建部署

[Docker-compose快速创建pgxc集群](https://github.com/BigCat-byebye/pgxc-in-docker-compose/tree/main)

## Tchouse数据库

pg中各个对象(比如表, 字段, 角色, 等等)之间使用oid进行关联, 该oid为同一个数据库内的唯一标识符, oid为隐藏字段, 不显示在正常的select语句中

### 系统表(常用)

文档参考: [PG12中文-系统目录](http://www.postgres.cn/docs/12/catalogs.html)

| 表名             | 作用               |
| ---------------- | ------------------ |
| pg_class         | 存储系统中各类对象 |
| pg_namespace     | 系统中模式信息     |
| pa_tables        | 系统中的表名       |
| pg_attribute     | 系统中各项属性     |
| pg_settings      | 系统配置           |
| pg_stat_activity | 系统活跃信息       |

### 表隐藏字段

| 列名     | 说明                           |
| -------- | ------------------------------ |
| tableoid | 表的oid值                      |
| ctid     | 行的物理位置                   |
| xmin     | 行插入id                       |
| xmax     | 删除修改事务id                 |
| cmin     | 事务标记号, 在1个事务中0,1,2,3 |
| cmax     | 事务标记号, 在1个事务中0,1,2,3 |
| Shardid  | 逻辑分片id                     |
| xmin_gts | 全局事务插入id                 |
| xmax_gtx | 全局事务删除修改id             |

### 系统表之间的关系


### Sql常用

```shell
# 查看分区表
postgres=# select pc.oid, pn.nspname, pc.relname , pc.relkind from pg_class pc inner join pg_namespace pn on pc.relnamespace = pn.oid where pc.relkind = 'p';
  oid  | nspname |       relname       | relkind
-------+---------+---------------------+---------
 82053 | public  | t_p_more_test       | p
 82070 | public  | t_p_more_test_1_10  | p
 82099 | public  | t_native_range      | p

#### 查看快捷指令实际运行的Sql, 可用于学习
postgres=# \set ECHO_HIDDEN on
postgres=# \dt
********* QUERY **********
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'm' THEN 'materialized view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 's' THEN 'special' WHEN 'f' THEN 'foreign table' WHEN 'p' THEN 'table' WHEN 'I' THEN 'index' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r','p','')
      AND n.nspname <> 'pg_catalog'
      AND n.nspname <> 'information_schema'
      AND n.nspname !~ '^pg_toast'
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
**************************

                  List of relations
 Schema |            Name             | Type  | Owner
--------+-----------------------------+-------+-------
 public | lwltest1                    | table | tbase
 public | t_hash_partition            | table | tbase
 public | t_hash_partition_1          | table | tbase
 public | t_p_more_test               | table | tbase

### 展示系统中所有参数
postgres=# show all ;
                  name                   |                                                           setting                                                            |
                                                                           description
-----------------------------------------+------------------------------------------------------------------------------------------------------------------------------+---------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 account_lock_threshold                  | 10                                                                                                                           | Account will be locked after failed  account_lock_threshold times
 account_lock_time                       | 10min                                                                                                                        | The time which user account will be locked.
 account_lock_track_count                | 1000                                                                                                                         | The limit to store user account info. aka the max num of hash table entry.
 advance_verification_action             | error                                                                                                                        | A common option for detecting uncertain bugs.

### 查询实例中参数
postgres=# select * from pg_settings limit 1;
          name          | setting | unit |       category        |                            short_desc                             | extra_desc |  context  | vartype | source  | min_val |  max_val   | enumvals | boot_val | reset_val | sourcefile | source
line | pending_restart
------------------------+---------+------+-----------------------+-------------------------------------------------------------------+------------+-----------+---------+---------+---------+------------+----------+----------+-----------+------------+-------
-----+-----------------
 account_lock_threshold | 10      |      | Reporting and Logging | Account will be locked after failed  account_lock_threshold times |            | superuser | integer | default | 1       | 2147483647 |          | 10       | 10        |            |
     | f
(1 row)

postgres=# select * from pg_settings where name like '%log%' limit 2;
          name          | setting | unit |               category               |                                    short_desc                                     | extra_desc |  context   | vartype | source  | min_val | max_val | enumvals | boot_val | re
set_val | sourcefile | sourceline | pending_restart
------------------------+---------+------+--------------------------------------+-----------------------------------------------------------------------------------+------------+------------+---------+---------+---------+---------+----------+----------+---
--------+------------+------------+-----------------
 alog_common_cache_size | 64      | kB   | Reporting and Logging / Where to Log | Size of common audit log local buffer for each audit worker, kilobytes.           |            | postmaster | integer | default | 8       | 2097151 |          | 64       | 64
        |            |            | f
 alog_common_queue_size | 64      | kB   | Reporting and Logging / Where to Log | Size of share memory queue for each backend to store common audit log, kilobytes. |            | postmaster | integer | default | 8       | 2097151 |          | 64       | 64
        |            |            | f
(2 rows)



```
