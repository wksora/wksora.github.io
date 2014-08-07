---
layout: post
title: "如何等待多个分片任务的执行结果——分享一个关于混用同步与异步机制的错误"
description: ""
category: 
tags: ["java", "多线程", "同步", "异步"]
---
{% include JB/setup %}

分享一个自己在处理等待分片任务执行时犯的错误。我将这个错误的原因归结为没有妥善的同时使用好Java的同步和异步机制。

### 开发中遇到的需求 ###
开发过程中有这样一个需求，主线程有一个步骤需要拆分为多个任务线程，然后主线程等待多个任务线程全部执行完后，收集任务线程的结果，然后继续主线程的逻辑。其中要求可以控制这段等待的总时间，即如果超过一定时间还没有完成所有的任务，就直接收集已经结束的线程运算的结果，然后继续主线程的逻辑。

这算是一个比较常见的通过多线程来加快任务执行的场景（在具体业务里面分片任务涉及IO阻塞，所以同步执行下会比较慢）。

### 初始的想法与实现 ###
在分割任务之前已知总的任务数有多少，为了满足限定总体时长，我选择了`CountDownLatch`作为锁等待所有线程执行完毕。为了得到线程的执行结果，我使用了`Futrue`获取。我构造的线程类如下：

```java
public class WorkThread implements Callable<Object> {

    /** 控制任务完成的闭锁 */
    private CountDownLatch finishLatch;

    public WorkThread(CountDownLatch finishLatch) {
        this.finishLatch = finishLatch;
    }

    /**
     * @see java.util.concurrent.Callable#call()
     */
    public Object call() throws Exception {
        try {
            //TODO Work
            return result;
        } finally {
            finishLatch.countDown();
        }
    }

}
```

在主线程里面利用一个`List<Future<Object>>`存放线程执行的结果，`CountDownLatch`的限时`await`方法来等待所有的线程执行完成。等待闭锁阻塞完成后，收集已完成的任务。

```java
final CountDownLatch latch = new CountDownLatch(n);
final ThreadPoolExecutors executors = Executors.newCachedThreadPool();
List<Future<Object>> futureList = new ArrayList<Future<Object>>();
// 添加执行异步任务
for (int i = 0; i < n; i++) {
	futureList.add(executors.submit(new WorkThread(latch)));
}

// 等待任务完成，搜集结果
try {
    boolean res = latch.await(MAX_EXCUTE_TIME, TimeUnit.MILLISECONDS);
    for (Future<Object> future : futureList) {
        if (future.isDone()) {
            try {
                // 有isDone()判断，肯定已完成
                Object result = future.get();
                // handle the result
            } catch (Exception e) {
            }
        } else {
            future.cancel(true);
        }
    }

} catch (InterruptedException e) {
}

```

### 问题 ###

在实际测试时发现程序总是会漏掉一些子任务，漏掉的概率和被漏的任务是随机的，便觉得很奇怪。我在所有会引起任务丢失的地方加了断点，在Debug时发现有部分任务在时间足够的情况下进入了```future.cancel(true);```分支，于是意识到了问题，我在同步场景下乱用了异步工具方法。```WorkThread```即线程类的执行是由线程池来控制的，这里提交了异步任务执行，线程类任务完成后```Future```的```isDone()```状态的更新操作是由线程池控制。在主线程中，等到闭锁为0时立即开始下面的逻辑，而future的完成状态由另一个线程控制，因此已完成的任务也有可能没有被及时的更改为已完成的状态而让主线程执行了cancel的分支。


### 解决方案 ###
1. 去掉future.isDone()判断，直接用future.get()的时限版获取结果，设定一个较短的时间。但是这种写法其实还是没有满足控制整体任务执行超时的限制，因为存在未完成的任务的可能，尽管设定了一个较短的时间，也有可能因为较长的任务队列使得总体运行时间延长。但是有一个好处，如果超时未获取到结果，就可以采取cancel的策略。
2. 利用一个线程安全的List存放结果，取代Future的结构。即将线程类修改为如下。

```java
public class WorkThread implements Runnable {

    /** 控制任务完成的闭锁 */
    private CountDownLatch finishLatch;
    /** 存放结果的同步List */
    private List<Object> resultList;

    public WorkThread(List<Object> resultList, CountDownLatch finishLatch) {
        this.resultList = resultList;
        this.finishLatch = finishLatch;
    }

    /**
     * @see java.util.concurrent.Callable#call()
     */
    public void run() {
        try {
            //TODO Work
            resultList.add(result);
        } finally {
            finishLatch.countDown();
        }
    }

}
```
其中保证对resultList的修改操作是线程安全的。比如采用：

```java
List<Object> resultList = Collections
            .synchronizedList(new ArrayList<Object>());
```

**除了文章中有特别说明，均为本人wksora原创文章，转载请以链接形式注明出处。**
