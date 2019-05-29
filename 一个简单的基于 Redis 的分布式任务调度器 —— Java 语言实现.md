---
title: 一个简单的基于 Redis 的分布式任务调度器 —— Java 语言实现
date: 2019-04-30 00:00:00
tags: [Java,分布式]
published: true
hideInList: false
feature: https://user-gold-cdn.xitu.io/2019/4/29/16a682144a242fdf?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文来自于 [https://juejin.im/post/5cc6ade4f265da038d0b4fd6](https://juejin.im/post/5cc6ade4f265da038d0b4fd6) 

折腾了一周的 `Java Quartz`集群任务调度，很遗憾没能搞定，网上的相关文章也少得可怜，在多节点（多进程）环境下 Quartz 似乎无法动态增减任务，恼火。无奈之下自己撸了一个简单的任务调度器，结果只花了不到 2天时间，而且感觉非常简单好用，代码量也不多，扩展性很好。

![](https://user-gold-cdn.xitu.io/2019/4/29/16a682144a242fdf?imageView2/0/w/1280/h/960/ignore-error/1)

实现一个分布式的任务调度器有几个关键的考虑点

1. 单次任务和循环任务好做，难的是 cron 表达式的解析和时间计算怎么做？
1. 多进程同一时间如何保证一个任务的互斥性？
1. 如何动态变更增加和减少任务？

## 代码实例

在深入讲解实现方法之前，我们先来看看这个调度器是如何使用的
```
class Demo {
    public static void main(String[] args) {
        var redis = new RedisStore();
        // sample 为任务分组名称
        var store = new RedisTaskStore(redis, "sample");
        // 5s 为任务锁寿命
        var scheduler = new DistributedScheduler(store, 5);
        // 注册一个单次任务
        scheduler.register(Trigger.onceOfDelay(5), Task.of("once1", () -> {
            System.out.println("once1");
        }));
        // 注册一个循环任务
        scheduler.register(Trigger.periodOfDelay(5, 5), Task.of("period2", () -> {
            System.out.println("period2");
        }));
        // 注册一个 CRON 任务
        scheduler.register(Trigger.cronOfMinutes(1), Task.of("cron3", () -> {
            System.out.println("cron3");
        }));
        // 设置全局版本号
        scheduler.version(1);
        // 注册监听器
        scheduler.listener(ctx -> {
            System.out.println(ctx.task().name() + " is complete");
        });
        // 启动调度器
        scheduler.start();
    }
}
```

当代码升级任务需要增加减少时（或者变更调度时间），只需要递增全局版本号，现有的进程中的任务会自动被重新调度，那些没有被注册的任务（任务减少）会自动清除。新增的任务（新任务）在老代码的进程里是不会被调度的（没有新任务的代码无法调度），被清除的任务（老任务）在老代码的进程里会被取消调度。

比如我们要取消 period2 任务，增加 period4 任务
```
class Demo {
    public static void main(String[] args) {
        var redis = new RedisStore();
        // sample 为任务分组名称
        var store = new RedisTaskStore(redis, "sample");
        // 5s 为任务锁寿命
        var scheduler = new DistributedScheduler(store, 5);
        // 注册一个单次任务
        scheduler.register(Trigger.onceOfDelay(5), Task.of("once1", () -> {
            System.out.println("once1");
        }));
        // 注册一个 CRON 任务
        scheduler.register(Trigger.cronOfMinutes(1), Task.of("cron3", () -> {
            System.out.println("cron3");
        }));
        // 注册一个循环任务
        scheduler.register(Trigger.periodOfDelay(5, 10), Task.of("period4", () -> {
            System.out.println("period4");
        }));
        // 递增全局版本号
        scheduler.version(2);
        // 注册监听器
        scheduler.listener(ctx -> {
            System.out.println(ctx.task().name() + " is complete");
        });
        // 启动调度器
        scheduler.start();
    }
}
```

## cron4j

```
<dependency>
	<groupId>it.sauronsoftware.cron4j</groupId>
	<artifactId>cron4j</artifactId>
	<version>2.2.5</version>
</dependency>
```

这个开源的 library 包含了基础的 cron 表达式解析功能，它还提供了任务的调度功能，不过这里并不需要使用它的调度器。我只会用到它的表达式解析功能，以及一个简单的方法用来判断当前的时间是否匹配表达式（是否该运行任务了）。

我们对 cron 的时间精度要求很低，1 分钟判断一次当前的时间是否到了该运行任务的时候就可以了。
```
class SchedulingPattern {
    // 表达式是否有效
    boolean validate(String cronExpr);
    // 是否应该运行任务了(一分钟判断一次)
    boolean match(long nowTs);
}
```

## 任务的互斥性

因为是分布式任务调度器，多进程环境下要控制同一个任务在调度的时间点只能有一个进程运行。使用 Redis 分布式锁很容易就可以搞定。锁需要保持一定的时间（比如默认 5s）。

所有的进程都会在同一时间调度这个任务，但是只有一个进程可以抢到锁。因为分布式环境下时间的不一致性，不同机器上的进程会有较小的时间差异窗口，锁必须保持一个窗口时间，这里我默认设置为 5s（可定制），这就要求不同机器的时间差不能超过 5s，超出了这个值就会出现重复调度。
```
public boolean grabTask(String name) {
    var holder = new Holder<Boolean>();
    redis.execute(jedis -> {
        var lockKey = keyFor("task_lock", name);
        var ok = jedis.set(lockKey, "true", SetParams.setParams().nx().ex(lockAge));
        holder.value(ok != null);
    });
    return holder.value();
}
```

## 全局版本号

我们给任务列表附上一个全局的版本号，当业务上需要增加或者减少调度任务时，通过变更版本号来触发进程的任务重加载。这个重加载的过程包含轮询全局版本号（Redis 的一个key），如果发现版本号变动，立即重新加载任务列表配置并重新调度所有的任务。
```
private void scheduleReload() {
    // 1s 对比一次
    this.scheduler.scheduleWithFixedDelay(() -> {
        try {
            if (this.reloadIfChanged()) {
                this.rescheduleTasks();
            }
        } catch (Exception e) {
            LOG.error("reloading tasks error", e);
        }
    }, 0, 1, TimeUnit.SECONDS);
}
```

重新调度任务先要取消当前所有正在调度的任务，然后调度刚刚加载的所有任务。

```
private void rescheduleTasks() {
    this.cancelAllTasks();
    this.scheduleTasks();
}

private void cancelAllTasks() {
    this.futures.forEach((name, future) -> {
        LOG.warn("cancelling task {}", name);
        future.cancel(false);
    });
    this.futures.clear();
}
```

因为需要将任务持久化，所以设计了一套任务的序列化格式，这个也很简单，使用文本符号分割任务配置属性就行。

```
// 一次性任务(startTime)
ONCE@2019-04-29T15:26:29.946+0800
// 循环任务，(startTime,endTime,period)，这里任务的结束时间是天荒地老
PERIOD@2019-04-29T15:26:29.949+0800|292278994-08-17T15:12:55.807+0800|5
// cron 任务，一分钟一次
CRON@*/1 * * * *

$ redis-cli
127.0.0.1:6379> hgetall sample_triggers
1) "task3"
2) "CRON@*/1 * * * *"
3) "task2"
4) "PERIOD@2019-04-29T15:26:29.949+0800|292278994-08-17T15:12:55.807+0800|5"
5) "task1"
6) "ONCE@2019-04-29T15:26:29.946+0800"
7) "task4"
8) "PERIOD@2019-04-29T15:26:29.957+0800|292278994-08-17T15:12:55.807+0800|10"
```

## 线程池

时间调度会有一个单独的线程（单线程线程池），任务的运行由另外一个线程池来完成（数量可定制）。
```
class DistributedScheduler {
    private ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
    private ExecutorService executor = Executors.newFixedThreadPool(threads);
}
```

之所以要将线程池分开，是为了避免任务的执行（IO）影响了时间的精确调度。

## FixedDelay vs FixedRate

Java 的内置调度器提供两种调度策略 FixedDelay 和 FixedRate。FixedDelay 保证同一个任务的连续两次运行有相等的时延（nextRun.startTime - lastRun.endTime），FixedRate 保证同一个任务的连续运行有确定的间隔（nextRun.startTime - lastRun.startTime）。

![](https://user-gold-cdn.xitu.io/2019/4/29/16a68a816b58ca9e?imageView2/0/w/1280/h/960/ignore-error/1)

FixedDelay 就好比你加班到深夜12点，可以第二天12点再来上班(保证固定的休息时间)，而 FixedRate 就没那么体贴了，第二天你继续 9点过来上班。如果你不走运到第二天 9 点了还在加班，那你今天就没有休息时间了，继续上班吧。
```
class ScheduledExecutorService {
    void scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);
    void scheduleAtFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);
}
```

分布式调度器要求有精确的调度时间，所以必须采用 FixedRate 模式，保证多节点同一个任务在同一时间被争抢。如果采用 FixedDelay 模式，会导致不同进程的调度时间错开了，分布式锁的默认 5s 时间窗口将起不到互斥作用。

## 支持无互斥任务

互斥任务要求任务的单进程运行，无互斥任务就是没有加分布式锁的任务，可以多进程同时运行。默认需要互斥。
```
class Task {
    /**
     * 是否需要考虑多进程互斥（true表示不互斥，多进程能同时跑）
     */
    private boolean concurrent;
    private String name;
    private Runnable runner;
    ...
    public static Task of(String name, Runnable runner) {
        return new Task(name, false, runner);
    }

    public static Task concurrent(String name, Runnable runner) {
        return new Task(name, true, runner);
    }
}
```

## 增加回调接口

考虑到调度器的使用者可能需要对任务运行状态进行监控，这里增加了一个简单的回调接口，目前功能比较简单。能汇报运行结果（成功还是异常）和运行的耗时
```
class TaskContext {
    private Task task;
    private long cost;  // 运行时间
    private boolean ok;
    private Throwable e;
}

interface ISchedulerListener {
    public void onComplete(TaskContext ctx);
}
```

## 支持存储扩展

目前只实现了 Redis 和 Memory 形式的任务存储，扩展到 zk、etcd、关系数据库也是可行的，实现下面的接口即可。
```
interface ITaskStore {
  public long getRemoteVersion();
  public Map<String, String> getAllTriggers();
  public void saveAllTriggers(long version, Map<String, String> triggers);
  public boolean grabTask(String name);
}
```

## 代码地址

[github.com/pyloque/tas…](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fpyloque%2Ftaskino)

