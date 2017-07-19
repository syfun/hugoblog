+++
thumbnail = ""
tags = ["其他"]
categories = ["其他"]
date = "2015-05-10T19:18:40+08:00"
description = ""
title = "使用pgpool+流复制搭建Postgresql高可用环境"

+++


研究了2周的pgpool搭建终于有点了成果，在这里总结下整个过程。

## 一.环境

db1 10.0.0.2  ubuntu 14.04 server

db2 10.0.0.3  ubuntu 14.04 server

watchdog VIP： 10.0.0.4

pg版本：    9.4.0

pgpool版本：3.4.1

<!--more-->

```sh
# 创建postgres用户，并创建用于安装pg的目录
root@ubuntu:~# groupadd postgres
root@ubuntu:~# useradd -g postgres -d /home/postgres -m -s /bin/bash postgres
root@ubuntu:~# passwd postgres
root@ubuntu:~# su - postgres
postgres@ubuntu:~ mkdir -p /home/postgres/db/data
```

修改两个节点的hosts文件，设置postgres用户之间无密码连接，使用ssh-keygen和ssh-copy-id两个命令。

依赖包，源码在两个节点上都需要安装。

## 二.依赖包的安装


```sh
# 编译postgresql源码需要zlib1g-dev libreadline-dev flex bison make
root@ubuntu:~# apt-get install --yes --force-yes zlib1g-dev libreadline-dev flex bison make

# pgpool watchdog需要arping
root@ubuntu:~# apt-get install --yes --force-yes arping
```

## 三.源码安装

```sh
# 编译pg源码并安装
root@ubuntu:~# tar xvf postgresql-9.4.0.tar.bz2 -C /tmp
root@ubuntu:~# cd /tmp/postgresql-9.4.0
root@ubuntu:/tmp/postgresql-9.4.0~# ./configure --prefix && make && make install

# 将pg加入环境变量
root@ubuntu:~# sed -i '1i\PATH=\$PATH:/home/postgres/db/bin \nexport PGDATA=/home/postgres/db/data' /home/postgres/.bashrc
root@ubuntu:~# sed -i '1i\PATH=\$PATH:/home/postgres/db/bin \nexport PGDATA=/home/postgres/db/data' /root/.bashrc
root@ubuntu:~# source ~/.bashrc

# 编译pgpool源码并安装
root@ubuntu:~# tar xvf pgpool-II-3.4.1.tar.gz -C /tmp
root@ubuntu:~# cd /tmp/pgpool-II-3.4.1
root@ubuntu:/tmp/pgpool-II-3.4.1~# ./configure --prefix && make && make install
# 安装pgpool-recovery，用于在线恢复
# pg9.4版本中已经含有pgpool-regclass，所以这里不用安装
root@ubuntu:/tmp/pgpool-II-3.4.1~# cd src/sql/pgpool-recovery && make install
```

拷贝pcp.conf，pgpool.conf和pool_hba.conf，默认在/usr/local/etc下

创建/var/run/pgpool，/var/log/pgpool，修改目录权限

```
root@ubuntu:~# mkdir /var/run/pgpool /var/log/pgpool
root@ubuntu:~# chown -R postgres:postgres /var/run/pgpool /var/log/pgpool /usr/local/etc
```


在/home/postgres/db/bin下添加failover_stream.sh脚本（注意执行权限），该脚本用于
故障切换。

failover_stream.sh

```sh
#!/usr/bin/env bash
# Execute command by failover.
# special values:
# 特殊字符	描述
# %d	断开连接的节点的后台 ID。
# %h	断开连接的节点的主机名。
# %p	断开连接的节点的端口号。
# %D	断开连接的节点的数据库实例所在目录。
# %M	旧的主节点 ID。
# %m	新的主节点 ID。
# %H	新的主节点主机名。
# %P	旧的第一节点 ID。
# %r    新的主节点端口
# %R    新的主节点数据库实例目录
# %%	'%' 字符

# ---------------------------------------------------------------------
# prepare
# ---------------------------------------------------------------------

SCRIPT_LOG="/var/log/pgpool/failover.log"

FAILED_NODE_ID=${1}
FAILED_NODE_HOST=${2}
FAILED_NODE_PORT=${3}
FAILED_NODE_PGDATA=${4}
NEW_MASTER_NODE_ID=${5}
OLD_MASTER_NODE_ID=${6}
NEW_MASTER_NODE_HOST=${7}
OLD_PRIMARY_NODE_ID=${8}
NEW_MASTER_NODE_PORT=${9}
NEW_MASTER_NODE_PGDATA=${10}

echo "----------------------------------------------------------------------" >> ${SCRIPT_LOG}
date >> ${SCRIPT_LOG}
echo "----------------------------------------------------------------------" >> ${SCRIPT_LOG}
echo "" >> ${SCRIPT_LOG}

echo "
[ node which failed ]
FAILED_NODE_ID           ${FAILED_NODE_ID}
FAILED_NODE_HOST         ${FAILED_NODE_HOST}
FAILED_NODE_PORT         ${FAILED_NODE_PORT}
FAILED_NODE_PGDATA       ${FAILED_NODE_PGDATA}

[ before failover ]
OLD_PRIMARY_NODE_ID      ${OLD_PRIMARY_NODE_ID}
OLD_MASTER_NODE_ID       ${OLD_MASTER_NODE_ID}

[ after faiover ]
NEW_MASTER_NODE_ID       ${NEW_MASTER_NODE_ID}
NEW_MASTER_NODE_HOST     ${NEW_MASTER_NODE_HOST}
NEW_MASTER_NODE_PORT     ${NEW_MASTER_NODE_PORT}
NEW_MASTER_NODE_PGDATA   ${NEW_MASTER_NODE_PGDATA}
" >> ${SCRIPT_LOG}

# ---------------------------------------------------------------------
# Do promote only when the primary node failes
# ---------------------------------------------------------------------

if [ "${FAILED_NODE_ID}" == "${OLD_PRIMARY_NODE_ID}" ]; then
    PROMOTE_COMMAND="pg_ctl -D ${NEW_MASTER_NODE_PGDATA} promote"

    echo "The primary node (node ${OLD_PRIMARY_NODE_ID}) dies." >> ${SCRIPT_LOG}
    echo "Node ${NEW_MASTER_NODE_ID} takes over the primary." >> ${SCRIPT_LOG}

    echo "Execute: ${PROMOTE_COMMAND}" >> ${SCRIPT_LOG}
    ssh postgres@${NEW_MASTER_NODE_HOST} -T "${PROMOTE_COMMAND}" >> ${SCRIPT_LOG}

else
    echo "Node ${FAILED_NODE_ID} dies, but it's not the primary node. This script doesn't anything." >> ${SCRIPT_LOG}
fi

echo "" >> ${SCRIPT_LOG}

```
在/home/postgres/db/data下添加basebackup.sh和pgpool\_remote\_start，用于在线恢复。

basebackup.sh，db1上$host修改成db1，db2上同理。

```sh
#!/usr/bin/env bash
# Recovery script for streaming replication.
# This script assumes that DB node 0 is primary, and 1 is standby.
#
datadir=$1
desthost=$2
destdir=$3

psql -c "SELECT pg_start_backup('Streaming Replication', true)" postgres

ssh -T postgres@$desthost mv $destdir/basebackup.sh $destdir/../
ssh -T postgres@$desthost rm -rf $destdir/*
ssh -T postgres@$desthost pg_basebackup -D $destdir -Fp -Xs -v -P -h $host -U repl
ssh -T postgres@$desthost mv $destdir/../basebackup.sh $destdir/

ssh -T postgres@$desthost mv $destdir/recovery.done $destdir/recovery.conf
ssh -T postgres@$desthost "sed -i \"s/[# ]*primary_conninfo[ ]*=.*/primary_conninfo = 'host=$host port=5432 user=$PG_REPL_USER'/g\" $destdir/recovery.conf"

psql -c "SELECT pg_stop_backup()" postgres
```

pgpool\_remote\_start

```sh
#!/usr/bin/env bash
# start postmaster on the recoveried node

if [ $# -ne 2 ]
then
    echo "pgpool_remote_start remote_host remote_datadir"
    exit 1
fi

SCRIPT_LOG="/var/log/pgpool/remote_start.log"

DEST_HOST=$1

echo "----------------------------------------------------------------------" >> ${SCRIPT_LOG}
date >> ${SCRIPT_LOG}
echo "----------------------------------------------------------------------" >> ${SCRIPT_LOG}
echo "" >> ${SCRIPT_LOG}

COMMAND="pg_ctl -w  start > /dev/null 2>&1"

echo "
DEST_HOST         ${DEST_HOST}
COMMAND           ${COMMAND}
" >> ${SCRIPT_LOG}

echo "remote start" >> ${SCRIPT_LOG}
ssh postgres@${DEST_HOST} -T "${COMMAND}"

ps auwx | grep postgres >> ${SCRIPT_LOG}
echo "" >> ${SCRIPT_LOG}
```

## 四.配置db1

初始化并启动

```sh
root@ubuntu~# su - postgres
postgres@ubuntu:~$ initdb -U postgres
```

修改postgresql.conf配置文件

```ini
log_destination = 'csvlog' # 使用csv格式的log,因为一般会按大小和时间自动切割
logging_collector = on # 如果使用csvlog，那么要开启日志收集
listen_addresses = '*'
wal_level = hot_standby # 日志级别使用hot_standby
archive_mode = on
archive_command = 'cp %p /home/postgres/db/archive/%f'
max_wal_senders = 3
```

启动pg，添加pgpool_recovery扩展

```sh
postgres@ubuntu:~$ pg_ctl start > /dev/null 2>&1
postgres@ubuntu:~$ psql template1 -c 'create extension pgpool_recovery;'
```

创建用于备份的用户

```sh
postgres@ubuntu:~$ createuser rep -l --replication -E -P
```

修改pg_hba.conf，加入

```
host  all      rep      10.0.0.0/24    trust
host  all      postgres 10.0.0.0/24    trust
```

## 五.配置db2

通过pg_basebackup和db1同步

```sh
postgres@ubuntu:~$ pg_basebackup -D /home/postgres/db/data -Fp -Xs -v -P -h db1 -U rep
```

从/home/postgres/db/share下拷贝recovery.conf文件到data目录下，并修改

```ini
standby_mode = on
primary_conninfo = 'host=db1 port=5432 user=rep'
```

修改postgresql.conf配置：

```ini
hot_standby = on # 开启热备模式
```

启动pg

```sh
postgres@ubuntu:~$ pg_ctl start > /dev/null 2>&1
```

## 六.配置pgpool

修改db1上pgpool.conf：

```ini
listen_addresses = '*'
pcp_listen_addresses = '*'
enable_pool_hba = on
pool_passwd = 'pool_passwd'
replication_mode = off # 关闭复制模式
load_balance_mode = on # 打开负载均衡
master_slave_mode = on # 打开主备
log_destination = 'syslog' # 日志使用syslog
master_slave_sub_mode='stream' # 使用流复制
sr_check_user = 'rep'
sr_check_password = 'rep' # 创建rep用户的时候指定
health_check_user = 'rep'
health_check_password = 'rep'
failover_command = '/home/postgres/db/failover_stream.sh %d %h %p %D %m %M %H %P %r %R'
backend_hostname0 = 'db1'
backend_port0 = 5432
backend_weight0 = 1
backend_data_directory0 = '/home/postgres/db/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = 'db2'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/home/postgres/db/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'
use_watchdog = on # 开启watchdog
wd_hostname = 'db1'
delegate_IP = '10.0.0.4'
heartbeat_destination0 = 'db2'
other_pgpool_hostname0 = 'db2'
other_pgpool_port0 = 9999
other_wd_port0 = 9000
recovery_user = 'postgres'
recovery_password = ''
recovery_1st_stage_command = 'basebackup.sh'
```

修改db2上pgpool.conf（和db1中有区别的）:

```ini
wd_hostname = 'db2'
heartbeat_destination0 = 'db1'
other_pgpool_hostname0 = 'db1'
```

db1和db2上的pool_hba.conf中添加

```ini
host all postgres 10.0.0.0/24 trust
```

db1和db2上的/etc/rsyslog.conf添加，并重启rsyslog

```ini
local0.*    /var/log/pgpool.log
```

使用pg_md5生成密钥，db1和db2上的密钥可能不同:

```sh
postgres@ubuntu:~$ pg_md5 postgres
21232f297a57a5a743894a0e4a801fc3

# 在pcp.conf中添加postgres:21232f297a57a5a743894a0e4a801fc3
```


在db1和db2上启动pgpool

```sh
postgres@ubuntu:~$ pgpool
```

至此，安装完成，下面进行简单的测试。

## 七.测试

```sh
# 使用VIP登录pg，发现db1主，db2备
postgres@ubuntu:~$ psql -h 10.0.0.4 -p 9999
psql (9.4.0)
Type "help" for help.

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role
---------+----------+------+--------+-----------+---------
 0       | db1      | 5432 | 2      | 0.500000  | primary
 1       | db2      | 5432 | 2      | 0.500000  | standby
(2 rows)

# 关闭db1上的pg服务，再次查看nodes，故障切换完成，db1变成备，db2变成主

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role
---------+----------+------+--------+-----------+---------
 0       | db1      | 5432 | 3      | 0.500000  | standby
 1       | db2      | 5432 | 2      | 0.500000  | primary
(2 rows)

# 恢复db1，注意要进行恢复操作，必须使用主节点上的pgpool服务
# 这里db2目前是主节点，所以pgpool的hostname使用db2
postgres@ubuntu:~$ pcp_recovery_node -d 5 db2 9898 postgres postgres 0

postgres=# show pool_nodes;
 node_id | hostname | port | status | lb_weight |  role
---------+----------+------+--------+-----------+---------
 0       | db1      | 5432 | 2      | 0.500000  | standby
 1       | db2      | 5432 | 2      | 0.500000  | primary
(2 rows)
```

其中status的值如下：

0：从未使用，直接忽略

1：server已经启动，但是连接池中没有连接

2：server已经启动，并且在连接池中存在连接

3：server没有启动或者联系不上
