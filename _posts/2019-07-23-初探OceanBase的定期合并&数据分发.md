---
layout:     post   				    # 使用的布局（不需要改）
title:      初探OceanBase的定期合并&数据分发 				# 标题 
subtitle:   MergeServer/UpdateServer/ChunkServer #副标题
date:       2019-07-23 				# 时间
author:     BY 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 数据库
---

# 初探OceanBase的定期合并&数据分发

## MergeServer

MergeServer的功能主要包括：协议解析、SOL.解析、请求转发、结果合并、多表操作等。
OceanBase客户端与MergeServer之间的协议为MySQL协议。MergeServer首先解析MySQL协议，从中提取出用户发送的SQL语句，接着进行词法分析和语法分析，生成SQL语句的逻辑查询计划和物理查询计划，最后根据物理查询计划调用OceanBase内部的各种操作符。

MergeServer缓存了子表分布信息，根据请求涉及的子表将请求转发给该子表所在的ChunkServer。如果是写操作，还会转发给UpdateServer。某些请求需要跨多个子表。此时MergeServer会将请求拆分后发送给多台ChunkServer，并合并这些ChunkSever返回的结果。如果请求涉及多个表格，MergeServer 需要首先从ChunkServer获取每个表格的数据，接着再执行多表关联或者嵌套查询等操作。

MergeServer 支持并发请求多台ChunkServer，即将多个请求发给多台ChunkServer，再一次性等待所有请求的应答。另外，在SQL执行过程中，如果某个子表所在的ChunkServer出现故障，MergeServer会将请求转发给该子表的其他副本所在的ChunkServer。这样，ChunkServer故障是不会影响用户查询的。

MergeServer本身是没有状态的，因此，MergeServer宕机不会对使用者产生影响，客户端会自动将发生故障的MergeServer屏蔽掉。
## UpdateServer
UpdateServer是集群中唯一能够接受写入的模块，每个集群中只有一个主UpdateServer。UpdateServer中的更新操作首先写人到内存表，当内存表的数据量超过一定值时，可以生成快照文件并转储到SSD中。快照文件的组织方式与ChunkServer中的SSTable类似，因此，这些快照文件也称为SSTable。另外，由于数据行的某些列被更新，某些列没被更新，SSTable中存储的数据行是稀疏的，称为稀疏型SSTable。

为了保证可靠性，主UpdateServer 更新内存表之前需要首先写操作日志，并同步到备UpdateServer。当主UpdateServer 发生故障时，RootServer上维护的租约将失效，此时，RootServer将从备UpdateServer列表中选择一台最新的备UpdateServer切换为主UpdateServer 继续提供写服务。UpdateServer岩机重启后需要首先加载转储的快照文件（SSTable文件），接着回放快照点之后的操作日志。

由于集群中只有一台主UpdateServer提供写服务，因此，OceanBase很容易地实现了跨行跨表事务，而不需要采用传统的两阶段提交协议。当然，这样也带来了一系列的问题。由于整个集群所有的读写操作都必须经过UpdateServer，UpdateServer的性能至关重要。OceanBase集群通过定期合并和数据分发这两种机制将UpdateServer一段时间之前的增量更新源源不断地分散到ChunkServer，而UpdateServer只需要服务最新一小段时间新增的数据，这些数据往往可以全部存放在内存中。另外，系统实现时也需要对UpdateServer的内存操作、网络框架、磁盘操作做大量的优化。

## ChunkServer
ChunkServer的功能包括：存储多个子表，提供读取服务，执行定期合并以及数据分发。

OceanBase将大表划分为大小约为256MB的子表，每个子表由一个或者多个SSTable组成（一般为一个），每个SSTable由多个块（Block，大小为4KB~64KB之间，可配置）组成，数据在SSTable中按照主键有序存储。查找某一行数据时，需要首先定位这一行所属的子表，接着在相应的SSTable中执行二分查找。SSTable支持两种缓存模式，块缓存（Block Cache）以及行缓存（Row Cache）。块缓存以块为单位缓存是近读取的数据，行缓存以行为单位缓存最近读取的数据。

MergeServer将每个子表的读取请求发送到子表所在的ChunkServer，ChunkServer首先读取SSTable中包含的基线数据，接着请求UpdateServer获取相应的增量更新数据，并将基线数据与增量更新融合后得到最终结果。

由于每次读取都需要从UpdateServer中获取最新的增量更新，为了保证读取性能，需要限制UpdateServer中增量更新的数据量，最好能够全部存放在内存中。由于每次读取都需要从UpdateServer中获取最新的增量更新，为了保证读取性能，需要限制UpdateServer中增量更新的数据量，最好能够全部存放在内存中。

OceanBase内部会定期触发合并或者数据分发操作，在这个过程中，ChunkServer将从UpdateServer获取一段时间之前的更新操作。通常情况下，OceanBase 集群会在每天的服务低峰期（凌晨1:00开始，可配置）执行一次合并操作。这个合并提作往往也称为每日合并。

## 定期合并&数据分发
定期合并和数据分发都是将UpdateServer中的增量更新分发到ChunkServer中的手段，二者的整体流程比较类似：
- UpdateServer冻结当前的活跃内存表（Active MemTable），生成冻结内存表，并开启新的活跃内存表，后续的更新操作都写入新的活跃内存表。
- UpdateServer通知RootServer数据版本发生了变化，之后RootServer通过心跳消息通知ChunkServer。
- 每台ChunkServer启动定期合并或者数据分发操作，从UpdateServer获取每个子表对应的增量更新数据。

定期合并与数据分发两者之间的不同点在于，数据分发过程中ChunkServer只是将UpdateServer中冻结内存表中的增量更新数据缓存到本地，而定期合并过程中ChunkServer 需要将本地SSTable中的基线数据与冻结内存表的增量更新数据执行一次多路归并，融合后生成新的基线数据并存放到新的SSTable中。定期合并对系统服务能力影响很大，往往安排在每天服务低峰期执行（例如凌晨1点开始），而数据分发可以不受限制。

如下图，活跃内存表冻结后生成冻结内存表，后续的写操作进入新的活跃内存表。定期合并过程中ChunkServer 需要读取UpdateServer中冻结内存表的数据、融合后生成新的子表，即:

**新子表=旧子表+冻结内存表**

![定期合并不停读服务](https://oss-weslie.oss-cn-shanghai.aliyuncs.com/%E5%9B%BE%E7%89%87/github%E5%8D%9A%E5%AE%A2%E5%9B%BE/b490ad636c6ba1479d1a6ebd47029d7a8cb2ddf4.jpeg)

虽然定期合并过程中各个ChunkServer的各个子表合并时间和完成时间可能都不相同。但并不影响读取服务。如果子表没有合并完成，那么使用旧子表，并且读取UpdateServer中的冻结内存表以及新的活跃内存表；否则，使用新子表，只读取新的活跃内存表，即：
**查询结果=旧子表+冻结内存表+新的活跃内存表=新子表+新的活跃内存表**
