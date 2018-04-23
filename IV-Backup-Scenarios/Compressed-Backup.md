Percona XtraBackup 实现了对压缩备份的支持。它可以使用 [xbstream](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-xbstream) 压缩、解压缩本地备份或流式备份。

# 创建压缩备份

为了进行压缩备份，您需要使用 `xtrabackup --compress` 选项：

```
$ xtrabackup --backup --compress --target-dir=/data/compressed/
```

如果您想加快压缩速度，可以使用并行压缩，可以使用 `xtrabackup --compress-threads` 选项启用并行压缩。以下示例将使用四个线程进行压缩：

```
$ xtrabackup --backup --compress --compress-threads=4 \
--target-dir=/data/compressed/
```

输出看起来像这样

```
...
170223 13:00:38 [01] Compressing ./test/sbtest1.frm to /tmp/compressed/test/sbtest1.frm.qp
170223 13:00:38 [01]        ...done
170223 13:00:38 [01] Compressing ./test/sbtest2.frm to /tmp/compressed/test/sbtest2.frm.qp
170223 13:00:38 [01]        ...done
...
170223 13:00:39 [00] Compressing xtrabackup_info
170223 13:00:39 [00]        ...done
xtrabackup: Transaction log of lsn (9291934) to (9291934) was copied.
170223 13:00:39 completed OK!
```

# 准备备份

在准备备份之前，您需要解压所有文件。 Percona XtraBackup 提供 `xtrabackup --decompress` 选项，可用于解压备份。

{% hint style='info' %}
注意
在继续操作之前，您需要确保已安装 [qpress](http://www.quicklz.com/)。它可以从 [Percona软件仓库](https://www.percona.com/doc/percona-xtrabackup/LATEST/installation.html#installing-from-binaries)获得
{% endhint %}

```
$ xtrabackup --decompress --target-dir=/data/compressed/
```

{% hint style='info' %}
注意
`xtrabackup --parallel` 可与 `xtrabackup --decompress` 选项一起使用,可同时解压缩多个文件。
{% endhint %}

Percona XtraBackup 不会自动删除压缩文件。你可以使用 `xtrabackup --remove-original` 选项清理备份目录。即使它们未被删除，如果使用了 `xtrabackup --copy-back` 或 `xtrabackup --move-back`，这些文件也不会被复制或移动到  `datadir`。

当文件解压后，可以使用 `xtrabackup --prepare` 选项准备备份：

```
$ xtrabackup --prepare --target-dir=/data/compressed/
```

你应该检查确认信息：

```
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 9293846
170223 13:39:31 completed OK!
```

现在 `/data/compressed/` 中的文件已准备好供服务器使用了。

# 恢复备份

**xtrabackup** 有一个 `xtrabackup --copy-back` 选项，它执行将备份还原到服务器的 `datadir` 目录中：

```
$ xtrabackup --copy-back --target-dir=/data/backups/
```

它会将所有与数据相关的文件复制回服务器的 `datadir` 目录中，`datadir` 的位置由服务器的 `my.cnf` 配置文件决定。您应该检查输出的最后一行以获取成功消息：

```
170223 13:49:13 completed OK!
```

复制数据后应检查文件权限。你可能需要用类似的方法来调整它们：

```
$ chown -R mysql:mysql /var/lib/mysql
```

现在 [datadir](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-datadir) 包含还原的数据。您现在可以启动服务器了。