**xtrabackup** 和 **innobackupex** 工具都支持增量备份，这意味着它们可以仅复制自上次备份以来发生更改的数据。

您可以在每次完整备份之间执行多次增量备份，因此您可以这样设置备份过程，例如每周一次完整备份和每天增量备份，或者每天完整备份和每小时增量备份。

增量备份的工作原理是每个 InnoDB 页都包含日志序列号既 LSN。 LSN 是整个数据库的系统版本号。每个页的 LSN 显示了它最近的变化。

增量备份复制那些 LSN 比之前增量备份或完全备份的 LSN 更新的所有页面。有两种算法用于查找要复制的这类页面的集合。第一种算法可用于所有服务器类型和版本，它通过读取所有数据页直接检查页面 LSN。 第二种方法适用于 Percona Server 它启用服务器上[更改页面跟踪](https://www.percona.com/doc/percona-server/5.6/management/changed_page_tracking.html)功能，该功能会在页面被更改时记录下来。这些信息将被写入一个紧凑的单独的所谓的位图文件中。 **xtrabackup** 二进制文件将使用该文件只读取增量备份所需的数据页面，这潜在地节省了很多读取请求。如果**xtrabackup** 程序找到了位图文件，则默认启用后一种算法。即使位图数据可用，也可以指定`xtrabackup --incremental-force-scan` 来读取所有页面。

增量备份实际上不会将数据文件与先前备份的数据文件进行比较。实际上，如果您知道 LSN，则可以使用 `xtrabackup --incremental-lsn` 执行增量备份，而无需拥有以前的备份。增量备份只需读取页面并将其 LSN 与最后一个备份的 LSN 进行比较。但是，您仍需要完整备份才能恢复增量更改;如果没有完整的备份作为基础，增量备份将毫无用处。

# 创建增量备份

要进行增量备份，请照例先开始进行完整备份。 **xtrabackup** 程序将名为 `xtrabackup_checkpoints` 的文件写入备份的目标目录。该文件包含一行内容显示 `to_lsn`，这是备份结束时数据库的 LSN。使用以下命令创建完整备份：

```
$ xtrabackup --backup --target-dir=/data/backups/base
```

如果您查看 `xtrabackup_checkpoints` 文件，您应该看到以下类似的内容，具体取决于您的 LSN 号：

```
backup_type = full-backuped
from_lsn = 0
to_lsn = 1626007
last_lsn = 1626007
compact = 0
recover_binlog_info = 1
```

现在您已经完成了一个完整备份，您可以在它上面进行增量备份。使用以下命令：

```
$ xtrabackup --backup --target-dir=/data/backups/inc1 \
--incremental-basedir=/data/backups/base
```

`/data/backups/inc1/` 目录现在应该包含增量文件，例如 `ibdata1.delta` 和 `test/table1.ibd.delta`。这些文件代表了自 LSN 1626007 以来所做的更改。如果检查此目录中的 `xtrabackup_checkpoints` 文件，则应该看到与以下内容类似的内容：

```
backup_type = incremental
from_lsn = 1626007
to_lsn = 4124244
last_lsn = 4124244
compact = 0
recover_binlog_info = 1
```

`from_lsn` 是备份的起始 LSN，对于增量备份，它必须与先前基本备份的 `to_lsn`（如果它是最后一个检查点的话）相同。

现在可以将此目录用作另一个增量备份的基础：

```
$ xtrabackup --backup --target-dir=/data/backups/inc2 \
--incremental-basedir=/data/backups/inc1
```

该文件夹同样包含 `xtrabackup_checkpoints`：

```
backup_type = incremental
from_lsn = 4124244
to_lsn = 6938371
last_lsn = 7110572
compact = 0
recover_binlog_info = 1
```

{% hint style='info' %}
注意
在这种情况下，您可以看到 `to_lsn` （上次检查点 LSN）和 `last_lsn` （上次复制的 LSN）之间存在差异，这意味着备份过程中服务器上存在一些操作流量。
{% endhint %}

# 准备增量备份

增量备份的 `xtrabackup --prepare` 步骤与完整备份的步骤不同。在完整备份中，执行两种类型的操作以使数据库保持一致：从日志文件中针对数据文件重演已提交的事务，并回滚未提交的事务。在准备增量备份时，您必须跳过未提交事务的回滚，因为备份时未提交的事务可能正在进行，并且很可能会在下一次增量备份中提交。您应该使用 `xtrabackup --apply-log-only` 选项来阻止回滚阶段。

{% hint style='danger' %}
警告
如果您不使用 `xtrabackup --apply-log-only` 选项来阻止回滚阶段，那么您的增量备份将毫无用处。事务回滚后，也不能应用进一步的增量备份。
{% endhint %}

从创建完整备份开始，您就可以准备它，然后将增量差异应用于该备份。回想一下，您有以下备份：

```
/data/backups/base
/data/backups/inc1
/data/backups/inc2
```

要准备基础备份，您需要照例运行 `xtrabackup --prepare` ，但要阻止回滚阶段：

```
$ xtrabackup --prepare --apply-log-only --target-dir=/data/backups/base
```

输出应该与下面的文本结束：

```
InnoDB: Shutdown completed; log sequence number 1626007
161011 12:41:04 completed OK!
```

日志序列号应该与您之前看到的基础备份的 `to_lsn` 匹配。

{% hint style='info' %}
注意
即使回滚阶段已被跳过，此备份实际上仍然可以安全地恢复到现在的状态。如果你恢复并启动 MySQL，InnoDB 会检测到回滚阶段没有执行，并且它会在后台执行，就像它为崩溃恢复启动时做的那样。它会通知您数据库未正常关闭。
{% endhint %}

要将第一次增量备份应用到完全备份，请运行以下命令：

```
$ xtrabackup --prepare --apply-log-only --target-dir=/data/backups/base \
--incremental-dir=/data/backups/inc1
```

这会将增量文件应用于 `/data/backups/base` 中，这会将它们向前滚到增量备份的时间。然后像往常一样将重做日志应用于结果。最终数据位于 `/data/backups/base` 目录中，不在增量目录中。您应该看到类似于以下内容的输出：

```
incremental backup from 1626007 is enabled.
xtrabackup: cd to /data/backups/base
xtrabackup: This target seems to be already prepared with --apply-log-only.
xtrabackup: xtrabackup_logfile detected: size=2097152, start_lsn=(4124244)
...
xtrabackup: page size for /tmp/backups/inc1/ibdata1.delta is 16384 bytes
Applying /tmp/backups/inc1/ibdata1.delta to ./ibdata1...
...
161011 12:45:56 completed OK!
```

同样，LSN 应该与您在早期检查第一次增量备份时看到的一致。如果您从 `/data/backups/base` 恢复文件，则应该在第一次增量备份时看到数据库的状态。

准备第二次增量备份是一个类似的过程：对（已修改的）基础备份应用增量文件，并且及时将数据前滚到第二次增量备份的位置：

```
$ xtrabackup --prepare --target-dir=/data/backups/base \
--incremental-dir=/data/backups/inc2
```

{% hint style='info' %}
注意
合并除最后一个之外的所有增量备份时，都应使用 `xtrabackup --apply-log-only`。这就是为什么上一行不包含 `xtrabackup --apply-log-only` 选项。即使在最后一步使用了 `xtrabackup --apply-log-only` ，备份仍然是一致的，但在这种情况下，服务器将执行回滚阶段。
{% endhint %}

一旦像完整备份那样准备好增量备份，它们就可以以完整备份那样的方式恢复。