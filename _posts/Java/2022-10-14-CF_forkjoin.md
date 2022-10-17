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
### 以CompletableFuture.thenApply为例
```

```
