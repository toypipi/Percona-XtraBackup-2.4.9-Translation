Percona XtraBackup 需要能够连接到数据库服务器，并在创建备份、某些场景下进行准备以及恢复时需要在服务器和[数据目录](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-datadir)上执行操作。为此，必须满足其执行操作时的权限和许可要求。

权限是指允许在数据库服务器中执行操作的系统用户。**它们设置在数据库服务器上，仅适用于数据库服务器中的用户**。

许可是允许用户在系统上执行操作的权限，例如读取、写入某个目录或启动、停止系统服务。**它们设置在系统级别，仅适用于系统用户**。

无论使用 **xtrabackup** 还是 **innobackupex** ，都有两个参与者：用户调用程序既系统用户，用户在数据库服务器中执行操作既数据库用户。请注意，这两个用户的作用域是不同的，尽管他们可能有相同的用户名。

在本文档中，**innobackupex** 和 **xtrabackup** 的所有调用都假定系统用户已经具有适当的权限，并且除了要执行的操作选项外，您还提供了用于连接数据库服务器的相关选项，并且数据库用户具有足够的权限。

# 连接到服务器

用于连接服务器的数据库用户及其密码由 `xtrabackup --user` 和 `xtrabackup --password` 选项指定：
```
$ xtrabackup --user=DVADER --password=14MY0URF4TH3R --backup \
  --target-dir=/data/bkps/
$ innobackupex --user=DBUSER --password=SECRET /path/to/backup/dir/
$ innobackupex --user=LUKE --password=US3TH3F0RC3 --stream=tar ./ | bzip2 -
```

如果您不使用 `xtrabackup --user` 选项，Percona XtraBackup 将假定数据库用户名为正在执行备份的系统用户名。

# 其他连接选项

根据您的系统环境，您可能需要指定一个或多个以下选项才能连接到服务器：

| 选项 | 说明 |
| ---|---|
| -port | 使用 TCP/IP 连接到数据库服务器时使用的端口。|
| -socket | 连接到本地数据库时使用的套接字。|
| -host | 使用 TCP/IP 连接到数据库服务器时使用的主机。|

这些选项不加改变地传递给 mysql 子进程，详情参见 `mysql --help` 。

{% hint style='info' %}
注意
在多服务器实例的情况下，必须指定正确的连接参数（port, socket, host）以便 **xtrabackup** 与正确的服务器通信。
{% endhint %}

# 需要的许可和权限

一旦连接到服务器，为了执行备份，您需要在服务器的 `datadir` 中具有文件系统级别的 `READ`，`WRITE` 和 `EXECUTE` 权限。

数据库用户需要在要备份的表和数据库上具有以下权限：

* `RELOAD` 和 `LOCK TABLES`（除非指定了 `--no-lock` 选项），为了在开始复制文件之前，使用 `FLUSH TABLES WITH READ LOCK` 和 `FLUSH ENGINE LOGS`，当使用 [Backup Locks](https://www.percona.com/doc/percona-server/5.6/management/backup_locks.html) 时 `LOCK TABLES FOR BACKUP` 和 `LOCK BINLOG FOR BACKUP` 需要此权限，
* `REPLICATION CLIENT`为了获得二进制日志的位置，
* `CREATE TABLESPACE` 为了导入数据表（请参阅 [恢复个别数据表](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/restoring_individual_tables_ibk.html#imp-exp-ibk)），
* `PROCESS` 为了运行 `SHOW ENGINE INNODB STATUS`（这是必需的），并且可以选择查看服务器上正在运行的所有线程（请参阅 [改进的 FLUSH TABLES WITH READ LOCK 处理](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/improved_ftwrl.html#improved-ftwrl)），
* `SUPER` 为了在复制环境中启动或停止从属线程，使用用于[增量备份](https://www.percona.com/doc/percona-xtrabackup/LATEST/xtrabackup_bin/incremental_backups.html#xb-incremental)的[XtraDB 改变页面跟踪](https://www.percona.com/doc/percona-server/5.6/management/changed_page_tracking.html) 和[改进的 FLUSH TABLES WITH READ LOCK 处理](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/improved_ftwrl.html#improved-ftwrl)，
* `CREATE` 权限为了创建 `PERCONA_SCHEMA.xtrabackup_history` 数据库和表，
* `INSERT` 权限，以便将历史记录添加到 `PERCONA_SCHEMA.xtrabackup_history` 表中，
* `SELECT` 权限以便使用 `innobackupex --incremental-history-name` 或 `innobackupex --incremental-history-uuid` ，以便该功能查找` PERCONA_SCHEMA.xtrabackup_history` 表中的 `innodb_to_lsn` 值。

在[Percona XtraBackup 如何工作](https://toypipi.github.io/2018/03/23/Percona-XtraBackup-%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C/)中可以找到何时使用它们的解释。

使用完整备份所需的最小权限创建数据库用户的SQL示例为：
```
mysql> CREATE USER 'bkpuser'@'localhost' IDENTIFIED BY 's3cret';
mysql> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO
       'bkpuser'@'localhost';
mysql> FLUSH PRIVILEGES;
```
