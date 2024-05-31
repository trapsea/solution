# 华为云mq重复消费问题排查记录

topic：pos_create_pos_cooperate_sale_order

消费者：pos-provider

服务器：172.66.6.2（smp-rest1）、172.66.6.3（smp-rest2）、172.66.6.5（smp-provider2）

mq-broker：172.66.1.29、172.66.1.30、172.66.1.31、172.66.1.33

出现时间：8.31-9.5日凌晨2-5点之间



最早接手问题的时候先从链路里能看到handleMessage的链路有重复，其中有两个重点

1、重复消费的时间间隔较大，8.31消费过的消息在9.1号凌晨又重复消费

这里可以排除并发消费的可能了，基本能确定是拉取消息的offset出了问题，只是当时mq的日志3小时清空一次，在这个时间无法查到更多信息 

2、消费者是不同ip

mq消息的消费有两种模式，推和拉，这里出现不同ip的消费者第一反应是排除推送消息模式导致消费者变更，最终从代码层面确认，只有从控制台点击发送消息这一个触发推模式即排除这种可能性。 那么问题必定出现在消费者端，消费者端消息消费的方式只有主动拉取消息一种，推也是基于拉取做的定时轮询。 





先讲结论：网络超时导致的重复消费



在修改了mq日志保存时间以后，发现rocketmq_client日志显示在凌晨出现连接broker和nameServer超时，正是请求超时导致了重复消费，以下是问题发生过程(根据时间顺序描述)



以topic：pos_create_pos_cooperate_sale_order，messageId: AC42020D00016267C3BB043EAE03BEF9 入手排查，这个消息在2023-09-01 19:46:56生成，并且正常消费，在2023-09-05 03:21:42被重复消费

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693897977891-af366e15-03c3-4b6f-875a-63e5b4de40d1.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)



S1ZE:2E,"VENDORSHOPLD":L9545,"WAREHOUSE1D":B0E

DDBEAAEEAEESNENRCHASEORDERID:GGE

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693898053403-1076ffb6-7cdb-4c8e-9223-dbca375a9580.png?x-oss-process=image%2Fformat%2Cwebp)



并且可以得知这个消息的位置信息：broker：172.66.1.31:10911、queueId：0、queueOffset：65，以及正常消费者：172.66.6.3上的pos-provider

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693898266430-39bc59dc-f556-4358-8f72-c40450763463.png?x-oss-process=image%2Fformat%2Cwebp)



而重复消费时的消费者是172.66.6.5的pos-provider，查到这里其实已经能发现，这个消息被不同ip的消费者消费，可能是重平衡机制导致队列重新分配。

简单讲下重平衡的机制，消费者在启动后会向broker定时发送心跳，broker端会维护存在心跳的消费者队列，而重平衡机制是消费者在启动后以及定时轮询做的决定该消费者拉取broker端哪些队列，比如一个topic有t-1到t-4的4个队列，有两个消费者A和B，重平衡的结果就是要保证t1-t2被消费者A消费t3和t4被消费者B消费，核心工作原理是消费者触发重平衡时从broker端拉取消费者列表，以及topic的队列，经过排序后再以指定的方式(一致性hash、轮询等)算出该消费者需要消费的队列，对这些应该自己消费的队列，消费者会向broker端请求获取该topic、consumeGroup、queueId下的offset，这里的offset默认是下次应该消费的offset，理应不会重复消费，再以topic、consumeGroup、queueId、nextOffset等信息构建一个定时拉取消息的对象pullRequest放到另一个线程池的队列里定时触发拉取消息。



继续排查，顺着重平衡的问题排查172.66.6.3，日志显示在凌晨2.40-3.22期间出现大量的请求broker和nameserver超时

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693899368802-c6a0668e-4e14-44ad-a98a-923fe7842513.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)



网络超时的原因也分为服务导致和网络本身的问题，到这里的时候经过排查排除了broker端、pos-provider端、服务器cpu、内存等因素导致的可能，当然，网络超时是并不是触发重复消费的直接原因。

这里还需要知道一点，网络超时是频发，但不是完全无法请求，可理解为存在比较大的网络波动。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693899741084-c4375d4b-83e0-414f-855a-fec1737b27cc.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693899756429-0beb23cd-1381-463f-b36d-e8fdd0907aed.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)





然后在2023-09-05 03:04:44的时候由于172.66.6.3这台服务器网络波动发送心跳超时，重平衡机制使得broker：172.66.1.31:10911、queueId：0这个队列被分配到172.66.6.5这个消费者,并且此时的消费刻度是从broker端拉取到的最新消费刻度

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693899979701-1b77cc88-46c1-4b99-b02c-ba1205b61665.png?x-oss-process=image%2Fformat%2Cwebp)



在2023-09-05 03:19:04的时候172.66.6.3这台消费者又通过重平衡重新消费到队列，并且消费刻度是108没有问题，但是在这个时间网络波动仍然存在

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693900051388-87916f46-3b00-4507-b6d1-651d067021e7.png?x-oss-process=image%2Fformat%2Cwebp)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693900311141-842ad1a6-d79e-4255-961d-7a5cde32e53d.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)





直到2023-09-05 03:19:51能看到这里在fetchConsumeOffsetFromBroker超时后把offset设置为了-1

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693900439400-20ce0b9b-8fba-43a5-af45-57129972bc20.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693900897211-2d065f6a-a4b4-47b5-8bbe-2961ccf3b85f.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693900934894-bcd31beb-bea8-4c8d-9287-1a3adfd30aa0.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0)





然后请求到broker端拉取消息broker识别到offset为-1非法，会将offset设置为该queue的minOffset：65

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693901918496-caf9b3ad-d5d1-4517-94fe-4a285a0c851e.png?x-oss-process=image%2Fformat%2Cwebp)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693901019142-9f1fee8b-6daf-435d-8a2a-dfa31ef5e9f6.png?x-oss-process=image%2Fformat%2Cwebp)



再然后2023-09-05 03:21:20消费者将offset为65反写到broker端，broker端识别到当前的offset：65小于broker端已存在的offset：108，并打了个日志然后覆盖了。

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693902019308-d76e984a-b10d-49f3-b5fa-f12bd35d9f55.png?x-oss-process=image%2Fformat%2Cwebp)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693901517848-ddae9091-ee14-4ed8-8873-2696866621e9.png?x-oss-process=image%2Fformat%2Cwebp)





此后网络波动仍然存在，这期间能看到172.66.6.5这台消费者不断在尝试获取队列的锁，但没能创建拉取消息的对象

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693902358346-4f5b05bc-fdf7-443f-9576-9bd6a6a23416.png?x-oss-process=image%2Fformat%2Cwebp)





2023-09-05 03:21:27时创建了pullrequest对象开始拉取消息消费，但这时仍然没拿到队列的锁，导致拉取不了消息，直到2023-09-05 03:21:42拿到队列的锁开始消费，2023-09-05 03:21:46,046提交了第一个commitOffset：66

![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693902521298-939fe47f-6fc9-487c-b5c7-71ed5e0244d1.png?x-oss-process=image%2Fformat%2Cwebp)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693902653370-759b01ad-270a-4117-8e70-5ae6c8a878b8.png?x-oss-process=image%2Fformat%2Cwebp)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693902671954-e74e3532-95b0-44f6-a682-0690f777ba49.png?x-oss-process=image%2Fformat%2Cwebp)



![image.png](https://cdn.nlark.com/yuque/0/2023/png/2387107/1693902683310-e154dd0e-4de0-47ac-aa8c-98fcafe44c96.png?x-oss-process=image%2Fformat%2Cwebp)





至此重复消费的经过排查完毕，其根本原因由网络波动导致，后续会对网络情况进行进一步排查。