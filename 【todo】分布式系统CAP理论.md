# 1 前言
参考文章：

分布式系统之CAP理论      https://www.cnblogs.com/hxsyl/p/4381980.html

CAP一开始作为提出时，是指：一致性（consistency）、可用性（Availability）、分区容错（partition-tolerance）。后来有人证明，这CAP三这不可能同时兼得，CAP理论成为了NoSql数据库管理系统构建的基础。

在证明CAP不可能三者同时满足的过程中，逐渐开始有新的理解：

* 一致性逐渐有了新的理解。CAP中的一致性要求，被认为其实就是数据库的ACID：

一个用户请求要么成功、要么失败，不能处于中间状态（Atomic）；

一旦一个事务完成，将来的所有事务都必须基于这个完成后的状态（Consistent）；

未完成的事务不会互相影响（Isolated）；

一旦一个事务完成，就是持久的（Durable）。

* 对于Availability，其概念没有变化，指的是对于一个系统而言，所有的请求都应该‘成功’并且收到‘返回’。

* 对于Partition-tolerance，所指就是分布式系统的容错性。节点crash或者网络分片都不应该导致一个分布式系统停止服务。

# 2 CAP简介
CAP定律说的是在一个分布式计算机系统中，一致性，可用性和分区容错性这三种保证无法同时得到满足，最多满足两个。

