---
layout: post
title: TiDB的分布式事务原理探究
date: 2019-05-09
categories: TiDB
tags: TiDB 数据库 分布式事务
cover: /assets/img/tidb/db.jpg
---



# TiDB分布式事务原理探究



## 事务开启



获取全局授时作为startTS构建一个tikvTxn对象（包括snapshot）。



## 事务写



txn.Set方法本质上将kv值写入了一个内存缓存(即kv/memdb_buffer.go中的memDbBuffer)中。该内存kv数据库利用的是golevel提供的功能。



## 事务回滚



直接将tikvTxn的valid字段置为false，之后如果用户再执行提交或者回滚操作，会检查valid，如果为false则直接返回错误。



## 事务提交



### 操作映射



遍历kv数据库中所有的key，并将每个key和其操作组装到一起成为一个mutationEx对象，并且将其放到一个map中（map由key映射到mutationEx），最后将这个map放入twoPhaseCommitter的mutations字段。



操作也是通过key的内容推断出来的，推断逻辑如下：

- key的长度为0，则操作为删除（Op_Del）
- 如果key的长度大于0，且不需要延迟检查或者延迟检查不通过（value的当前长度大于0），则操作为Op_Put（即新增key或者更新key）
- 如果key的长度大于0，且延迟检查(value的当前长度等于0)通过，则操作为Op_Insert（即新增key，不允许更新key）
- 对于tikvTxn中的lockKeys字段中的key，如果他在kv数据库中不存在的话，则给予Op_Lock（即单纯的作为锁，事务结束就将这个key删除）



### Prewrite



Percolator事务模型有primary和secondaries的概念，TiDB的实现中直接将第一个key作为Primary，剩下的Key全部作为secondaries



TiDB上的操作：



- 将所有的key按照Region进行分组（从Region缓存或者PD中获取key所处的Region）
- 将每组的key再拆分成Batch（每个Batch在16k作为，主要目的是为了缩小RPC packet的大小），并发地对每个batch进行处理（即给TiKV发送Prewrite指令）



注意：其实在Prewrite阶段的实现并不太能看出primary和secondaries的区别，他们都被一起打成batch并发处理了。



Tikv接收到指令之后对每个Batch分别进行Prewrite：



遍历batch中每个元素的mutationEx（之前**操作映射**时组装的），然后分别进行如下操作：

- 如果操作是Op_Insert的话，则以事务开始时间startTs进行快照读检查key是否重复，如果重复则标记错误，看batch中下一个元素
- 编码出一把锁（所谓“锁”就是指key的version为全0的64位bit，正常情况下是时间戳，所以锁永远排在第一个）
- 检查是否有其他事务给该key上锁（即查看是否有version为全0的key），如果有则事务冲突
- 上面的检查通过了的话，则查看Rocksdb上紧接着锁的下一个key（即最新的key），查看其时间戳是否大于等于startTs，如果这样的话，说明有其他事务先提交了，事务冲突。
- 上面的检查也通过了的话则将自己的锁（"锁"的信息包括startTs，primary，value以及操作码等等，详见store/mockstore/mocktikv/mvcc.go中的mvccLock结构）插入进去



### Commit



TiDB中的逻辑：



- 重新获得一个全局授时作为提交时间戳commitTs
- Region分组，Batch拆分和上面是一样的
- 先提交Primary
- 然后在后台提交secondaries



TiKV中的逻辑：



新建一个Rocksdb的Batch进行批量的增删，然后对于每个key

- 除了Op_Lock操作的Key，都以CommitTS作为Key的版本号插入进去，组装Value的时候将TiKV的操作码转成底层mvcc store的操作码（将Op_put转成typePut，剩下的除了不可能出现的Op_Lock，都转换成typeDelete），然后删除锁
- 对于Op_Lock操作的Key则直接删除锁即可



