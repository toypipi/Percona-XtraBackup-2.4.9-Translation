Percona XtraBackup 基于 InnoDB 的崩溃恢复功能。它复制你的 InnoDB 数据文件时，会导致内部不一致的数据;但是它会对文件执行崩溃恢复，以使其再次成为一个一致的可用数据库。

这是可行的，因为 InnoDB 维护一个重做日志，也称为事务日志。它包含对 InnoDB 数据的每个变更的记录。当 InnoDB 启动时，它会检查数据文件和事务日志，并执行两个步骤。它将已提交的事务日志条目应用于数据文件，并对任何修改了数据但未提交的事务执行撤销操作。

Percona XtraBackup 通过在启动时记住日志序列号（LSN）来工作，然后复制数据文件。这需要一些时间来完成，所以如果文件正在改变，那么它们会在不同的时间点反映数据库的状态。与此同时，Percona XtraBackup 运行一个后台进程，用于监视事务日志文件，并从中复制更改。 Percona XtraBackup 需要不断做到这一点，因为事务日志是以循环方式写入的，并且可以在一段时间后重新使用。 Percona XtraBackup 自启动以后，每次对数据文件的更改都需要事务日志记录。

Percona XtraBackup 将使用[备份锁](https://www.percona.com/doc/percona-server/5.6/management/backup_locks.html)作为 `FLUSH TABLES WITH READ LOCK` 的轻量级替代产品。此功能在 Percona Server 5.6+ 中可用。 Percona XtraBackup 自动使用它来复制非 InnoDB 数据，以避免阻止修改 InnoDB 表的DML查询。当服务器支持备份锁时，xtrabackup 将首先复制 InnoDB 数据，运行 `LOCK TABLES FOR BACKUP` 并复制 MyISAM 表和 .frm 文件。一旦完成，文件的备份将开始。它将备份 .frm，.MRG，.MYD，.MYI，.TRG，.TRN，.ARM，.ARZ，.CSM，.CSV，.par 和 .opt 文件。

{% hint style="danger" %}
注意
仅在 MyISAM 和其他非 InnoDB 表上进行锁定，并且只有在 Percona XtraBackup 完成备份所有 InnoDB / XtraDB 数据和日志后才执行锁定。 Percona XtraBackup 将使用备份锁作为 `FLUSH TABLES WITH READ LOCK` 的轻量级替代产品。此功能在 Percona Server 5.6+ 中可用。 Percona XtraBackup 自动使用它来复制非 InnoDB 数据，以避免阻止修改 InnoDB 表的 DML 查询。
{% endhint %}

之后，**xtrabackup** 将使用 `LOCK BINLOG FOR BACKUP` 来阻止所有可能更改二进制日志位置或 `SHOW MASTER` 或 `SLAVE STATUS`打印的 `Exec_Master_Log_Pos` 或 `Exec_Gtid_Set`（即对应主库二进制日志坐标的当前复制从库上的SQL线程状态）的操作。然后 **xtrabackup** 将完成复制 REDO 日志文件并获取二进制日志坐标。完成此操作后，**xtrabackup** 将解锁二进制日志和表。

最后，二进制日志位置将打印到 `STDERR`，如果一切正常，**xtrabackup** 将退出并返回0。

请注意，**xtrabackup** 的 `STDERR` 不会写入任何文件。您必须将其重定向到一个文件，例如 `xtrabackup OPTIONS 2> backupout.log`。

它还将在备份目录中创建以下文件。

在准备阶段，Percona XtraBackup 使用复制的事务日志文件对复制的数据文件执行崩溃恢复。完成此操作后，数据库已准备好进行恢复和使用。

备份的 MyISAM 和 InnoDB 表最终将保持一致，因为在准备（恢复）过程之后，InnoDB 的数据将前滚到备份完成的位置，而不会回滚到备份开始的位置。此时间点与 `FLUSH TABLES WITH READ LOCK` 相匹配，因此 MyISAM 数据和准备好的 InnoDB 数据保持同步。

**xtrabackup** 和 **innobackupex** 工具都提供了前面解释中没有提到的许多功能。每个工具的功能都在手册中进一步详细解释。简而言之，这些工具允许您通过复制数据文件，复制日志文件和将日志应用于数据的各种组合来执行如流式备份和增量式备份的各种备份。

# 恢复备份

要使用 **xtrabackup** 恢复备份，可以使用 `xtrabackup --copy-back` 或 `xtrabackup --move-back` 选项。

**xtrabackup** 将从 my.cnf 读取变量 datadir, innodb_data_home_dir , innodb_data_file_path , innodb_log_group_home_dir 并检查目录是否存在。

它将首先复制MyISAM表，索引等（.frm , .MRG , .MYD , .MYI , .TRG , .TRN , .ARM , .ARZ , .CSM , .CSV , par 和 .opt 文件），然后复制 InnoDB 表和索引，最后是日志文件。它将在复制时保留文件的属性，备份文件将由创建备份的用户拥有，因此，在启动数据库服务器之前你可能需要将文件的所有者更改为 mysql 。

或者，您还可以使用 `xtrabackup --move-back` 用于恢复备份。 该选项与 `xtrabackup --copy-back` 类似，唯一的区别在于它不是复制文件而是将它们移动到目标位置。 由于此选项删除备份文件，因此必须谨慎使用。 当没有足够的可用磁盘空间来容纳数据文件及其备份副本时，这很有用。
