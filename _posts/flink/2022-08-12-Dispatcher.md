---
title: Flink1.15 Dispatcher源码分析
tagline: ""
category : Flink1.15
layout: post
tags : [flink, realtime]
---
AM启动类YarnApplicationClusterEntryPoint<br />YarnApplicationClusterEntryPoint.main<br />ClusterEntrypoint.runClusterEntrypoint<br />ClusterEntrypoint.startCluster<br />ClusterEntrypoint.runCluster
```
private void runCluster(Configuration configuration, PluginManager pluginManager)
throws Exception {
    synchronized (lock) {
        initializeServices(configuration, pluginManager);

        // write host information into configuration
        configuration.setString(JobManagerOptions.ADDRESS, commonRpcService.getAddress());
        configuration.setInteger(JobManagerOptions.PORT, commonRpcService.getPort());
        //创建dispatcherResourceManagerComponentFactory
        final DispatcherResourceManagerComponentFactory
        dispatcherResourceManagerComponentFactory =
        createDispatcherResourceManagerComponentFactory(configuration);
        //创建DispatcherResourceManagerComponent
        clusterComponent =
        dispatcherResourceManagerComponentFactory.create(
            configuration,
            resourceId.unwrap(),
            ioExecutor,
            commonRpcService,
            haServices,
            blobServer,
            heartbeatServices,
            metricRegistry,
            executionGraphInfoStore,
            new RpcMetricQueryServiceRetriever(
                metricRegistry.getMetricQueryServiceRpcService()),
            this);

        clusterComponent
        .getShutDownFuture()
        .whenComplete(
            (ApplicationStatus applicationStatus, Throwable throwable) -> {
                if (throwable != null) {
                    shutDownAsync(
                        ApplicationStatus.UNKNOWN,
                        ShutdownBehaviour.GRACEFUL_SHUTDOWN,
                        ExceptionUtils.stringifyException(throwable),
                        false);
                } else {
                    // This is the general shutdown path. If a separate more
                    // specific shutdown was
                    // already triggered, this will do nothing
                    shutDownAsync(
                        applicationStatus,
                        ShutdownBehaviour.GRACEFUL_SHUTDOWN,
                        null,
                        true);
                }
            });
    }
}
```
createDispatcherResourceManagerComponentFactory这里用的是DefaultDispatcherResourceManagerComponentFactory<br />另外注意下这里的参数SessionDispatcherFactory.INSTANCE
```
//注意下SessionDispatcherFactory
protected DispatcherResourceManagerComponentFactory
createDispatcherResourceManagerComponentFactory(final Configuration configuration) {
    return new DefaultDispatcherResourceManagerComponentFactory(
        new DefaultDispatcherRunnerFactory(
            ApplicationDispatcherLeaderProcessFactoryFactory.create(
                configuration, SessionDispatcherFactory.INSTANCE, program)),
        resourceManagerFactory,
        JobRestEndpointFactory.INSTANCE);
}
```
SessionDispatcherFactory后边生成dispatcher是StandaloneDispatcher
```
public enum SessionDispatcherFactory implements DispatcherFactory {
    INSTANCE;

    @Override
    public StandaloneDispatcher createDispatcher(
        RpcService rpcService,
        DispatcherId fencingToken,
        Collection<JobGraph> recoveredJobs,
        Collection<JobResult> recoveredDirtyJobResults,
        DispatcherBootstrapFactory dispatcherBootstrapFactory,
        PartialDispatcherServicesWithJobPersistenceComponents
        partialDispatcherServicesWithJobPersistenceComponents)
    throws Exception {
        // create the default dispatcher
        return new StandaloneDispatcher(
            rpcService,
            fencingToken,
            recoveredJobs,
            recoveredDirtyJobResults,
            dispatcherBootstrapFactory,
            DispatcherServices.from(
                partialDispatcherServicesWithJobPersistenceComponents,
                JobMasterServiceLeadershipRunnerFactory.INSTANCE,
                CheckpointResourcesCleanupRunnerFactory.INSTANCE));
    }
}
```
DefaultDispatcherResourceManagerComponentFactory.create<br />启动dispatcher<br />启动RM
```
log.debug("Starting Dispatcher.");
dispatcherRunner =
dispatcherRunnerFactory.createDispatcherRunner(
    highAvailabilityServices.getDispatcherLeaderElectionService(),
    fatalErrorHandler,
    new HaServicesJobPersistenceComponentFactory(highAvailabilityServices),
    ioExecutor,
    rpcService,
    partialDispatcherServices);

log.debug("Starting ResourceManagerService.");
resourceManagerService.start();

resourceManagerRetrievalService.start(resourceManagerGatewayRetriever);
dispatcherLeaderRetrievalService.start(dispatcherGatewayRetriever);

return new DispatcherResourceManagerComponent(
    dispatcherRunner,
    resourceManagerService,
    dispatcherLeaderRetrievalService,
    resourceManagerRetrievalService,
    webMonitorEndpoint,
    fatalErrorHandler,
    dispatcherOperationCaches);
```
DefaultDispatcherRunnerFactory.createDispatcherRunner
```
public class DefaultDispatcherRunnerFactory implements DispatcherRunnerFactory {
    private final DispatcherLeaderProcessFactoryFactory dispatcherLeaderProcessFactoryFactory;

    public DefaultDispatcherRunnerFactory(
        DispatcherLeaderProcessFactoryFactory dispatcherLeaderProcessFactoryFactory) {
        this.dispatcherLeaderProcessFactoryFactory = dispatcherLeaderProcessFactoryFactory;
    }
    //application
    @Override
    public DispatcherRunner createDispatcherRunner(
        LeaderElectionService leaderElectionService,
        FatalErrorHandler fatalErrorHandler,
        JobPersistenceComponentFactory jobPersistenceComponentFactory,
        Executor ioExecutor,
        RpcService rpcService,
        PartialDispatcherServices partialDispatcherServices)
    throws Exception {
        //SessionDispatcherLeaderProcessFactory
        final DispatcherLeaderProcessFactory dispatcherLeaderProcessFactory =
        dispatcherLeaderProcessFactoryFactory.createFactory(
            jobPersistenceComponentFactory,
            ioExecutor,
            rpcService,
            partialDispatcherServices,
            fatalErrorHandler);

        return DefaultDispatcherRunner.create(
            leaderElectionService, fatalErrorHandler, dispatcherLeaderProcessFactory);
    }
    //session 
    public static DefaultDispatcherRunnerFactory createSessionRunner(
        DispatcherFactory dispatcherFactory) {
        return new DefaultDispatcherRunnerFactory(
            SessionDispatcherLeaderProcessFactoryFactory.create(dispatcherFactory));
    }
    //job
    public static DefaultDispatcherRunnerFactory createJobRunner(
        JobGraphRetriever jobGraphRetriever) {
        return new DefaultDispatcherRunnerFactory(
            JobDispatcherLeaderProcessFactoryFactory.create(jobGraphRetriever));
    }
}
```
application模式<br />ApplicationDispatcherLeaderProcessFactoryFactory.createDispatcherRunner
```
public DispatcherRunner createDispatcherRunner(
    LeaderElectionService leaderElectionService,
    FatalErrorHandler fatalErrorHandler,
    JobPersistenceComponentFactory jobPersistenceComponentFactory,
    Executor ioExecutor,
    RpcService rpcService,
    PartialDispatcherServices partialDispatcherServices)
throws Exception {

    final DispatcherLeaderProcessFactory dispatcherLeaderProcessFactory =
    dispatcherLeaderProcessFactoryFactory.createFactory(
        jobPersistenceComponentFactory,
        ioExecutor,
        rpcService,
        partialDispatcherServices,
        fatalErrorHandler);

    return DefaultDispatcherRunner.create(
        leaderElectionService, fatalErrorHandler, dispatcherLeaderProcessFactory);
}
```
DefaultDispatcherRunner实现LeaderContender选举回调grantLeadership
```
public void grantLeadership(UUID leaderSessionID) {
    runActionIfRunning(
        () -> {
            LOG.info(
                "{} was granted leadership with leader id {}. Creating new {}.",
                getClass().getSimpleName(),
                leaderSessionID,
                DispatcherLeaderProcess.class.getSimpleName());
            startNewDispatcherLeaderProcess(leaderSessionID);
        });
}
```
DefaultDispatcherRunner.startNewDispatcherLeaderProcess
```
private void startNewDispatcherLeaderProcess(UUID leaderSessionID) {
    stopDispatcherLeaderProcess();

    dispatcherLeaderProcess = createNewDispatcherLeaderProcess(leaderSessionID);

    final DispatcherLeaderProcess newDispatcherLeaderProcess = dispatcherLeaderProcess;
    FutureUtils.assertNoException(
        previousDispatcherLeaderProcessTerminationFuture.thenRun(
            newDispatcherLeaderProcess::start));
}
```
AbstractDispatcherLeaderProcess.start<br />AbstractDispatcherLeaderProcess.startInternal
```
public final void start() {
    runIfStateIs(State.CREATED, this::startInternal);
}

private void startInternal() {
    log.info("Start {}.", getClass().getSimpleName());
    state = State.RUNNING;
    onStart();
}
```
SessionDispatcherLeaderProcess.onStart()
```
protected void onStart() {
    startServices();

    onGoingRecoveryOperation =
    createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults();
}
```
createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults
```
    private CompletableFuture<Void>
            createDispatcherBasedOnRecoveredJobGraphsAndRecoveredDirtyJobResults() {
        final CompletableFuture<Collection<JobResult>> dirtyJobsFuture =
                CompletableFuture.supplyAsync(this::getDirtyJobResultsIfRunning, ioExecutor);

        return dirtyJobsFuture
                .thenApplyAsync(
                        dirtyJobs ->
                                this.recoverJobsIfRunning(
                                        dirtyJobs.stream()
                                                .map(JobResult::getJobId)
                                                .collect(Collectors.toSet())),
                        ioExecutor)
                .thenAcceptBoth(dirtyJobsFuture, this::createDispatcherIfRunning)
                .handle(this::onErrorIfRunning);
    }
```
SessionDispatcherLeaderProcess.createDispatcherIfRunning<br />SessionDispatcherLeaderProcess.createDispatcher
```
private void createDispatcherIfRunning(
    Collection<JobGraph> jobGraphs, Collection<JobResult> recoveredDirtyJobResults) {
    runIfStateIs(State.RUNNING, () -> createDispatcher(jobGraphs, recoveredDirtyJobResults));
}

private void createDispatcher(
    Collection<JobGraph> jobGraphs, Collection<JobResult> recoveredDirtyJobResults) {

    final DispatcherGatewayService dispatcherService =
    dispatcherGatewayServiceFactory.create(
        DispatcherId.fromUuid(getLeaderSessionId()),
        jobGraphs,
        recoveredDirtyJobResults,
        jobGraphStore,
        jobResultStore);

    completeDispatcherSetup(dispatcherService);
}
```
ApplicationDispatcherGatewayServiceFactory.create
```
public AbstractDispatcherLeaderProcess.DispatcherGatewayService create(
    DispatcherId fencingToken,
    Collection<JobGraph> recoveredJobs,
    Collection<JobResult> recoveredDirtyJobResults,
    JobGraphWriter jobGraphWriter,
    JobResultStore jobResultStore) {

    final List<JobID> recoveredJobIds = getRecoveredJobIds(recoveredJobs);

    final Dispatcher dispatcher;
    try {
        dispatcher =
        dispatcherFactory.createDispatcher(
            rpcService,
            fencingToken,
            recoveredJobs,
            recoveredDirtyJobResults,
            (dispatcherGateway, scheduledExecutor, errorHandler) ->
            new ApplicationDispatcherBootstrap(
                application,
                recoveredJobIds,
                configuration,
                dispatcherGateway,
                scheduledExecutor,
                errorHandler),
            PartialDispatcherServicesWithJobPersistenceComponents.from(
                partialDispatcherServices, jobGraphWriter, jobResultStore));
    } catch (Exception e) {
        throw new FlinkRuntimeException("Could not create the Dispatcher rpc endpoint.", e);
    }

    dispatcher.start();

    return DefaultDispatcherGatewayService.from(dispatcher);
}
```
这里SessionDispatcherFactory生成StandaloneDispatcher
```
public enum SessionDispatcherFactory implements DispatcherFactory {
    INSTANCE;

    @Override
    public StandaloneDispatcher createDispatcher(
            RpcService rpcService,
            DispatcherId fencingToken,
            Collection<JobGraph> recoveredJobs,
            Collection<JobResult> recoveredDirtyJobResults,
            DispatcherBootstrapFactory dispatcherBootstrapFactory,
            PartialDispatcherServicesWithJobPersistenceComponents
                    partialDispatcherServicesWithJobPersistenceComponents)
            throws Exception {
        // create the default dispatcher
        return new StandaloneDispatcher(
                rpcService,
                fencingToken,
                recoveredJobs,
                recoveredDirtyJobResults,
                dispatcherBootstrapFactory,
                DispatcherServices.from(
                        partialDispatcherServicesWithJobPersistenceComponents,
                        JobMasterServiceLeadershipRunnerFactory.INSTANCE,
                        CheckpointResourcesCleanupRunnerFactory.INSTANCE));
    }
}
```
StandaloneDispatcher是Dispatcher的子类，本身什么也没干，逻辑都在Dispatcher
```
public class StandaloneDispatcher extends Dispatcher {
    public StandaloneDispatcher(
            RpcService rpcService,
            DispatcherId fencingToken,
            Collection<JobGraph> recoveredJobs,
            Collection<JobResult> recoveredDirtyJobResults,
            DispatcherBootstrapFactory dispatcherBootstrapFactory,
            DispatcherServices dispatcherServices)
            throws Exception {
        super(
                rpcService,
                fencingToken,
                recoveredJobs,
                recoveredDirtyJobResults,
                dispatcherBootstrapFactory,
                dispatcherServices);
    }
}
```
 Dispatcher继承了RpcEndpoint，start方法
```
public final void start() {
    rpcServer.start();
}
```
启动RPC后会调用onStart
```
public void onStart() throws Exception {
    try {
        startDispatcherServices();
    } catch (Throwable t) {
        final DispatcherException exception =
                new DispatcherException(
                        String.format("Could not start the Dispatcher %s", getAddress()), t);
        onFatalError(exception);
        throw exception;
    }

    startCleanupRetries();
    //
    startRecoveredJobs();

    this.dispatcherBootstrap =
            this.dispatcherBootstrapFactory.create(
                    getSelfGateway(DispatcherGateway.class),
                    this.getRpcService().getScheduledExecutor(),
                    this::onFatalError);
}
```
主要看startRecoveredJobs
```
private void startRecoveredJobs() {
    for (JobGraph recoveredJob : recoveredJobs) {
        runRecoveredJob(recoveredJob);
    }
    recoveredJobs.clear();
}

private void runRecoveredJob(final JobGraph recoveredJob) {
    checkNotNull(recoveredJob);
    try {
        runJob(createJobMasterRunner(recoveredJob), ExecutionType.RECOVERY);
    } catch (Throwable throwable) {
        onFatalError(
                new DispatcherException(
                        String.format(
                                "Could not start recovered job %s.", recoveredJob.getJobID()),
                        throwable));
    }
}
```

