---
layout:     post
title:      "JoyQueue源码-Leader选举"
date:       2021-06-23 20:07:00
categories: 技术
tags: JoyQueue
---

开篇诗两句：

​	**白兔捣药秋复春，嫦娥孤栖与谁邻？**

​	**今人不见古时月，今月曾经照古人。**	——李白《把酒问月》

<br/>


## Leader选举

JoyQueue的leader副本选举与Kafka方式不同，采用的raft协议，同时数据一致性的保证也是采用的raft协议保证的

**ReplicaGroup通过**RaftLeaderElection进行leader选举的相关操作。

RaftLeaderElection启动时回启动选举定时器，当选举定时器超时，会重新触发一次leader选举。

```java
private synchronized void resetElectionTimer() {
    if (electionTimerFuture != null && !electionTimerFuture.isDone()) {
        electionTimerFuture.cancel(true);
        electionTimerFuture = null;
    }
    electionTimerFuture = electionTimerExecutor.schedule(this::handleElectionTimeout,
            getElectionTimeoutMs(), TimeUnit.MILLISECONDS);
}
```

选举超时时间是随机生成的，根据raft协议，通过随机时间，尽量减少选举冲突导致频繁选举的情况出现

```java
private int getElectionTimeoutMs() {
    Random random = new Random();
    return electionConfig.getElectionTimeout() + random.nextInt(electionConfig.getElectionTimeout());

}
```

当选举超时之后，会调用handleElectionTimeout方法进行leader选举：

（1）该方法中判断，如果节点列表只有自己，那么直接选举leader，becomeLeader方法

（2）执行preVote方法，进行预选举

```java
private synchronized void handleElectionTimeout() {
      if (!isStarted()) {
          throw new IllegalStateException("Election timeout, election service not start");
      }

      if (electionTimerFuture == null) {
          logger.info("Partition group {}/node {} election timeout, timer future is null",
                  topicPartitionGroup, localNode);
      }

      /*
       * 如果只有一个节点，直接设置该节点为leader
       */
      if (getAllNodes().size() == 1) {
          becomeLeader();
          return;
      }

      if (state() != FOLLOWER) {
          logger.info("Partition group {}/node {} election timeout, state is {}",
                  topicPartitionGroup, localNode, state());
           if (state() == LEADER) {
              return;
           }
      }

      logger.info("Partition group {}/node {} election timeout, current term is {}.",
              topicPartitionGroup, localNode, currentTerm);

      leaderId = INVALID_NODE_ID;

      try {
          preVote();
      } catch (Throwable t) {
          logger.warn("Partition group {}/node {} preVote fail",
                  topicPartitionGroup, localNode);
      }
      resetElectionTimer();
  }
```

preVote方法向所有其他节点发送预选举命令，并等待VoteRequestCallBack回调

```java
private void preVote() {
    localNode.setVoteGranted(true);

    int lastLogTerm = getLastLogTerm();
    long lastLogPos = getLastLogPosition();

    for (ElectionNode node : getAllNodes()) {
        if (node.equals(localNode)) {
            continue;
        }

        electionExecutor.submit(() -> {
            node.setVoteGranted(false);
            VoteRequest voteRequest = new VoteRequest(topicPartitionGroup, currentTerm, localNodeId,
                    lastLogTerm, lastLogPos, true);
            JoyQueueHeader header = new JoyQueueHeader(Direction.REQUEST, CommandType.RAFT_VOTE_REQUEST);
            Command command = new Command(header, voteRequest);

            logger.info("Partition group {}/node{} send prevote request to node {}",
                    topicPartitionGroup, localNode, node);

            try {
                electionManager.sendCommand(node.getAddress(), command,
                        electionConfig.getSendCommandTimeout(), new VoteRequestCallback(currentTerm, node));
            } catch (Exception e) {
                logger.info("Partition group {}/node{} send pre vote request to node {} fail",
                        topicPartitionGroup, localNode, node, e);
            }
        });
    }
}
```

在回调方法中，会判断当前是预选举还是真实选举：

（1）预选举：调用handlePreVoteResponse方法

（2）真实选举：调用handleVoteResponse方法

handlePreVoteResponse方法中处理预选举的响应结果：

主要逻辑就是根据响应结果，判断其他节点选举自己的数量是否超过一半，如果超过一半则调用electSelf方法执行真实选举流程

```java
private synchronized void handlePreVoteResponse(Command command) {
    if (command == null) {
        logger.warn("Partition group {}/node{} receive pre vote response is null",
                topicPartitionGroup, localNode);
        return;
    }

    if (!(command.getPayload() instanceof VoteResponse)) {
        logger.info("Partition group {}/node{} receive pre vote response object type error",
                topicPartitionGroup, localNode);
        return;
    }
    VoteResponse voteResponse = (VoteResponse)command.getPayload();

    logger.info("Partition group {}/node{} receive pre vote response from {}, term is {}, " +
                "vote candidateId is {}, vote granted is {}",
            topicPartitionGroup, localNode, voteResponse.getVoteNodeId(), voteResponse.getTerm(),
            voteResponse.getCandidateId(), voteResponse.isVoteGranted());

    if (state() != FOLLOWER) {
        logger.info("Partition group {}/node {} receive pre vote response, state is {}",
                topicPartitionGroup, localNode, state());
        return;
    }
    if (voteResponse.getTerm() > currentTerm) {
        logger.info("Partition group {}/node{} receive pre vote response, current term is {}, " +
                    "response term is {}",
                topicPartitionGroup, localNode, currentTerm, voteResponse.getTerm());
        stepDown(voteResponse.getTerm());
        return;
    }

    ElectionNode voteNode = getNode(voteResponse.getVoteNodeId());
    voteNode.setVoteGranted(voteResponse.isVoteGranted());
    int voteGranted = 0;
    for (ElectionNode node : getAllNodes()) {
        logger.info("Partition group {}/node {} pre vote voteGranted is {}",
                topicPartitionGroup, node, node.isVoteGranted());
        if (node.isVoteGranted()) {
            voteGranted++;
        }
    }

    logger.info("Partition group {}/node {} receive {} pre votes",
            topicPartitionGroup, localNode, voteGranted);

    // if granted quorum, become leader
    if (voteGranted > (getAllNodes().size()) / 2) {
        logger.info("Partition group {}/node{} receive {} pre votes, start vote, term is {}.",
                topicPartitionGroup, localNode, voteGranted, currentTerm);
        electSelf();
    }
}
```

electSelf方法首先选举自己为候选者，然后向其他节点发送投票请求，主要流程如下：

（1）选举自己为候选者CONDIDATE

（2）重置选举计时器，如果选举计时器超时，则取消选举，重新进行预选举

（3）向所有节点发送选举请求，并执行VoteRequestCallback回调

```java
private void electSelf() {
    currentTerm++;
    transitionTo(CONDIDATE);
    leaderId = INVALID_NODE_ID;
    votedFor = localNode.getNodeId();
    localNode.setVoteGranted(true);

    nodeOffline(currentTerm);

    updateElectionMetadata();
    electionEventManager.add(new ElectionEvent(START_ELECTION,
            currentTerm, INVALID_NODE_ID, topicPartitionGroup));

    resetVoteTimer();

    int lastLogTerm = getLastLogTerm();
    long lastLogPos = getLastLogPosition();

    for (ElectionNode node : getAllNodes()) {
        if (node.equals(localNode)) {
            continue;
        }

        electionExecutor.submit(() -> {
            node.setVoteGranted(false);
            VoteRequest voteRequest = new VoteRequest(topicPartitionGroup, currentTerm, localNodeId,
                    lastLogTerm, lastLogPos, false);
            JoyQueueHeader header = new JoyQueueHeader(Direction.REQUEST, CommandType.RAFT_VOTE_REQUEST);
            Command command = new Command(header, voteRequest);

            logger.info("Partition group {}/node{} send vote request to node {}",
                    topicPartitionGroup, localNode, node);

            try {
                electionManager.sendCommand(node.getAddress(), command,
                        electionConfig.getSendCommandTimeout(), new VoteRequestCallback(currentTerm, node));
            } catch (Exception e) {
                logger.info("Partition group {}/node{} send vote request to node {} fail",
                        topicPartitionGroup, localNode, node, e);
            }
        });
    }
}
```

回调中，会调用handleVoteResponse方法处理选举响应

该方法中判断，如果投票数量超过一半则调用becomeLeader方法成为leader

```java
private synchronized void handleVoteResponse(Command command) {
    if (command == null) {
        logger.warn("Partition group {}/node{} receive vote response is null",
                topicPartitionGroup, localNode);
        return;
    }

    if (!(command.getPayload() instanceof VoteResponse)) {
        logger.info("Partition group {}/node{} receive vote response object type error",
                topicPartitionGroup, localNode);
        return;
    }
    VoteResponse voteResponse = (VoteResponse)command.getPayload();

    logger.info("Partition group {}/node{} receive vote response from {}, term is {}, " +
                "vote candidateId is {}, vote granted is {}",
            topicPartitionGroup, localNode, voteResponse.getVoteNodeId(), voteResponse.getTerm(),
            voteResponse.getCandidateId(), voteResponse.isVoteGranted());
    if (state() != CONDIDATE) {
        logger.warn("Partition group {}/node{} receive vote response, local node state is {}",
                topicPartitionGroup, localNode, state());
        return;
    }

    if (voteResponse.getTerm() > currentTerm) {
        logger.info("Partition group {}/node{} receive vote response, current term is {}, " +
                    "response term is {}",
                topicPartitionGroup, localNode, currentTerm, voteResponse.getTerm());
        stepDown(voteResponse.getTerm());
    }

    ElectionNode voteNode = getNode(voteResponse.getVoteNodeId());
    voteNode.setVoteGranted(voteResponse.isVoteGranted());
    int voteGranted = 0;
    for (ElectionNode node : getAllNodes()) {
        logger.info("Partition group {}/node {} voteGranted is {}",
                topicPartitionGroup, node, node.isVoteGranted());
        if (node.isVoteGranted()) {
            voteGranted++;
        }
    }

    logger.info("Partition group {}/node {} receive {} votes",
            topicPartitionGroup, localNode, voteGranted);

    // if granted quorum, become leader
    if (voteGranted > (getAllNodes().size()) / 2) {
        logger.info("Partition group {}/node{} receive {} votes, become leader term is {}.",
                topicPartitionGroup, localNode, voteGranted, currentTerm);
        becomeLeader();
    }
}
```

