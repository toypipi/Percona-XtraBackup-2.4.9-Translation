# 创建备份

要创建备份，请使用 `xtrabackup --backup` 选项运行 **xtrabackup** 。您还需要使用 `xtrabackup --target-dir` 选项指定备份文件存储的目录，如果 InnoDB 数据或日志文件未存储在同一目录中，则还需要指定他们的位置。如果备份的目标目录不存在， **xtrabackup** 会创建它。如果目录确实存在并且为空，则 **xtrabackup** 将成功运行。 如果目录不为空， **xtrabackup** 不会覆盖现有的文件，它会执行失败，并报操作系统错误码17，`file exists` 。

要开始备份请运行：

```sql
$ xtrabackup --backup --target-dir=/data/backups/
```

这会将备份存储在 `/data/backups/` 目录中。如果您指定的是相对路径，则目标目录将与您当前目录有关。

在备份过程中，您应会看到大量输出显示正在复制的数据文件，以及日志文件线程反复扫描日志文件并从中复制日志文件。下面是一个例子，它显示了在后台扫描日志的日志线程以及在 `ibdata1`  文件上工作的文件复制线程：

```
160906 10:19:17 Finished backing up non-InnoDB tables and files
160906 10:19:17 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '62988944'
xtrabackup: Stopping log copying thread.
.160906 10:19:18 >> log scanned up to (137343534)
160906 10:19:18 Executing UNLOCK TABLES
160906 10:19:18 All tables unlocked
160906 10:19:18 Backup created in directory '/data/backups/'
160906 10:19:18 [00] Writing backup-my.cnf
160906 10:19:18 [00]        ...done
160906 10:19:18 [00] Writing xtrabackup_info
160906 10:19:18 [00]        ...done
xtrabackup: Transaction log of lsn (26970807) to (137343534) was copied.
160906 10:19:18 completed OK!
```

备份的最后您应会看到以下内容，其中 `<LSN>` 的值为与您系统有关的数字：

```
xtrabackup: Transaction log of lsn (<SLN>) to (<LSN>) was copied.
```

{% hint style='info' %}
注意
日志复制线程每秒检查一次事务日志，查看是否有任何需要复制的新日志记录，但是日志复制线程可能无法跟上写入的大量事务日志，并且当日志记录在被读取之前被覆盖时将会遇到错误。
{% endhint %}

备份完成后，目标目录将包含以下文件，假设您有一个 InnoDB 表 `test.tbl1`，并且您使用了 MySQL 的 `innodb_file_per_table` 选项：

```
$ ls -lh /data/backups/
total 182M
drwx------  7 root root 4.0K Sep  6 10:19 .
drwxrwxrwt 11 root root 4.0K Sep  6 11:05 ..
-rw-r-----  1 root root  387 Sep  6 10:19 backup-my.cnf
-rw-r-----  1 root root  76M Sep  6 10:19 ibdata1
drwx------  2 root root 4.0K Sep  6 10:19 mysql
drwx------  2 root root 4.0K Sep  6 10:19 performance_schema
drwx------  2 root root 4.0K Sep  6 10:19 sbtest
drwx------  2 root root 4.0K Sep  6 10:19 test
drwx------  2 root root 4.0K Sep  6 10:19 world2
-rw-r-----  1 root root  116 Sep  6 10:19 xtrabackup_checkpoints
-rw-r-----  1 root root  433 Sep  6 10:19 xtrabackup_info
-rw-r-----  1 root root 106M Sep  6 10:19 xtrabackup_logfile
```

备份可能需要很长时间，时间长短具体取决于数据库的大小。随时取消备份是安全的，因为备份不会修改数据库。

下一步是让备份文件为恢复做好准备。

# 准备备份

在使用 `xtrabackup --backup` 选项进行备份后，首先需要准备它以恢复备份。数据文件在准备好之前不是时间点一致的，因为它们在程序运行的不同时间被复制，并且在复制时它们可能已被更改。如果您尝试使用这些数据文件启动 InnoDB ，它将检测到损坏并自行崩溃，以防止您在损坏的数据上运行。 `xtrabackup --prepare` 步骤使文件在某个瞬间完美一致，因此您可以在其上运行 InnoDB。

您可以在任何机器上运行准备操作;它不需要位于开始备份的服务器或您想要恢复的服务器上。您可以将备份复制到通用服务器并在那里进行准备。

{% hint style='info' %}
注意
您可以使用较新的 Percona XtraBackup 版本准备一个由较旧的 Percona XtraBackup 版本创建的备份，但是反之则不行。在不支持的服务器版本上准备备份应该使用支持该服务器版本的最新 Percona XtraBackup 版本完成。例如，如果使用由 Percona XtraBackup 1.6 创建的 MySQL 5.0 备份，则不支持使用 Percona XtraBackup 2.3 准备备份，因为在 Percona XtraBackup 2.1 中删除了对 MySQL 5.0 的支持。相反，应该使用 2.0 系列中的最新版本。
{% endhint %}

在准备操作期间，**xtrabackup** 启动一种内嵌的修改的 InnoDB （与之链接的库）。这些修改对于禁用 InnoDB 的标准安全检查是必要的，例如报告日志文件大小不正确，这些文件不适合用于备份。这些修改仅适用于 **xtrabackup** 二进制文件;您不需要修改后的 InnoDB 就可以使用 **xtrabackup** 进行备份。

准备步骤通过 “内嵌的 InnoDB” 使用复制的日志文件对复制的数据文件执行崩溃恢复。`prepare` 步骤使用起来非常简单：只需使用 `xtrabackup --prepare` 选项运行 **xtrabackup**，并告诉它要准备哪个目录，例如，准备刚才执行的备份：

```
$ xtrabackup --prepare --target-dir=/data/backups/
```

完成后，您应会看到 `InnoDB shutdown`，并显示如下信息，`LSN` 的值同样取决于您的系统：
```
InnoDB: Shutdown completed; log sequence number 137345046
160906 11:21:01 completed OK!
```

以下所有准备工作都不会更改已经准备好的数据文件，您将会看到输出显示：

```
xtrabackup: This target seems to be already prepared.
xtrabackup: notice: xtrabackup_logfile was already used to '--prepare'.
```

建议不要在准备备份时中断 xtrabackup 进程，因为它可能会导致数据文件损坏，并且备份将变得不可用。如果准备过程中断，则不保证备份的有效性。

{% hint style='info' %}
注意
如果您打算将备份作为进一步增量备份的基础，则应在准备备份时使用 `xtrabackup --apply-log-only` 选项，否则将无法对其应用增量备份。有关更多详细信息，请参阅准备[增量备份](https://www.percona.com/doc/percona-xtrabackup/LATEST/backup_scenarios/incremental_backup.html#incremental-backup)的文档。
{% endhint %}

# 恢复备份

{% hint style='danger' %}
警告
备份在恢复之前需要做好准备。
{% endhint %}

为了方便使用 **xtrabackup** 有一个 `xtrabackup --copy-back` 选项，它将备份复制到服务器的 `datadir` 目录：
```
$ xtrabackup --copy-back --target-dir=/data/backups/
```

如果您不想保存备份，则可以使用 `xtrabackup --move-back` 选项，它将备份的数据移动到 `datadir` 目录。

如果您不想使用上述任何选项，则可以另外使用 **rsync** 或 **cp** 来恢复文件。
{% hint style='info' %}
注意
恢复备份前， `datadir` 目录必须为空。另外需要注意的是，在执行恢复之前需要关闭 MySQL 服务器。您无法将备份还原到正在运行的 mysqld 实例的`datadir` 目录（导入部分备份时除外）。
{% endhint %}

可用于恢复备份的 **rsync** 命令示例如下所示：
```
$ rsync -avrP /data/backup/ /var/lib/mysql/
```

您应该检查还原的文件是否具有正确的所有权和许可权。

由于文件的属性将被保留，在大多数情况下，备份将由创建备份的用户拥有，因此您需要在启动数据库服务器之前将文件的所有权更改为 `mysql`：
```
$ chown -R mysql:mysql /var/lib/mysql
```

数据现在已恢复，您可以启动服务器了。
