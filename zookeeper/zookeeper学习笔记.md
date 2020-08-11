# Zookeeper学习笔记
## Zookeeper选举源码总结
### 选举核心类包路径
![](/img/选举类包路径.png)

![](/img/FastLeaderElection.png)

### 选举核心类属性介绍
#### 1.final static int finalizeWait=200

&emsp; zk源码中FastLeaderElection为选举的核心类，其中final static int finalizeWait=200;规定了zk选主过程的最大时间上限为200毫秒，最长不超过final static int maxNotificationInterval=60000;60秒。

#### 3.Notification
&emsp; Notification是一个让其他Server知道当前Server已经改变投票的通知消息，改变投票会在以下几个场景产生：
&emsp; 1.Server参与了新的一轮选举，此时它改变投票，投向自己。
&emsp; 2.Server知道当前有更大zxid或者zxid相同，但是serverId更大的server，进而把投票转投给他，然后通知其他server，它要修改选票。

#### 2.QuorumCnxManager manager;

&emsp; 该对象为zk的连接管理器，FastLeaderElection使用TCP管理两个同辈Server的通信，并且QuorumCnxManager还管理着这些连接。

&emsp; 因为两个Server同时发起连接，会产生两个TCP链接，QuorumCnxManager会根据IP简单的断开其中一个。让两个Server之间仅存在一个TCP链接。对于每一个peer,QuorumCnxManager会维护一个消息队列，当一方断开连接后，未发送的消息会放在队列中。待重新连接后，会把消息队列中的消息进行重发。
&emsp; 下图为QuorumCnxManager维护的消息队列Map结构图，Key为具体某个serverId,value为消息队列。

![](/img/消息队列.png)

&emsp;若所有队列不为空，说明该server与其他server通讯都出了问题，证明该server失联。
&emsp;若所有队列均为空，说明所有的消息都发送成功，说明当前server与集群连接正常。
&emsp;若有一个队列为空，说明当前server与zk集群的连接没有问题。

### 选举方法介绍(lookForLeader)
#### 1.创建选举需要的对象
Time.currentElapsedTime()获取当前时间的方法，JDK自带的时间类，做差值运算时通过它获取start和end的时间节点，能避免系统时间被篡改的问题。类似于JDK提供的计时器。zk中用到的时间计算方法都用的是这个。

```Java
//recvset用于存放外部选票，key为投票者的serverid,value为选票
HashMap<Long,Vote> recvset=new HashMap<Long,Vote>();

//其中存放的是非法选票，即投票者的状态不是Looking
HashMap<Long,Vote> outofelection=new HashMap<Long,Vote>();

//notificationTimeout是通知的超时时间
int notTimeout=finalizeWait;
```
#### 2.将自己作为初始leader广播出去
```java
synchronized(this){
//逻辑时钟
logicalclock.incrementAndGet();
//更新当前server推荐信息
//getInitId(),获取当前serverId,getInitLastLoggeZxid获取自己的zxid,getPeerEpoch获取自己的epoch
//updateProposal将自己更新为leader发送出去
updateProposal(getInitId(),getInitLastLoggeZxid(),getPeerEpoch());
}
```
#### 3.验证自己与大家的投票，看看谁更适合做leader
![](/img/getVotingView.png)
getVotingView()方法，获取集群中所有Server，View就代表一个Server，然后过滤掉Observe的view，剩下Participant的view，即有权参与投票的view。
![](/img/sendNotification.png)
第三节1   41分钟

#### 4.判断本轮选举是否可以结束
#### 5.无需选举的情况