---
title:
- JanusGraph Index Problem
archives:
- /archives
date:
- 2020-06-14 14:00:00
categories:
- janusgraph
- index
tags:
- janusgraph
- index
comments:
- false
photos:
- /static/images/zelda.jpg
---

# JanusGraph索引问题整理

## 一. 解决JanusGraph索引更新失败导致数据不一致的场景

<p style="text-indent: 2em">JanusGraph
虽然可以支持事务,但其原子性仅保证了更新存储后端时该事务是原子操作,当更新索引后端数据时可能存在失败的场景,此时若未出现其他问题,仅更新索引失败,则事务不会回滚.因此,可能导致索引后端和存储后端数据不一致的场景.</p>

<!-- more -->
### 1. 服务版本列表

| 服务名 | Version |
| :---: | :---: |
|JanusGraph|0.4.x|
|Cassandra|3.11.x|
|ElasticSearch|5.6.x|

### 2. JanusGraph索引操作非事务操作

<p style="text-indent: 2em">JanusGraph提交事务时,若索引操作失败,不会导致事务回滚,因此存在一定的数据不一致的风险,详见代码(Commit indexes部分,仅会打印索引操作的错误日志)</p>

```java StandardJanusGraph.java
   public void commit(final Collection<InternalRelation> addedRelations,
                     final Collection<InternalRelation> deletedRelations, final StandardJanusGraphTx tx) {
        if (addedRelations.isEmpty() && deletedRelations.isEmpty()) return;
        //1. Finalize transaction
        log.debug("Saving transaction. Added {}, removed {}", addedRelations.size(), deletedRelations.size());
        if (!tx.getConfiguration().hasCommitTime()) tx.getConfiguration().setCommitTime(times.getTime());
        final Instant txTimestamp = tx.getConfiguration().getCommitTime();
        final long transactionId = txCounter.incrementAndGet();

        //2. Assign JanusGraphVertex IDs
        if (!tx.getConfiguration().hasAssignIDsImmediately())
            idAssigner.assignIDs(addedRelations);

        //3. Commit
        BackendTransaction mutator = tx.getTxHandle();
        final boolean acquireLocks = tx.getConfiguration().hasAcquireLocks();
        final boolean hasTxIsolation = backend.getStoreFeatures().hasTxIsolation();
        final boolean logTransaction = config.hasLogTransactions() && !tx.getConfiguration().hasEnabledBatchLoading();
        final KCVSLog txLog = logTransaction?backend.getSystemTxLog():null;
        final TransactionLogHeader txLogHeader = new TransactionLogHeader(transactionId,txTimestamp, times);
        ModificationSummary commitSummary;

        try {
            //3.1 Log transaction (write-ahead log) if enabled
            if (logTransaction) {
                //[FAILURE] Inability to log transaction fails the transaction by escalation since it's likely due to unavailability of primary
                //storage backend.
                Preconditions.checkNotNull(txLog, "Transaction log is null");
                txLog.add(txLogHeader.serializeModifications(serializer, LogTxStatus.PRECOMMIT, tx, addedRelations, deletedRelations),txLogHeader.getLogKey());
            }

            //3.2 Commit schema elements and their associated relations in a separate transaction if backend does not support
            //    transactional isolation
            boolean hasSchemaElements = !Iterables.isEmpty(Iterables.filter(deletedRelations,SCHEMA_FILTER))
                    || !Iterables.isEmpty(Iterables.filter(addedRelations,SCHEMA_FILTER));
            Preconditions.checkArgument(!hasSchemaElements || (!tx.getConfiguration().hasEnabledBatchLoading() && acquireLocks),
                    "Attempting to create schema elements in inconsistent state");

            if (hasSchemaElements && !hasTxIsolation) {
                /*
                 * On storage without transactional isolation, create separate
                 * backend transaction for schema aspects to make sure that
                 * those are persisted prior to and independently of other
                 * mutations in the tx. If the storage supports transactional
                 * isolation, then don't create a separate tx.
                 */
                final BackendTransaction schemaMutator = openBackendTransaction(tx);

                try {
                    //[FAILURE] If the preparation throws an exception abort directly - nothing persisted since batch-loading cannot be enabled for schema elements
                    commitSummary = prepareCommit(addedRelations,deletedRelations, SCHEMA_FILTER, schemaMutator, tx, acquireLocks);
                    assert commitSummary.hasModifications && !commitSummary.has2iModifications;
                } catch (Throwable e) {
                    //Roll back schema tx and escalate exception
                    schemaMutator.rollback();
                    throw e;
                }

                try {
                    schemaMutator.commit();
                } catch (Throwable e) {
                    //[FAILURE] Primary persistence failed => abort and escalate exception, nothing should have been persisted
                    log.error("Could not commit transaction ["+transactionId+"] due to storage exception in system-commit",e);
                    throw e;
                }
            }

            //[FAILURE] Exceptions during preparation here cause the entire transaction to fail on transactional systems
            //or just the non-system part on others. Nothing has been persisted unless batch-loading
            commitSummary = prepareCommit(addedRelations,deletedRelations, hasTxIsolation? NO_FILTER : NO_SCHEMA_FILTER, mutator, tx, acquireLocks);
            if (commitSummary.hasModifications) {
                String logTxIdentifier = tx.getConfiguration().getLogIdentifier();
                boolean hasSecondaryPersistence = logTxIdentifier!=null || commitSummary.has2iModifications;

                //1. Commit storage - failures lead to immediate abort

                //1a. Add success message to tx log which will be committed atomically with all transactional changes so that we can recover secondary failures
                //    This should not throw an exception since the mutations are just cached. If it does, it will be escalated since its critical
                if (logTransaction) {
                    txLog.add(txLogHeader.serializePrimary(serializer,
                                        hasSecondaryPersistence?LogTxStatus.PRIMARY_SUCCESS:LogTxStatus.COMPLETE_SUCCESS),
                            txLogHeader.getLogKey(),mutator.getTxLogPersistor());
                }

                try {
                    mutator.commitStorage();
                } catch (Throwable e) {
                    //[FAILURE] If primary storage persistence fails abort directly (only schema could have been persisted)
                    log.error("Could not commit transaction ["+transactionId+"] due to storage exception in commit",e);
                    throw e;
                }

                if (hasSecondaryPersistence) {
                    LogTxStatus status = LogTxStatus.SECONDARY_SUCCESS;
                    Map<String,Throwable> indexFailures = ImmutableMap.of();
                    boolean userlogSuccess = true;

                    try {
                        //2. Commit indexes - [FAILURE] all exceptions are collected and logged but nothing is aborted
                        indexFailures = mutator.commitIndexes();
                        if (!indexFailures.isEmpty()) {
                            status = LogTxStatus.SECONDARY_FAILURE;
                            for (Map.Entry<String,Throwable> entry : indexFailures.entrySet()) {
                                log.error("Error while committing index mutations for transaction ["+transactionId+"] on index: " +entry.getKey(),entry.getValue());
                            }
                        }
                        //3. Log transaction if configured - [FAILURE] is recorded but does not cause exception
                        if (logTxIdentifier!=null) {
                            try {
                                userlogSuccess = false;
                                final Log userLog = backend.getUserLog(logTxIdentifier);
                                Future<Message> env = userLog.add(txLogHeader.serializeModifications(serializer, LogTxStatus.USER_LOG, tx, addedRelations, deletedRelations));
                                if (env.isDone()) {
                                    try {
                                        env.get();
                                    } catch (ExecutionException ex) {
                                        throw ex.getCause();
                                    }
                                }
                                userlogSuccess=true;
                            } catch (Throwable e) {
                                status = LogTxStatus.SECONDARY_FAILURE;
                                log.error("Could not user-log committed transaction ["+transactionId+"] to " + logTxIdentifier, e);
                            }
                        }
                    } finally {
                        if (logTransaction) {
                            //[FAILURE] An exception here will be logged and not escalated; tx considered success and
                            // needs to be cleaned up later
                            try {
                                txLog.add(txLogHeader.serializeSecondary(serializer,status,indexFailures,userlogSuccess),txLogHeader.getLogKey());
                            } catch (Throwable e) {
                                log.error("Could not tx-log secondary persistence status on transaction ["+transactionId+"]",e);
                            }
                        }
                    }
                } else {
                    //This just closes the transaction since there are no modifications
                    mutator.commitIndexes();
                }
            } else { //Just commit everything at once
                //[FAILURE] This case only happens when there are no non-system mutations in which case all changes
                //are already flushed. Hence, an exception here is unlikely and should abort
                mutator.commit();
            }
        } catch (Throwable e) {
            log.error("Could not commit transaction ["+transactionId+"] due to exception",e);
            try {
                //Clean up any left-over transaction handles
                mutator.rollback();
            } catch (Throwable e2) {
                log.error("Could not roll-back transaction ["+transactionId+"] after failure due to exception",e2);
            }
            if (e instanceof RuntimeException) throw (RuntimeException)e;
            else throw new JanusGraphException("Unexpected exception",e);
        }
    }

```

### 3. 索引操作执行失败的场景总结

#### 3.1 索引并发更新时锁冲突

```text
154481719 [gremlin-server-exec-6] ERROR org.janusgraph.diskstorage.es.rest.RestElasticSearchClient  - Failed to execute ES query: {type=version_conflict_engine_exception, reason=[byFuzzySearchMixedIndex][12jjc]: version conflict, current version [3] is different than the one provided [2], index_uuid=Zi4DOgwcT8WEjQu-bSJemw, shard=4, index=test__byfuzzysearchmixedindex}
```

<p style="text-indent: 2em">该场景的原理比较简单,主要是程序并发修改索引的某一文档时(此时不同请求获取的版本号一致),当其中一个请求修改成功后,版本号改变,其他请求修改时会首先判断版本号是否一致,若不一致,则该次请求失败.</p>

<p style="text-indent: 2em">以上场景最简单的解决方案就是重试,但老版本(目前0.5.x以前的版本应该都没有,只确认了一部分)的JanusGraph中Es
的客户端实现中没有添加重试的配置,最快速的方法时升级JanusGraph版本至0.5.0以上</p>

<p style="text-indent: 2em">es客户端重试配置的添加可见: {% link issues#1797 https://github.com/JanusGraph/janusgraph/issues/1797 %} </p>

#### 3.2 es某时间段编译大量脚本报错

```text
154481719 [gremlin-server-exec-6] ERROR org.janusgraph.diskstorage.es.ElasticSearchIndex  - Failed to execute bulk Elasticsearch mutation
java.io.IOException: Failure(s) in Elasicsearch bulk request: [{type=illegal_argument_exception, reason=failed to execute script, caused_by={type=general_script_exception, reason=Failed to compile inline script [ctx._source.remove("*hidden:bp");ctx._source.remove("*hidden:bp__STRING");ctx._source.remove("*hidden:dp");ctx._source.remove("*hidden:dp__STRING");ctx._source.remove("*hidden:up");ctx._source.remove("*hidden:up__STRING");ctx._source.remove("*hidden:nc");ctx._source.remove("*hidden:nc__STRING");ctx._source.remove("*hidden:tu");ctx._source.remove("*hidden:tu__STRING");] using lang [painless], caused_by={type=circuit_breaking_exception, reason=[script] Too many dynamic script compilations within one minute, max: [15/min]; please use on-disk, indexed, or scripts with parameters instead; this limit can be changed by the [script.max_compilations_per_minute] setting, bytes_wanted=0, bytes_limit=0}}}]
	at org.janusgraph.diskstorage.es.rest.RestElasticSearchClient.bulkRequest(RestElasticSearchClient.java:258)
	at org.janusgraph.diskstorage.es.ElasticSearchIndex.mutate(ElasticSearchIndex.java:601)
	at org.janusgraph.diskstorage.indexing.IndexTransaction$1.call(IndexTransaction.java:160)
	at org.janusgraph.diskstorage.indexing.IndexTransaction$1.call(IndexTransaction.java:157)
	at org.janusgraph.diskstorage.util.BackendOperation.executeDirect(BackendOperation.java:69)
	at org.janusgraph.diskstorage.util.BackendOperation.execute(BackendOperation.java:55)
	at org.janusgraph.diskstorage.indexing.IndexTransaction.flushInternal(IndexTransaction.java:157)
	at org.janusgraph.diskstorage.indexing.IndexTransaction.commit(IndexTransaction.java:138)
	at org.janusgraph.diskstorage.BackendTransaction.commitIndexes(BackendTransaction.java:141)
	at org.janusgraph.graphdb.database.StandardJanusGraph.commit(StandardJanusGraph.java:765)
	at org.janusgraph.graphdb.transaction.StandardJanusGraphTx.commit(StandardJanusGraphTx.java:1374)
	at org.janusgraph.graphdb.tinkerpop.JanusGraphBlueprintsGraph$GraphTransaction.doCommit(JanusGraphBlueprintsGraph.java:272)
	at org.apache.tinkerpop.gremlin.structure.util.AbstractTransaction.commit(AbstractTransaction.java:105)
	at org.apache.tinkerpop.gremlin.structure.Transaction$commit$4.call(Unknown Source)
	at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCall(CallSiteArray.java:48)
	at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:113)
	at org.codehaus.groovy.runtime.callsite.AbstractCallSite.call(AbstractCallSite.java:117)
	at Script27489.run(Script27489.groovy:1)
	at org.apache.tinkerpop.gremlin.groovy.jsr223.GremlinGroovyScriptEngine.eval(GremlinGroovyScriptEngine.java:843)
	at org.apache.tinkerpop.gremlin.groovy.jsr223.GremlinGroovyScriptEngine.eval(GremlinGroovyScriptEngine.java:548)
	at javax.script.AbstractScriptEngine.eval(AbstractScriptEngine.java:233)
	at org.apache.tinkerpop.gremlin.groovy.engine.ScriptEngines.eval(ScriptEngines.java:120)
	at org.apache.tinkerpop.gremlin.groovy.engine.GremlinExecutor.lambda$eval$0(GremlinExecutor.java:290)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

<p style="text-indent: 2em">该错误主要由于某段时间编译了大量的脚本,导致触发了ES的保护机制,拒绝了部分脚本的编译请求.</p>
<p style="text-indent: 2em">最简单的解决方案就是调大限制,但是不可避免的会给ES集群造成压力,因此,不采用当前方案;另一种方案就是想办法从根本上解决大量脚本编译的问题.</p>
<p style="text-indent: 2em">在我们的使用场景下,通常在大量删除节点的属性(相关属性创建了MixedIndex索引)时会出现这个问题,对具体的场景进行分析发现,当我们在删除节点属性时,JanusGraph
会向ES发送fields删除的脚本.首先,这部分脚本没有进行参数化;其次,JanusGraph
构造脚本时,属性列表顺序并不是有序的,因此,虽然我们的使用场景中每次删除的都是不同节点上的相同属性,但可能每次执行删除操作时构造的脚本都不同,发送至ES后,ES每次都需重新编译,所以触发了这个问题.
</p>
<p style="text-indent: 2em">在JanusGraph0.5.x版本后,修复了这个问题,删除Fields时,使用的是参数化的语句,且脚本已经提前提交至ES进行了编译.</p>
<p style="text-indent: 2em">但是很可惜,我们的使用场景中无法升级JanusGraph版本至0.5.x,因为该版本以后JanusGraph不在支持ES5,我们的项目还不能升级ES
版本,因此只能对之前的JanusGraph的代码进行修改.相关的修改如下:
</p>


未完待续!