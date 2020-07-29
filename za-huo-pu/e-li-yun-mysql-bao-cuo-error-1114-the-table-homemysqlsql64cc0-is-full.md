---
description: '2020-7-29 21:46:10'
---

# 阿里云Mysql报错: ERROR 1114 The table '/home/mysql/\*\#sql\_64cc\_0' is full

> 问题现象

今天，开发同学说用小黄油连接数据库看不到数据库，命令行连接后 `show databases;` 返回如下报错

{% hint style="info" %}
用其他数据库GUI软件打开没报错的时候，试试用命令行~
{% endhint %}

```text
$ show databases;

ERROR 1114 (HY000): The table '/home/mysql/log/tmp/#sql_64cc_0' is full
```

```text
MySQL [(none)]> show status like '%tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 5     |
| Created_tmp_tables      | 5     |
+-------------------------+-------+
3 rows in set (0.00 sec)

MySQL [(none)]> show variables like "%tmp%";
+----------------------------+--------------------------------+
| Variable_name              | Value                          |
+----------------------------+--------------------------------+
| default_tmp_storage_engine | InnoDB                         |
| innodb_tmpdir              | /1805115-1/home/mysql/data/log |
| max_tmp_tables             | 32                             |
| slave_load_tmpdir          |                                |
| tmp_table_size             | 67108864                       |
| tmpdir                     |                                |
+----------------------------+--------------------------------+
6 rows in set (0.00 sec)
```

> 原因

在SQL查询进行`group by`、`order by`、`distinct`、`union`、多表更新、`group_concat`、`count(distinct)`、子查询或表连接的情况下，MySQL 有可能会使用内部临时表。

MySQL 首先在内存中创建 `Memory` 引擎临时表，当临时表的尺寸过大时，会自动转换为磁盘上的 `MyISAM` 引擎临时表（当查询涉及到 Blob 或 Text 类型字段，MySQL 会直接使用磁盘临时表）。

这个错误信息，说明磁盘上的临时表 `#sql_64cc_0` 的物理尺寸受到限制，已经无法再继续扩展了。

导致这个错误信息的原因是查询语句使用的内部磁盘临时表（MyISAM 引擎表）总大小已经达到了实例参数 `loose_rds_max_tmp_disk_space` 指定的限制（默认 10 GB）（仅限MySQL 5.5或5.6支持该参数）。

> 怎么处理？

修改 `loose_rds_max_tmp_disk_space` 和 `tmp_table_size` 的大小咯，单位是`Byte` 。

{% hint style="info" %}
`loose_rds_max_tmp_disk_space:` Used to limit the max space can be occupied by tmp dir.

用于限制tmp目录所能占用的最大空间。

tmp\_table\_size: The maximum size of internal in-memory temporary tables.

内存中内部临时表的最大大小。
{% endhint %}

```text
loose_rds_max_tmp_disk_space = 53687091200
tmp_table_size = 67108864
```

修改这2个参数不需要重启。

> 参考

1. [RDS MySQL出现“the table '/home/mysql/xxxx/xxxx/\#tab\_name' is full”报错的解决方法](https://help.aliyun.com/knowledge_detail/41746.html)
2. [参数调优建议](https://help.aliyun.com/document_detail/63255.html)

> 思考

因为业务需要，导致实例中有9000+的库，因此缩短库名或者SQL调优都用一定的效果。

