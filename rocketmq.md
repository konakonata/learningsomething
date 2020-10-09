卡夫卡：吞吐量最高，支持消息的大量堆积

rocketMQ：丰富的消息拉取模式，实时的消息订阅机制，支持事务消息，高效的订阅者水平扩展能力

解压rocketMQ的压缩包，修改文件名

**在host文件中映射ip**

vi /etc/hosts文件

192.168.0.14 rocketmq-nameserver1

**创建文件夹：**
logs：存储日志

store：存储数据文件

commitlog：存储消息信息

consumequeue、index：存储消息的索引数据

**conf目录配置文件说明：**

2m-2s-async：2主2从异步

2m-2s-sync：2主2从同步

2m-noslave：2主没从

配置单节点，修改2m-2s-async的配置，进入2m-2s-async目录，修改第一个配置文件broker-a.properties文件

```properties
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不通的配置文件填写的不一样
brokerName=broker-a
#0标识Master，> 0表示slave
brokerId=0
#nameServer 地址、分号分割，rocketmq-nameserver1映射的ip名称
namesrvAddr=rocketmq-nameserver1:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许broker自动创建topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许broker自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#broker对外服务的坚挺端口
listenPort=10911
#删除文件时间点，默认是凌晨4点
deleteWhen=04
#文件保留时间，默认48小时
fileReservedTime=120
#commitLog每个文件的大小默认1g
mapedFileSizeCommitLog=1073741824
#consumerqueue每个文件默认存30w条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径,store文件夹路径
storePathRootDir=
#commitLog路径
storePathCommitLog=
#consumeQueue路径
storePathConsumeQueue=
#index路径
storePathIndex=
#checkpoint文件存储路径,store下的子目录
storeCheckPoint=
#abort文件存储路径,store下的子目录
abortFile=
#限制消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThroughInterval=10000
#flushConsumeQueueThroughInterval=60000
#broker的角色
#-ASYNC_MASTER异步复制master
#-SYNC_MASTER同步复制master
#-SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#-ASYNC_FLUSH 异步刷盘
#-SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128


```

进入conf目录，替换所有xml中的${user.home},保证日志路径正确

**sed -i 's#${user.home}#rocketmq的路径#g' *.xml**

sed -i 在这里起一个批量替换的作用

rocketmq对内存的要求比较高，最少1g，需要修改bin目录下的runbroker.sh和runserver.sh文件

runbroker.sh

JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn1g" 

runserver.sh

JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn1g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=128m"



**启动** 

1.启动namesrv

nohup sh mqnamesrv &

2.再启动broker

nohup sh mqborker -c broker-a.properties的路径 > /dev/null 2>&1 &



**RocketMQ集群**

一主一从：

在主从两台机器上都如上述安装好rocketmq。

配置conf/2m-2s-async目录下的broker-a-s.properties,

吧上面的配置文件内容拷贝，修改brokerId=1（大于0就可以），brokerRole=ASYNC_SLAVE

在java项目的配置文件中，添加从节点的ip加端口，中间用分号隔开

![1572490589453](C:\Users\konata\AppData\Roaming\Typora\typora-user-images\1572490589453.png)

**双主双从，异步刷盘，同步复制**：





