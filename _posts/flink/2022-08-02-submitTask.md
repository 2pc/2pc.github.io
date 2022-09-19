---
title: Flink1.15 Task源码分析
tagline: ""
category : Flink1.15
layout: post
tags : [flink, realtime]
---
DefaultScheduler.allocateSlotsAndDeploy
```
public void allocateSlotsAndDeploy(
    final List<ExecutionVertexDeploymentOption> executionVertexDeploymentOptions) {
    validateDeploymentOptions(executionVertexDeploymentOptions);

    final Map<ExecutionVertexID, ExecutionVertexDeploymentOption> deploymentOptionsByVertex =
    groupDeploymentOptionsByVertexId(executionVertexDeploymentOptions);

    final List<ExecutionVertexID> verticesToDeploy =
    executionVertexDeploymentOptions.stream()
    .map(ExecutionVertexDeploymentOption::getExecutionVertexId)
    .collect(Collectors.toList());

    final Map<ExecutionVertexID, ExecutionVertexVersion> requiredVersionByVertex =
    executionVertexVersioner.recordVertexModifications(verticesToDeploy);

    transitionToScheduled(verticesToDeploy);
    //分配slots
    final List<SlotExecutionVertexAssignment> slotExecutionVertexAssignments =
    allocateSlots(executionVertexDeploymentOptions);

    final List<DeploymentHandle> deploymentHandles =
    createDeploymentHandles(
        requiredVersionByVertex,
        deploymentOptionsByVertex,
        slotExecutionVertexAssignments);
    //调用submitTask提交task
    waitForAllSlotsAndDeploy(deploymentHandles);
}
```
DefaultScheduler.waitForAllSlotsAndDeploy
```
private void waitForAllSlotsAndDeploy(final List<DeploymentHandle> deploymentHandles) {
    FutureUtils.assertNoException(
        assignAllResourcesAndRegisterProducedPartitions(deploymentHandles)
        .handle(deployAll(deploymentHandles)));
}
```
DefaultScheduler.deployAll
```
private BiFunction<Void, Throwable, Void> deployAll(
    final List<DeploymentHandle> deploymentHandles) {
    return (ignored, throwable) -> {
        propagateIfNonNull(throwable);
        for (final DeploymentHandle deploymentHandle : deploymentHandles) {
            final SlotExecutionVertexAssignment slotExecutionVertexAssignment =
            deploymentHandle.getSlotExecutionVertexAssignment();
            final CompletableFuture<LogicalSlot> slotAssigned =
            slotExecutionVertexAssignment.getLogicalSlotFuture();
            checkState(slotAssigned.isDone());
            //命名deployOrHandleError
            FutureUtils.assertNoException(
                slotAssigned.handle(deployOrHandleError(deploymentHandle)));
        }
        return null;
    };
}
```
DefaultScheduler.deployOrHandleError
```
private BiFunction<Object, Throwable, Void> deployOrHandleError(
    final DeploymentHandle deploymentHandle) {
    final ExecutionVertexVersion requiredVertexVersion =
    deploymentHandle.getRequiredVertexVersion();
    final ExecutionVertexID executionVertexId = requiredVertexVersion.getExecutionVertexId();

    return (ignored, throwable) -> {
        if (executionVertexVersioner.isModified(requiredVertexVersion)) {
            log.debug(
                "Refusing to deploy execution vertex {} because this deployment was "
                + "superseded by another deployment",
                executionVertexId);
            return null;
        }

        if (throwable == null) {
            deployTaskSafe(executionVertexId);
        } else {
            handleTaskDeploymentFailure(executionVertexId, throwable);
        }
        return null;
    };
}
```
DefaultScheduler.deployTaskSafe<br />依据executionVertexId获取到executionVertex
```
private void deployTaskSafe(final ExecutionVertexID executionVertexId) {
    try {
        final ExecutionVertex executionVertex = getExecutionVertex(executionVertexId);
        executionVertexOperations.deploy(executionVertex);
    } catch (Throwable e) {
        handleTaskDeploymentFailure(executionVertexId, e);
    }
}
//依据JobVertexId和SubtaskIndex从executionGraph获取ExecutionVertex
public ExecutionVertex getExecutionVertex(final ExecutionVertexID executionVertexId) {
    return executionGraph
    .getAllVertices()
    .get(executionVertexId.getJobVertexId())
    .getTaskVertices()[executionVertexId.getSubtaskIndex()];
}
```
DefaultExecutionVertexOperations内部实际调用的executionVertex.deploy()
```
public class DefaultExecutionVertexOperations implements ExecutionVertexOperations {

    @Override
    public void deploy(final ExecutionVertex executionVertex) throws JobException {
        executionVertex.deploy();
    }

    @Override
    public CompletableFuture<?> cancel(final ExecutionVertex executionVertex) {
        return executionVertex.cancel();
    }

    @Override
    public void markFailed(final ExecutionVertex executionVertex, final Throwable cause) {
        executionVertex.markFailed(cause);
    }
}
```
ExecutionVertex.deploy()
```
public void deploy() throws JobException {
    //这里的currentExecution是Execution
    currentExecution.deploy();
}
```
Execution.deploy();<br />经过slot以及状态校验后，提交给TM
```
final TaskDeploymentDescriptor deployment =
TaskDeploymentDescriptorFactory.fromExecutionVertex(vertex, attemptNumber)
.createDeploymentDescriptor(
    slot.getAllocationId(),
    taskRestore,
    producedPartitions.values());

// null taskRestore to let it be GC'ed
taskRestore = null;

final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();

final ComponentMainThreadExecutor jobMasterMainThreadExecutor =
vertex.getExecutionGraphAccessor().getJobMasterMainThreadExecutor();

getVertex().notifyPendingDeployment(this);
// We run the submission in the future executor so that the serialization of large TDDs
// does not block
// the main thread and sync back to the main thread once submission is completed.
//RpcTaskManagerGateway
CompletableFuture.supplyAsync(
    () -> taskManagerGateway.submitTask(deployment, rpcTimeout), executor)
.thenCompose(Function.identity())
.whenCompleteAsync(
    (ack, failure) -> {
        if (failure == null) {
            vertex.notifyCompletedDeployment(this);
        } else {
            final Throwable actualFailure =
            ExceptionUtils.stripCompletionException(failure);

            if (actualFailure instanceof TimeoutException) {
                String taskname =
                vertex.getTaskNameWithSubtaskIndex()
                + " ("
                + attemptId
                + ')';

                markFailed(
                    new Exception(
                        "Cannot deploy task "
                        + taskname
                        + " - TaskManager ("
                        + getAssignedResourceLocation()
                        + ") not responding after a rpcTimeout of "
                        + rpcTimeout,
                        actualFailure));
            } else {
                markFailed(actualFailure);
            }
        }
    },
    jobMasterMainThreadExecutor);
```
RpcTaskManagerGateway.submitTask

```
public CompletableFuture<Acknowledge> submitTask(TaskDeploymentDescriptor tdd, Time timeout) {
    return taskExecutorGateway.submitTask(tdd, jobMasterId, timeout);
}
```
TaskExecutorGatewayDecoratorBase.submitTask
```
public CompletableFuture<Acknowledge> submitTask(
    TaskDeploymentDescriptor tdd, JobMasterId jobMasterId, Time timeout) {
    return originalGateway.submitTask(tdd, jobMasterId, timeout);
}
```
TaskExecutor.submitTask
```
Task task =new Task();
try {
    //判断下是否分配了slot，且slot是否存活
    taskAdded = taskSlotTable.addTask(task);
} catch (SlotNotFoundException | SlotNotActiveException e) {
    throw new TaskSubmissionException("Could not submit task.", e);
}
    ///如果taskSlotTable添加任务成功,则启动task，
if (taskAdded) {
    task.startTaskThread();

    setupResultPartitionBookkeeping(
        tdd.getJobId(), tdd.getProducedPartitions(), task.getTerminationFuture());
    return CompletableFuture.completedFuture(Acknowledge.get());
} else {
    final String message =
    "TaskManager already contains a task for id " + task.getExecutionId() + '.';

    log.debug(message);
    throw new TaskSubmissionException(message);
}
```
Task.startTaskThread()<br />这里的executingThread实际上是封装的Task
```
// finally, create the executing thread, but do not start it
executingThread = new Thread(TASK_THREADS_GROUP, this, taskNameWithSubtask);
public void startTaskThread() {
    executingThread.start();
}
```
所以逻辑还是在Task的run（）方法
```
public void run() {
    try {
        doRun();
    } finally {
        terminationFuture.complete(executionState);
    }
}
```
Task.doRun的代码有点长，主要逻辑
```
try {
    // now load and instantiate the task's invokable code
    invokable =
    loadAndInstantiateInvokable(
        userCodeClassLoader.asClassLoader(), nameOfInvokableClass, env);
} finally {
    FlinkSecurityManager.unmonitorUserSystemExitForCurrentThread();
}

// ----------------------------------------------------------------
//  actual task core work
// ----------------------------------------------------------------

// we must make strictly sure that the invokable is accessible to the cancel() call
// by the time we switched to running.
this.invokable = invokable;

restoreAndInvoke(invokable);
```
Task.restoreAndInvoke 如其名主要两步<br />finalInvokable::restore和finalInvokable::invoke
```
private void restoreAndInvoke(TaskInvokable finalInvokable) throws Exception {
    try {
        // switch to the INITIALIZING state, if that fails, we have been canceled/failed in the
        // meantime
        if (!transitionState(ExecutionState.DEPLOYING, ExecutionState.INITIALIZING)) {
            throw new CancelTaskException();
        }

        taskManagerActions.updateTaskExecutionState(
            new TaskExecutionState(executionId, ExecutionState.INITIALIZING));

        // make sure the user code classloader is accessible thread-locally
        executingThread.setContextClassLoader(userCodeClassLoader.asClassLoader());

        runWithSystemExitMonitoring(finalInvokable::restore);

        if (!transitionState(ExecutionState.INITIALIZING, ExecutionState.RUNNING)) {
            throw new CancelTaskException();
        }

        // notify everyone that we switched to running
        taskManagerActions.updateTaskExecutionState(
            new TaskExecutionState(executionId, ExecutionState.RUNNING));

        runWithSystemExitMonitoring(finalInvokable::invoke);
    } catch (Throwable throwable) {
        try {
            runWithSystemExitMonitoring(() -> finalInvokable.cleanUp(throwable));
        } catch (Throwable cleanUpThrowable) {
            throwable.addSuppressed(cleanUpThrowable);
        }
        throw throwable;
    }
    runWithSystemExitMonitoring(() -> finalInvokable.cleanUp(null));
}
```
StreamTask.restore
```
public final void restore() throws Exception {
    restoreInternal();
}
```
StreamTask.invoke
```
public final void invoke() throws Exception {
    // Allow invoking method 'invoke' without having to call 'restore' before it.
    if (!isRunning) {
        LOG.debug("Restoring during invoke will be called.");
        restoreInternal();
    }

    // final check to exit early before starting to run
    ensureNotCanceled();

    scheduleBufferDebloater();

    // let the task do its work
    runMailboxLoop();

    // if this left the run() method cleanly despite the fact that this was canceled,
    // make sure the "clean shutdown" is not attempted
    ensureNotCanceled();

    afterInvoke();
}
//
public void runMailboxLoop() throws Exception {
    mailboxProcessor.runMailboxLoop();
}
```
mailboxProcessor.runMailboxLoop()
```
public void runMailboxLoop() throws Exception {
    suspended = !mailboxLoopRunning;

    final TaskMailbox localMailbox = mailbox;

    checkState(
        localMailbox.isMailboxThread(),
        "Method must be executed by declared mailbox thread!");

    assert localMailbox.getState() == TaskMailbox.State.OPEN : "Mailbox must be opened!";

    final MailboxController defaultActionContext = new MailboxController(this);

    while (isNextLoopPossible()) {
        // The blocking `processMail` call will not return until default action is available.
        processMail(localMailbox, false);
        if (isNextLoopPossible()) {
            mailboxDefaultAction.runDefaultAction(
                defaultActionContext); // lock is acquired inside default action as needed
        }
    }
}
```
mailboxDefaultAction是MailboxProcessor构造函数参数
```
public MailboxProcessor(
    MailboxDefaultAction mailboxDefaultAction,
    TaskMailbox mailbox,
    StreamTaskActionExecutor actionExecutor) {
    this.mailboxDefaultAction = Preconditions.checkNotNull(mailboxDefaultAction);
    this.actionExecutor = Preconditions.checkNotNull(actionExecutor);
    this.mailbox = Preconditions.checkNotNull(mailbox);
    this.mailboxLoopRunning = true;
    this.suspendedDefaultAction = null;
}
this.mailboxProcessor =
new MailboxProcessor(this::processInput, mailbox, actionExecutor);
```
也就是mailboxDefaultAction是StreamTask.processInput
```
protected void processInput(MailboxDefaultAction.Controller controller) throws Exception {
    DataInputStatus status = inputProcessor.processInput();
}
```
StreamMultipleInputProcessor.processInput<br />StreamOneInputProcessor.processInput
```
public DataInputStatus processInput() throws Exception {
    DataInputStatus status = input.emitNext(output);

    if (status == DataInputStatus.END_OF_DATA) {
        endOfInputAware.endInput(input.getInputIndex() + 1);
        output = new FinishedDataOutput<>();
    } else if (status == DataInputStatus.END_OF_RECOVERY) {
        if (input instanceof RecoverableStreamTaskInput) {
            input = ((RecoverableStreamTaskInput<IN>) input).finishRecovery();
        }
        return DataInputStatus.MORE_AVAILABLE;
    }

    return status;
}
```
AbstractStreamTaskNetworkInput.emitNext()
```
public DataInputStatus emitNext(DataOutput<T> output) throws Exception {

    while (true) {
        // get the stream element from the deserializer
        if (currentRecordDeserializer != null) {
            RecordDeserializer.DeserializationResult result;
            try {
                result = currentRecordDeserializer.getNextRecord(deserializationDelegate);
            } catch (IOException e) {
                throw new IOException(
                    String.format("Can't get next record for channel %s", lastChannel), e);
            }
            if (result.isBufferConsumed()) {
                currentRecordDeserializer = null;
            }

            if (result.isFullRecord()) {
                processElement(deserializationDelegate.getInstance(), output);
                return DataInputStatus.MORE_AVAILABLE;
            }
        }

        Optional<BufferOrEvent> bufferOrEvent = checkpointedInputGate.pollNext();
        if (bufferOrEvent.isPresent()) {
            // return to the mailbox after receiving a checkpoint barrier to avoid processing of
            // data after the barrier before checkpoint is performed for unaligned checkpoint
            // mode
            if (bufferOrEvent.get().isBuffer()) {
                processBuffer(bufferOrEvent.get());
            } else {
                return processEvent(bufferOrEvent.get());
            }
        } else {
            if (checkpointedInputGate.isFinished()) {
                checkState(
                    checkpointedInputGate.getAvailableFuture().isDone(),
                    "Finished BarrierHandler should be available");
                return DataInputStatus.END_OF_INPUT;
            }
            return DataInputStatus.NOTHING_AVAILABLE;
        }
    }
}
```

