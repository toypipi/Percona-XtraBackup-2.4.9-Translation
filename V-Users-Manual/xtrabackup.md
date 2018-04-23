**xtrabackup** 二进制程序是一个已经编译好的，链接到 InnoDB 库和标准 MySQL 客户端库的 C 程序。 InnoDB 库提供了将日志文件应用于数据文件所需的功能，MySQL 客户端库提供了命令行选项解析，配置文件解析等，以使 **xtrabackup**  二进制具有 MySQL 客户端类似的外观。

该工具以 `--backup` 或 `--prepare` 模式运行，分别对应于它执行的两个主要功能。这些函数有几种不同的变体来完成不同的任务，并且有两种不常用的模式 `--stats` 和 `--print-param`。

# 其他类型的备份

## 增量备份
**xtrabackup** 和 **innobackupex** 工具都支持增量备份，这意味着它可以只复制自上次完全备份以来更改过的数据。您可以在每个完全备份之间执行多次增量备份，因此您可以设置这样的备份过程，例如每周一次完全备份和每天增量备份，或每天完全备份和每小时增量备份。

增量备份基于每个 InnoDB 页面（通常大小为16kb）都包含日志序列号或 [LSN](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-lsn) 而工作。 LSN 是整个数据库的系统版本号。每个页面的 LSN 显示了最近的数据变化。增量备份复制那些 LSN 比先前增量或完全备份的 LSN 更新的每个页面。有两种算法用于查找要复制的这类页面的集合。第一种可用于所有服务器类型和版本，通过读取所有数据页直接检查页面 LSN。 第二种方法仅适用于 Percona Server，Percona Server 通过启用服务器上更改页面跟踪功能，该功能会记录更改的页面。这些信息将被写入一个紧凑的单独的所谓的位图文件中。 **xtrabackup** 二进制程序将使用该文件读取增量备份所需的数据页面，这会节省很多读取请求。如果 **xtrabackup** 二进制程序找到位图文件，则默认启用后一种算法。即使位图文件可用，也可以指定 `--incremental-force-scan` 来读取所有页面。

增量备份实际上不会将数据文件与先前备份的数据文件进行比较。实际上，如果您知道 LSN，则可以使用 `--incremental-lsn` 执行增量备份，而无需使用以前的备份。增量备份只需读取页面并将其 LSN 与最后一个备份的 LSN 进行比较。但是，您仍需要完整备份才能恢复增量备份;如果没有完整的备份作为基础，增量备份将毫无用处。

### 创建增量备份

要进行增量备份，照例先进行完整备份。 **xtrabackup** 二进制程序将名为 `xtrabackup_checkpoints` 的文件写入备份的目标目录中。该文件包含一行显示`to_lsn`，这是备份结束时数据库的 LSN。使用以下命令创建完整备份：

```
xtrabackup --backup --target-dir=/data/backups/base --datadir=/var/lib/mysql/
```

如果您想要一个可用的完整备份，请使用 **innobackupex**，因为 xtrabackup 本身不会复制表定义，触发器或其他任何不是 .ibd 的文件。
 
如果您查看 `xtrabackup_checkpoints` 文件，您应该看到以下类似的内容：

```
backup_type = full-backuped
from_lsn = 0
to_lsn = 1291135
```

现在您已经完成了一个完整备份，您可以根据它进行增量备份。使用如下命令：

```
xtrabackup --backup --target-dir=/data/backups/inc1 \
--incremental-basedir=/data/backups/base --datadir=/var/lib/mysql/
```

`/data/backups/inc1/` 目录现在应该包含增量文件，例如 `ibdata1.delta` 和 `test/table1.ibd.delta`。这些表示自 `LSN 1291135` 以来数据库所做的更改。如果检查此目录中的 `xtrabackup_checkpoints` 文件，您应该看到以下类似内容：

```
backup_type = incremental
from_lsn = 1291135
to_lsn = 1291340
```

含义是不言而喻的。现在可以将此目录用作另一个增量备份的基础：

```
xtrabackup --backup --target-dir=/data/backups/inc2 \
--incremental-basedir=/data/backups/inc1 --datadir=/var/lib/mysql/
```

### 准备增量备份

增量备份的 `--prepare` 步骤与完全备份的步骤不同。在完全备份中，执行两种类型的操作来使数据库保持一致：在数据文件上应用日志文件中已提交的事务，并回滚未提交的事务。在准备增量备份时，您必须跳过未提交事务的回滚，因为备份时未提交的事务可能正在进行，并且很可能会在下一次增量备份中提交。您应该使用`--apply-log-only` 选项来防止回滚阶段。

**如果您不使用 `--apply-log-only` 选项来阻止回滚阶段，那么您的增量备份将毫无用处。** 事务回滚后，也不能对其进行进一步的增量备份。

从创建完整备份开始准备，然后将增量差异应用于该备份。回想一下，您有以下三个备份：

```
/data/backups/base
/data/backups/inc1
/data/backups/inc2
```

要准备完全备份，您需要像往常一样运行 `--prepare`，但要防止回滚阶段：

```
xtrabackup --prepare --apply-log-only --target-dir=/data/backups/base
```

输出应该类似于下面的文本：

```
101107 20:49:43  InnoDB: Shutdown completed; log sequence number 1291135
```

日志序列号应该与您之前看到的基础备份的 `to_lsn` 一样。

即使回滚阶段已被跳过，此备份实际上仍然可以安全地恢复到现在的状态。如果您恢复并启动 MySQL，InnoDB 会检测到回滚阶段没有执行，并且会在后台执行它，就像在启动时通常用于崩溃恢复一样。它会通知您数据库未正常关闭。

要将第一次增量备份应用于完整备份，您应该使用以下命令：

```
xtrabackup --prepare --apply-log-only --target-dir=/data/backups/base \
--incremental-dir=/data/backups/inc1
```

这会将增量文件应用于 `/data/backups/base` 中的文件，这些文件会将它们及时前滚到增量备份的时间。然后像照例将重做日志应用于结果。最终数据位于/  `/data/backups/base` 中，不在增量目录中。您应该看到如下的输出：

```
incremental backup from 1291135 is enabled.
xtrabackup: cd to /data/backups/base/
xtrabackup: This target seems to be already prepared.
xtrabackup: xtrabackup_logfile detected: size=2097152, start_lsn=(1291340)
Applying /data/backups/inc1/ibdata1.delta ...
Applying /data/backups/inc1/test/table1.ibd.delta ...
.... snip
101107 20:56:30  InnoDB: Shutdown completed; log sequence number 1291340
```

再说一次，LSN 应该与先前检查第一次增量备份时看到的一致。如果您从 `/data/backups/base` 恢复文件，则应该可以看到第一次增量备份时的数据库状态。

准备第二次增量备份是一个类似的过程：对（已修改的）基础备份应用增量，并且及时将数据前滚到第二次增量备份时的状态：

```
xtrabackup --prepare --target-dir=/data/backups/base \
--incremental-dir=/data/backups/inc2
```

{% hint style='info' %}
注意
合并除最后一个之外的所有增量备份时，都应该使用 `--apply-log-only`。这就是上一行不包含 `--apply-log-only` 选项的原因。即使在最后一步使用了 `--apply-log-only` ，备份仍然是一致的，但在这种情况下，服务器将执行回滚阶段。
{% endhint %}

当您已将所需增量应用于基本备份时, 如果想避免收到关于 InnoDB 未正常关闭的通知。那么，可以再次运行 `--prepare` 而不禁用回滚阶段。

## 部分备份

当启用 `innodb_file_per_table` 选项时，**xtrabackup** 支持部分备份。有三种方法可以创建部分备份：使用正则表达式匹配表名，提供一个包含表明列表的文件或提供一个包含数据库和表名列表的文件。

{% hint style='danger' %}
警告
如果在备份过程中删除了任何匹配到的或已经列出的表，则 **xtrabackup** 将会执行失败。
{% endhint %}

为了达到演示本手册内容的目的，我们假设有一个名为 `test` 的数据库，其中包含名为 `t1` 和 `t2` 的表。

### 使用 `--tables` 选项

第一种备份方法是使用 `--tables` 选项。该选项的值是一个正则表达式，它以 `databasename.tablename` 的形式与完全限定的表名（包括数据库名称）相匹配。

仅备份 `test` 数据库中的表，可以使用以下命令：

```
$ xtrabackup --backup --datadir=/var/lib/mysql --target-dir=/data/backups/ \
--tables="^test[.].*"
```

仅备份 `test.t1` 表，可以使用以下命令：

```
$ xtrabackup --backup --datadir=/var/lib/mysql --target-dir=/data/backups/ \
--tables="^test[.]t1"
```

### 使用 `--tables-file` 选项

`--tables-file` 选项指定一个文件，该文件可以包含多张表名，文件中每行写入一张表名。只有在该文件中列出的表才会被备份。表名需要完全匹配，区分大小写，不能使用模式或正则表达式匹配。表名必须是完全限定的，采用 `databasename.tablename` 的格式。

```
$ echo "mydatabase.mytable" > /tmp/tables.txt
$ xtrabackup --backup --tables-file=/tmp/tables.txt
```

### 使用 `--databases` 和 `--databases-file` 选项

`--databases` 选项指定用空格分隔的数据库和表名列表，以 `databasename[.tablename]` 的形式。 `--databases-file`  选项指定一个包含上述格式的多张数据库和表文件，该文件中每行包含一张要备份的表名。只有列出的数据库和表将被备份。表名需要完全匹配，区分大小写，不能使用模式或正则表达式匹配。

### 准备备份

在部分备份上使用 `--prepare` 选项时，会看到有关表不存在的警告。这是因为这些表存在于 `InnoDB` 内的数据字典中，但相应的 `.ibd` 文件不存在。这些表没有被复制到备份目录中。当您恢复备份并启动 `InnoDB` 时，这些表将从数据字典中删除，不再存在，并且不会有任何错误或警告打印到日志文件中。

在准备阶段您将看到的错误消息的示例如下。

```
InnoDB: Reading tablespace information from the .ibd files...
101107 22:31:30  InnoDB: Error: table 'test1/t'
InnoDB: in InnoDB data dictionary has tablespace id 6,
InnoDB: but tablespace with that id or name does not exist. It will be removed from data dictionary.
```

## 紧凑备份

在进行 InnoDB 表的备份时，可以省略二级索引页面。这将使备份更加紧凑，占用更少的磁盘空间。这样做的缺点是准备备份的过程需要更长的时间，因为需要重新创建这些二级索引。备份大小的差异取决于二级索引的大小。

例如，以下分别为不使用和使用 `--compact` 选项的完整备份大小：

```
#backup size without --compact
2.0G  xb_backup

#backup size taken with --compact option
1.4G  xb_compact_backup
```

{% hint style='info' %}
注意
系统表空间不支持紧凑备份，因此为了能够正确运行备份程序，应该启用 innodb-file-per-table 选项。        
{% endhint %}

此功能在 *Percona XtraBackup 2.1* 中引入。

### 创建紧凑备份

要创建紧凑备份 innobackupex 启动时需要使用 `--compact` 选项：

```
$ xtrabackup --backup --compact --target-dir=/data/backups
```

这将在 `/data/backups` 目录中创建一个紧凑备份。

如果您检查 `target-dir` 文件夹中的 `xtrabackup-checkpoints` 文件，您应该看到如下所示的内容：

```
backup_type = full-backuped
from_lsn = 0
to_lsn = 2888984349
last_lsn = 2888984349
compact = 1
```

当未使用 `--compact` 时，`compact` 的值将为0。通过这种方式，很容易检查备份是否包含辅助索引页。

### 准备紧凑备份

准备紧凑备份需要重建索引。为了准备备份，应该使用一个新选项 `--rebuild-indexes`：

```
$ xtrabackup --prepare --rebuild-indexes /data/backups/
```

除了标准 **innobackupex** 输出之外，输出还应包含重建索引的信息，如：

```
[01] Checking if there are indexes to rebuild in table sakila/city (space id: 9)
[01]   Found index idx_fk_country_id
[01]   Rebuilding 1 index(es).
[01] Checking if there are indexes to rebuild in table sakila/country (space id: 10)
[01] Checking if there are indexes to rebuild in table sakila/customer (space id: 11)
[01]   Found index idx_fk_store_id
[01]   Found index idx_fk_address_id
[01]   Found index idx_last_name
[01]   Rebuilding 3 index(es).
```

此外，您还可以使用 `--rebuild-threads` 选项在重建索引时使用多线程，例如：

```
$ xtrabackup --prepare --rebuild-indexes --rebuild-threads=16 /data/backups/
```

在这种情况下，*Percona XtraBackup* 将一次为一张表创建 16 个工作线程，每个线程同时为一张表重建索引。它还会显示每条消息的线程ID

```
Starting 16 threads to rebuild indexes.

[09] Checking if there are indexes to rebuild in table sakila/city (space id: 9)
[09]   Found index idx_fk_country_id
[10] Checking if there are indexes to rebuild in table sakila/country (space id: 10)
[11] Checking if there are indexes to rebuild in table sakila/customer (space id: 11)
[11]   Found index idx_fk_store_id
[11]   Found index idx_fk_address_id
[11]   Found index idx_last_name
[11]   Rebuilding 3 index(es).
```

由于 *Percona XtraBackup* 在将增量备份应用于紧凑完整备份时没有任何提示信息，因此不论以后是否会应用更多的增量备份，每次将增量备份应用于完整备份时，都需要用户显示指定重建索引。在每次增量备份合并时无条件地重建索引不是一种好的选择，因为这是一项非常昂贵的操作。

### 恢复紧凑备份

**xtrabackup** 二进制程序没有提供任何恢复备份的功能。这是由用户来做的。您可以使用 **rsync** 或 **cp** 来恢复文件。您应该检查恢复的文件是否具有正确的所有权和许可权。

### 其他阅读

[功能预览：Percona XtraBackup 中的紧凑备份](https://www.percona.com/blog/2013/01/29/feature-preview-compact-backups-in-percona-xtrabackup/)

# 高级功能

## 节流备份

尽管 xtrabackup 不会阻止数据库操作，但任何备份都可以将负载添加到正在备份的系统中。 在没有太多备用 I/O 容量的系统上，调节xtrabackup 读取和写入数据的速率可能会有所帮助。 您可以使用 `--throttle` 选项执行此操作，该选将项限制每秒的 I/O 操作数为 1 MB。

下图显示了当 `--throttle = 1` 时节流备份是如何工作的。

![](http://oio4l6s3j.bkt.clouddn.com/image/20180410/throttle.png)

在 `--backup` 模式下，此选项限制 xtrabackup 每秒执行的读写操作对的数量。 如果您正在创建增量备份，则限制的是每秒读 IO 操作的次数。

默认情况下，不存在限制，并且 xtrabackup 尽可能快地读取和写入数据。 如果您对 I/O 操作的限制设置过于严格，备份可能会变得非常缓慢，以致于它永远无法跟上 InnoDB 写事务日志的速度，因此备份可能永远无法完成。

## 使用 xtrabackup 编写脚本备份

**xtrabackup** 工具具有若干功能，以便脚本在执行相关任务时对其进行控制。 [innobackupex 脚本](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/innobackupex_script.html)就是一个例子，但是也很容易用您自己的命令行脚本来控制 **xtrabackup**。

### 复制后暂停

在备份模式下，xtrabackup 通常会使用后台线程复制日志文件，使用前台线程复制数据文件，最后停止日志复制线程并结束。如果使用 `--suspend-at-end` 选项，xtrabackup 将不会停止日志线程并结束，而是继续复制日志文件，并在名为 `xtrabackup_suspended` 的目标目录中创建一个文件。只要该文件存在，xtrabackup 就会继续观察日志文件并将其复制到目标目录中的 `xtrabackup_logfile` 中。当该文件被移除时，**xtrabackup** 将完成复制日志文件并退出。

此功能对于协调 InnoDB 数据备份与其他操作非常有用。也许最明显的作用就是复制表定义（`.frm` 文件），以便可以恢复备份。您可以在后台启动 **xtrabackup**，等待创建 `xtrabackup_suspended` 文件，然后复制完成备份所需的任何其他文件。这正是 `innobackupex` 工具所做的（它也复制MyISAM 数据和其他文件）。 

### 生成元数据

备份包含恢复备份所需的所有信息是一个好主意。 **xtrabackup** 工具可以打印出需要恢复数据和日志文件的 `my.cnf` 文件的内容。如果添加 `--print-param` 选项，它将打印出如下所示的内容：

```
#  ThisMySQL options file was generated by XtraBackup.
[mysqld]
datadir = /data/mysql/
innodb_data_home_dir = /data/innodb/
innodb_data_file_path = ibdata1:10M:autoextend
innodb_log_group_home_dir = /data/innodb-logs/
```

您可以将此输出重定向到备份目标目录中的文件当中。

### 一致的备份源目录

可能存在缺省文件或其他因素导致 **xtrabackup** 不是从您预期的位置备份数据。为了防止这种情况发生，可以使用 `--print-param` 来询问将从何处复制数据。您可以使用输出来确保 **xtrabackup** 和您的脚本使用相同的数据集。

### 日志流

您可以指定 **xtrabackup** 省略复制数据文件，并简单地将日志文件流式传输到其标准输出，而不是使用 `--log-stream` 。这会自动添加 `--suspend-at-end` 选项。然后，您的脚本可以通过将日志文件传输到 SSH 连接并使用 **rsync** 或 [xbstream二进制程序](https://www.percona.com/doc/percona-xtrabackup/LATEST/xbstream/xbstream.html)等工具将数据文件复制到另一台服务器来执行流式传输远程备份等任务。

## 分析表统计信息

**xtrabackup** 程序可以在只读模式下分析 InnoDB 数据文件以提供统计信息。 要做到这一点，您应该使用 `--stats` 选项。 您可以将它与 `--tables` 选项结合使用，以限制要检查的文件。 它也适用于 `--use-memory` 选项。

您可以在正在运行的服务器上执行分析，但是由于分析过程中数据发生更改而可能导致出现错误。 或者，您也可以分析数据库的备份副本。 无论哪种方式，要使用统计功能，您需要一个包含正确大小的日志文件的数据库的干净副本，因此您需要执行两次 `--prepare` 才能在备份数据库上使用此功能。

在备份数据库上运行的结果可能如下所示：

```
<INDEX STATISTICS>
  table: test/table1, index: PRIMARY, space id: 12, root page 3
  estimated statistics in dictionary:
    key vals: 25265338, leaf pages 497839, size pages 498304
  real statistics:
     level 2 pages: pages=1, data=5395 bytes, data/pages=32%
     level 1 pages: pages=415, data=6471907 bytes, data/pages=95%
        leaf pages: recs=25958413, pages=497839, data=7492026403 bytes, data/pages=91%
```

这可以解释如下：

* 第一行仅显示了表格和索引名称及其内部标识符。如果您看到名为 `GEN_CLUST_INDEX` 的索引，即表的聚簇索引，它是自动创建的，因为您没有明确创建 `PRIMARY KEY` 。
* 字典信息中的估计统计信息类似于通过 InnoDB 内部的 `ANALYZE TABLE` 收集的数据，这些数据被存储为估计的基数统计信息，并传递给查询优化器。
* 真正的统计信息是扫描数据页和计算有关索引的确切信息的结果。
* `The level <X> pages` ：该输出表示该行对应索引树中的那个级别的页面的信息。越大的 `<X>` 离叶子页越远，它们是叶子页 0 级。第一行是根页面。
* `leaf pages` 输出显示叶子页的信息。这是表格数据的存储位置。
* `external pages` ：输出（上面未显示）显示保存那些其值太长而不适合存储于行本身的大型外部页面，如 `BLOB` 和 `TEXT` 类型值。
* `recs` 是叶子页中记录（行）的实际数量。
* `pages` 是页数。
* `data` 是页面中数据的总大小（以字节为单位）。
* `data/pages` 计算方法为 (`data` / (`pages` * `PAGE_SIZE`)) * 100%。由于为页眉和页脚预留了空间，所以它永远不会达到100％。

更详细的示例将作为 MySQL 性能博客文章[发布](https://www.percona.com/blog/2009/09/14/statistics-of-innodb-tables-and-indexes-available-in-xtrabackup/)。

### 格式化输出脚本

以下脚本可用于汇总和制表统计信息的输出：

```
tabulate-xtrabackup-stats.pl

#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';
my $script_version = "0.1";

my $PG_SIZE = 16_384; # InnoDB defaults to 16k pages, change if needed.
my ($cur_idx, $cur_tbl);
my (%idx_stats, %tbl_stats);
my ($max_tbl_len, $max_idx_len) = (0, 0);
while ( my $line = <> ) {
   if ( my ($t, $i) = $line =~ m/table: (.*), index: (.*), space id:/ ) {
      $t =~ s!/!.!;
      $cur_tbl = $t;
      $cur_idx = $i;
      if ( length($i) > $max_idx_len ) {
         $max_idx_len = length($i);
      }
      if ( length($t) > $max_tbl_len ) {
         $max_tbl_len = length($t);
      }
   }
   elsif ( my ($kv, $lp, $sp) = $line =~ m/key vals: (\d+), \D*(\d+), \D*(\d+)/ ) {
      @{$idx_stats{$cur_tbl}->{$cur_idx}}{qw(est_kv est_lp est_sp)} = ($kv, $lp, $sp);
      $tbl_stats{$cur_tbl}->{est_kv} += $kv;
      $tbl_stats{$cur_tbl}->{est_lp} += $lp;
      $tbl_stats{$cur_tbl}->{est_sp} += $sp;
   }
   elsif ( my ($l, $pages, $bytes) = $line =~ m/(?:level (\d+)|leaf) pages:.*pages=(\d+), data=(\d+) bytes/ ) {
      $l ||= 0;
      $idx_stats{$cur_tbl}->{$cur_idx}->{real_pages} += $pages;
      $idx_stats{$cur_tbl}->{$cur_idx}->{real_bytes} += $bytes;
      $tbl_stats{$cur_tbl}->{real_pages} += $pages;
      $tbl_stats{$cur_tbl}->{real_bytes} += $bytes;
   }
}

my $hdr_fmt = "%${max_tbl_len}s %${max_idx_len}s %9s %10s %10s\n";
my @headers = qw(TABLE INDEX TOT_PAGES FREE_PAGES PCT_FULL);
printf $hdr_fmt, @headers;

my $row_fmt = "%${max_tbl_len}s %${max_idx_len}s %9d %10d %9.1f%%\n";
foreach my $t ( sort keys %tbl_stats ) {
   my $tbl = $tbl_stats{$t};
   printf $row_fmt, $t, "", $tbl->{est_sp}, $tbl->{est_sp} - $tbl->{real_pages},
      $tbl->{real_bytes} / ($tbl->{real_pages} * $PG_SIZE) * 100;
   foreach my $i ( sort keys %{$idx_stats{$t}} ) {
      my $idx = $idx_stats{$t}->{$i};
      printf $row_fmt, $t, $i, $idx->{est_sp}, $idx->{est_sp} - $idx->{real_pages},
         $idx->{real_bytes} / ($idx->{real_pages} * $PG_SIZE) * 100;
   }
}
```

### 示例脚本输出

以上提到的博客文章中显示的示例运行上述 Perl 脚本时，输出将显示如下：

```
          TABLE           INDEX TOT_PAGES FREE_PAGES   PCT_FULL
art.link_out104                    832383      38561      86.8%
art.link_out104         PRIMARY    498304         49      91.9%
art.link_out104       domain_id     49600       6230      76.9%
art.link_out104     domain_id_2     26495       3339      89.1%
art.link_out104 from_message_id     28160        142      96.3%
art.link_out104    from_site_id     38848       4874      79.4%
art.link_out104   revert_domain    153984      19276      71.4%
art.link_out104    site_message     36992       4651      83.4%
```

这些列分别是表、索引、该索引中的页面总数，数据未实际占用的页数，以实际数据页面总大小的百分比表示的实际数据的字节数。 上述输出中的第一行（ `INDEX` 列为空）是整个表格的摘要。

## 使用二进制日志

**xtrabackup** 程序集成了 *InnoDB* 在其事务日志中存储的有关已提交事务的相应二进制日志位置的信息。 这使它能够打印出备份文件对应的二进制日志位置，因此您可以使用它来设置新的从库复制或执行时间点恢复。

### 查找二进制日志位置

准备备份完成后，您可以找到该备份对应的二进制日志位置。 **xtrabackup** 可以使用 `--prepare` , **innobackupex** 可以使用 `--apply-log` 。 如果备份文件来自于启用了二进制日志记录的服务器，则 **xtrabackup** 将在目标目录中创建一个名为 `xtrabackup_binlog_info` 的文件。 该文件包含二进制日志文件的名称和准备备份所对应的二进制日志中确切点的位置。

在准备阶段，您还会看到类似于以下内容的输出：

```
InnoDB: Last MySQL binlog file position 0 3252710, file name ./mysql-bin.000001
... snip ...
[notice (again)]
  If you use binary log and don't use any hack of group commit,
  the binary log position seems to be:
InnoDB: Last MySQL binlog file position 0 3252710, file name ./mysql-bin.000001
```

该输出也可以在 `xtrabackup_binlog_pos_innodb` 文件中找到，但只有在使用 *XtraDB* 或 *InnoDB* 作为存储引擎时才是正确的。

如果使用其他存储引擎（例如 *MyISAM*），则应使用 `xtrabackup_binlog_info` 文件检索位置。

上文提到的 `hack of group commit` 是指在 *Percona* 服务器中早期实现的模拟组提交。

### 时间点恢复

要从 `xtrabackup` 备份执行时间点恢复，应首先准备并恢复备份，然后从 `xtrabackup_binlog_info` 文件中显示的二进制日志位置重演二进制日志。

更详细的过程参见[这里](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/pit_recovery_ibk.html)（使用 **innobackupex**）。

### 设置新的从库复制

要设置新的从库复制，您应该首先准备好备份，并将其恢复到新从库复制的数据目录中。然后在您的 `CHANGE MASTER TO` 命令中，使用 `xtrabackup_binlog_info` 文件中显示的二进制日志文件名和位置开始复制。

更详细的步骤在[如何通过 Percona XtraBackup 轻松 6 步设置复制从库](https://www.percona.com/doc/percona-xtrabackup/LATEST/howtos/setting_up_replication.html)。

## 恢复单表

在 5.6 之前的服务器版本中，即使使用 `innodb_file_per_table`，也不可能通过复制文件在服务器之间复制表。 但是，使用 *Percona XtraBackup*，您可以从任何 *InnoDB* 数据库中导出单张表，然后将它们导入到 *Percona Server* 的 *XtraDB* 或 *MySQL 5.6* 中。 （导出库不一定是 *XtraDB* 或 *MySQL 5.6* ，但是导入库必须是。）这只适用于单独的 `.ibd` 文件，并且不能导出不包含在其自己的 `.ibd` 文件中的表。

我们来看看如何导出和导入下表：

```
CREATE TABLE export_test (
  a int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

{% hint style='info' %}
注意
如果您运行的是早于 `5.5.10-20.1` 的 *Percona Server* 版本，则应该使用变量 `innodb_expand_import` 而不是 `innodb_import_table_from_xtrabackup` 。
{% endhint %}

### 导出表格
这张表应该是在 `innodb_file_per_table` 模式下创建的，所以像 `xtrabackup --backup` 一样进行备份后，`.ibd` 文件应该存在于目标目录中：

```
$ find /data/backups/mysql/ -name export_test.*
/data/backups/mysql/test/export_test.ibd
```

准备备份时，将额外的参数 `xtrabackup --export` 添加到命令中。 这里是一个例子：

```
$ xtrabackup --prepare --export --target-dir=/data/backups/mysql/
```

{% hint style='info' %}
注意
如果你想恢复[加密的 InnoDB 表空间表](https://www.percona.com/doc/percona-xtrabackup/LATEST/advanced/encrypted_innodb_tablespace_backups.html#encrypted-innodb-tablespace-backups)，你还需要指定密钥文件：

```
xtrabackup --prepare --export --target-dir=/tmp/table \
--keyring-file-data=/var/lib/mysql-keyring/keyring
```
{% endhint %}

现在你应该在目标目录中看到一个 `.exp` 文件：

```
$ find /data/backups/mysql/ -name export_test.*
/data/backups/mysql/test/export_test.exp
/data/backups/mysql/test/export_test.ibd
/data/backups/mysql/test/export_test.cfg
```

将表导入运行 *XtraDB* 的 *Percona服务器* 或 *MySQL 5.7* 需要以上三个文件。在服务器使用 [`InnoDB Tablespace Encryption`](https://dev.mysql.com/doc/refman/5.7/en/innodb-tablespace-encryption.html) 的情况下，将会列出用于加密表的附加的 `.cfg` 文件。

{% hint style='info' %}
注意
*MySQL* 使用包含特殊格式的 *InnoDB* 字典转储的 `.cfg` 文件。这种格式与 *XtraDB* 中用于相同目的的 `.exp` 不同。严格地说，将表空间导入到 *MySQL 5.7* 或 *Percona Server 5.7* 中时不需要 `.cfg` 文件。即使是来自其他数据库版本的表空间也会成功导入，但如果相应的 `.cfg` 文件存在于同一目录中，*InnoDB* 将执行模式验证。
{% endhint %}

### 导入表格

在使用 `XtraDB` 引擎和启用 `innodb_import_table_from_xtrabackup` 选项的 *Percona Server* 的目标服务器，或者 *MySQL 5.6* 上，创建一张具有相同结构的表，然后执行以下步骤：

* 执行 `ALTER TABLE test.export_test DISCARD TABLESPACE`;
    如果看到以下消息，则必须启用 `innodb_file_per_table` 并再次创建该表：`ERROR 1030 (HY000): Got error -1 from storage engine`
* 将导出的文件复制到目标服务器的数据目录的 `test/` 子目录
* 执行 `ALTER TABLE test.export_test IMPORT TABLESPACE`;

该表现在应该已被导入，你应该能够从中执行 `SELECT` 语句并查看导入的数据。

{% hint style='info' %}
注意
除非在导入的表上运行 `ANALYZE TABLE` ，导入的表空间的持久性统计信息将为空。因为它们存储在系统表 `mysql.innodb_table_stats` 和 `mysql.innodb_index_stats` 中，并且在导入期间它们不会被服务器更新，所以它们是空的。这是由于上游错误 [#72368](https://bugs.mysql.com/bug.php?id=72368)。
{% endhint %}

## LRU 转储备份

此功能通过在数据库重启后从 `ib_lru_dump` 文件恢复缓冲池状态来减少预热时间。 *Percona XtraBackup* 会查找 `ib_lru_dump` 并自动备份它。

![](http://oio4l6s3j.bkt.clouddn.com/image/20180416/lru_dump.png)

如果在 `my.cnf` 文件中设置了启用缓冲区恢复选项，则备份恢复后缓冲池将处于预热好的状态。 要启用它，请在 Percona Server 5.5 中设置变量 `innodb_buffer_pool_restore_at_startup =1` 或在 Percona Server 5.1 中设置 `innodb_auto_lru_dump =1`。

# 实现

## 实现细节

该页面包含 **xtrabackup** 工具操作的内部实现原理的解释。

### 文件权限

**xtrabackup** 以读写模式打开源数据文件，虽然它并不修改源文件。这意味着您必须以拥有写入权限的用户运行 **xtrabackup** 。以读写模式打开文件的原因是 **xtrabackup** 使用嵌入式 *InnoDB* 库打开和读取文件， *InnoDB* 以读写模式打开它们，因为它通常假定它将写入数据。

### 调整 OS 缓冲区

由于 **xtrabackup** 从文件系统读取大量数据，因此它会尽可能使用 `posix_fadvise()`，以指示操作系统不要尝试缓存从磁盘读取的块。如果没有这个提示，操作系统会假设 **xtrabackup** 很可能需要它们而缓存块，但事实并非如此。缓存这些大文件会给操作系统的虚拟内存造成压力，并导致其他进程（如数据库）被换出。 **xtrabackup** 工具在源文件和目标文件上使用以下提示来避免这种情况：

```
posix_fadvise(file, 0, 0, POSIX_FADV_DONTNEED)
```

另外， **xtrabackup** 要求操作系统对源文件执行更积极的预读优化：

```
posix_fadvise(file, 0, 0, POSIX_FADV_SEQUENTIAL)
```

### 复制数据文件

将数据文件复制到目标目录时， **xtrabackup** 一次读取和写入 1MB 数据。这是不可配置的。在复制日志文件时， **xtrabackup** 每次读取和写入 512 个字节。这也无法配置，并且与 InnoDB 的复制行为相匹配（*Percona Server* 中存在解决方法，因为它可以选择调整 *XtraDB* 的 `innodb_log_block_size`，在这种情况下，*Percona XtraBackup* 将相应调整）。

从文件中读取数据后， **xtrabackup** 每次迭代 1MB 缓冲页，并使用 InnoDB 的 `buf_page_is_corrupted()` 函数检查每个页面上的页面损坏情况。如果页面已损坏，则每页重新读取并重试 10 次。它跳过对双写缓冲区的检查。

## xtrabackup 退出码

在没有错误发生时， **xtrabackup** 将在备份完成后以成功值 0 退出。 如果在备份过程中发生错误，则退出值为 1 。

在某些情况下，由于 *MySQL* 库包含的命令行选项代码，退出值可能不是 0 或 1 。 例如，未知的命令行选项会导致退出代码为 255。

# 参考

## xtrabackup 选项参考