Percona XtraBackup 是全球唯一的开源免费MySQL热备软件，可为 InnoDB 和 XtraDB 数据库执行无锁定备份。借助 Percona XtraBackup，您可以获得以下好处：

* 完成快速可靠的备份
* 备份期间不间断的事务处理
* 节省磁盘空间和网络带宽
* 自动备份验证
* 由于更快的恢复时间从而使服务器正常运行时间更长

Percona XtraBackup 为所有版本的 Percona Server，MySQL 和 MariaDB 进行 MySQL 热备份。它可以执行流式，压缩和增量 MySQL 备份。

Percona XtraBackup 可与 MySQL，MariaDB 和 Percona Server 配合使用。它支持 InnoDB，XtraDB 和 HailDB 存储引擎的完全无锁定备份。此外，它还可以通过在备份结束时暂停写入来备份以下存储引擎：MyISAM，[Merge](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-mrg) 和 [Archive](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-arm)，包括分区表，触发器和数据库选项。

Percona 的企业级商业 [MySQL 支持](https://www.percona.com/services/support/mysql-support)合同包括对 Percona XtraBackup 的支持。我们建议对关键的生产部署提供支持。

# MySQL备份工具功能比较

| 功能 | Percona XtraBackup | MySQL Enterprise backup |
| ---|---| ---|
| 许可 | GPL | 专有 |
| 价格 | 免费 | 订阅费用为5000美元/服务器 |
| 流和加密格式 | 开源 | 专有 |
| 支持的 MySQL 类型 | MySQL，Percona Server，MariaDB，Percona XtraDB 集群，MariaDB Galera 集群 | MySQL |
| 支持的操作系统 | Linux | Linux，Solaris，Windows，OSX，FreeBSD |
| 锁定 InnoDB 备份[1] | 是 | 是 |
| 锁定 MyISAM 备份 | 是 | 是 |
| 增量备份 | 是 | 是 |
| 完全压缩的备份 | 是 | 是 |
| 增量压缩备份 | 是 |  |
| 快速增量备份[2] | 是 | |
| Percona Server 的增量备份与归档日志功能 | 是 | |
| 仅限 REDO 日志的增量备份|  | 是 |
| 备份锁定[8] | 是 | |
| 加密备份 | 是 | 是[3]|
| 流式备份 | 是 | 是 |
| 并行本地备份 | 是 | 是 |
| 并行压缩 | 是 | 是 |
| 并行加密 | 是 | 是 |
| 并行使用日志 | 是 | |
| 并行复制 | | 是 |
| 部分备份 | 是 | 是 |
| 个别分区的部分备份 | 是 | |
| 节流[4] | 是 | 是 |
| 备份映像验证 | 是 | |
| 时间点恢复支持 | 是 | 是 |
| 安全的从库备份 | 是 |  |
| 紧凑备份[5] | 是 |　|
| 缓冲池状态备份 | 是 |  |
| 个别表导出 | 是 | 是[6] |
| 个别分区导出 | 是 |  |
| 将表恢复到不同的数据库[7] | 是 | 是 |
| 数据和索引文件统计 | 是 |  |
| InnoDB 二级索引碎片整理 | 是 | |
| rsync 支持最大限度地缩短锁定时间 | 是 | |
| 改进的 FTWRL 处理 | 是 | |
| 备份历史记录表 | 是 | 是 |
| 备份进程表 |  | 是 |
| 离线备份 | | 是 |
| 备份到磁带媒体管理器 |  | 是 |
| 云备份支持 | | Amazon S3 |
| 用于备份/恢复的外部图形用户界面 | Zmanda Recovery Manager for MySQL | MySQL Workbench，MySQL Enterprise Monitor |

# Percona XtraBackup 有哪些功能？

以下是 Percona XtraBackup 功能的简短列表。查看文档获取更多内容。

* 在不暂停数据库的情况下创建 InnoDB 热备份
* 进行 MySQL 的增量备份
* 将压缩的 MySQL 备份传输到另一台服务器
* 在 MySQL 服务器之间在线移动表格
* 轻松创建新的 MySQL 复制从库
* 在不增加服务器负载的情况下备份 MySQL


# 脚注
[1]在复制非 InnoDB 数据时，InnoDB 表仍处于锁定状态。   
[2]支持快速增量备份的 Percona Server 需启用 XtraDB 更改页面跟踪功能。  
[3] Percona XtraBackup 支持任何形式的加密备份。 MySQL Enterprise Backup 只支持单文件加密备份。  
[4] Percona XtraBackup 根据每秒IO操作的数量执行调节。 MySQL Enterprise Backup 支持操作之间的可配置休眠时间。  
[5] Percona XtraBackup 跳过二级索引页面，并在紧凑备份准备好时重新创建它们。 MySQL Enterprise Backup 跳过未使用的页面，并在准备阶段重新插入。  
[6]无论 InnoDB 版本如何，Percona XtraBackup 甚至可以从完整备份中导出单个表。 MySQL Enterprise Backup 仅在执行部分备份时使用 InnoDB 5.6 可传输表空间。  
[7]使用 Percona XtraBackup 导出的表可导入 Percona Server 5.1,5.5 或 5.6+ 或 MySQL 5.6+。使用 MySQL Enterprise Backup 创建的可移动表空间只能导入到 Percona Server 5.6+，MySQL 5.6+ 或 MariaDB 10.0+。  
[8]备份锁是 Percona Server 5.6+ 中的 `FLUSH TABLES WITH READ LOCK` 的轻量级替代品。 Percona XtraBackup 自动使用它们来复制非 InnoDB 数据，以避免锁定修改 InnoDB 表的 DML 查询。  
