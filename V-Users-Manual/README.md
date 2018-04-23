Percona XtraBackup 包含以下工具组：

[innobackupex](User-Manual/innobackupex.md)
**innobackupex** 是 **xtrabackup** 的符号链接。 **innobackupex** 仍支持 2.2 版本的所有功能和语法，但现在已被弃用，并将在下一个主要版本中删除。

[xtrabackup](User-Manual/xbcloud.md)
它是一个编译的 C 二进制文件，它提供了备份整个包含 MyISAM，InnoDB 和 XtraDB 表的 MySQL 数据库实例的功能。

[xbcrypt](User-Manual/xbcrypt.md)
用于加密和解密备份文件的实用程序。

[xbstream](User-Manual/xbstream.md)
用于从 [xbstream](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-xbstream) 格式流式存储和提取文件的实用程序。

[xbcloud](User-Manual/xtrabackup.md)
用于从云中下载或上传 [xbstream](https://www.percona.com/doc/percona-xtrabackup/LATEST/glossary.html#term-xbstream) 存档的全部或部分内容的实用程序。

Percona XtraBackup 2.3 发布后，建议使用 xtrabackup 脚本的方式备份。 有关脚本选项的更多信息可以在[如何使用 xtrabackup 中找到](https://www.percona.com/doc/percona-xtrabackup/LATEST/xtrabackup_bin/xtrabackup_binary.html)。