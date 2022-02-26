raft会随机休眠一段时间，ZAB则是不停地更新投票，如果自己收到选票但还没发起就直接赞同(同时苏醒就重新选)。数据提交和zab一样都是2阶段，脑裂就会写不成功，无法获得半数以上的支持。新版嵌入jraft
- 心跳
会携带少量key，leader会50个一组去向leader拿数据
## 4.com.alibaba.nacos.naming.consistency.persistent.raft.RaftCore#signalPublish
```java
public void signalPublish(String key, Record value) throws Exception {
    if (stopWork) {
        throw new IllegalStateException("old raft protocol already stop work");
    }
    if (!isLeader()) {
    	// 如果不是leader则进行转发
        ObjectNode params = JacksonUtils.createEmptyJsonNode();
        params.put("key", key);
        params.replace("value", JacksonUtils.transferToJsonNode(value));
        Map<String, String> parameters = new HashMap<>(1);
        parameters.put("key", key);
        
        final RaftPeer leader = getLeader();
        // 转发
        raftProxy.proxyPostLarge(leader.ip, API_PUB, params.toString(), parameters);
        return;
    }
    
    OPERATE_LOCK.lock();
    try {
        final long start = System.currentTimeMillis();
        final Datum datum = new Datum();
        datum.key = key;
        datum.value = value;
        if (getDatum(key) == null) {
            datum.timestamp.set(1L);
        } else {
            datum.timestamp.set(getDatum(key).timestamp.incrementAndGet());
        }
        
        ObjectNode json = JacksonUtils.createEmptyJsonNode();
        json.replace("datum", JacksonUtils.transferToJsonNode(datum));
        json.replace("source", JacksonUtils.transferToJsonNode(peers.local()));
        
        onPublish(datum, peers.local());
        
        final String content = json.toString();
        
        final CountDownLatch latch = new CountDownLatch(peers.majorityCount());
        for (final String server : peers.allServersIncludeMyself()) {
            if (isLeader(server)) {
                latch.countDown();
                continue;
            }
            final String url = buildUrl(server, API_ON_PUB);
            HttpClient.asyncHttpPostLarge(url, Arrays.asList("key", key), content, new Callback<String>() {
                @Override
                public void onReceive(RestResult<String> result) {
                    if (!result.ok()) {
                        Loggers.RAFT
                                .warn("[RAFT] failed to publish data to peer, datumId={}, peer={}, http code={}",
                                        datum.key, server, result.getCode());
                        return;
                    }
                    latch.countDown();
                }
                
                @Override
                public void onError(Throwable throwable) {
                    Loggers.RAFT.error("[RAFT] failed to publish data to peer", throwable);
                }
                
                @Override
                public void onCancel() {
                
                }
            });
            
        }
        
        if (!latch.await(UtilsAndCommons.RAFT_PUBLISH_TIMEOUT, TimeUnit.MILLISECONDS)) {
            // only majority servers return success can we consider this update success
            Loggers.RAFT.error("data publish failed, caused failed to notify majority, key={}", key);
            throw new IllegalStateException("data publish failed, caused failed to notify majority, key=" + key);
        }
        
        long end = System.currentTimeMillis();
        Loggers.RAFT.info("signalPublish cost {} ms, key: {}", (end - start), key);
    } finally {
        OPERATE_LOCK.unlock();
    }
}
```
## 4.1com.alibaba.nacos.naming.consistency.persistent.raft.RaftCore#onPublish
```java
public void onPublish(Datum datum, RaftPeer source) throws Exception {
    if (stopWork) {
        throw new IllegalStateException("old raft protocol already stop work");
    }
    RaftPeer local = peers.local();
    if (datum.value == null) {
        Loggers.RAFT.warn("received empty datum");
        throw new IllegalStateException("received empty datum");
    }
    
    if (!peers.isLeader(source.ip)) {
        Loggers.RAFT
                .warn("peer {} tried to publish data but wasn't leader, leader: {}", JacksonUtils.toJson(source),
                        JacksonUtils.toJson(getLeader()));
        throw new IllegalStateException("peer(" + source.ip + ") tried to publish " + "data but wasn't leader");
    }
    
    if (source.term.get() < local.term.get()) {
        Loggers.RAFT.warn("out of date publish, pub-term: {}, cur-term: {}", JacksonUtils.toJson(source),
                JacksonUtils.toJson(local));
        throw new IllegalStateException(
                "out of date publish, pub-term:" + source.term.get() + ", cur-term: " + local.term.get());
    }
    
    local.resetLeaderDue();
    
    // if data should be persisted, usually this is true:
    if (KeyBuilder.matchPersistentKey(datum.key)) {
    	// 刷盘
        raftStore.write(datum);
    }
    
    datums.put(datum.key, datum);
    
    if (isLeader()) {
        local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
    } else {
        if (local.term.get() + PUBLISH_TERM_INCREASE_COUNT > source.term.get()) {
            //set leader term:
            getLeader().term.set(source.term.get());
            local.term.set(getLeader().term.get());
        } else {
            local.term.addAndGet(PUBLISH_TERM_INCREASE_COUNT);
        }
    }
    raftStore.updateTerm(local.term.get());
    // 发布事件同步数据库
    NotifyCenter.publishEvent(ValueChangeEvent.builder().key(datum.key).action(DataOperation.CHANGE).build());
    Loggers.RAFT.info("data added/updated, key={}, term={}", datum.key, local.term);
}
```