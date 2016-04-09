---
layout: post
title: mysql使用binlog恢复数据
date: 2016-03-22
categories: blog
tags: [Mysql]
author: cqy
---


当mysql因为某些原因数据丢失时，可以使用binlog来恢复数据。
<!-- more -->
本文基于mysql5.6，主要记录一些binlog相关的内容。
### binlog
首先介绍mysql的binlog
binlog是binary log的缩写，是mysql记录数据库变动的二进制日志，会包含所有使数据和表结构发生变化的事件。根据官方文档，mysql的binary log主要有两个目的：

>1、实现复制集：replication中的master通过binary log记录数据变动然后发送给slave；
>2、恢复数据。

可以通过配置 ``--log-bin[=base_name]``变量启动mysql server以启用binlog。
可以通过``--binlog-format={ROW|STATEMENT|MIXED}``来指定mysql使用哪种binlog格式：

>1、statement-based格式的binlog下，replication中的slave通过执行master的binlog中的sql与master同步，这样的副本集也叫SBR(statement-based replication)。statement日志会记录sql、服务器信息、sql执行时间戳、sql执行时长等等信息；
>2、row-based格式的binlog下，master会把数据每一列的改动前后都记录下来，slave复制这些改动来同步master，这样的副本集叫做RBR(row-based replication)；
>3、mixed-based格式的binlog默认采用statement-based，只有在某些情况下会自动切换到row-based格式。
可以看出，row-based binlog的特点十分适合用于数据恢复与增量备份。

### mysqlbinlog
mysqlbinlog是mysql自带的binlog工具，可以直接将binlog用于数据恢复。
**将log_file的变动应用到远程mysql：**
```shell
shell> mysqlbinlog log_file | mysql -h server_name
```

**将数据恢复到某个时间点：**
假设要恢复数据的mysql开启了binlog并且有一个A时间点的备份，现在要把数据恢复到发生了数据丢失的B时间点前。那么就可以在A时间点的备份上应用A-B区间的binlog，得到一个B时间点前数据集。
具体操作可以用到以下命令：
```shell
shell> mysqlbinlog --start-datetime="2005-04-20 9:55:00" --stop-datetime="2005-04-20 10:05:00"  /var/log/mysql/bin.123456 | mysql -u root -p
```
同理还可以使用``--start-positon``、``--stop-position``达到同样目的。
可以看出这里的binlog其实也起着增量备份数据的作用。阿里云rds所实现的创建一个有效期内任意时间点的rds实例应该就是用了类似上面的方法。

**导出数据变动到文件：**
```shell
mysqlbinlog binlog_files.000001 > tmpfile //覆盖
mysqlbinlog binlog_files.000002 >> tmpfile //追加
```
导出的文件同样可以直接应用到mysql
```shell
mysql -u root -p < tmpfile
```
**导出可读的binlog：**
binlog是二进制日志，直接用mysqlbinlog binlog_file命令导出的日志是base64编码过的。
对于row格式的binlog可以用 ``-versbose(-v)``和``--base64-output=DECODE-ROWS``将binlog输出为可读的sql伪码，这些伪码会记录下每次数据操作造成的列改动：
```shell
shell> mysqlbinlog -v --base64-output=DECODE-ROWS log_file
...
# at 218
#080828 15:03:08 server id 1  end_log_pos 258   Write_rows: table id 17 flags: STMT_END_F
### INSERT INTO test.t
### SET
###   @1=1
###   @2='apple'
###   @3=NULL
...
# at 302
#080828 15:03:08 server id 1  end_log_pos 356   Update_rows: table id 17 flags: STMT_END_F
### UPDATE test.t
### WHERE
###   @1=1
###   @2='apple'
###   @3=NULL
### SET
###   @1=1
###   @2='pear'
###   @3='2009:01:01'
...
# at 400
#080828 15:03:08 server id 1  end_log_pos 442   Delete_rows: table id 17 flags: STMT_END_F
### DELETE FROM test.t
### WHERE
###   @1=1
###   @2='pear'
###   @3='2009:01:01'
```
### mysql server logs
除了用于复制集和恢复数据的binlog，mysql还提供了以下日志

|Log Type|	Information Written to Log|
|--------------|-----------------------|
|Error log|	Problems encountered starting, running, or stopping mysqld|
|General query log|	Established client connections and statements received from clients|
|Binary log|	Statements that change data (also used for replication)|
|Relay log|	Data changes received from a replication master server|
|Slow query log|	Queries that took more than long_query_time seconds to execute|
|DDL log| (metadata log)	Metadata operations performed by DDL statements|

### 小结

为了保证数据可恢复性，mysql官方文档提出了一些备份恢复的建议：
>1、永远开启``--log-bin``运行mysql server
>2、定时使用``mysqldump``全量备份
>3、定时使用``mysqladmin flush-logs [log_type ...]`` 增量备份

可以看到，阿里云的一系列备份恢复措施正是这些建议的实践。

为了应对一些突发状况造成的数据丢失，备份与恢复机制是数据基础建设必不可少的一环。
对应mysql来说，开启row-based的binlog并且定时备份数据是一个实用的方案。
但是在不可避免的意外造成的数据丢失之外还需要警惕的是某些人为误操作引起的数据丢失——在直接操作线上数据时要慎之又慎，尤其对于批量数据的修改最好要先dump或复制一份数据再操作。

参考文档：
[binary-log](http://dev.mysql.com/doc/refman/5.6/en/binary-log.html)
[mysqlbinlog](http://dev.mysql.com/doc/refman/5.6/en/mysqlbinlog.html)

