---
layout: post
title:  "阿里云RDS自建从数据库"
date:   2016-12-19 20:00:00 +0800
categories: jekyll update
---

需求：[将RDS的数据同步到本地数据库][RDS-slave]，并获取 binlog 。

### 一、获取备份

可通过 innobackupex 获取物理备份，这里使用的是 rds ，直接 wget 获取相应备份并解压到当前目录。

    $ tar zxf hinsxxxxxx_data_20161219032352.tar.gz

回滚未提交事务，同步已经提交事务到数据文件，默认使用 backup-my.cnf 中的参数，通常可用 --use-memory 参数指定可使用的内存大小。并可查看 xtrabackup_* 等相关文件记录的主从等信息。

    $ innobackupex  --apply-log  --use-memory=150M  ./
    ......innobackupex: completed OK!

将物理备份中的数据文件拷贝到 /etc/my.cnf 中定义的数据库存储目录。如服务器空间不足可使用 --move-back 替换 --copy-back 。

    $ sudo innobackupex --copy-back ./
    ......innobackupex: completed OK!

务必记得修改数据文件的属主后再启动 slave mysql。

    $ sudo chown  mysql.mysql /srv/mysqldata/ -R
    $ sudo service  mysqld  start
    $ mysql -h 127.0.0.1 -u root

### 二、建立主从关系

1、基本命令，可查询 mysql 相关参数、信息、状态等。

    > select @@server_uuid;
    > show global variables like 'server_id';
    > show global variables like '%bin%';
    > show global variables like '%gtid%';
    > show binary logs;
    > show binlog events;         ##  show binlog events in 'mysql-bin.000011' from 69603 limit 3;
    > flush logs                  ##  手动刷新，生成新的binlog
    > show processlist\G
    > show master status\G;
    > show slave status\G;

2、在 master 上建立复制账号。

    > GRANT REPLICATION SLAVE ON *.* TO 'user'@'slaveIP' IDENTIFIED BY 'passwd';

若 rds 不通过[高权限账号][RDS-super-user]建立此用户，直接建立的普通用户是默认具有 Repl_slave 和 Repl_client 权限的。

3、slave 上设置连接主的信息。

先清空这3张表。

    > truncate table  mysql.slave_relay_log_info;
    > truncate table  mysql.slave_master_info;
    > truncate table  mysql.slave_worker_info;
    $ sudo service  mysqld  restart

[可选]主从信息清理和二进制文件清理。

    > stop slave;

删除 master.info 和 relay-log.info 文件，并清空内存中主从信息，删除所有 relay-log ，并新生成一个 relay-log 文件；注意和 reset slave 的区别与使用场景。

    > reset slave all;

删除所有 bin-log ，并新生成一个 bin-log 文件，在 GTID 环境中，会清掉系统变量 gtid_purged 和 gtid_executed 的值； master 上慎用。

    > reset master;

slave 上设置 gtid_purged 。可在 master 上通过 show global variables like '%gtid_purged%'; 命令查询相应内容，或查看 xtrabackup_slave_info 文件中内容。

    > SET GLOBAL gtid_purged='xxxxx, xxxxx';

指定连接 master 。

    > change master to master_host='masterIP', master_user='user', master_password='passwd', master_port=3306, master_auto_position=1;

开启 slave，可查看到如下内容。

    > start  slave;
    Slave_IO_Running: Yes
    Slave_SQL_Running: Yes

查看 binlog 中 account 库相关信息。

    $ mysqlbinlog --no-defaults  mysql-bin.000011 -d account

### 三、slave /etc/my.cnf 配置文件

    $ cat /etc/my.cnf
    [client]
    default-character-set=utf8
    [mysql]
    default-character-set=utf8

    [mysqld]
    character_set_server=utf8
    collation-server=utf8_general_ci
    lower_case_table_names=1
    innodb_file_per_table=1
    datadir=/srv/mysqldata
    socket=/var/lib/mysql/mysql.sock

    ## 唯一 ##
    server-id=9753186420
    ## relay log 命名，不配置默认会包含主机名信息 ##
    relay-log=slave-relay-bin
    log-bin=mysql-bin
    ## statement/row/mixed 。由于 rds 传来的是 row 事件，这里只能设置为 row 或 mixed ##
    binlog_format=row
    binlog_checksum=CRC32
    binlog_row_image=full
    log_bin_trust_function_creators=on
    log_bin_use_v1_row_events=on
    ## rds 为主时报错，在 slave 上执行 show slave status; 可以查看到：Last_SQL_Error: Error executing row event: 'Table 'mysql.ha_health_check' doesn't exist' ##
    replicate-wild-ignore-table=mysql.%
    # replicate-wild-ignore-table=test.%
    # replicate-wild-ignore-table =information_schema.%
    # replicate-wild-ignore-table =performance_schema.%
    expire_logs_days=30
    max_binlog_size=300M
    sync_binlog=1
    read_only=on
    ## 默认是 file，写在 master.info 文件中，如果是 table ，写在 mysql.slave_master_info中 ##
    master-info-repository=file
    ## 默认是 file，写在 relay-log.info 文件中，如果是 table ，写在 mysql.slave_relay_log_info ##
    relay-log-info_repository=file
    ## GTID ##
    gtid_mode = on
    enforce_gtid_consistency = on
    log-slave-updates
    ## 重置密码：开启这条后不用密码即可登录，并 UPDATE mysql.user SET Password = PASSWORD('passwd') WHERE user = 'root'; ##
    # skip-grant-tables

    [mysqld_safe]
    log-error=/var/log/mysqld.log
    pid-file=/var/run/mysqld/mysqld.pid

[RDS-super-user]: https://help.aliyun.com/document_detail/26130.html
[RDS-slave]: https://yq.aliyun.com/articles/9044
