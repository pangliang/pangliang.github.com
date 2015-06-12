---
layout: post
categories : [lnmp]
tags : [mysql, timestamp]
excerpt: mysql 中 TIMESTAMP 列的几种声明方式各有什么不同?
---
{% include JB/setup %}

在CREATE TABLE语句中，第1个TIMESTAMP列可以用下面的任何一种方式声明：

* 如果定义时DEFAULT CURRENT_TIMESTAMP和ON UPDATE CURRENT_TIMESTAMP子句都有，列值为默认使用当前的时间戳，并且自动更新。

* 如果不使用DEFAULT或ON UPDATE子句，那么它等同于DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP。

* 如果只有DEFAULT CURRENT_TIMESTAMP子句，而没有ON UPDATE子句，列值默认为当前时间戳但不自动更新。

* 如果没用DEFAULT子句，但有ON UPDATE CURRENT_TIMESTAMP子句，列默认为0并自动更新。

* 如果有一个常量值DEFAULT，该列会有一个默认值，而且不会自动初始化为当前时间戳。如果该列还有一个ON UPDATE CURRENT_TIMESTAMP子句，这个时间戳会自动更新，否则该列有一个默认的常量但不会自动更新。