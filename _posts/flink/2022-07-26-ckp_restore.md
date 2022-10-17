---
title: Flink1.15 CheckPoint restore源码分析
tagline: ""
category : Flink1.15
layout: post
tags : [flink, realtime]
---

当所有task成功后，执行completePendingCheckpoint()操作

1. 将PendingCheckpoint 转成CompletedCheckpoint，并将CheckpointMetadata持久化
2. 更新completedCheckpointStore
3. 通知其他算子ack,执行commit
```
//CheckpointCoordinator
private void completePendingCheckpoint(PendingCheckpoint pendingCheckpoint)
throws CheckpointException {
    final long checkpointId = pendingCheckpoint.getCheckpointID();
    final CompletedCheckpoint completedCheckpoint;
    final CompletedCheckpoint lastSubsumed;
    final CheckpointProperties props = pendingCheckpoint.getProps();

    completedCheckpointStore.getSharedStateRegistry().checkpointCompleted(checkpointId);

    try {
        //将PendingCheckpoint 转成CompletedCheckpoint
        completedCheckpoint = finalizeCheckpoint(pendingCheckpoint);

        // the pending checkpoint must be discarded after the finalization
        Preconditions.checkState(pendingCheckpoint.isDisposed() && completedCheckpoint != null);
        //更新到completedCheckpointStore,
        //ZooKeeperStateHandleStore
        if (!props.isSavepoint()) {
            lastSubsumed =
            addCompletedCheckpointToStoreAndSubsumeOldest(
                checkpointId,
                completedCheckpoint,
                pendingCheckpoint.getCheckpointPlan().getTasksToCommitTo());
        } else {
            lastSubsumed = null;
        }

        reportCompletedCheckpoint(completedCheckpoint);
    } finally {
        pendingCheckpoints.remove(checkpointId);
        //异步ack
        scheduleTriggerRequest();
    }
    //ack-->sendAcknowledgeMessages-->notifyCheckpointComplete
    cleanupAfterCompletedCheckpoint(
        pendingCheckpoint, checkpointId, completedCheckpoint, lastSubsumed, props);
}

```
finalizeCheckpoint主要将PendingCheckpoint 转成CompletedCheckpoint，并持久化ckp元数据信息
```
public CompletedCheckpoint finalizeCheckpoint(
    CheckpointsCleaner checkpointsCleaner, Runnable postCleanup, Executor executor)
throws IOException {

    synchronized (lock) {
        checkState(!isDisposed(), "checkpoint is discarded");
        checkState(
            isFullyAcknowledged(),
            "Pending checkpoint has not been fully acknowledged yet");

        // make sure we fulfill the promise with an exception if something fails
        try {
            //填充operatorStates
            checkpointPlan.fulfillFinishedTaskStatus(operatorStates);
            //元数据信息
            // write out the metadata
            final CheckpointMetadata savepoint =
            new CheckpointMetadata(checkpointId, operatorStates.values(), masterStates);
            final CompletedCheckpointStorageLocation finalizedLocation;
            //持久化到文件,注意返回位置信息finalizedLocation
            try (CheckpointMetadataOutputStream out =
                 targetLocation.createMetadataOutputStream()) {
                Checkpoints.storeCheckpointMetadata(savepoint, out);
                finalizedLocation = out.closeAndFinalizeCheckpoint();
            }
            //包括ckp元数据信息位置finalizedLocation，统计信息stats
            CompletedCheckpoint completed =
            new CompletedCheckpoint(
                jobId,
                checkpointId,
                checkpointTimestamp,
                System.currentTimeMillis(),
                operatorStates,
                masterStates,
                props,
                finalizedLocation,
                toCompletedCheckpointStats(finalizedLocation));

            onCompletionPromise.complete(completed);

            // mark this pending checkpoint as disposed, but do NOT drop the state
            dispose(false, checkpointsCleaner, postCleanup, executor);

            return completed;
        } catch (Throwable t) {
            onCompletionPromise.completeExceptionally(t);
            ExceptionUtils.rethrowIOException(t);
            return null; // silence the compiler
        }
    }
}
```
addCompletedCheckpointToStoreAndSubsumeOldest 更新completedCheckpointStore<br />completedCheckpointStore的默认实现DefaultCompletedCheckpointStore，存储到zk<br />还有一个是StandaloneCompletedCheckpointStore，主要是JM的内存
```
private CompletedCheckpoint addCompletedCheckpointToStoreAndSubsumeOldest(
    long checkpointId,
    CompletedCheckpoint completedCheckpoint,
    List<ExecutionVertex> tasksToAbort)
throws CheckpointException {
    try {
        final CompletedCheckpoint subsumedCheckpoint =
        completedCheckpointStore.addCheckpointAndSubsumeOldestOne(
            completedCheckpoint, checkpointsCleaner, this::scheduleTriggerRequest);
        // reset the force full snapshot flag, we should've completed at least one full
        // snapshot by now
        this.forceFullSnapshot = false;
        return subsumedCheckpoint;
    } catch (Exception exception) {
        if (exception instanceof PossibleInconsistentStateException) {
            LOG.warn(
                "An error occurred while writing checkpoint {} to the underlying metadata"
                + " store. Flink was not able to determine whether the metadata was"
                + " successfully persisted. The corresponding state located at '{}'"
                + " won't be discarded and needs to be cleaned up manually.",
                completedCheckpoint.getCheckpointID(),
                completedCheckpoint.getExternalPointer());
        } else {
            // we failed to store the completed checkpoint. Let's clean up
            checkpointsCleaner.cleanCheckpointOnFailedStoring(completedCheckpoint, executor);
        }

        reportFailedCheckpoint(checkpointId, exception);
        sendAbortedMessages(tasksToAbort, checkpointId, completedCheckpoint.getTimestamp());
        throw new CheckpointException(
            "Could not complete the pending checkpoint " + checkpointId + '.',
            CheckpointFailureReason.FINALIZE_CHECKPOINT_FAILURE,
            exception);
    }
}
```
HA支持zookeeper或者k8s,对应两种实现ZooKeeperStateHandleStore或者KubernetesStateHandleStore
```
//DefaultCompletedCheckpointStore
public CompletedCheckpoint addCheckpointAndSubsumeOldestOne(
    final CompletedCheckpoint checkpoint,
    CheckpointsCleaner checkpointsCleaner,
    Runnable postCleanup)
throws Exception {
    Preconditions.checkState(running.get(), "Checkpoint store has already been shutdown.");
    checkNotNull(checkpoint, "Checkpoint");

    final String path =
    completedCheckpointStoreUtil.checkpointIDToName(checkpoint.getCheckpointID());
    //HA支持zookeeper或者k8s,对应两种实现ZooKeeperStateHandleStore或者KubernetesStateHandleStore
    // Now add the new one. If it fails, we don't want to lose existing data.
    checkpointStateHandleStore.addAndLock(path, checkpoint);

    completedCheckpoints.addLast(checkpoint);

    Optional<CompletedCheckpoint> subsume =
    CheckpointSubsumeHelper.subsume(
        completedCheckpoints,
        maxNumberOfCheckpointsToRetain,
        completedCheckpoint ->
        tryRemoveCompletedCheckpoint(
            completedCheckpoint,
            completedCheckpoint.shouldBeDiscardedOnSubsume(),
            checkpointsCleaner,
            postCleanup));
    unregisterUnusedState(completedCheckpoints);

    if (subsume.isPresent()) {
        LOG.debug("Added {} to {} without any older checkpoint to subsume.", checkpoint, path);
    } else {
        LOG.debug("Added {} to {} and subsume {}.", checkpoint, path, subsume);
    }
    return subsume.orElse(null);
}
```
ZooKeeperStateHandleStore
```
public RetrievableStateHandle<T> addAndLock(String pathInZooKeeper, T state)
throws PossibleInconsistentStateException, Exception {
    checkNotNull(pathInZooKeeper, "Path in ZooKeeper");
    checkNotNull(state, "State");
    final String path = normalizePath(pathInZooKeeper);
    final Optional<Stat> maybeStat = getStat(path);

    if (maybeStat.isPresent()) {
        if (isNotMarkedForDeletion(maybeStat.get())) {
            throw new AlreadyExistException(
                String.format("ZooKeeper node %s already exists.", path));
        }

        Preconditions.checkState(
            releaseAndTryRemove(path),
            "The state is marked for deletion and, therefore, should be deletable.");
    }
    这里store是FileSystemStateStorageHelper
    final RetrievableStateHandle<T> storeHandle = storage.store(state);
    final byte[] serializedStoreHandle = serializeOrDiscard(storeHandle);
    try {
        writeStoreHandleTransactionally(path, serializedStoreHandle);
        return storeHandle;
    } catch (KeeperException.NodeExistsException e) {
        // Transactions are not idempotent in the curator version we're currently using, so it
        // is actually possible that we've re-tried a transaction that has already succeeded.
        // We've ensured that the node hasn't been present prior executing the transaction, so
        // we can assume that this is a result of the retry mechanism.
        return storeHandle;
    } catch (Exception e) {
        if (indicatesPossiblyInconsistentState(e)) {
            throw new PossibleInconsistentStateException(e);
        }
        // In case of any other failure, discard the state and rethrow the exception.
        storeHandle.discardState();
        throw e;
    }
}
```
FileSystemStateStorageHelper
```
public RetrievableStateHandle<T> store(T state) throws Exception {
    Exception latestException = null;

    for (int attempt = 0; attempt < 10; attempt++) {
        Path filePath = getNewFilePath();
        //写文件
        try (FSDataOutputStream outStream =
             fs.create(filePath, FileSystem.WriteMode.NO_OVERWRITE)) {
            InstantiationUtil.serializeObject(outStream, state);
            return new RetrievableStreamStateHandle<T>(filePath, outStream.getPos());
        } catch (Exception e) {
            latestException = e;
        }
    }

    throw new Exception("Could not open output stream for state backend", latestException);
}
```
恢复
```
//创建 completedCheckpointStore
this.completedCheckpointStore =
SchedulerUtils.createCompletedCheckpointStoreIfCheckpointingIsEnabled(
    jobGraph,
    jobMasterConfiguration,
    checkNotNull(checkpointRecoveryFactory),
    ioExecutor,
    log);
//创建与
this.executionGraph =
createAndRestoreExecutionGraph(
    completedCheckpointStore,
    checkpointsCleaner,
    checkpointIdCounter,
    initializationTimestamp,
    mainThreadExecutor,
    jobStatusListener,
    vertexParallelismStore);
```
DefaultExecutionGraphFactory<br />如果checkpointCoordinator存在则尝试从Checkpoint恢复<br />如果ckp没有，则尝试从savepoint恢复
```
public ExecutionGraph createAndRestoreExecutionGraph(
    JobGraph jobGraph,
    CompletedCheckpointStore completedCheckpointStore,
    CheckpointsCleaner checkpointsCleaner,
    CheckpointIDCounter checkpointIdCounter,
    TaskDeploymentDescriptorFactory.PartitionLocationConstraint partitionLocationConstraint,
    long initializationTimestamp,
    VertexAttemptNumberStore vertexAttemptNumberStore,
    VertexParallelismStore vertexParallelismStore,
    ExecutionStateUpdateListener executionStateUpdateListener,
    Logger log)
throws Exception {
    ExecutionDeploymentListener executionDeploymentListener =
    new ExecutionDeploymentTrackerDeploymentListenerAdapter(executionDeploymentTracker);
    ExecutionStateUpdateListener combinedExecutionStateUpdateListener =
    (execution, previousState, newState) -> {
        executionStateUpdateListener.onStateUpdate(execution, previousState, newState);
        if (newState.isTerminal()) {
            executionDeploymentTracker.stopTrackingDeploymentOf(execution);
        }
    };

    final ExecutionGraph newExecutionGraph =
    DefaultExecutionGraphBuilder.buildGraph(
        jobGraph,
        configuration,
        futureExecutor,
        ioExecutor,
        userCodeClassLoader,
        completedCheckpointStore,
        checkpointsCleaner,
        checkpointIdCounter,
        rpcTimeout,
        blobWriter,
        log,
        shuffleMaster,
        jobMasterPartitionTracker,
        partitionLocationConstraint,
        executionDeploymentListener,
        combinedExecutionStateUpdateListener,
        initializationTimestamp,
        vertexAttemptNumberStore,
        vertexParallelismStore,
        checkpointStatsTrackerFactory,
        isDynamicGraph);

    final CheckpointCoordinator checkpointCoordinator =
    newExecutionGraph.getCheckpointCoordinator();
    //如果checkpointCoordinator存在则尝试从Checkpoint恢复
    if (checkpointCoordinator != null) {
        // check whether we find a valid checkpoint
        if (!checkpointCoordinator.restoreInitialCheckpointIfPresent(
            new HashSet<>(newExecutionGraph.getAllVertices().values()))) {

            // check whether we can restore from a savepoint
            tryRestoreExecutionGraphFromSavepoint(
                newExecutionGraph, jobGraph.getSavepointRestoreSettings());
        }
    }

    return newExecutionGraph;
}
```
