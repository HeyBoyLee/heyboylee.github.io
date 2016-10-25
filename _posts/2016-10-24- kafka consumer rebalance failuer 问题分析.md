---
layout: post
title:  "kafka consumer rebalance failure问题分析"
date:   2016-10-24 10:00:00
categories: 开发者手册
---
### 问题
	近期在使用kafka替代rabbtimq的过程中出现了几次consumer rebalance失败的问题，导致服务出现暂时不可用，送达延迟增大的问题。

### 问题主要有下面几个:
	kafka consumer过于“娇气”，一言不合就rebalance失败，例如发布、重启、partition变更、zk抖动等
	rebalance failure就罢了，但rebalance failure之后无法重启进程自动恢复就不能忍了。主要原因是这个异常在程序中难以捕获，因为rebalance操作是在一个异步线程中单独处理的，异常无法到达业务线程，无法进行捕获
	由于目前我们业务中大量使用了kafka，这个rebalance的问题是个极大的隐患。
	目前我们仍在使用0.8版本的kafka server，看介绍说0.9的kafka的rebalance机制发生了很大的变化，说不定不会再有这种情况。但咨询了负责kafka运营的同事，没有足够的精力去升级kafka的版本。
	因此，决定花一些精力研究一些kafka consumer rebalance机制，看能否通过一些配置或对客户端源码的简单改动解决这个隐患。

### rebalance触发时机
	目前有以下几种情况会触发rebalance
	有broker发生增加、减少或其他变更时（/brokers/ids）
	zookeeper session expired重新建立时
	当前consumer group有新的consumer加入或者有consumer退出时   (/consumers/${groupid}/ids/*)
	当有新的topic创建时   /brokers/topics/*

### rebalance流程
	对rebalance流程，最多重试rebalance.max.retries次，每次尝试失败后，则等待rebalance.backoff.ms 毫秒
	查看是否有broker存在，如果没有任何broker存在，则退出  （/brokers/ids）
	关闭fetchers，停止从broker读取数据，将所有的offset信息写到zk   (/consumers/${groupid}/offsets/${topic}/${partitionid})
	释放partition与consumer thread的绑定关系，删除节点 (/consumers/${groupid}/owners/${topic}/${partitionid})
	对每个topic+ consumer线程重新分配partition
	对consumer thread（consumerid）按照字典顺序排序    假设partion数量为P， consumer thread数量为C
	根据 { P/C } 计算出平均每个consumer thread应消费的partition个数   => N = P/C
	根据 { P%C } 计算出额外多消费一个partition的consumer thread数量   => M = P%C
	前M个consumer thread应消费 N+1 个partition，后面的consumer thread消费N个partition
	计算出当前的每个consumer thread应该消费的partition范围  [ i*N+min(i,M), (i+1)*N + min(i+1,M) )
	当consumer thread数量大于partition数量时，>=M的consumer thread则不消费任何partition数据
	对每个consumer thread进行partition绑定
	从zk读取offset信息    /consumers/${groupid}/offsets/${topic}/${partitioned}
	尝试将从属关系写到zk  /consumers/${groupid}/owners/${topic}/${partitionid}
	如果绑定关系失败（此时节点被其他consumer绑定，创建失败）则返回失败，等待下次继续尝试
	如果其他原因导致绑定失败，则释放所有已经绑定成功的消费关系

### rebalance failure优化思路
	目前rebalance机制有很多不足之处，显著的两点是：
	kafka consumer rebalance过于依赖zookeeper，kafka服务器及每个consumer的信息都存储在zookeeper上，一旦发生zookeeper不可连，则无法进行rebalance
	目前kafka consumer rebalance完全是一种客户端行为，每个客户端按照统一的规则去决定自己消费哪个partition。而每个consumer的rebalance行为又依赖于其他的consumer是否已经释放了即将消费的partition，从而产生连锁反应，在consumer数量比较多时，很容易导致重试了若干次仍然没有成功，而导致rebalance failure

	优化思路主要从减少rebalance发生的概率，以及发生rebalance failure之后能及时恢复两方面去考虑。
	增大zookeeper session过期时间，减少因为zk过期而导致的rebalance。也防止了用户进程发生长时间GC时zk超时的情况。
	减少每个consumer group下面节点的数量。因为相比其他的原因导致的rebalance，consumer变动导致rebalance的几率大很多，减少同一个consumer group下面consumer数量可以很大的降低因为consumer变动导致的rebalance。每个consumer group下面只有同一个topic的consumer是比较合适的做法。另外，避免同一个topic下面consumer太多也可以降低rebalance发生的概率，并且可以减少rebalance failure的连锁反应。
	增大重试次数也是一个可选方案。rebalance failure发生主要原因是在绑定消费关系时，该partition的绑定关系没有被其他的consumer释放。理论上，经过足够多次的重试，每个consumer都应该能够rebalance成功，只是consumer group的规模越大，所需要的尝试次数越多。

### 捕获ConsumerRebalanceFailedException
	目前的主要问题不是rebalance failure的发生，而是rebalance failure发生之后无法自动恢复。而不能自动恢复的原因是kafka rebalance线程抛出的Exception我们无法捕获并做一定的处理。
	想到一个比较hack的方法，就是在log4j的appender中比较日志中的exception，如果是我们认为致命的异常，就通知程序退出重启。重启时需要注意一点，最好使用单独的线程去处理重启操作，否则很容易产生清理线程与调用线程访问同一把锁而产生死锁的现象。
