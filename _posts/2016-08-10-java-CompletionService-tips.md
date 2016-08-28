---
layout: post
title:  "CompletionService小技巧"
date:   2016-08-10 16:50:39
comments: true
ads: true
categories: 软件技术
tags: [Java, 多线程]
---

在上一篇blogs中，我详细的解释了`CompletionService`的使用方法和`ExecutorCompletionService`的详细实现，这篇blogs中，我就介绍使用它的一个小技巧，算是对上一篇blogs的一个补完。在开始之前我们先回顾一下它的实现。

<!--more-->

首先，在初始化`ExecutorCompletionService`的时候我们需要传入一个`Executor`，作为`ExecutorCompletionService`执行任务的容器。

```java
public ExecutorCompletionService(Executor executor) {
    [......]
}

public ExecutorCompletionService(Executor executor,
                                 BlockingQueue<Future<V>> completionQueue) {
    [......]
}

```
然后，调用`submit`方法，向它提交任务。`submit`方法会将我们提交的任务包装成一个`QueueingFuture`并提交给`Executor`来执行。

```java
public Future<V> submit(Callable<V> task) {  
    if (task == null) throw new NullPointerException();  
    RunnableFuture<V> f = newTaskFor(task);  
    executor.execute(new QueueingFuture(f));  
    return f;  
}  
```

接着，`QueueingFuture`会在任务执行完成后把执行结果放到队列中。

```java
private class QueueingFuture extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task) {
        super(task, null);
        this.task = task;
    }
    protected void done() { completionQueue.add(task); }
    private final Future<V> task;
}
```

最后，我们通过`take`或者`poll`方法就能拿到任务执行的结果。

下面让我们设想一个场景，我需要从网络上下载几张图片和视频并最后把它们渲染到页面上去，由于下载图片和视频都比较耗时，所以我希望能以多线程的形式进行下载。但是由于资源有限，下载的并发度不能太大，所以需要限制线程池的并发线程大小。但如果将可用线程数平均分给下载图片和下载视频的线程池，当某线程池的所有任务执行完成后，另外一个线程池也无法获取到它所释放的资源。那怎么办呢？我们可以创建一个统一的线程池，然后把两个CompletionService绑定上去，让CompletionService作为一个句柄来使用。

```java
final ExecutorService pool = Executors.newFixedThreadPool(5);

final ExecutorCompletionService<Image> imageCompletionService = new ExecutorCompletionService<>(pool);
for (final String site : imageSites) {
    completionService.submit(new Callable<Image>() {
        @Override
        public String call() throws Exception {
            return IOUtils.toString(new URL("http://" + site), StandardCharsets.UTF_8);
        }
    });
}

final ExecutorCompletionService<Video> vidoeCompletionService = new ExecutorCompletionService<>(pool);
for (final String site : videoSites) {
    completionService.submit(new Callable<Video>() {
        @Override
        public String call() throws Exception {
            return IOUtils.toString(new URL("http://" + site), StandardCharsets.UTF_8);
        }
    });
}

List<Image> images = new ArrayList<>();
for(int i = 0; i < topSites.size(); ++i) {
    final Future<String> future = completionService.take();
    try {
        images.add(future.get());
    } catch (ExecutionException e) {
        log.warn("Error while downloading", e.getCause());
    }
}

List<Video> videos = new ArrayList<>();
for(int i = 0; i < topSites.size(); ++i) {
    final Future<String> future = completionService.take();
    try {
        videos.add(future.get());
    } catch (ExecutionException e) {
        log.warn("Error while downloading", e.getCause());
    }
}
// ... do process content
```
