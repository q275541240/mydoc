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
```Java
public Vote lookForLeader() throws InterruptedException {
        // ----------------------- 1 创建选举对象，做选举前的初始化工作 ---------------------
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(
                    self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
            self.start_fle = Time.currentElapsedTime();
        }
        try {
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

            int notTimeout = finalizeWait;

            // ----------------------- 2 将自己作为初始leader投出去 ---------------------
            synchronized(this){
                logicalclock.incrementAndGet();
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() +
                    ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            sendNotifications();

            // ----------------------- 3 验证自己与大家的投票谁更适合做leader ---------------------
            /*
             * Loop in which we exchange notifications until we find a leader
             */

            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)){
                /*
                 * Remove next notification from queue, times out after 2 times
                 * the termination time
                 */
                Notification n = recvqueue.poll(notTimeout,
                        TimeUnit.MILLISECONDS);

                /*
                 * Sends more notifications if haven't received enough.
                 * Otherwise processes new notification.
                 */
                if(n == null){
                    if(manager.haveDelivered()){
                        sendNotifications();
                    } else {
                        manager.connectAll();
                    }

                    /*
                     * Exponential backoff
                     */
                    int tmpTimeOut = notTimeout*2;
                    notTimeout = (tmpTimeOut < maxNotificationInterval?
                            tmpTimeOut : maxNotificationInterval);
                    LOG.info("Notification time out: " + notTimeout);
                }
                else if(validVoter(n.sid) && validVoter(n.leader)) {
                    /*
                     * Only proceed if the vote comes from a replica in the
                     * voting view for a replica in the voting view.
                     */
                    switch (n.state) {
                        case LOOKING:
                            // If notification > current, replace and send messages out
                            if (n.electionEpoch > logicalclock.get()) {
                                logicalclock.set(n.electionEpoch);
                                recvset.clear();
                                if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                    updateProposal(n.leader, n.zxid, n.peerEpoch);
                                } else {
                                    updateProposal(getInitId(),
                                        getInitLastLoggedZxid(),
                                        getPeerEpoch());
                                }
                                sendNotifications();
                            } else if (n.electionEpoch < logicalclock.get()) {
                                if(LOG.isDebugEnabled()){
                                    LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                        + Long.toHexString(n.electionEpoch)
                                        + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                                }
                                break;
                            } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                proposedLeader, proposedZxid, proposedEpoch)) {
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                                sendNotifications();
                            }

                            if(LOG.isDebugEnabled()){
                                LOG.debug("Adding vote: from=" + n.sid +
                                    ", proposed leader=" + n.leader +
                                    ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                    ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                            }

                            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                            // ----------------------- 4 判断本轮选举是否可以结束了 ---------------------
                            if (termPredicate(recvset,
                                new Vote(proposedLeader, proposedZxid,
                                        logicalclock.get(), proposedEpoch))) {

                                // Verify if there is any change in the proposed leader
                                while((n = recvqueue.poll(finalizeWait,
                                    TimeUnit.MILLISECONDS)) != null){
                                    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                        proposedLeader, proposedZxid, proposedEpoch)){
                                        recvqueue.put(n);
                                        break;
                                    }
                                }

                                /*
                             * This predicate is true once we don't read any new
                             * relevant message from the reception queue
                             */
                                if (n == null) {
                                    self.setPeerState((proposedLeader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                    Vote endVote = new Vote(proposedLeader,
                                                        proposedZxid,
                                                        logicalclock.get(),
                                                        proposedEpoch);
                                    leaveInstance(endVote);
                                    return endVote;
                                }
                            }
                            break;
                        // ----------------------- 5 处理无需选举的情况 ---------------------
                        case OBSERVING:
                            LOG.debug("Notification from observer: " + n.sid);
                            break;
                        case FOLLOWING:
                        case LEADING:
                        /*
                         * Consider all notifications from the same epoch
                         * together.
                         */
                        if(n.electionEpoch == logicalclock.get()){
                            recvset.put(n.sid, new Vote(n.leader,
                                                          n.zxid,
                                                          n.electionEpoch,
                                                          n.peerEpoch));
                           
                            if(ooePredicate(recvset, outofelection, n)) {
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(n.leader, 
                                        n.zxid, 
                                        n.electionEpoch, 
                                        n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }

                        /*
                         * Before joining an established ensemble, verify
                         * a majority is following the same leader.
                         */
                        outofelection.put(n.sid, new Vote(n.version,
                                                            n.leader,
                                                            n.zxid,
                                                            n.electionEpoch,
                                                            n.peerEpoch,
                                                            n.state));
           
                        if(ooePredicate(outofelection, outofelection, n)) {
                            synchronized(this){
                                logicalclock.set(n.electionEpoch);
                                self.setPeerState((n.leader == self.getId()) ?
                                        ServerState.LEADING: learningState());
                            }
                            Vote endVote = new Vote(n.leader,
                                                    n.zxid,
                                                    n.electionEpoch,
                                                    n.peerEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                        break;
                    default:
                        LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)",
                                n.state, n.sid);
                        break;
                    }
                } else {
                    if (!validVoter(n.leader)) {
                        LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                    }
                    if (!validVoter(n.sid)) {
                        LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                    }
                }
            }
            return null;
        } finally {
            try {
                if(self.jmxLeaderElectionBean != null){
                    MBeanRegistry.getInstance().unregister(
                            self.jmxLeaderElectionBean);
                }
            } catch (Exception e) {
                LOG.warn("Failed to unregister with JMX", e);
            }
            self.jmxLeaderElectionBean = null;
            LOG.debug("Number of connection processing threads: {}",
                    manager.getConnectionThreadCount());
        }
    }
```
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
sid代表接受消息通知的服务id,该方法遍历所有服务，构造发送的通知对象，丢进队列中。

![](/img/validVoter.png)

该方法验证sid，判断是否有权参与投票以及投票。

```java
switch (n.state) {
//判断接收方的状态，如果是Looking则判断谁更适合做主
    case LOOKING:
        // If notification > current, replace and send messages out
        //如果收到的选票轮数大于自己的轮数，说明自己的选举轮数已经落后，所以设置当前轮数为收到的轮数，清空选票箱(说明可能已经掉线，这时候不知道轮数是多少，所以以收到的选票中最大的为准)
        if (n.electionEpoch > logicalclock.get()) {
            logicalclock.set(n.electionEpoch);
            recvset.clear();
            //判断n和自己谁的选票有效，如果返回true则说明n更适合
            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                    getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                    //修改自己的选票，选择和n选的一样
                updateProposal(n.leader, n.zxid, n.peerEpoch);
            } else {
            //如果自己更适合，修改选票为自己的
                updateProposal(getInitId(),
                        getInitLastLoggedZxid(),
                        getPeerEpoch());
            }
            //发送更改选票通知
            sendNotifications();
        } else if (n.electionEpoch < logicalclock.get()) {
        //如果n的选举轮数小于自己的选举轮数，说明是废票
            if (LOG.isDebugEnabled()) {
                LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                        + Long.toHexString(n.electionEpoch)
                        + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
            }
            break;
        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                proposedLeader, proposedZxid, proposedEpoch)) {
                //如果选举轮数相等，并且判断n更适合，则修改选票成和n的一致，发送修改选票通知
            updateProposal(n.leader, n.zxid, n.peerEpoch);
            sendNotifications();
        }
        。。。
}
```
1.如果发现接收到的选票epoch大于自己投出去的epoch，则判断谁推荐的leader更合适，如果是n推荐的更合适，则修改自己的选票和n一样，发送通知。否则修改自己的epoch和n一致,重新发送通知。清空售票箱，改变自己的选票，发送通知。
2.如果n的epoch小于自己的epoch，则跳出该选票的处理。
3.如果n的epoch和自己的相等，说明是同一轮选举，如果发现n选的更适合当leader则修改自己的选票和n一致，发送通知。

#### 4.判断本轮选举是否可以结束
```java
recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
// ----------------------- 4 判断本轮选举是否可以结束了 ---------------------
if (termPredicate(recvset,
        new Vote(proposedLeader, proposedZxid,
                logicalclock.get(), proposedEpoch))) {
    // Verify if there is any change in the proposed leader
    while ((n = recvqueue.poll(finalizeWait,
            TimeUnit.MILLISECONDS)) != null) {
        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                proposedLeader, proposedZxid, proposedEpoch)) {
            recvqueue.put(n);
            break;
        }
    }
```
```java
protected boolean termPredicate(
            HashMap<Long, Vote> votes,
            Vote vote) {

        HashSet<Long> set = new HashSet<Long>();

        /*
         * First make the views consistent. Sometimes peers will have
         * different zxids for a server depending on timing.
         */
        for (Map.Entry<Long, Vote> entry : votes.entrySet()) {
            if (vote.equals(entry.getValue())) {
                set.add(entry.getKey());
            }
        }

        return self.getQuorumVerifier().containsQuorum(set);
    }
```
1.接收到的选票有效，则放入选票箱，recvset为一个Map，可以覆盖。
2.将n的选票进行校验，如果和n选票一致的票数过半，则进入while循环。
3.while循环将剩下的所有选票取出，如果剩下的选票没有更适合当leader的，则选举结束，产生leader，但是如果剩下更适合当leader的选票，则重新放回队列。break while循环，在下一次会出发修改选票，则队列会重新收到选票，再次进行选举

#### 5.无需选举的情况