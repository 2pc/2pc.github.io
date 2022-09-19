---
title: Flink1.15 CKP InputData&OutPutData源码分析
tagline: ""
category : Flink1.15
layout: post
tags : [flink, realtime]
---
AlternatingCollectingBarriers.alignmentTimeout
```
public BarrierHandlerState alignmentTimeout(
    Controller controller, CheckpointBarrier checkpointBarrier)
throws IOException, CheckpointException {
    state.prioritizeAllAnnouncements();
    CheckpointBarrier unalignedBarrier = checkpointBarrier.asUnaligned();
    //初始化ckp
    controller.initInputsCheckpoint(unalignedBarrier);
    for (CheckpointableInput input : state.getInputs()) {
        input.checkpointStarted(unalignedBarrier);
    }
    controller.triggerGlobalCheckpoint(unalignedBarrier);
    //转换位Collecting
    return new AlternatingCollectingBarriersUnaligned(true, state);
}
```
AlternatingWaitingForFirstBarrierUnaligned.barrierReceived
```
public BarrierHandlerState barrierReceived(
    Controller controller,
    InputChannelInfo channelInfo,
    CheckpointBarrier checkpointBarrier,
    boolean markChannelBlocked)
throws CheckpointException, IOException {

    // we received an out of order aligned barrier, we should book keep this channel as blocked,
    // as it is being blocked by the credit-based network
    if (markChannelBlocked
        && !checkpointBarrier.getCheckpointOptions().isUnalignedCheckpoint()) {
        channelState.blockChannel(channelInfo);
    }

    CheckpointBarrier unalignedBarrier = checkpointBarrier.asUnaligned();
    //初始化
    controller.initInputsCheckpoint(unalignedBarrier);
    for (CheckpointableInput input : channelState.getInputs()) {
        input.checkpointStarted(unalignedBarrier);
    }
    //出发ckp
    controller.triggerGlobalCheckpoint(unalignedBarrier);
    if (controller.allBarriersReceived()) {
        for (CheckpointableInput input : channelState.getInputs()) {
            input.checkpointStopped(unalignedBarrier.getId());
        }
        return stopCheckpoint();
    }
    return new AlternatingCollectingBarriersUnaligned(alternating, channelState);
}
```

```
public void initInputsCheckpoint(CheckpointBarrier checkpointBarrier)
throws CheckpointException {
    checkState(subTaskCheckpointCoordinator != null);
    long barrierId = checkpointBarrier.getId();
    subTaskCheckpointCoordinator.initInputsCheckpoint(
        barrierId, checkpointBarrier.getCheckpointOptions());
}
//SubtaskCheckpointCoordinatorImpl
public void initInputsCheckpoint(long id, CheckpointOptions checkpointOptions)
throws CheckpointException {
    //非对齐模式
    if (checkpointOptions.isUnalignedCheckpoint()) {
        channelStateWriter.start(id, checkpointOptions);

        prepareInflightDataSnapshot(id);
    }
}
```
prepareInflightDataSnapshot主要是将将上游数据添加到inputData,见StreamTask
```
//prepareInputSnapshot这个是StreamTask实例化就生成的，也就是构造方法里
private void prepareInflightDataSnapshot(long checkpointId) throws CheckpointException {
    prepareInputSnapshot
    .apply(channelStateWriter, checkpointId)
    .whenComplete(
        (unused, ex) -> {
            if (ex != null) {
                channelStateWriter.abort(
                    checkpointId,
                    ex,
                    false /* result is needed and cleaned by getWriteResult */);
            } else {
                channelStateWriter.finishInput(checkpointId);
            }
        });
}
```
StreamTask
```
//StreamTask 实例化subtaskCheckpointCoordinator
this.subtaskCheckpointCoordinator =
new SubtaskCheckpointCoordinatorImpl(
    checkpointStorageAccess,
    getName(),
    actionExecutor,
    getCancelables(),
    getAsyncOperationsThreadPool(),
    environment,
    this,
    configuration.isUnalignedCheckpointsEnabled(),
    configuration
    .getConfiguration()
    .get(
        ExecutionCheckpointingOptions
        .ENABLE_CHECKPOINTS_AFTER_TASKS_FINISH),
    this::prepareInputSnapshot);
//StreamTask
private CompletableFuture<Void> prepareInputSnapshot(
    ChannelStateWriter channelStateWriter, long checkpointId) throws CheckpointException {
    if (inputProcessor == null) {
        return FutureUtils.completedVoidFuture();
    }
    return inputProcessor.prepareSnapshot(channelStateWriter, checkpointId);
}
```
StreamMultipleInputProcessor.prepareSnapshot<br />StreamOneInputProcessor.prepareSnapshot<br />StreamTaskNetworkInput.prepareSnapshot<br />channelStateWriter.addInputData
```
//StreamTaskNetworkInput
public CompletableFuture<Void> prepareSnapshot(
    ChannelStateWriter channelStateWriter, long checkpointId) throws CheckpointException {
    for (Map.Entry<
         InputChannelInfo,
         SpillingAdaptiveSpanningRecordDeserializer<
         DeserializationDelegate<StreamElement>>>
         e : recordDeserializers.entrySet()) {

        try {
            //
            channelStateWriter.addInputData(
                checkpointId,
                e.getKey(),
                ChannelStateWriter.SEQUENCE_NUMBER_UNKNOWN,
                e.getValue().getUnconsumedBuffer());
        } catch (IOException ioException) {
            throw new CheckpointException(CheckpointFailureReason.IO_EXCEPTION, ioException);
        }
    }
    return checkpointedInputGate.getAllBarriersReceivedFuture(checkpointId);
}
```
向下游广播checkpointEvent<br />SubtaskCheckpointCoordinatorImpl,这里broadcastEvent跟channelStateWriter.finishOutput连起来看<br />看注释是说broadcastEvent已经将数据写到output data了
```
//SubtaskCheckpointCoordinatorImpl
// Step (2): Send the checkpoint barrier downstream
operatorChain.broadcastEvent(
    new CheckpointBarrier(metadata.getCheckpointId(), metadata.getTimestamp(), options),
    options.isUnalignedCheckpoint());
// Step (3): Prepare to spill the in-flight buffers for input and output
if (options.isUnalignedCheckpoint()) {
    // output data already written while broadcasting event
    channelStateWriter.finishOutput(metadata.getCheckpointId());
}
//OperatorChain
public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {
    for (RecordWriterOutput<?> streamOutput : streamOutputs) {
        streamOutput.broadcastEvent(event, isPriorityEvent);
    }
}
//RecordWriterOutput
public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {
    if (isPriorityEvent
        && event instanceof CheckpointBarrier
        && !supportsUnalignedCheckpoints) {
        final CheckpointBarrier barrier = (CheckpointBarrier) event;
        event = barrier.withOptions(barrier.getCheckpointOptions().withUnalignedUnsupported());
        isPriorityEvent = false;
    }
    recordWriter.broadcastEvent(event, isPriorityEvent);
}
//RecordWriter
public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {
    targetPartition.broadcastEvent(event, isPriorityEvent);

    if (flushAlways) {
        flushAll();
    }
}
```
targetPartition可以式pipeline,比如BufferWritingResultPartition
```
public void broadcastEvent(AbstractEvent event, boolean isPriorityEvent) throws IOException {
    checkInProduceState();
    finishBroadcastBufferBuilder();
    finishUnicastBufferBuilders();

    try (BufferConsumer eventBufferConsumer =
         EventSerializer.toBufferConsumer(event, isPriorityEvent)) {
        totalWrittenBytes += ((long) eventBufferConsumer.getWrittenBytes() * numSubpartitions);
        for (ResultSubpartition subpartition : subpartitions) {
            // Retain the buffer so that it can be recycled by each channel of targetPartition
            subpartition.add(eventBufferConsumer.copy(), 0);
        }
    }
}
//PipelinedSubpartition
public void addRecovered(BufferConsumer bufferConsumer) throws IOException {
    NetworkActionsLogger.traceRecover(
        "PipelinedSubpartition#addRecovered",
        bufferConsumer,
        parent.getOwningTaskName(),
        subpartitionInfo);
    if (add(bufferConsumer, Integer.MIN_VALUE) == -1) {
        throw new IOException("Buffer consumer couldn't be added to ResultSubpartition");
    }
}
//PipelinedSubpartition
public int add(BufferConsumer bufferConsumer, int partialRecordLength) {
    return add(bufferConsumer, partialRecordLength, false);
}
//PipelinedSubpartition
private int add(BufferConsumer bufferConsumer, int partialRecordLength, boolean finish) {
    checkNotNull(bufferConsumer);

    final boolean notifyDataAvailable;
    int prioritySequenceNumber = -1;
    int newBufferSize;
    synchronized (buffers) {
        if (isFinished || isReleased) {
            bufferConsumer.close();
            return -1;
        }
        //注意这里addBuffer
        // Add the bufferConsumer and update the stats
        if (addBuffer(bufferConsumer, partialRecordLength)) {
            prioritySequenceNumber = sequenceNumber;
        }
        updateStatistics(bufferConsumer);
        increaseBuffersInBacklog(bufferConsumer);
        notifyDataAvailable = finish || shouldNotifyDataAvailable();

        isFinished |= finish;
        newBufferSize = bufferSize;
    }

    if (prioritySequenceNumber != -1) {
        notifyPriorityEvent(prioritySequenceNumber);
    }
    if (notifyDataAvailable) {
        notifyDataAvailable();
    }

    return newBufferSize;
}
```
看注释addBuffer里处理bufferConsumer
```
private boolean addBuffer(BufferConsumer bufferConsumer, int partialRecordLength) {
    assert Thread.holdsLock(buffers);
    //优先级类型，非对齐模式
    if (bufferConsumer.getDataType().hasPriority()) {
        return processPriorityBuffer(bufferConsumer, partialRecordLength);
    }
    buffers.add(new BufferConsumerWithPartialRecordLength(bufferConsumer, partialRecordLength));
    return false;
}
```
解析barrier时间，手机buffers内部数据，inflightBuffers写入outputData
```
private boolean processPriorityBuffer(BufferConsumer bufferConsumer, int partialRecordLength) {
    buffers.addPriorityElement(
        new BufferConsumerWithPartialRecordLength(bufferConsumer, partialRecordLength));
    final int numPriorityElements = buffers.getNumPriorityElements();
    //解析CheckpointBarrier
    CheckpointBarrier barrier = parseCheckpointBarrier(bufferConsumer);
    if (barrier != null) {
        checkState(
            barrier.getCheckpointOptions().isUnalignedCheckpoint(),
            "Only unaligned checkpoints should be priority events");
        final Iterator<BufferConsumerWithPartialRecordLength> iterator = buffers.iterator();
        Iterators.advance(iterator, numPriorityElements);
        List<Buffer> inflightBuffers = new ArrayList<>();
        //迭代buffers，即BufferConsumer
        while (iterator.hasNext()) {
            BufferConsumer buffer = iterator.next().getBufferConsumer();

            if (buffer.isBuffer()) {
                //如果式buffer,则加入inflightBuffers
                try (BufferConsumer bc = buffer.copy()) {
                    inflightBuffers.add(bc.build());
                }
            }
        }
        if (!inflightBuffers.isEmpty()) {
            //将inflightBuffers下入outputData
            channelStateWriter.addOutputData(
                barrier.getId(),
                subpartitionInfo,
                ChannelStateWriter.SEQUENCE_NUMBER_UNKNOWN,
                inflightBuffers.toArray(new Buffer[0]));
        }
    }
    return numPriorityElements == 1
    && !isBlocked; // if subpartition is blocked then downstream doesn't expect any
    // notifications
}
```
