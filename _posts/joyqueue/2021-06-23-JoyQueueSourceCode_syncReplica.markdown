---
layout:     post
title:      "JoyQueue源码-副本同步"
date:       2021-06-23 20:35:00
catalog: true
tags: JoyQueue
---

开篇诗两句：

​	**葡萄美酒夜光杯，欲饮琵琶马上催。**

​	**醉卧沙场君莫笑，古来征战几人回？**	——王翰《凉州词》

<br/>


## 副本同步

JoyQueue中，多个副本共用同一个partition的日志文件，所以存在TopicPartitionGroup的概念，对应Kafka中Partition的概念 

ReplicaGroup是和TopicPartitionGroup对应的，主要用于副本复制 

```java
ReplicaGroup
leaderId：leader副本ID
replicaExecutor：用于执行副本同步的线程池
replicas：副本列表
replicasWithoutLearners：非learner的副本列表
```

因为JoyQueue的副本同步采用的Raft协议，只需要多数节点同步到数据，就可以提交，同时副本可能存在Learner角色，所以在数据同步的过程中，需要排除learner角色 

ReplicaGroup启动的时候会创建一个延迟队列，并启动一个副本同步线程 

```java
@Override
public void doStart() throws Exception {
    super.doStart();

    replicateResponseQueue = new DelayQueue<>();

    replicateThread = new ReplicateThread("ReplicateThread-" + topicPartitionGroup.toString());
    replicateThread.start();
}
```

ReplicateThread副本同步线程的主要工作：

```java
class ReplicateThread extends Thread {
    private ReplicateThread(String name) {
        super(name);
    }

    @Override
    public void run() {
        initResponseQueue();
        while (true) {
            try {
                //TODO 能否优化一下，非leader节点不要起复制线程
                if (!ReplicaGroup.this.isStarted() || (state != LEADER && state != TRANSFERRING)) {
                    Thread.sleep(100);
                    continue;
                }
                if (neednotReplicate()) {
                    return;
                }
                DelayedCommand command = replicateResponseQueue.take();
                if (command.replicaId() == localReplicaId) {
                    replicateLocal();
                    continue;
                }
                if (!replicas.contains(getReplica(command.replicaId()))) {
                    logger.info("Partition group {}/node {} not contain this node {}",
                            topicPartitionGroup, localReplicaId, command.replicaId());
                    continue;
                }
                replicateMessage(getReplica(command.replicaId()));
                //maybeReplicateConsumePos(getReplica(command.replicaId()));

            } catch (InterruptedException ie) {
                logger.info("Partition group {}/node {} replicate interrupted",
                        topicPartitionGroup, localReplicaId, ie);
                break;
            } catch (Throwable t) {
                logger.warn("Partition group {}/node {} replicate fail",
                        topicPartitionGroup, localReplicaId, t);
                try {
                    Thread.sleep(1000);
                } catch (Exception ignored) {
                }
            }
        }
    }
}
```

（1）初始化响应队列，initResponseQueue()方法 

只有延迟队列中对应副本的同步命令，同步线程才会执行副本同步的逻辑，所以初始化过程中，手动添加各个副本的同步命令 

```java
private void initResponseQueue() {
    replicateResponseQueue.clear();
    replicas.forEach((r) -> replicateResponseQueue.put(
            new DelayedCommand(0, r.replicaId())));
}
```

（2）通过replicateMessage()方法，实现对各个副本的数据同步操作 

该方法首先获取需要同步的数据，生成对应的请求以及command，并将command发送到对应的副本，并通过AppendEntriesRequestCallback回调来处理副本同步响应 

```java
private void replicateMessage(Replica replica) {
    try {
        replicateExecutor.submit(() -> {
            try {
                long startTimeUs = usTime();

                AppendEntriesRequest request = generateAppendEntriesRequest(replica);
                if (request == null) {
                    replicateResponseQueue.put(new DelayedCommand(ONE_MS_NANO, replica.replicaId()));
                    return;
                }

                JoyQueueHeader header = new JoyQueueHeader(Direction.REQUEST, CommandType.RAFT_APPEND_ENTRIES_REQUEST);


                if (!replica.isMatch() || logger.isDebugEnabled()) {
                    logger.info("Partition group {}/node {} send append entries request {} to node {}, " +
                                    "read entries elapse {} us",
                            topicPartitionGroup, leaderId, request, replica.replicaId(), usTime() - startTimeUs);
                }

                this.sendCommand(replica.getAddress(), new Command(header, request),
                        electionConfig.getSendCommandTimeout(),
                        new AppendEntriesRequestCallback(replica, startTimeUs, request.getEntriesLength()));

            } catch (Throwable t) {
                logger.warn("Partition group {}/ node {} send append entries to {} fail",
                        topicPartitionGroup, localReplicaId, replica.replicaId(), t);
                replicateResponseQueue.put(new DelayedCommand(ONE_SECOND_NANO, replica.replicaId()));
            }
        });
    } catch (Exception e) {
        logger.info("Partition group {}/node {} replicate message to {} fail",
                topicPartitionGroup, localReplicaId, replica.replicaId(), e);
        replicateResponseQueue.put(new DelayedCommand(ONE_SECOND_NANO, replica.replicaId()));
    }
}
```

（3）AppendEntriesRequestCallback回调中，如果数据同步成功，回执行onSuccess方法 

该方法中主要的处理逻辑是调用processAppendEntriesResponse方法，该方法主要用来处理数据同步的位置 

```java
@Override
public void onSuccess(Command request, Command response) {
    try {
        if (!(request.getPayload() instanceof AppendEntriesRequest)
                || !(response.getPayload() instanceof AppendEntriesResponse)) {
            return;
        }

        AppendEntriesRequest appendEntriesRequest = (AppendEntriesRequest)request.getPayload();
        AppendEntriesResponse appendEntriesResponse = (AppendEntriesResponse)response.getPayload();


        if (logger.isDebugEnabled() || usTime() - startTimeUs > MAX_PROCESS_TIME) {
            logger.info("Partition group {}/node {} receive append entries response from {}, " +
                            "success is {}, next position is {}, write position is {}, elapse {} us",
                    topicPartitionGroup, localReplicaId, replica.replicaId(), appendEntriesResponse.isSuccess(),
                    appendEntriesResponse.getNextPosition(), appendEntriesResponse.getWritePosition(),
                    usTime() - startTimeUs);
        }

        if (appendEntriesRequest.getTerm() != currentTerm) {
            logger.info("Partition group {}/node {} append entries request term {} not equals current term {}",
                    topicPartitionGroup, localReplicaId, appendEntriesRequest.getTerm(), currentTerm);
            return;
        }
        if (appendEntriesResponse.getTerm() > currentTerm) {
            logger.info("Partition group {}/node {} append entries response term {} not equals current term {}",
                    topicPartitionGroup, localReplicaId, appendEntriesResponse.getTerm(), currentTerm);
            leaderElection.stepDown(appendEntriesResponse.getTerm());
            return;
        }

        processAppendEntriesResponse(appendEntriesResponse, replica);

        brokerMonitor.onReplicateMessage(topicPartitionGroup.getTopic(), topicPartitionGroup.getPartitionGroupId(),
                1, entriesLength, usTime() - startTimeUs);

    } catch (Exception e) {
        logger.info("Partition group {}/node {} process append entries reponse fail",
                topicPartitionGroup, localReplicaId, e);
    } finally {
        replicateResponseQueue.put(new DelayedCommand(0, replica.replicaId()));
    }
}
```

（4）processAppendEntriesResponse方法首先更新同步数据的副本的同步位置，然后更新leader副本的写入位置（即rightPosition，可能有新的数据写入） 

然后是获取最终的commitPosition，获取方式是对replicasWithoutLearners进行排序，获取中间副本的writePosition作为commitPosition，保证多数副本同步 

最后提交commitPosition，同时触发发送请求的回调，因为有些请求的写入的日志级别是REPLICATION，需要多数副本同步完成之后才算发送成功 

```java
private synchronized void processAppendEntriesResponse(AppendEntriesResponse response, Replica replica) {
    replica.lastAppendSuccessTime(SystemClock.now());

    if (!response.isSuccess()) {
        if (response.getNextPosition() == -1L) {
            replica.nextPosition(getPrevPosition(replica.nextPosition()));
        } else {
            replica.nextPosition(getPrevPosition(response.getNextPosition()));
        }
        return;
    }

    replica.writePosition(response.getWritePosition());
    replica.nextPosition(response.getNextPosition());
    replica.setMatch(true);

    if (transferee != ElectionNode.INVALID_NODE_ID && replica.nextPosition() >= timeoutNowPosition) {
        sendTimeoutNowRequest(transferee);
    }
    // sync leader write position by the way
    getReplica(leaderId).writePosition(replicableStore.rightPosition());
    replicasWithoutLearners.sort((r1, r2) ->
            Long.compare(r2.writePosition(), r1.writePosition()));

    long commitPosition = replicasWithoutLearners.get(replicasWithoutLearners.size() / 2).writePosition();
    replicableStore.commit(commitPosition);

    if (null != brokerConfig && brokerConfig.getLogDetail(topicPartitionGroup.getTopic())) {
        replicas.forEach(r -> logger.info("Partition group {}/node {}", topicPartitionGroup, r));
        logger.info("Partition group {}/node {} commit position is {}",
                topicPartitionGroup, localReplicaId, replicableStore.commitPosition());
    }
}
```

