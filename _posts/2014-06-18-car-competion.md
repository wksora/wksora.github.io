---
layout: post
title: "多个线程如何同时启动？Java并发编程初体验"
description: ""
category: 
tags: ["java", "多线程", "并发", "concurrent", "AbstractQueuedSynchronizer"]
---
{% include JB/setup %}

### 一道面试题 ###
这是我朋友问我的一题面试题，题目大意是，给你n个赛车，让他们都在起跑线上就绪后，同时出发，让你用Java多线程的技术把这种情况写出来。

搜索一下如何让多个线程同时启动，答案里面无非都是在说不可能会真的做到同时，然后略略的谈了CPU调度等之类的泛泛。首先这是一个面试题所以我们得清楚，面试官其实是想问我们多线程某一方面的知识点。作为面试，还是应该就一些问题跟面试官做一些交流，比如我也在一开始就跟同学提出了我的疑问，就算某种方式能让他们同时出发，但是cpu时间段的分配还是无法做到同时。我也觉得如果能提出这种疑问，也是面试加分项。

### 第一感觉 ###
言归正题，我们来看看怎么做这个面试题。显然题意在勾引我们应该抽象每个赛车为每个线程，我们注意到这些线程执行赛车启动方法之前都在等待一个状态，即所有参加比赛的赛车都在起跑线了。当这个状态达到之后，所有的赛车线程开始执行赛车启动、运行、到达终点的过程。多线程里面如何表达这样一个过程呢？也即如何让所有的线程等待某一个状态，而当某个状态达到目标状态之后让所有线程重新执行呢？粗粗的想应该会有感觉跟对象的wait和notify有关系，然后就会想到一个方法，来一辆赛车就加上一把锁，并修改对应的操作数，如果没有全部就绪就等待，并释放锁，直到最后一辆赛车到场后唤醒所有的赛车线程。直接贴代码：

```java
public class CarCompetion {
	// 参赛赛车的数量
	protected final int totalCarNum = 10;

	// 当前在起跑线的赛车数量
	protected int nowCarNum = 0;
}
```

这个类用于充当锁对象和维护状态的对象，简单起见先直接用protected。

```java
public class Car implements Runnable{

	private int carNum;
	
	private CarCompetion competion = null;
	
	public Car(int carNum, CarCompetion competion) {
		this.carNum = carNum;
		this.competion = competion;
	}
	
	@Override
	public void run() {
		synchronized (competion) {
			competion.nowCarNum++;
			while (competion.nowCarNum < competion.totalCarNum) {
				try {
					competion.wait();
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
			competion.notifyAll();
		}
		startCar();
	}
	
	private void startCar() {
		System.out.println("Car num " + this.carNum + " start to run.");
		try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("Car num " + this.carNum + " get to the finish line.");
	}

}
```

这个Car类表示一辆赛车，实现了Runnable，参数构造传入了赛车的id和赛场，私有方法startCar表示启动赛车的函数。重点就在于合适去调用这个函数。

定位到我在实现Car线程类的run函数的地方，方法首先对competion赛场类加锁，修改当前到赛场赛车的数量，判断是否已经达到赛场的同时启动的要求，如果没有到达就调用competion的wait方法，释放competion锁，让其他Car线程可以来修改状态。

最后一辆车到场的时候会符合启动要求，于是最后一辆车会调用notifyAll方法通知赛场上的所有其他的赛车可以开始跑啦，问题初步的解决了。

下面给一下执行的main方法：

```java
public static void main(String[] args) {
	CarCompetion carCompetion = new CarCompetion();
	
	final ExecutorService carPool = 
		Executors.newFixedThreadPool(carCompetion.totalCarNum);
	for (int i = 0; i < carCompetion.totalCarNum; i++) {
		carPool.execute(new Car(i, carCompetion));
	}
}
```

其实在这个问题上类似的解法还有很多种，其他解法自行搜索吧。

### 进一步思考 ###
前面标题是第一感觉，所以这个解法并不是我觉得满意的。题目要求所有赛车同时，我们就应该尽可能的在cpu层，jvm调度层之上让所有线程同时，在之前这个解法里面有一个小问题，我们看到是最后一辆赛车通知的所有线程可以开始跑，换言之他其实占了大便宜不是么，其实他是第一个开始偷跑的。那我们想想怎么改善这个问题吧，把notifyAll这个事情外放是个直观的方法。还有一些问题存在，比如向这段代码的写法太固定太死板了，只能10辆车参加比赛，设计的灵活一点会更加好。

就算达到上面的这些要求，还不能让我自己觉得满意。刚才提到之前的解法最后一辆到场的赛车其实是第一个开始调用startCar()的，那么后面的赛车就都是被同时调用了么？其实不是，咱们先撇开cpu调度那套，看看notifyAll()干了什么。

> JDK中 Object.notifyAll方法的释义
> 
> public final void notifyAll()
>
> 唤醒在此对象监视器上等待的所有线程。线程通过调用其中一个 wait 方法，在对象的监视器上等待。
>
> 直到当前线程放弃此对象上的锁定，才能继续执行被唤醒的线程。被唤醒的线程将以常规方式与在该对象上主动同步的其他所有线程进行竞争；例如，唤醒的线程在作为锁定此对象的下一个线程方面没有可靠的特权或劣势。

看出来了么，至少我感觉到了我之前解法的不足，notifyAll看名字好像是通知到所有其他的赛车，让他们开始启动了！其实不然，它是让所有在赛车上排队的赛车们醒来了，但是这些赛车们还是得竞争赛场的锁，最后只有一个赛车会胜出（线程仍为阻塞状态，所处锁对象的锁池），其他赛车乖乖的继续wait吧（线程仍为阻塞状态，所处锁对象的等待池）。等前一辆赛车的run方法，执行完synchronized同步块后，锁才真正的交给它（从锁池中拿一个线程出来，转为运行态），然后他才执行notifyAll通知剩下的赛车，最后跟着前一辆先不客气的跑了。这里需要看一下[Java线程生命周期](http://www.cnblogs.com/mengdd/archive/2013/02/20/2917966.html)这篇文章，会对Java线程的生命周期有初步的了解。

Jvm对运行态的线程有两种定义，一个是Runnable一个是Running，其中Running线程可以通过`Thread.currentThread()`确认自己是否是当前正在运行的，运行态之间的切换是Jvm与CPU在调度。那我们还能做的是什么呢，刚才的解法里面所有的Car线程都呆在锁对象的等待池，是不是可以尽早让他们进入锁池里面去等待呢？更进一步，能不能让他们唤醒的时候就可以得到调度呢？

### Java并发框架—AbstractQueuedSynchronizer ###
可能你还会想到join()之类线程操作，但是就我思考而言靠Java的语法糖我们不能再在这个问题上更进一步了。JDK在1.4起，为了保证java在多线程方面的能力，就在`java.util.concurrent`为我们准备了很多礼物，其中一个即`java.util.concurrent.locks.AbstractQueuedSynchronizer`是我们在这个问题上能更进一步的目标。

这个类可以说构成了j.u.c所有内容的基础类了，遍布了`Lock`，`Condition`，`Semaphore`以及各种并发容器等等的实现，而且它更像一种框架，充满了神秘色彩，调用了`sun.misc.Unsafe`，要讲清楚我的能力还有限，而且一般程序员，比如我，在这一辈子都可能不会碰到会跟它扯上关系的业务开发，但我会稍微扯扯，更多的需要自己去google，以及看相关的源码分析，这里还有一篇论文是关于这个的[The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)。

先列一下`AbstractQueuedSynchronizer`的方法，有点多根据需要会介绍（未包含所有的，比如可中断的acquire方法等）：
> To use this class as the basis of a synchronizer, redefine the following methods, as applicable, by inspecting and/or modifying the synchronization state using getState(), setState(int) and/or compareAndSetState(int, int):
> 
>
> protected boolean tryAcquire(int)
>
> protected boolean tryRelease(int)
>
> protected boolean tryAcquireShared(int)
>
> protected boolean tryReleaseShared(int)
> 
> public final void acquire(int)
>
> public final void acquireShared(int)
>
> public final void release(int)
>
> public final void releaseShared(int)

`AbstractQueuedSynchronizer`本身不再依靠java的`synchronized`关键字做同步控制了，思考一下同步本质在做什么事情呢？我们同步线程的时候往往需要对对象加锁，加锁的目的是为了让该对象的值或者该对象守护的值进行“同步修改”，即你修改的过程不会被干扰，别人直到你修改后才会读到你修改后的值，所以从某个角度来说，同步的目的是为了同步状态。没有了语法上的帮助，AQS采用了CPU级别的并发原语，也就是介绍中的compareAndSetState(int, int)方法，即比较并设置操作（或者比较并替换compare and swap，有些地方这么称，简称都是CAS操作）。我们通过`getState()`拿到了一个状态A，和状态隐含的地址&A，需要对它进行计算f并得到新的状态B，我们现在要B写到&A上，我们要保证此时的&A上的值仍为读取用于计算的原来的A才进行赋值，如果相等则赋值&A地址上的值为B，如果不相等则重新`getState()`，计算f，并再次CAS，这里是一个**自旋锁**（即不等待或者等待短时间就重新进行操作，区别于阻塞通知模型）的过程。这里举一个例子表达一下刚才的过程。

```java
for(;;) {
	int A = getState();
	int B = A + val;
	if (compareAndSet(A, B))
		break;
}
```

在AQS的首要功能就是维护这么一个状态，我们的赛车问题状态就是还差几辆车到场，信号量里面的资源数量就是状态等等，状态在AQS里面形容成资源可能更恰当，acquire方法表意申请资源，release方法表意释放资源，对应的分别有try版本以及shared版本。其中try版本的方法是final修饰的，即不要尝试去修改，try版本的方法会首先去尝试调用非try版的方法，因此非try版的方法是允许我们根据实际场景的不同做修改的。shared版本表明资源获取是非独占的，**独占**就类似于synchronized做的事情，我获取锁你就拿不到，除非我释放；**非独占**就类似信号量，每个线程都可以pv操作，不需要对信号量去加锁。

线程通过acquire方法去调用tryAcquire尝试申请资源，这个方法会返回boolean值表明是否申请到（我们就在这里做文章），如果申请不到就会被压入阻塞队列等待，一直等到有线程调用release释放了资源，acquire方法会继续，判断是否申请得到资源，如果可以了，就退出阻塞队列，并返回true，否则继续等待。因此AQS里面也维护了一个阻塞线程的队列，它是一个链表的形式存在的，对这个链表的操作也用到了CAS，不细讲了，不过需要注意的是线程的公平竞争与非公平竞争与这个阻塞队列的实现是有关系的。

介绍到这里，应该有点思路了吧。我们要为赛车问题设计一个同步量（同步量就类似于信号量、Lock对象等等用于同步的对象），也就是闭锁，等到某个状态的锁。首先赛车这个问题对应的是一个非独占的同步量，每辆车都可以同时去申请是否可以出发这个状态，因此应该用AQS的shared版本。状态怎么设计呢，我们把状态定义为当前还有几辆车没有到场，那么显然当状态为0的时候，所有的acquireShared方法得以继续，这里有一点与synchronized的区别点，在这里被唤醒的acquireShared方法按照公平或者非公平的顺序从AQS维护的线程阻塞队列里面的里面被唤醒后，直接继续去验证是否可以通过状态（tryAcquireShared）并继续执行该线程的方法，也即他们在同时的检查状态，而不是在synchronized中那样等一个线程检查完之后再交给下一个线程检查。

### 基于AQS的实现 ###
代码时间，先贴一下我实现的闭锁：

```java
public class CompetionLatch {
	
	public void await() {
		sync.acquireShared(0);
	}
	
	public void countDown() {
		sync.releaseShared(0);
	}
	
	public CompetionLatch(int total) {
		this.sync = new Sync(total);
	}
	
	private Sync sync = null;
	
	@SuppressWarnings("serial")
	private class Sync extends AbstractQueuedSynchronizer {
		public Sync(int num) {
			super();
			setState(num);
		}
		
		protected int tryAcquireShared(int ignore) {
			// 如果状态为0就返回通过，否则返回失败。
			return getState() == 0? 1: -1;
		}
		
		protected boolean tryReleaseShared(int ignore) {
			for (;;) {
				// 获取状态
				int a = getState();
				// 如果状态为0就退出
				if (a == 0) {
					break;
				}
				// 思考一下，如果这个时候别的线程把状态修改为0了，会发生什么？
				// 检查之前获取的状态与现在是否一致，一致则修改并退出；否则自旋重试
				if (compareAndSetState(a, a - 1)) {
					break;
				}
			}
			// 简单考虑，每次release都唤醒队列
			// 思考点，是否可以判断状态是否为0来确定是否应该去唤醒队列？
			return true;
		}
	}
}
```

然后还是赛车线程类：

```java
public class Car implements Runnable {

	private int carNum;
	
	private CompetionLatch latch = null;
	
	public Car(int carNum, CompetionLatch latch) {
		this.carNum = carNum;
		this.latch = latch;
	}
	
	@Override
	public void run() {
		this.latch.countDown();
		this.latch.await();
		startCar();
	}

	private void startCar() {
		System.out.println("Car num " + this.carNum + " start to run.");
		try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("Car num " + this.carNum + " get to the finish line.");
	}
}
```

最后是main程序：

```java
public class CarCompetion {
	
	public static final int CAR_NUM = 10;
	
	public static void main(String[] args) {
		// 构造同步量
		final CompetionLatch latch = new CompetionLatch(CarCompetion.CAR_NUM);
		final ExecutorService carPool = 
			Executors.newFixedThreadPool(CarCompetion.CAR_NUM);
		for (int i = 0; i < CarCompetion.CAR_NUM; i++) {
			carPool.execute(new Car(i, latch));
		}
	}
}

```

需要解释的基本在注释里面了，如果有兴趣的话，可以想想怎么再利用闭锁，让所有赛车都到终点后，宣布比赛结束呢？

### 好吧，公布正确答案 ###
说了这么多，其实有些对j.u.c熟悉的人一开始就会表示知道java本身就提供了这个问题的实现`CountDownLatch`和`CyclicBarrier`。前者是一次性的，用于类似赛车的同时启动场景，或者等待所有线程执行结束。后者是可以多次用的，可以处理线程运行间的依赖，比如执行某个线程，必须保证前置的几个线程任务完成。API都比较简单，这里简单说下`CountDownLatch`的用法，具体可以查看JDK说的会更详细。

构造`CountDownLatch`的时候需要传递构造函数即有几辆车：

```java
CountDownLatch competion = new CountDownLatch(10);
```

每个赛车进程先后去调用CountDownLatch的countDown方法与await方法（为了与Object.wait区分）。即在赛车进程的run函数里面有代码：

```java
competion.countDown();
competion.await();
```

这里要注意的是countDown()当计数已经为0时，不做任何操作；操作后为0则通知所有阻塞线程。await()在计数不为0时，阻塞等待计数为0的通知；为0时不阻塞，直接返回。await()的方法可以被中断，且有时限版的api。

> public boolean await(long timeout,TimeUnit unit)
throws InterruptedException

示例代码如前一节，我实现的闭锁api与CountDownLatch一致。

### 你看完了 ###
本文从一个面试题出发，引出了一个闭锁的实现问题，首先给出了java语法层实现的方法，并指出不足，然后介绍了`j.u.c.locks.AQS`，并基于它做了新的示例，这下可以更接近题目要求的“**同时**”了。最后告诉你其实java本身就提供了闭锁的实现。不过在过程中知道这个闭锁里面是怎么工作的之后，应该多少会对`j.u.c`充满了好奇和学习的冲动吧。如果有兴趣就开始扩展阅读吧！


**除了文章中有特别说明，均为本人bevoid原创文章，转载请以链接形式注明出处。**
