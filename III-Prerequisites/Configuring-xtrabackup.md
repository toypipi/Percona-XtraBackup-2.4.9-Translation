所有 **xtrabackup** 配置都是通过选项完成的，这些选项的行为与标准 MySQL 程序选项完全相同：它们可以在命令行或通过诸如 `/etc/my.cnf` 的文件中指定。

**xtrabackup** 以 [mysqld] 和 [xtrabackup] 的顺序从任意配置文件中读取配置选项。因此它可以从现有的 MySQL 安装中读取已经配置的选项，例如 datadi r或一些 InnoDB 选项。如果您想要覆盖这些内容，只需在 [xtrabackup] 部分中指定它们，因为它稍后被读取，所以优先级更高。

如果你不想这么做，你也可以不用在 my.cnf 中添加任何配置。您可以简单地在命令行上指定选项。通常，您可能会发现在 my.cnf 文件中的 [xtrabackup] 使用 target_dir 选项来指定默认备份目录会很方便，例如：
```
[xtrabackup]
target_dir = /data/backups/mysql/
```

本手册假定您没有任何基于文件的 **xtrabackup** 配置，因此它将始终显示使用的命令行选项。有关所有配置选项的详细信息，请参阅[选项和变量参考](https://www.percona.com/doc/percona-xtrabackup/LATEST/xtrabackup_bin/xbk_option_reference.html#xbk-option-reference)。

**xtrabackup** 不支持 my.cnf 文件中与 mysqld 服务器二进制文件完全相同的语法。由于历史原因，mysqld 服务器程序允许带有 `--set-variable=<variable>=<value> ` 语法的参数，**xtrabackup** 不支持这种配置。如果你的 my.cnf 文件中有这样的配置指令，你应该用 `--variable=value` 语法重写它们。

# 系统配置和NFS卷

**xtrabackup** 工具在大多数系统上不需要特殊配置。但是，调用 `fsync()` 时，`xtrabackup --target-dir` 指定的存储必须正常运行。特别地，我们注意到，不使用 `sync` 选项挂载的 NFS 卷可能不会真正地同步数据。因此，如果您将数据备份到使用 async 选项安装的 NFS 卷，然后尝试从另一个也安装该卷的服务器准备备份，则数据可能看起来已损坏。您可以使用 `sync mount` 选项来避免此问题。