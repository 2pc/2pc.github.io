---
title: Flink1.15 InputChannel源码分析
tagline: ""
category : Flink1.15
layout: post
tags : [flink, realtime]
---
```
this.subtaskCheckpointCoordinator = new SubtaskCheckpointCoordinatorImpl(checkpointStorageAccess, this.getName(), actionExecutor, this.getCancelables(), this.getAsyncOperationsThreadPool(), environment, this, this.configuration.isUnalignedCheckpointsEnabled(), (Boolean)this.configuration.getConfiguration().get(ExecutionCheckpointingOptions.ENABLE_CHECKPOINTS_AFTER_TASKS_FINISH), this::prepareInputSnapshot);

private CompletableFuture<Void> prepareInputSnapshot(ChannelStateWriter channelStateWriter, long checkpointId) throws CheckpointException {
    return this.inputProcessor == null ? FutureUtils.completedVoidFuture() : this.inputProcessor.prepareSnapshot(channelStateWriter, checkpointId);
}
//StreamOneInputProcessor
public CompletableFuture<Void> prepareSnapshot(ChannelStateWriter channelStateWriter, long checkpointId) throws IOException {
    return this.input.prepareSnapshot(channelStateWriter, checkpointId);
}
//StreamTwoInputProcessor
public CompletableFuture<Void> prepareSnapshot(ChannelStateWriter channelStateWriter, long checkpointId) throws IOException {
    return CompletableFuture.allOf(this.processor1.prepareSnapshot(channelStateWriter, checkpointId), this.processor2.prepareSnapshot(channelStateWriter, checkpointId));
}
//StreamMultipleInputProcessor
public CompletableFuture<Void> prepareSnapshot(ChannelStateWriter channelStateWriter, long checkpointId) throws IOException {
    CompletableFuture<?>[] inputFutures = new CompletableFuture[this.inputProcessors.length];

    for(int index = 0; index < inputFutures.length; ++index) {
        inputFutures[index] = this.inputProcessors[index].prepareSnapshot(channelStateWriter, checkpointId);
    }

    return CompletableFuture.allOf(inputFutures);
}

```

Task中创建SingleInputGate
```
// consumed intermediate result partitions
final IndexedInputGate[] gates =
shuffleEnvironment
.createInputGates(taskShuffleContext, this, inputGateDeploymentDescriptors)
.toArray(new IndexedInputGate[0]);

this.inputGates = new IndexedInputGate[gates.length];
int counter = 0;
for (IndexedInputGate gate : gates) {
    inputGates[counter++] =
    new InputGateWithMetrics(
        gate, metrics.getIOMetricGroup().getNumBytesInCounter());
}
//NettyShuffleEnvironment
public List<SingleInputGate> createInputGates(
    ShuffleIOOwnerContext ownerContext,
    PartitionProducerStateProvider partitionProducerStateProvider,
    List<InputGateDeploymentDescriptor> inputGateDeploymentDescriptors) {
    synchronized (lock) {
        Preconditions.checkState(
            !isClosed, "The NettyShuffleEnvironment has already been shut down.");

        MetricGroup networkInputGroup = ownerContext.getInputGroup();
        @SuppressWarnings("deprecation")
        InputChannelMetrics inputChannelMetrics =
        new InputChannelMetrics(networkInputGroup, ownerContext.getParentGroup());

        SingleInputGate[] inputGates =
        new SingleInputGate[inputGateDeploymentDescriptors.size()];
        for (int gateIndex = 0; gateIndex < inputGates.length; gateIndex++) {
            final InputGateDeploymentDescriptor igdd =
            inputGateDeploymentDescriptors.get(gateIndex);
            SingleInputGate inputGate =
            singleInputGateFactory.create(
                ownerContext.getOwnerName(),
                gateIndex,
                igdd,
                partitionProducerStateProvider,
                inputChannelMetrics);
            InputGateID id =
            new InputGateID(
                igdd.getConsumedResultId(), ownerContext.getExecutionAttemptID());
            inputGatesById.put(id, inputGate);
            inputGate.getCloseFuture().thenRun(() -> inputGatesById.remove(id));
            inputGates[gateIndex] = inputGate;
        }

        registerInputMetrics(config.isNetworkDetailedMetrics(), networkInputGroup, inputGates);
        return Arrays.asList(inputGates);
    }
}
```
