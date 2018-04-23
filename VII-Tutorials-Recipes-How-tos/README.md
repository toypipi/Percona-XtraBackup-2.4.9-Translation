# [**innobackupex** 的用法](Recipes/Recipes-for-innobackupex.md)

* 制作本地完整备份（创建，准备和还原）
* 制作流式备份
* 制作增量备份
* 制作压缩备份
* 备份和还原单个分区

# [**xtrabackup** 的用法](Recipes/Recipes-for-xtrabackup.md)

* 制作完整备份
* 制作增量备份
* 还原备份

# [入门指南](Recipes/How-Tos.md)

* 如何使用 Percona XtraBackup 轻松 6 步设置用于复制的从库
* 使用复制和 pt-checksum 验证备份
* 如何创建一个新的（或修复一个坏的）基于 GTID 的从库

# [辅助指南](Recipes/Auxiliary-Guides.md)

* 允许服务器通过 TCP/IP 进行通信
* 用户的特权和权限
* 安装和配置 SSH 服务器

# 本节中的假设

大多数情况下，上下文将使用法或教程更容易理解。为了保证以上内容，此处将给出本节中出现的假设，名称以及“其他事物”的列表。他们将在每个用法或教程的开头指定，以使它能被更快读取和更易实用。

`HOST`
基于 MySQL 安装，配置和运行的服务器系统。我们将假设关于这个系统的以下内容：

* MySQL 服务器能够[通过标准的 TCP/IP 端口与其他应用进行通信](https://www.percona.com/doc/percona-xtrabackup/LATEST/howtos/enabling_tcp.html);
* 安装并配置了 SSH 服务 - 如果不是，请参阅[此处](https://www.percona.com/doc/percona-xtrabackup/LATEST/howtos/ssh_server.html);
* 您在系统中拥有适当[权限](https://www.percona.com/doc/percona-xtrabackup/LATEST/howtos/permissions.html)的用户帐户
* 你有一个 MySQL 用户帐户，该账户具有适当的 [Connection 和 Privileges 权限](https://www.percona.com/doc/percona-xtrabackup/LATEST/using_xtrabackup/privileges.html#privileges)。

`USER`
    操作系统中具有 shell 访问权限和执行任务的适当权限的用户帐户。检查他们的指南在[这里](https://www.percona.com/doc/percona-xtrabackup/LATEST/howtos/permissions.html)。
    
`DB-USER`
    数据库服务器中具有适当任务权限的用户帐户。检查他们的指南在[这里](https://www.percona
