Percona XtraBackup 已经实现了对加密备份的支持。它可用于使用 `xbstream` 选项加密、解密本地备份或流式备份（不支持流式 tar 备份），以便为备份添加另一层保护。加密是通过 `libgcrypt` 库完成的。

#创建加密备份

要创建一个加密备份，需要指定以下选项（选项 `xtrabackup --encrypt-key` 和 `xtrabackup --encrypt-key-file` 是互斥的，即只需要提供其中的一个就行）：

* `--encrypt=ALGORITHM` - 目前支持的算法有：`AES128`，`AES192` 和 `AES256`
* `--encrypt-key=ENCRYPTION_KEY` - 使用适当长度的加密密钥。不建议在通过命令行访问机器的情况下使用此选项，因为该密钥在执行过程中会被打印输出。
* `--encrypt-key-file=KEYFILE` - 可从中读取适当长度的原始密钥的文件的名称。该文件必须是一个简单的二进制（或文本）文件，其中准确存储了需要使用的密钥。

`xtrabackup --encrypt-key` 选项和` xtrabackup --encrypt-key-file` 选项均可用于指定加密密钥。加密密钥可以使用如下命令生成：

```
$ openssl rand -base64 24
```

该命令的输出示例如下所示：

```
GCHFLrDFVx6UAsRb88uLVbAVWbK+Yzfs
```
该值稍后可以用作加密密钥

# 使用 `--encrypt-key` 选项

使用 `xtrabackup --encrypt-key` 的 xtrabackup 命令示例应如下所示：

```
$ xtrabackup --backup --target-dir=/data/backups --encrypt=AES256 \
--encrypt-key="GCHFLrDFVx6UAsRb88uLVbAVWbK+Yzfs"
```

# 使用 `--encrypt-key-file` 选项

使用 `xtrabackup --encrypt-key-file` 的 xtrabackup 命令示例应如下所示：

```
$ xtrabackup --backup --target-dir=/data/backups/ --encrypt=AES256 \
--encrypt-key-file=/data/backups/keyfile
```

{% hint style='info' %}
注意
在某些情况下，使用不同的文本编辑器生成 `KEYFILE` ，文本文件可能包含 CRLF，这将导致密钥大小增大，从而使其无效。建议使用如下方法创建密钥文件：`echo -n "GCHFLrDFVx6UAsRb88uLVbAVWbK+Yzfs" > /data/backups/keyfile`
{% endhint %}

# 优化加密过程

有两个可用于加速加密过程的加密备份选项。分别是 `xtrabackup --encrypt-threads` 和 `xtrabackup --encrypt-chunk-size`。通过使用 `xtrabackup --encrypt-threads` 选项，可以指定多个线程用于并行加密。选项 `xtrabackup --encrypt-chunk-size` 可用于指定每个加密线程的加密缓冲区的大小（以字节为单位）（默认值为64K）。

# 解密加密备份

Percona XtraBackup 已经实现了用于解密备份的解密选项 `xtrabackup --decrypt `：

```
$ xtrabackup --decrypt=AES256 --encrypt-key="GCHFLrDFVx6UAsRb88uLVbAVWbK+Yzfs"\
--target-dir=/data/backups/
```

Percona XtraBackup 不会自动删除加密文件。为了清理备份目录，用户应该自行删除 `*.xbcrypt` 文件。在 Percona XtraBackup 2.4.6 中，您可以使用 `xtrabackup --remove-original` 选项在解密后删除加密文件。要在解密文件后删除文件，您应该运行：

```
$ xtrabackup --decrypt=AES256 --encrypt-key="GCHFLrDFVx6UAsRb88uLVbAVWbK+Yzfs"\
--target-dir=/data/backups/ --remove-original
```

{% hint style='info' %}
注意
`xtrabackup --parallel` 可与 `xtrabackup --decrypt` 选项一起使用，同时解密多个文件。
{% endhint %}

当文件已经被解密时，可以准备备份了。

# 准备加密备份

备份解密后，可以使用 `xtrabackup --prepare` 选项以与标准完整备份相同的方式进行准备：

```
$ xtrabackup --prepare --target-dir=/data/backups/
```

# 恢复加密的备份

**xtrabackup** 有一个 `xtrabackup --copy-back` 选项，可用于将备份还原到服务器的 `datadir`：

```
$ xtrabackup --copy-back --target-dir=/data/backups/
```

它会将所有与数据相关的文件复制到服务器的 `datadir`，需要复制的文件由服务器的 `my.cnf` 配置文件决定。您应该检查输出的最后一行以获取成功消息：

```
170214 12:37:01 completed OK!
```

# 其他阅读

* [Libgcrypt 参考手册](https://www.gnupg.org/documentation/manuals/gcrypt/)
