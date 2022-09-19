---
title: RocketMQ DLeger
tagline: ""
category : rocketmq
layout: post
tags : [rocketmq]
---
启动BrokerStartup
```go
//main方法启动入口
public static void main(String[] args) {
    start(createBrokerController(args));
}

public static BrokerController start(BrokerController controller) {
    try {

        controller.start();//启动BrokerController

        String tip = "The broker[" + controller.getBrokerConfig().getBrokerName() + ", "
        + controller.getBrokerAddr() + "] boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();

        if (null != controller.getBrokerConfig().getNamesrvAddr()) {
            tip += " and name server is " + controller.getBrokerConfig().getNamesrvAddr();
        }

        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```
createBrokerController会实例化一个BrokerController，并初始化
```go
final BrokerController controller = new BrokerController(
    brokerConfig,
    nettyServerConfig,
    nettyClientConfig,
    messageStoreConfig);
// remember all configs to prevent discard
controller.getConfiguration().registerConfig(properties);

boolean initResult = controller.initialize();//完成BrokerController初始化工作
```
在BrokerController初始化阶段会实例化DefaultMessageStore,DefaultMessageStore直接与CommitLog关联
```go
public boolean initialize() throws CloneNotSupportedException {
    try {
        //实例化messageStore
        //如果开启DLeger,会实例化一个DLedgerCommitLog，关联DLedgerServer,否则使用原CommitLog
        this.messageStore =
        new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                                this.brokerConfig);
        //如果开启了DLeger，需要添加roleChangeHandler
        if (messageStoreConfig.isEnableDLegerCommitLog()) {
            DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
            ((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
        }
        this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
        //load plugin
        MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
        this.messageStore = MessageStoreFactory.build(context, this.messageStore);
        this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
    } catch (IOException e) {
        result = false;
        log.error("Failed to initialize", e);
    }
}
```
DefaultMessageStore这里会实例化commitLog，如果开启DLedger，则是DLedgerCommitLog
```go
public DefaultMessageStore(final MessageStoreConfig messageStoreConfig, final BrokerStatsManager brokerStatsManager,
                           final MessageArrivingListener messageArrivingListener, final BrokerConfig brokerConfig) throws IOException {
    //开启 DLedger，则使用DLedgerCommitLog
    if (messageStoreConfig.isEnableDLegerCommitLog()) {
        this.commitLog = new DLedgerCommitLog(this);
    } else {
        this.commitLog = new CommitLog(this);
    }
}
```
DLedgerCommitLog内部有一个DLedgerServer
```go
public DLedgerCommitLog(final DefaultMessageStore defaultMessageStore) {
    super(defaultMessageStore);
    dLedgerConfig = new DLedgerConfig();
    dLedgerConfig.setEnableDiskForceClean(defaultMessageStore.getMessageStoreConfig().isCleanFileForciblyEnable());
    dLedgerConfig.setStoreType(DLedgerConfig.FILE);
    dLedgerConfig.setSelfId(defaultMessageStore.getMessageStoreConfig().getdLegerSelfId());
    dLedgerConfig.setGroup(defaultMessageStore.getMessageStoreConfig().getdLegerGroup());
    dLedgerConfig.setPeers(defaultMessageStore.getMessageStoreConfig().getdLegerPeers());
    dLedgerConfig.setStoreBaseDir(defaultMessageStore.getMessageStoreConfig().getStorePathRootDir());
    dLedgerConfig.setMappedFileSizeForEntryData(defaultMessageStore.getMessageStoreConfig().getMappedFileSizeCommitLog());
    dLedgerConfig.setDeleteWhen(defaultMessageStore.getMessageStoreConfig().getDeleteWhen());
    dLedgerConfig.setFileReservedHours(defaultMessageStore.getMessageStoreConfig().getFileReservedTime() + 1);
    id = Integer.valueOf(dLedgerConfig.getSelfId().substring(1)) + 1;
    //实例化DLedgerServer
    dLedgerServer = new DLedgerServer(dLedgerConfig);
    dLedgerFileStore = (DLedgerMmapFileStore) dLedgerServer.getdLedgerStore();
    DLedgerMmapFileStore.AppendHook appendHook = (entry, buffer, bodyOffset) -> {
        assert bodyOffset == DLedgerEntry.BODY_OFFSET;
        buffer.position(buffer.position() + bodyOffset + MessageDecoder.PHY_POS_POSITION);
        buffer.putLong(entry.getPos() + bodyOffset);
    };
    dLedgerFileStore.addAppendHook(appendHook);
    dLedgerFileList = dLedgerFileStore.getDataFileList();
    this.messageSerializer = new MessageSerializer(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());

}
```
DLedgerServer就是开源项目DLedger中的了
```go
public DLedgerServer(DLedgerConfig dLedgerConfig) {
    this.dLedgerConfig = dLedgerConfig;
    this.memberState = new MemberState(dLedgerConfig);//状态，candidate,currTerm,currVoteFor等
    this.dLedgerStore = createDLedgerStore(dLedgerConfig.getStoreType(), this.dLedgerConfig, this.memberState);
    dLedgerRpcService = new DLedgerRpcNettyService(this);
    dLedgerEntryPusher = new DLedgerEntryPusher(dLedgerConfig, memberState, dLedgerStore, dLedgerRpcService);
    dLedgerLeaderElector = new DLedgerLeaderElector(dLedgerConfig, memberState, dLedgerRpcService);
}
```
controller.start()启动过程中最终也会启动DLedgerServer
```go
    public void startup() {
        this.dLedgerStore.startup();
        this.dLedgerRpcService.startup();
        this.dLedgerEntryPusher.startup();
        this.dLedgerLeaderElector.startup();//领导选举
    }
```
dLedgerLeaderElector领导选举
```go
public DLedgerLeaderElector(DLedgerConfig dLedgerConfig, MemberState memberState, DLedgerRpcService dLedgerRpcService) {
    this.dLedgerConfig = dLedgerConfig;
    this.memberState = memberState;
    this.dLedgerRpcService = dLedgerRpcService;
    refreshIntervals(dLedgerConfig);
}

public void startup() {
    stateMaintainer.start();//raft选举循环
    for (RoleChangeHandler roleChangeHandler : roleChangeHandlers) {
        roleChangeHandler.startup();
    }
}
```
ShutdownAbleThread是线程实现类，最终调用StateMaintainer的doWork方法
```go
public class StateMaintainer extends ShutdownAbleThread {

    public StateMaintainer(String name, Logger logger) {
        super(name, logger);
    }

    @Override public void doWork() {
        try {
            if (DLedgerLeaderElector.this.dLedgerConfig.isEnableLeaderElector()) {
                DLedgerLeaderElector.this.refreshIntervals(dLedgerConfig);
                DLedgerLeaderElector.this.maintainState();
            }
            sleep(10);
        } catch (Throwable t) {
            DLedgerLeaderElector.logger.error("Error in heartbeat", t);
        }
    }

}
```
maintainState内典型的raft角色
```go
private void maintainState() throws Exception {
    if (memberState.isLeader()) {
        maintainAsLeader();
    } else if (memberState.isFollower()) {
        maintainAsFollower();
    } else {
        maintainAsCandidate();
    }
}
```
初始状态Candidate走maintainAsCandidate，发起投投票
```go
private List<CompletableFuture<VoteResponse>> voteForQuorumResponses(long term, long ledgerEndTerm,
                                                                     long ledgerEndIndex) throws Exception {
    List<CompletableFuture<VoteResponse>> responses = new ArrayList<>();
    for (String id : memberState.getPeerMap().keySet()) {
        VoteRequest voteRequest = new VoteRequest();
        voteRequest.setGroup(memberState.getGroup());
        voteRequest.setLedgerEndIndex(ledgerEndIndex);
        voteRequest.setLedgerEndTerm(ledgerEndTerm);
        voteRequest.setLeaderId(memberState.getSelfId());
        voteRequest.setTerm(term);
        voteRequest.setRemoteId(id);
        CompletableFuture<VoteResponse> voteResponse;
        if (memberState.getSelfId().equals(id)) {
            voteResponse = handleVote(voteRequest, true);//投给自己，无需rpc远程调用
        } else {
            //async
            voteResponse = dLedgerRpcService.vote(voteRequest);//rpc异步投给其他节点id
        }
        responses.add(voteResponse);

    }
    return responses;
}
```
经过投票后，如果获得多数投票，则转换为Leader
```go
if (parseResult == VoteResponse.ParseResult.PASSED) {
    logger.info("[{}] [VOTE_RESULT] has been elected to be the leader in term {}", memberState.getSelfId(), term);
    changeRoleToLeader(term);
}
```
