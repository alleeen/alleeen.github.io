---
layout: drafts
title: Spark 任务的生命周期
date: 2016-09-18 22:09:10
comments: true
ads: true
categories: [大数据,Spark]
tags: [Spark,Scala,大数据]
---

在日常的工作中，相信诸位小伙伴已经把 RDD 的 API 玩得相当熟练了，也了解了 transform 和 action 的区别。那你知道当你触发一个 action 后，在 Spark 内部究竟发生了什么么？接下来让我们打开 Spark 的源码揭示一个 Spark 任务内部的奥秘。

## 一些概念
在开始之前我们先来回顾一些 Spark 的概念：

- Jobs 是提交到调度器（scheduler）的顶层工作项（the top-level work items）。举个例子，当用户调用诸如`count()`这样的 action 方法时，Spark 会通过`submitJob`方法将这个 Job 提交到调度器。每个 Job 可能需要执行多个 Stage 来构建它的过程数据。

- 

*
*  - Jobs (represented by [[ActiveJob]]) are the top-level work items submitted to the scheduler.
*    For example, when the user calls an action, like count(), a job will be submitted through
*    submitJob. Each Job may require the execution of multiple stages to build intermediate data.
*
*  - Stages ([[Stage]]) are sets of tasks that compute intermediate results in jobs, where each
*    task computes the same function on partitions of the same RDD. Stages are separated at shuffle
*    boundaries, which introduce a barrier (where we must wait for the previous stage to finish to
*    fetch outputs). There are two types of stages: [[ResultStage]], for the final stage that
*    executes an action, and [[ShuffleMapStage]], which writes map output files for a shuffle.
*    Stages are often shared across multiple jobs, if these jobs reuse the same RDDs.
*
*  - Tasks are individual units of work, each sent to one machine.
*
*  - Cache tracking: the DAGScheduler figures out which RDDs are cached to avoid recomputing them
*    and likewise remembers which shuffle map stages have already produced output files to avoid
*    redoing the map side of a shuffle.
*
*  - Preferred locations: the DAGScheduler also computes where to run each task in a stage based
*    on the preferred locations of its underlying RDDs, or the location of cached or shuffle data.
*
*  - Cleanup: all data structures are cleared when the running jobs that depend on them finish,
*    to prevent memory leaks in a long-running application.
