---
layout: post
categories : [lnmp]
tags : []
excerpt: innodb-memcached.html文档翻译
---
{% include JB/setup %}

官方文档:

http://dev.mysql.com/doc/refman/5.6/en/innodb-memcached.html

当前版本提供的功能:

* memcached as a daemon plugin of mysqld: both mysqld and memcached run in the same process space, with very low latency access to data.

当作mysqld的一个守护进程: mysqld 和memcached 运行在同一个进程空间,对数据访问有非常低的延迟;

* Direct access to InnoDB tables, bypassing the SQL parser, the optimizer, and even the Handler API layer.

绕过SQL语句的解析,优化,甚至整个SQL Handler API, 直接从InnoDB 表(文件)读取数据

* Standard memcached protocols, both the text-based protocol and the binary protocol. The memcached+ InnoDB combination passes all 55 compatibility tests from the memcapable command.

(使用)标准的memcached协议, (支持)字符数据和二进制数据; memcached+InnoDB的组合(还有其他组合?) 通过了所有55项memcapable 命令的兼容性测试;

* Multi-column support: you can map multiple columns into the “value” part of the key/value store, with column values delimited by a user-specified separator character.

多列支持:可以把多个列使用用户指定的分隔符链接起来 , 作为key/value存储系统中value的一部分,(存储起来)

* By default, you use the memcached protocol to read and write data directly to InnoDB, and let MySQL manage the in-memory caching through the InnoDB buffer pool. Advanced users can also configure the system as a traditional memcached server, with all data cached only in memory, or use a combination ofmemcached caching and InnoDB persistent storage. The default settings represent the combination of high reliability with the fewest surprises for database applications. For example, the default settings avoid uncommitted data on the database side, or stale data returned for memcached get requests.

默认情况, 你可以使用memecached协议来直接读写数据到InnoDB表, 并让mysql 通过 InnoDB 缓冲池 将数据缓存到内存里. 高级用户还可以进行设置 ,选择是让数据是像传统memcached 一样只缓存到内存 , 还是使用一个memcached缓存+InnoDB持久化的组合. 这种组合的默认设置就给数据库应用带来了高可靠性. 例如 ,避免了数据只在持久化一端(只存在数据库中,而没有被缓存) ,又避免了当使用memcache的get命令获取到旧的数据; (感觉整体其实就是说 ,这个插件帮你管了 持久化端 的时候 memcached 端的更新, 不用你自己手动去管了.)

* You can control how often data is passed back and forth between InnoDB and memcached operations through thedaemon_memcached_r_batch_size and daemon_memcached_w_batch_size configuration options. Both of these options default to a value of 1 for maximum reliability.

你可以通过修改daemon_memcached_r_batch_size 和 daemon_memcached_w_batch_size 这两个配置选项来控制InnoDB 端和memcached 端 交互数据的频率 .这两个配置的默认值都是1 ,以获得最高的可靠性;

* You can specify any memcached configuration options through the MySQL configuration variabledaemon_memcached_option. For example, you might change the port that memcached listens on, reduce the maximum number of simultaneous connections, change the maximum memory size for a key/value pair, or enable debugging messages for the error log.

你可以通过mysql 配置变量 daemon_memcached_option特别指定 memcached的配置 . 例如 ,改变memcache的监听端口, 修改最大连接数, 修改最大内存大小 或者 打开 debug 信息;

* A configuration option innodb_api_trx_level lets you control the transaction isolation level on queries processed by the memcached interface. Although memcached has concept of transactions, you might use this property to control how soon memcached sees changes caused by SQL statements, if you issue DML statements on the same table that memcached interfaces with. By default, it is set to READ UNCOMMITTED.

配置项 innodb_api_trx_level 让你控制当通过memcached接口访问数据时的隔离级别. (虽然memcached也有事务的概念) ,你可以用此控制当数据被 SQL 端修改后 ,memcached端多快能看到(同时被更新); 默认, 这个设置的默认是 READ UNCOMMITTED 级别;

