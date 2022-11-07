---
title: CompletableFuture与ForkJoinPool
tagline: ""
category : Java
layout: post
tags : [Java, File, tools，OOM]
---
## CompletableFuture
### CompletableFuture常用方法
```
//thenApply
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}
//thenApplyAsync
public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}
//thenCompose
public <U> CompletableFuture<U> thenCompose(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(null, fn);
}
//thenComposeAsync
public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(asyncPool, fn);
}
//supplyAsync
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                    Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}
//runAsync
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(asyncPool, runnable);
}

```
### 以CompletableFuture.thenApplyAsync为例
大致流程
![CompletableFuture&ForkJoinPool](https://raw.githubusercontent.com/2pc/mydrawio/master/Java/export/CompletableFuture_ForkJoinPool-第-1-页.png)
```

```

### get相关方法
```
public T get() throws InterruptedException, ExecutionException {
    Object r;
    ////result为null,继续等待结果
    return reportGet((r = result) == null ? waitingGet(true) : r);
}
```
#### waitingGet()方法
1，首先进行自旋

```
private Object waitingGet(boolean interruptible) {
    Signaller q = null;
    boolean queued = false;
    int spins = -1;
    Object r;
    while ((r = result) == null) {
        //spins < 0开始进入自旋
        if (spins < 0)
            spins = (Runtime.getRuntime().availableProcessors() > 1) ?
                1 << 8 : 0; // Use brief spin-wait on multiprocessors
        else if (spins > 0) {、
            //随机数，并不是每次都一样
            if (ThreadLocalRandom.nextSecondarySeed() >= 0)
                --spins;
        }
        //到这里说明自旋次数spins==0
        else if (q == null)//还没有生成Signaller，
            q = new Signaller(interruptible, 0L, 0L);
        else if (!queued)//将Signaller入栈
            queued = tryPushStack(q);
        else if (interruptible && q.interruptControl < 0) {//interruptible是否允许中断，且已经中断，直接返回null
            q.thread = null;
            cleanStack();//清理stack,依赖的stage
            return null;
        }
        else if (q.thread != null && result == null) {
            try {
                ForkJoinPool.managedBlock(q);//进行阻塞，通过LockSupport.park阻塞
            } catch (InterruptedException ie) {
                q.interruptControl = -1;
            }
        }
    }
    //阻塞已经唤醒，有可能是中断，需要处理下中断
    if (q != null) {
        q.thread = null;
        if (q.interruptControl < 0) {//interruptControl < 0表示已经中断了
            if (interruptible)
                r = null; // report interruption
            else
                Thread.currentThread().interrupt();
        }
    }
    //非中断，正常唤醒
    postComplete();
    return r;
}
```
#### ForkJoinPool.managedBlock阻塞
内部调用Signaller的block进行阻塞，看起来主要是调用LockSupport进行阻塞
```
public boolean block() {
    if (isReleasable())
        return true;
    else if (deadline == 0L)
        LockSupport.park(this);
    else if (nanos > 0L)
        LockSupport.parkNanos(this, nanos);
    return isReleasable();
}
final boolean isLive() { return thread != null; }
}
```
#### Signaller唤醒
既然阻塞是通过LockSupport.park，那么唤醒理应是LockSupport.unpark，这个在Signaller的tryFire内部
```
final CompletableFuture<?> tryFire(int ignore) {
    Thread w; // no need to atomically claim
    if ((w = thread) != null) {
        thread = null;
        LockSupport.unpark(w);
    }
    return null;
}
```
再看下的定义
```
static final class Signaller extends Completion
    implements ForkJoinPool.ManagedBlocker {
    long nanos;                    // wait time if timed
    final long deadline;           // non-zero if timed
    volatile int interruptControl; // > 0: interruptible, < 0: interrupted
    volatile Thread thread;

    Signaller(boolean interruptible, long nanos, long deadline) {
        this.thread = Thread.currentThread();//当前线程
        this.interruptControl = interruptible ? 1 : 0;//支持中断默认值为1
        this.nanos = nanos;
        this.deadline = deadline;
    }
    final CompletableFuture<?> tryFire(int ignore) {
        Thread w; // no need to atomically claim
        if ((w = thread) != null) {
            thread = null;
            LockSupport.unpark(w);//唤醒当前线程
        }
        return null;
    }
    public boolean isReleasable() {
        if (thread == null)
            return true;
        if (Thread.interrupted()) {
            int i = interruptControl;
            interruptControl = -1;
            if (i > 0)
                return true;
        }
        if (deadline != 0L &&
            (nanos <= 0L || (nanos = deadline - System.nanoTime()) <= 0L)) {
            thread = null;
            return true;
        }
        return false;
    }
    public boolean block() {
        if (isReleasable())
            return true;
        else if (deadline == 0L)
            LockSupport.park(this);
        else if (nanos > 0L)
            LockSupport.parkNanos(this, nanos);
        return isReleasable();
    }
    final boolean isLive() { return thread != null; }
} 
```
Signaller继承了Completion
Completion继承了ForkJoinTask，实现Runnable
```
abstract static class Completion extends ForkJoinTask<Void>
    implements Runnable, AsynchronousCompletionTask {
    volatile Completion next;      // Treiber stack link

    /**
        * Performs completion action if triggered, returning a
        * dependent that may need propagation, if one exists.
        *
        * @param mode SYNC, ASYNC, or NESTED
        */
    abstract CompletableFuture<?> tryFire(int mode);

    /** Returns true if possibly still triggerable. Used by cleanStack. */
    abstract boolean isLive();

    public final void run()                { tryFire(ASYNC); }//Runnable触发tryFire
    public final boolean exec()            { tryFire(ASYNC); return true; }//由ForkJoinTask调用
    public final Void getRawResult()       { return null; }
    public final void setRawResult(Void v) {}
}
```


