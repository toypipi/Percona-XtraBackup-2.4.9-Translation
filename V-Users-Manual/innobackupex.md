# 制作本地完整备份（创建，准备和还原）

## 创建备份
这是最简单的用例。 它将所有的 *MySQL* 数据复制到指定的目录中。 以下是如何备份 `my.cnf` 中指定的 `datadir` 中的所有数据库。 它会将备份置于 `/data/backups/` 由时间标记的子目录中，在本例中为 `/data/backups/2010-03-13_02-42-44`，
```
$ innobackupex /data/backups
```
备份过程中有很多输出，但你需要确保在备份结束时看到以下内容。 如果你没有看到这个输出，那么表示你的备份失败：
> 100313 02:43:07  innobackupex: completed OK!

## 准备备份
要准备备份，请使用 `--apply-log` 选项并指定备份的时间戳子目录。 为了加快应用日志的过程，我们推荐使用 `--use-memory` 选项：
```
$ innobackupex --use-memory=4G --apply-log /data/backups/2010-03-13_02-42-44/
```
您应该检查确认消息：
> 100313 02:51:02  innobackupex: completed OK!

现在，位于 `/data/backups/2010-03-13_02-42-44` 的文件已经准备好可以使用了。

## 还原备份
要还原刚才准备好的备份，首先需要停止数据库服务器，然后使用 **innobackupex** 函数 `--copy-back`
```
innobackupex --copy-back /data/backups/2010-03-13_02-42-44/
## Use chmod to correct the permissions, if necessary!
```
这将会把数据拷贝回你在 `my.cnf` 文件中定义的 `datadir` 目录中。

{% hint style='info' %}
注意
`datadir` 目录必须为空; 目录不为空时，*Percona XtraBackup* `innobackupex --copy-back` 选项不会复制现有文件，除非指定了 `innobackupex --force-non-empty-directories` 选项。 另外需要注意的是，在执行恢复之前需要关闭 *MySQL* 服务器。 您无法将备份还原到一个正在运行的 mysqld 实例的 `datadir` 目录（导入部分备份时除外）。
{% endhint %}

确认信息：
> 100313 02:58:44  innobackupex: completed OK!

您应该在复制数据后检查文件权限。 您可能需要使用类似的方法调整它们：
```
$ chown -R mysql:mysql /var/lib/mysql
```

现在 `datadir` 目录已经包含还原后的数据。 您可以启动服务器了。

# 制作流式备份

流式备份将备份以 tar 格式发送到 `STDOUT` ，而不是将其复制到第一个参数指定的目录中。 您可以将输出传送到 **gzip**，或通过网络传送到另一台服务器。

要提取生成的tar文件，**必须**使用 `-i` 选项，例如 `tar -ixvf backup.tar`。

{% hint style='danger' %}
警告
请记住使用 `-i` 选项来提取备份的 tar 文件。 有关更多信息，请参阅[流式备份和压缩备份](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/streaming_backups_innobackupex.html)。
{% endhint %}

以下是使用 `tar` 选项进行流式处理的一些示例：

* 将流式备份传输到名为 'backup.tar' 的 tar 归档文件中  
```
$ innobackupex --stream=tar ./ > backup.tar
```

* 同上，压缩备份文件
```
$ innobackupex --stream=tar ./ | gzip - > backup.tar.gz
```

* 加密备份文件
```
$ innobackupex --stream=tar . | gzip - | openssl des3 -salt -k "password" > backup.tar.gz.des3
```

* 将备份文件传输到另一台服务器而不是保存在本地
```
$ innobackupex --stream=tar ./ | ssh user@desthost "cat - > /data/backups/backup.tar"
```

* 用 "netcat" 实现上述功能
```
## 在目标主机上执行:
$ nc -l 9999 | cat - > /data/backups/backup.tar
## 在源主机上执行:
$ innobackupex --stream=tar ./ | nc desthost 9999
```

* 一条指令执行上述功能
```
$ ssh user@desthost "( nc -l 9999 > /data/backups/backup.tar & )" \
&& innobackupex --stream=tar ./  |  nc desthost 9999
```

* 将吞吐量限制为 10MB 每秒。 这需要 'pv' 工具; 你可以在[官方站点](http://www.ivarch.com/programs/quickref/pv.shtml)找到它或者从分发包安装它（`apt-get install pv`）
```
$ innobackupex --stream=tar ./ | pv -q -L10m \
| ssh user@desthost "cat - > /data/backups/backup.tar"
```

* 在流式备份过程中检查校验和
```
## On the destination host:
$ nc -l 9999 | tee >(sha1sum > destination_checksum) > /data/backups/backup.tar
## On the source host:
$ innobackupex --stream=tar ./ | tee >(sha1sum > source_checksum) | nc desthost 9999
## compare the checksums
## On the source host:
$ cat source_checksum
65e4f916a49c1f216e0887ce54cf59bf3934dbad  -
## On the destination host:
$ destination_checksum
65e4f916a49c1f216e0887ce54cf59bf3934dbad  -
```

使用 [xbstream](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-xbstream) 选项进行流式处理的示例：

* 将流式备份传输到名为 'backup.tar' 的 tar 归档文件中  
```
innobackupex --stream=xbstream ./ > backup.xbstream
```

* 同上，压缩备份文件
```
innobackupex --stream=xbstream --compress ./ > backup.xbstream
```

* 解压缩备份文件到当前目录
```
xbstream -x <  backup.xbstream
```

* 压缩备份文件然后直接传输到另一台服务器并解压缩
```
innobackupex --compress --stream=xbstream ./ | ssh user@otherhost "xbstream -x"
```

* 并行复制备份的并行压缩
```
innobackupex --compress --compress-threads=8 --stream=xbstream --parallel=4 ./ > backup.xbstream
```

# 制作增量备份

每次增量备份都需要从一个完整备份开始，我们将之称为 *基础备份*。
```
innobackupex --user=USER --password=PASSWORD /path/to/backup/dir/
```

请注意，完整备份将位于 `/path/to/backup/dir/`（例如 `/path/to/backup/dir/2011-12-24_23-01-00/` 中）的时间戳子目录中。

假设变量 `$FULLBACKUP` 表示 `/path/to/backup/dir/2011-5-23_23-01-18` ，让我们在一小时后执行增量备份：
```
innobackupex --incremental /path/to/inc/dir \
  --incremental-basedir=$FULLBACKUP --user=USER --password=PASSWORD
```

现在，增量备份将位于 `/path/to/inc/dir/2011-12-25_00-01-00/` 中。我们将之称为 `$INCREMENTALBACKUP=2011-5-23_23-50-10`。

准备增量备份与准备完整备份稍稍有些不同。

首先，你必须在每个备份上重放已提交的事务，
```
innobackupex --apply-log --redo-only $FULLBACKUP \
 --use-memory=1G --user=USER --password=PASSWORD
```

201/5000
`--use-memory` 选项不是必需的，（如果提供的 RAM 数量可用）的话使用它可以加速进程。

如果一切正常，您应该看到类似于以下的输出：
> 111225 01:10:12 InnoDB: Shutdown completed; log sequence number 91514213

现在，通过发出以下命令将增量备份应用于基础备份：
```
innobackupex --apply-log --redo-only $FULLBACKUP
 --incremental-dir=$INCREMENTALBACKUP
 --use-memory=1G --user=DVADER --password=D4RKS1D3
```
注意 `$INCREMENTALBACKUP` 的使用。

*最终数据将位于基础备份目录中*，而不是增量备份目录中。在本例中，位于 `/path/to/backup/dir/2011-12-24_23-01-00` 或 `$FULLBACKUP` 中。

如果您想要应用更多增量备份，请重复此步骤。 按照备份完成的时间顺序执行此操作非常重要。

您可以检查每个备份目录下的文件 xtrabackup_checkpoints。

它们应该如下所示:(在基础备份中）
> backup_type = full-backuped  
> from_lsn = 0  
> to_lsn = 1291135  

在增量备份中：
> backup_type = incremental  
> from_lsn = 1291135  
> to_lsn = 1291340  

其中，`to_lsn` 编号一定与下一次备份的 `from_lsn` 一致。

将所有备份放在一起后，您可以再次准备完整备份（此时的完整备份是基础备份和增量备份的集合）以回滚未提交的事务：
```
innobackupex-1.5.1 --apply-log $FULLBACKUP --use-memory=1G \
  --user=$USERNAME --password=$PASSWORD
```

现在，您的备份可以在恢复后立即使用。 这个准备步骤是可选的，如果你没有执行这个步骤，数据库服务器将假设数据库发生了崩溃，并开始回滚未提交的事务（这将会产生一些本可以避免的停机时间）。

# 制作压缩备份

为了制作一个压缩备份，您需要使用 `--compress` 选项。
```
$ innobackupex --compress /data/backup
```

如果您想加速压缩，那么可以使用并行压缩，可以使用 `--compress-threads=#` 选项启用并行压缩。 以下示例将使用四个线程进行压缩：
```
$ innobackupex --compress --compress-threads=4 /data/backup
```

输出类似于以下内容
> ...                                                              
> [01] Compressing ./imdb/comp_cast_type.ibd to /data/backup/2013-08-01_11-24-04/./imdb/comp_cast_type.ibd.qp          
> [01]        ...done                                                                                          
> [01] Compressing ./imdb/aka_name.ibd to /data/backup/2013-08-01_11-24-04/./imdb/aka_name.ibd.qp         
> [01]        ...done                                                                    
> ...                                 
> 130801 11:50:24  innobackupex: completed OK              

## 准备备份
在准备备份之前，您需要使用 [qpress](http://www.quicklz.com/) 解压缩所有文件（可从 [Percona 软件库](https://www.percona.com/doc/percona-xtrabackup/2.1/installation.html#using-percona-software-repositories))中获取）。 您可以使用以下内容来解压缩所有文件：
```
$ for bf in `find . -iname "*\.qp"`; do qpress -d $bf $(dirname $bf) && rm $bf; done
```

在 *Percona XtraBackup* 2.1.4 中新增了 `innobackupex --decompress` 选项，可用于解压备份：
```
$ innobackupex --decompress /data/backup/2013-08-01_11-24-04/
```

{% hint style='info' %}
注意
为了成功使用 `innobackupex --decompress` 选项，需要安装 qpress 二进制文件并包含在路径当中。 `innobackupex --parallel` 可以和 `innobackupex --decompress` 选项一起使用同时解压缩多个文件。
{% endhint %}

一旦文件解压完成后，您可以使用 `--apply-log` 准备备份。
```
$ innobackupex --apply-log /data/backup/2013-08-01_11-24-04/
```

您需要检查备份完成信息：
```
130802 02:51:02  innobackupex: completed OK!
```

现在，位于 `/data/backups/2013-08-01_11-24-0` 目录中的文件已经可以被数据库使用了。

{% hint style='info' %}
注意
*Percona XtraBackup* 不会自动删除压缩文件。为了清理备份目录，用户需要手动删除 `*.qp` 文件。
{% endhint %}

## 还原备份
一旦备份准备完成，您可以使用 `--copy-back` 选项还原备份。
```
$ innobackupex --copy-back /data/backups/2013-08-01_11-24-04/
```

这将会把准备好的数据拷贝回您在 `my.cnf` 文件中定义的 `datadir` 目录中。

还原后输出确认信息。
> 130802 02:58:44  innobackupex: completed OK!

您需要在文件拷贝回数据目录中后检查文件的所有权。您可能需要一下命令调整文件的所有权。
```
$ chown -R mysql:mysql /var/lib/mysql
```

现在 `datadir` 目录中包含还原后的数据。您可以启动数据库了。

# 备份和还原单个分区

*Percona XtraBackup* 具有[部分备份](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/partial_backups_innobackupex.html)功能，这意味着您可以备份单个分区，因为从存储引擎的角度来看，分区是具有特殊格式名称的常规表。此功能的唯一要求是在服务器中启用 `innodb_file_per_table` 选项。

使用这种备份只有一个警告：您无法将准备好的备份复制回数据目录。应通过导入表来恢复部分备份，而不是使用传统的 `--copy-back` 选项。尽管在某些情况下可以通过复制文件来完成恢复，但这在很多情况下可能会导致数据库不一致，因此建议不要这样做。

## 创建备份
有三种方式可以指定备份整个数据的哪一部分：正则表达式（`--include`），在文件中枚举需要备份的表（`--tables-file`）或提供数据库列表（`--databases`）。 在这个例子中 `--include` 选项将被使用。

提供给此选项的正则表达式将与表的完全限定表名（包括数据库名）匹配，以 `databasename.tablename` 的形式。

例如，以下命令将从数据库 `imdb` 中备份 `name` 表的 `p4` 分区：
```
$ innobackupex --include='^imdb[.]name#P#p4' /mnt/backup/
```

这将创建一个包含 `innobackupex` 创建的常用文件的时间戳目录，但只有与表相关的数据文件才会被创建。

innobackupex的输出将列出跳过的表格

> ...
> [01] Skipping ./imdb/person_info.ibd
> [01] Skipping ./imdb/name#P#p5.ibd
> [01] Skipping ./imdb/name#P#p6.ibd
> ...
> imdb.person_info.frm is skipped because it does not match ^imdb[.]name#P#p4.
> imdb.title.frm is skipped because it does not match ^imdb[.]name#P#p4.
> imdb.company_type.frm is skipped because it does not match ^imdb[.]name#P#p4.
> ...

请注意，此选项将传递给 `xtrabackup --tables`，并且与每个数据库的每张表匹配，即使没有匹配到，每个数据库的目录也将被创建。

## 准备备份
准备部分备份，该过程类似于[恢复单个表](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/restoring_individual_tables_ibk.html)：应用日志并使用 `--export` 选项：
```
$ innobackupex --apply-log --export /mnt/backup/2012-08-28_10-29-09
```

您可能会在输出中看到关于表不存在的的警告。 这是因为基于 *InnoDB* 的引擎将其数据字典存储在除 `.frm` 文件之外的表空间文件中。 **innobackupex** 将使用 **xtrabackup** 从数据字典中删除缺少的表（指那些未在部分备份中选择的表），以避免将来出现警告或错误：

> InnoDB: in InnoDB data dictionary has tablespace id 51,
> InnoDB: but tablespace with that id or name does not exist. It will be removed from data dictionary.
> 120828 10:25:28  InnoDB: Waiting for the background threads to start
> 120828 10:25:29 Percona XtraDB (http://www.percona.com) 1.1.8-20.1 started; log sequence number 10098323731
> xtrabackup: export option is specified.
> xtrabackup: export metadata of table 'imdb/name#P#p4' to file `./imdb/name#P#p4.exp` (1 indexes)
> xtrabackup:     name=PRIMARY, id.low=73, page=3

您还应该看到为部分备份中包含的每张表导入（`.exp` 文件）所需的创建文件的通知：
> xtrabackup: export option is specified.
> xtrabackup: export metadata of table 'imdb/name#P#p4' to file `./imdb/name#P#p4.exp` (1 indexes)
> xtrabackup:     name=PRIMARY, id.low=73, page=3

请注意，您可以将 `--export` 选项和 `--apply-log` 用于已经准备好的备份，以创建 `.exp` 文件。

最后，检查输出中的确认消息：
> 120828 19:25:38  innobackupex: completed OK!

## 从备份文件还原表
应通过将部分备份中的表[导入服务器](https://www.percona.com/doc/percona-xtrabackup/LATEST/innobackupex/restoring_individual_tables_ibk.html)来完成还原。

{% hint style='info' %}
注意
改进的表/分区导入仅在 *Percona Server* 和 *MySQL 5.6* 中可用，这意味着也可以导入从不同服务器备份的分区。 对于早于 *MySQL 5.6* 的版本，只能从该服务器导入分区，并且还有另外一些重要限制。 在进行备份和导入分区之间不应该发出 DROP/CREATE/TRUNCATE/ALTER TABLE 命令。
{% endhint %}


第一步是创建要恢复数据的新表
```
mysql> CREATE TABLE `name_p4` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`name` text NOT NULL,
`imdb_index` varchar(12) DEFAULT NULL,
`imdb_id` int(11) DEFAULT NULL,
`name_pcode_cf` varchar(5) DEFAULT NULL,
`name_pcode_nf` varchar(5) DEFAULT NULL,
`surname_pcode` varchar(5) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2812744 DEFAULT CHARSET=utf8
```

要从备份文件中还原数据，需要丢弃该表的表空间：
```
mysql>  ALTER TABLE name_p4 DISCARD TABLESPACE;
```

接下来需要将 `.exp` 和 *ibd* 文件从备份目录中拷贝到 *MySQL* 数据目录中：
```
$ cp /mnt/backup/2012-08-28_10-29-09/imdb/name#P#p4.exp /var/lib/mysql/imdb/name_p4.exp
$ cp /mnt/backup/2012-08-28_10-29-09/imdb/name#P#p4.ibd /var/lib/mysql/imdb/name_p4.ibd
```

{% hint style='info' %}
注意
确保复制的文件可以被运行 *MySQL* 数据库的用户读取。
{% endhint %}

如果您运行的是 `Percona Server`,请确保 *innodb_import_table_from_xtrabackup* 变量已经启用。
```
SET GLOBAL innodb_import_table_from_xtrabackup=1;
```

最后一步是导入表空间：
```
mysql>  ALTER TABLE name_p4 IMPORT TABLESPACE;
```

## 在版本 5.6 中还原备份
服务器版本高于 5.5 的问题在于没有服务器支持导入单个分区或分区表的所有分区，因此分区只能作为独立表导入。 在 *MySQL* 和 *Percona Server*  5.6 中，可以使用 `ALTER TABLE ... EXCHANGE PARTITION` 命令将单个表变为单个分区。

{% hint style='info' %}
注意
在 *Percona Server* 5.6 中，变量 `innodb_import_table_from_xtrabackup` 已被删除，以支持 *MySQL* [可传输表空间](https://dev.mysql.com/doc/refman/5.6/en/tablespace-copying.html)实现。
{% endhint %}

导入整个分区表时，首先将所有（子）分区导入为独立表：
```
mysql> CREATE TABLE `name_p4` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`name` text NOT NULL,
`imdb_index` varchar(12) DEFAULT NULL,
`imdb_id` int(11) DEFAULT NULL,
`name_pcode_cf` varchar(5) DEFAULT NULL,
`name_pcode_nf` varchar(5) DEFAULT NULL,
`surname_pcode` varchar(5) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2812744 DEFAULT CHARSET=utf8
```

要从备份文件中还原数据，需要丢弃该表的表空间：
```
mysql>  ALTER TABLE name_p4 DISCARD TABLESPACE;
```

接下来需要将 `.cfg` 和 *ibd* 文件从备份目录中拷贝到 *MySQL* 数据目录中：
```
$ cp /mnt/backup/2013-07-18_10-29-09/imdb/name#P#p4.cfg /var/lib/mysql/imdb/name_p4.cfg
$ cp /mnt/backup/2013-07-18_10-29-09/imdb/name#P#p4.ibd /var/lib/mysql/imdb/name_p4.ibd
```

最后一步是导入表空间：
```
mysql>  ALTER TABLE name_p4 IMPORT TABLESPACE;
```

现在我们可以使用与导入的表完全相同的模式创建空分区表：
```
mysql> CREATE TABLE name2 LIKE name;
```

然后将新创建的表中的空分区与对应于先前步骤中已导出/导入的分区的单独表交换
```
mysql> ALTER TABLE name2 EXCHANGE PARTITION p4 WITH TABLE name_p4;
```

必须满足[这些条件](https://dev.mysql.com/doc/refman/5.6/en/partitioning-management-exchange.html)才能使此操作成功。

