# Java Concurrency in Practice (2012) by Brian Goetz

- 线程安全性: 永远不发生糟糕的事情
- 线程活跃性: 某件正确的事情最终会发生 
    - 如: 无限循环带来的活跃性问题

# 2. 线程安全性

(p.12)

***"要编写线程安全的代码, 其核心在于要对状态访问操作管理"***

- **共享 (shared)**: 指变量可以由多个线程访问
- **可变 (mutable)**: 指变量的值在其生命周期内可以变化

## 线程安全性

"当多个线程访问某个类时, 不管运行环境利用何种调度方式, 或这些线程将如何交替执行...这个类都能表
现出正确的行为, 那么这个类就是线程安全的。" 同时, 该类内部封装了必要的同步机制, 使客户端不需要
额外的同步.

## 竞争条件 Race Condition

当某个计算的正确性取决于多个线程的交替执行时序.

如： 

- check-then-act
- 复合操作

## 多个原子状态变量

要保持状态的一致性, 就需要在单个原子操作中更新所有相关的状态变量. (p.20)

## 使用 Java 对象作为锁

- 同步代码块 synchronized block
- 内置锁 intrinsic lock
- 监视器锁 monitor lock

## 锁的重入

如果某个线程试图获得一个已经由它持有的锁, 这个请求会成功, 如, 为每一个锁关联一个计数器和持有者线程, 当计数为0, 则无线程持有锁, 如遇锁的重入, 则增加计数, 当线程推出同步代码块, 则减少该计数.

# 3. 对象的共享

(p.27)

## synchronized 同步代码块

- 原子性
    - 要求线程进入代码块前必须获得锁, 从此每一个同步代码块都形成不可分割的单元, 形成原子操作.
- 内存可见性 memory visibility
    - 每一个进入当前同步代码块的线程, 都能看到上一线程对数据的变更 (**Happens-Before**).

## Volatile 变量

当变量被声明为 volatile 时, compile 和 runtime 都不会进行 reordering. 同时, volatile 
变量不会被缓存到寄存器中, 所以读取时总是会返回-最新写入的值, 只有当 volatile 可以简化代码时才
应该使用, volatile 只保证可见性, 而不是原子性.

当且仅当满足以下条件时才应该使用 volatile:

1. 对变量的写入不依赖变量当前的值, 或只有单一线程更新变量的值 
2. 该变量不会与其他状态变量一起纳入不变性条件中 (一致的约束)
3. 访问变量不需要加锁

## 发布与逃逸 Publish & Escape

(p.33)

**发布 (Publish)**: 

- 将对象发布, 使其能在当前作用域之外的代码中使用, 可以认为是公布该引用, 例如, 通过 
constructor

**逃逸 (Escape)**:

- 当某个不应该被发布的对象被发布时, 就称之为逃逸, 例如, 未完全构造的对象在 constructor 中引
用被不小心发布出去

对象的发布和逃逸破坏封装性 (Encapsulation), 导致难以维护不确定的线程安全问题, 类的设计者应假
设, 发布或逃逸的对象会被其他线程误用

## 安全的对象构作过程

(p.34)

**`this`** 引用不应该在构造器中发布或逃逸, 如果 this 在构造的过程中逃逸, 那么这种对象被称为不正确构造的对象. 可以用生命周期方法或工厂-方法去正确地构造对象然后发布引用。

## 线程封闭 Thread Confinement

(p.35)

线程封闭是一种对变量封闭到单一线程从而避免同步的, 对线程安全的实现方式. 例如, 使用 
ThreadLocal 或 volatile 用于读, 然后将写操作 confine 到单一线程中。

## 不变性 Immutability

(p.38)

不可变的对象一定是线程安全的. 当满足以下条件时, 对象才是不可变的:

1. 对象创建以后其状态就不能修改
2. 对象所有域都是 final 的
3. 对象是正确构建的

**Effective Java 中相关的 item:**   

- 除非需要更高可见读, 否则所有域都应该声明为私有
- 除非需要改变某个域, 否则应将其声明为 final

# 4. 对象的组合

(p.46)

在设计线程安全类的过程中, 需要包含以下三个元素:

1. 找出构成对象的所有变量
2. 找出约束状态量的不变性条件 (一致的约束)
3. 建立对象状态的并发访问管理策略

## 同步策略 Synchronization Policy

- 定义如何在不违背对象不变条件和后验条件 (post-condition) 的情况下对其状态的访问操作进行协同, 此策略应写成正式文档.

## 实例封闭 Instance Confinement

- 如果某一实例不是线程安全的, 我们可以用 Decorator Pattern, 将该实例封闭到一个线程安全的实例中, 同时确保不会发生对象逃逸的情况.

# 5. 基础构建模块

(p.66)

- **ConcurrentHashMap**

    - **ConcurrentHashMap** 实现了 **ConcurrentMap**, 内部使用了分段锁 (**Lock Stripping**), 使得 *"...任意数量的读取线程可以并发地访问Map, 执行读取操作的线程和执行写入操作的线程可以并发的访问 Map, 并且一定数量的写入线程可以并发的修改 Map"*. 但是对这个 Map 对象的加锁并不能提供独占访问, 只有应用需要对 Map 加锁时才应放弃使用 ConcurrentHashMap. **分段锁只用于早于1.8版本**, **1.8版本改为使用数组+链表+红黑树** 的方式实现.

- **CopyOnWriteArrayList**

    - **CopyOnWriteArrayList** 对应 List, 它提供了 **写时复制** 功能, 在访问该 List 时, 因为发布的 List 时 effectively immutable 的, 所以并不需要额外的同步, 但在修改 List 时, List 会复制一份原数组, 并在此新数组上更改. 这使得 **CopyOnWriteArrayList** 更适合大量迭代少量修改的场景, 例如, 注册监听器. 简单的来说, 就是读不加锁, 只用一个锁且用于修改, 修改基本上就是做一个全量的复制然后修改.

- **BlockingQueue**

    - **LinkedBlockingQueue**

        - 使用 **two locks, double condition**, 两个锁分别对 `put` 操作和 `take` 操作加锁 (`takeLock`, `putLock`), 同时结合使用两个 **Condition** `notEmpty` 和 `notFull` 一起使用.
        - 内部使用 LinkedList, 所以要保护的点在于 head node 和 tail node, 而两个锁 `takeLock` 和 `putLock` 则分别保护他们. 当某一线程尝试从 queue 中取值, 这一线程必须获得 `takeLock`, 获得锁以后, 该线程检查是否还有值可以拿, 如果没有就通过 `notEmpty.await` 将线程挂起同时释放锁. 如果当前 LinkedList 不为空, 则取出 `head` 节点, 如果取出以后 LinkedList 不为空 (取的同时有新的节点在 `tail` 插入) 则继续通过 `notEmpty.signal` 唤醒 `take` 操作挂起的线程, 如果 LinkedList 还有空间 (小于规定的 capacity), 则通过 `notFull.signal` 唤醒 `put` 操作挂起的线程.

    - **ArrayBlockingQueue**

        - 使用 **single lock, double condition**, 对于整个对象使用一个锁进行锁同步, 同时结合使用两个-**Condition** `notEmpty` 和 `notFull` 一起使用. 
        - 首先, `ArrayBlockingQueue` 内部储存的容器就是 array, 无论何时要更改这个 array, 线程必须争取同一个 mutex lock. 当某一线程尝试从 queue 中取值, 这一线程必须获得这个 mutex lock, 获得锁以后如果 queue 不是空的, 则获取值且放开锁, 但如果此时 queue 是空的, 则将通过 `notEmpty.await` 将线程挂起, 调用 `Condition.await` 方法在挂起线程的同时也会释放掉手中的锁, 只有当 `notEmpty.signal()` 方法调用时, 该线程才会再次唤醒, 继续-工作。当由多个线程被挂起时, `Condition.signal` 只会唤醒某一个线程.

    - **PriorityBlockingQueue**

        - 内部包含 `PriorityQueue`, 使用单一重入锁和 Condition `notEmpty` 配合使用. 

    - **SynchronousQueue**

        - 特例, 不维护存储空间而是线程, 从而提供更低延迟. Size 永远是 0, more like a handoff or action between publisher and consumer. `put` will never return until a `take` is called such that the 'transferring' happens.

- **BlockingDeque**

    - 非常适合 **Work Stealing** 算法, tasks are added to the tail and consumed from the head, one can 'steal' tasks from others tail to keep workers busy.
    - **LinkingBlockingDeque**
        - 使用 **single lock, double condition**, 对于整个对象使用一个锁进行锁同步, 同时结合使用两个 **Condition** `notEmpty` 和 `notFull` 一起使用. 
        - 基本与 `ArrayBlockingQueue` 一致

## 线程的中断 

(p.77)

Thread 的 `interrupt` 方法用于中断线程, 实际是线程中的一个 boolean 状态 `is_interrupted`. 中断是一种协作机制, 线程 A 不能强制中断线程 B, 只能请求线程 B 在能中断的时候中断, 当代码中抛出 `InterruptedException`, 有两种处理方式:

1. 传递 `InterruptedException`:
    - 将 `InterruptedException` 传递给方法调用者, 在传递前可以将一场捕获, 做一些清理工作, 然后再抛出
2. 恢复中断:
    - 如果不希望当前线程被中断, 可捕获该异常然后调用 `Thread.currentThread().interrupt()`, 该方法会再次改变 `is_interrupted` 状态, 使得线程继续工作.

```
try {

    ...

} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

***最不应该做的是, 没有理由的 suppress InterruptedException.***

## 闭锁 Latch

(p.79)

闭锁用于延迟线程的进度知道其到达终止状态. 闭锁的作用相当于一扇门, 在闭锁到达结束状态前, 这扇门一直是关闭的, 没有线程可以通过, 当到达闭锁的结束状态时, 这扇门将打开. 总而言之, 闭锁用于确保某些活动知道其他活动都完成才继续进行.

JDK 内部提供实现 **CountDownLatch**, 核心API:

- `await()`  
    - 等待 `CountDownLatch` 计数到 0
- `countDown()`
    - `CountDownLatch` 计数 -1

## Future and FutureTask

(p.80)

**FutureTask** 表示可完成的由结果的计算, 有以下三种状态:

- waiting to run
- running
- completed

**FutureTask** 的计算是通过 **Callable** 来实现的, 实际上 **Callable** 就是有结果返回值且可以抛出 checked exception 版本的 **Runnable**. **Callable** 中抛出的异常都会被封装到 **ExecutionException** 中, 并且在 **Future.get()** 中抛出。值的注意的是, **FutureTask** 是 **Future** 的实现, **FutureTask** 内部记录了关于这个 **Callable** 的状态, 当状态仍是进行中时, **Future.get()** 会堵塞/一直loop, 直到结果已经出现, 或者异常抛出. 

**FutureTask** 状态包括以下, 当 states > COMPLETING 则为处理完毕.

- int NEW = 0;
- int COMPLETING   = 1;
- int NORMAL       = 2;
- int EXCEPTIONAL  = 3;
- int CANCELLED    = 4;
- int INTERRUPTING = 5;
- int INTERRUPTED  = 6;

**FutureTask** 的实现方式和 Reactive Programming 中异步不太一样, **FutureTask** 不是等到 **Future.get()** 才执行运算的, **FutureTask.run()** 本身执行运算, 当某一线程使用 **Future.get()** 时, 运算可能结束也可能未结束. 而这个 **FutureTask.run()** 可能是被线程池所触发的. Reactive Programming paradigm 中, 像 Reactor 是等 subscribe 的时候才运算的, subscribe 以后结果才慢慢流到 subscription 中. 

## Semaphore 信号量

(p.82)

Semaphore 或者 Counting Semaphore 用来控制同时访问某个资源或进行某个操作的量. Semaphore 中管理着彝族虚拟的许和 (**permit**), 在执行操作前, 线程需要获得许可 (通过 **Semaphore.acquire()**). 当线程无法获得许可时 (许可都已经被分配), 该线程会被阻塞直到拿到许可, 当某一线程获得许可并且完成工作后, 该线程通过 **Semaphore.release()** 释放许可. 在 **Semaphore** 的内部记录了 `permit` 的数量, `acquire()` 则-1, `release()` 则+1, 如果在 acquire 时没有则一直loop直到有为止. 实现方式来说, **Semaphore** 与 **CountDownLatch** 很相似.

例如, 只允许同时有三个线程做 doSomthing()

```
    Semaphore sem = new Semaphore(3);

    void doSomething() {
        sem.acquire();

        // ....

        sem.release();
    }
```

## 栅栏 Barrier

(p.83)

栅栏与闭锁很像, 但闭锁是一次性的. 当闭锁进入结束状态以后就不会更改状态 (如, CountDownLatch 到 0 以后). 同时, 闭锁注重等待一系列的事件, 而栅栏要求所有线程同时到达栅栏继续执行, 也可以说栅栏用于等待其他的线程. 具体的实现方式也不是特别复杂, 基本上就是用一个 **ReentrantLock** 来作为 mutex lock 控制整个 **CyclicBarrier**, 每个线程在完成自己的任务时都会调用 **CyclicBarrier.await()**, 在该方法内, 首先, 该线程会尝试获得独占锁, 获得锁以后会对 **CyclicBarrier** 内计数减一, 并且通过计数查看是否其他线程也已经结束 (看计数是否等于0), 如果没有结束, 使用公用的 **Condition** 对象来将当前线程挂起, 这个 **Condition** 就像前面使用的 `notFull`, `notEmpty` 一样. 等最后一个线程进入 **CyclicBarrier.await()**, 计数减一后等于0, 这代表所有线程都在栅栏等待, 此时执行栅栏的工作 (如下面例子中的 `commitResult()`), 并且通过 `Condition.signalAll` 方法唤醒所有线程.

例如, 使用 **CyclicBarrier** 来确保三个线程都同时 (或者说, 等待未完成) 通过栅栏. 假如我们有一个工作可以被切分为3个部分来计算, 我们创建了三个线程各自运算一部分, 我们要求三个部分都完成的情况下才 commit 结果, 代码则会如下.

```
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    commitResult();
});

void doCalculation() {
    calculate();
    barrier.await();
}
```

# 6. 任务执行

(p.93)

使用 **Executor** 和 **ExecutorService** 将任务的提交与任务的执行分离开. Executor 基于 **Producer-Consumer** 模式, 将任务提交到 **Executor** 的行为相当于 task producer. **Executor** 内部线程执行任务相当于 consumer, 使用 **Executor** 使得无须太大困难就可以更改 consumer的执行策略.

## 执行策略

- what: 在什么线程中执行
- what: 按什么顺序执行任务
- how many: 多少任务能并发
- which / how: 如果过载需要拒绝一个任务, 应该拒绝哪个? 如何通知任务被拒绝了?
- what: 执行任务前后应作什么

## 线程池

线程池的核心就是 **Work Queue** 和 **Worker Thread**. **Work Queue** 是用来装提交到线程池的 Runnable, **Worker Thread** 就是执行任务的可重用线程. 线程池的一个核心的实现就是 **ThreadPoolExecutor**, 当要创建 **ThreadPoolExecutor** 时, 构造器会传入一个 **`BlockingQueue<Runnable>`** 代表 **Work Queue**, 每次提交任务到这个线程池时, 线程都会检查是否需要添加一个新的 **Worker**, 不管是否添加, 任务都会由 **Worker** 线程执行, 其内部也就是一个不中断的 loop, 不断的去从 **BlockingQueue** 中拿 **Runnable** 然后执行的线程.

## Executor 和 ThreadPool 的类型 (使用 Executors.new...() 创建)

- newFixedThreadPool()
    - 固定长度线程池
- newCachedThreadPool()
    - 可缓存线程池, 会回收空闲线程也会添加新线程, 不限制规模, 内部使用 SynchronousQueue, 所以任务不会真的被 enqueue, 而是阻塞直到有 worker consume 这个任务
- newSingleThreadExecutor()
    - 单线程
- newScheduledThreadPool()
    - 固定长度, 使用延迟或定时的方式执行任务 (基于相对时间)

## ExecutorService 生命周期

- void shutdown() 
    - 关闭 **ExecutorService**, 会触发已提交的任务, 但是不会等待他们完成 
- boolean isShutdown()
    - **ExecutorService** 是否已关闭
- boolean isTerminated() 
    - 是否所有任务在 **ExecutorService** 关闭后都结束了
- shutdownNow() 
    - 立刻关闭 **ExecutorService**, 将未完成的任务已 **Runnable** 的方式返回
- boolean awaitTermination(...)
    - 关闭 **ExecutorService** 之前等待一段时间

## CompletionService 和 ExecutorCompletionService

当我们尝试并发 N 个任务时, 我们可以将 N 个任务包到 **Callable** 中然后提交到 **Executor** 中, 提交完成后我们将得到 N 个 **Future**, 如果我们要想获得已完成任务的结果, 我们只能将 **Future.get()** 的 timeout 设为 0, 并且不断的轮询这 N 个 **Future**. **CompletionService** 用于解决这个问题, **ExecutorCompletionService** 是 **CompletionService** 的实现. 同时, 多个 **CompletionService** 可以共用同一个 **Executor**.

如:

```
CompletionService<?> cs = new ExecutorCompletionService<?>(executor);

// submit N callable

Future<?> completed = cs.take();
```

实际上 **ExecutorCompletionService** 的实现也非常简单, 每个提交的任务完成后都会自己的结果放入一个装结果的 **BlockingQueue** 中 (在 **FutureTask.done()** 方法中放入), 我们使用 **CompletionService** 拿结果其实就是从这个 **BlockingQueue** 中拿结果而已.


# 7. 取消与关闭

(p.111)

Java 没有提供安全终止线程的方法, 但是提供一种 **Interruption** 的协同机制.

***不要使用 Thread.stop 和 Thread.suspend 方法***

## 任务取消

**Cancellable**

- 如果外部代码能在当前操作完成之前终止该操作, 则此操作为 Cancellable
- 如果要实现任务取消, 则需要通过某种类似于 **Cancellation Requested** 的协同机制

如:

```
volatile boolean canceled;

while(!canceled){
    // ...
}
```

## 任务中断

中断任务依赖于 **Interruption** 机制, 如果在线程内的操作是 blocked 的, 我们就无法通过检查 boolean 值来取消任务, 我们只能通过在另一线程-使用线程的中断机制, interrupt 该线程. 也就是调用 **Thread.interrupt()**. 但这不保证线程会立刻中断, 当线程中任务代码遇到了 **InterruptionException**, 这代表该线程被中断了, 这是代码有两种解决方案:

1. 抛出 `InterruptedException`, 更明智, 不处理.
2. 恢复中断状态, 再次调用 **Thread.interrupt()**.

注意, 只有实现了线程中断策略的代码 (知道线程会被中断, 且有对应的相应方案) 才可以 suppress `InterruptedException`.

## 关闭 ExecutorService

(p.127)

**ExecutorService** 提供两种关闭方式:

- shutdown
- shutdownNow 

同时也可以使用 **Poison Pill** 来关闭:

- 保存一个特殊的对象作为 **Poison Pill**, 当 **ExecutorService** 接受到 **Poison Pill** 就会停止接受任务关闭 ExecutorService

## JVM 关闭

(p.135)

**正常关闭**:

- 最后一个非守护线程关闭
- System.exit()
- SIGINT 或 Ctrl+C

**强行关闭**:

- Halt
- SIGKILL

## Shutdown Hook

关闭钩子通过 **Runtime.addShutdownHook(Runnable)** 注册 callback. JVM 在关闭前会调用注册的 shutdown hook, 调用顺序不能保证, 当 JVM 要关闭时, 即使这些 hook 仍未运行完, 也会强制关闭. 关闭钩子的代码也应该是线程安全的, 且应尽快运行完毕, 为了避免并发问题, 可以在同一个钩子里完成所有操作, 这样也能保证执行的顺序.

## 守护线程 Daemon Thread

普通线程与守护线程的差异仅在于当线程退出时的操作, 当普通线程退出时, JVM 检查还有没有普通线程, 如果没有则直接退出.

## 终结器 Finalizer

***Object.finalize() 不要用***

# 8. 线程池的使用

(p.139)

## 线程饥饿死锁 Thread Starvation Deadlock

在线程池中, 如果任务依赖于其他任务, 那么可能产生死锁, 只要线程池中的任务需要无限期的等待一些其他在池中的任务提供的条件则可能会发生. 同样, 运行时间过长, 或使用了栅栏但没有足够线程数等, 也会导致

## 设置线程池的大小

要避免过大或过小的线程池

- 过大: 大量线程在 CPU 和内存上竞争
- 过少: 空闲处理器

## 计算密集型任务

- N 个 CPU, 线程池大小可为 N+1

## IO密集型任务

- N 个 CPU, 线程池大小大于 N+1, 可用具体公式观察

```
N_cpu: CPU 数量
U_cpu: 0~1 CPU 使用率
W_time: 等待时间
C_time: 计算时间

N_threads = N_cpu * U_cpu * ( 1 + W_time/C_time)
```

## 任务队列

基本任务队列有三种:

1. 无界队列: LinkedBlockingQueue
2. 有界队列: ArrayBlockingQueue
3. 同步移交队列 Synchronous Handoff: SynchronousQueue
    - 对于 SynchronousQueue, 它没有存储能力, 相反它要求同时有 producer 和 consumer 然后进行同步移交

## 饱和策略 Saturation Policy

- 中止 Abort Policy
    - 抛出异常, 直接拒绝任务
- 抛弃 Discard Policy
    - 安静的抛弃任务
- 抛弃最老 Discard Oldest Policy
    - 抛弃最老的任务
- 调用者运行 Caller Runs Policy
    - 使用调用者线程运行任务

## 动态锁顺序死锁  

当我们有锁 A 和 B, 我们可以很容易的确定统一的锁顺序避免死锁, 但如果我们特定场景时对两个业务对象加锁, 则很容易出现 **动态** 的顺序死锁.

如, 在转账时, 我们可能会:

```
synchronized (fromAcc) {
    synchronized (toAcc) {
        // doTransfer(...)
    }
}
```

但在某一笔交易中, 当前的 toAcc 可能会成为 fromAcc, 从而发生动态顺序死锁. 同样的,解决的方法是确保顺序一致, 如, 使用 `Object.hashCode()`

如, 永远都先锁 hashCode 较小的对象, 当 hashCode 相同时用第三锁

```
if (fromAccHash < toAccHash) {
    synchronized (fromAcc) {
        synchronized (toAcc) {
            // doTransfer(...)
        }
    }
} else if (fromAccHash > toAccHash) {
    synchronized (toAcc) {
        synchronized (fromAcc) {
            // doTransfer(...)
        }
    }
} else {
    synchronized (tieBreaking) {
        synchronized (fromAcc) {
            synchronized (toAcc) {
                // doTransfer(...)
            }
        }
    }
}
```

## 协作对象之间发生死锁

协作对象之间发生的死锁本质上也还是顺序死锁, 只是更为隐秘.

如, 有下面两个互相协作的类, 一个是 taxi dispatcher, 另一个代表 taxi.

```
class Taxi {

    synchronized Point getLocation();

    synchronized void setLocation(Point p){
        dispatcher.notify(this);
    }
}

class Dispatcher {

    synchronized Statistics getStat() {
        Statistics stat = new Statistics();
        for (Taxi t : taxis)
            stat.add(t.getLocation());
    }
}
```

假如某一时刻有两个线程 T1 和 T2. T1 在同一时刻调用 `Dispatcher.getStat()`, 且刚好调用了同一个 Taxi 的 `getLocation()`. 而 T2 也在同一时刻调用了同一个 Taxi 的 `setLocation(Point)`. 方法 `getStat()` 锁住了 Dispatcher, 而 `setLocation()` 也锁住了 Taxi, 使得 T1 在 `t.getLocation()` 锁死, T2 在 `dispatcher.notify(this)` 锁死.

```
T1
---> Dispatcher.getStat()

T2
---> Taxi.setLocation(Point)
```

## 开放调用 Open Call

开放调用指的是调用某方法不需要持有锁, 依赖于开放调用的类通常能表现的更好, 也更利于进行死锁分析.

## 资源死锁

资源死锁有两种:

1. 多个资源池, 线程之间以不同的顺序抢占资源池中资源
2. 线程饥饿死锁 Thread Starvation Deadlock
    - 线程互相等待对方的结果, 这往往是不同类型任务使用同一线程池的原因

## 避免死锁

1. 使用 Two-Part Strategy
    - 找出什么地方将使用多个锁
    - 对这些实例进行全局分析
2. 使用支持定时的锁

# 11. 性能可伸缩性

**Premature Optimization**:

- 避免不成熟的优化, 首先使程序正确, 再提高速度

## 减少锁的竞争

- 减少锁的持有时间
- 减少粒度过小的锁 (降低请求频率)
    - 锁分段
    - 锁分解
- 使用带有协调机制的独占锁 
    - ReadWriteLock

# 13. 显式锁

(p.227)

**Lock** API 指定了显式的, 更灵活的锁接口, 而 **ReentrantLock** 提供更为高级的加锁机制, 但也更难使用. **ReentrantLock** 不是 **synchronized** 的替代.

如:

- 使用 `Lock.tryLock()` 和轮询实现的轮询锁
- 使用 `Lock.tryLock(long, TimeUnit)` 实现的定时锁

总体来说, 非块的锁更灵活也更危险

***在大多数情况下, 非公平锁的性能高于公平锁的性能***

**读写锁 ReentrantReadWriteLock**
- 多个读操作 
- 单个写操作
- 读写不能同时进行

# 15. 原子变量与非阻塞同步机制

**CAS Compare-and-Swap**:

1. 读值 (内存) `V`
2. 比较值 `A`
3. 当比较匹配时写入值 `B`

**原子变量类**:

- AtomicInteger
- AtomicReference
- ...

***原子变量类是更好的 volatile, 支持 CAS 比使用锁更快***

# 16. Java 内存模型

(p.276)

**JMM Java Memory Model** 内存模型规定了一组保证, 这组保证规定对变量的写入操作在何时将对其他线程可见.

## 平台的内存模型

在内存共享的多处理器架构中, 每个处理器都拥有自己的缓存, 并且定期与主内存协调. 不同的架构提供不同级别的 **缓存一致性 Cache Coherence** (也就是CPU之间看到变量一致的程度).

大部分时间里, 缓存并不需要一致, 所以处理器会适当放宽一致型, 平台/架构定义的内存模型描述出如何从内存系统中获得怎样的保证, 同时还提供了一些特殊的指令用于协调 Cache Coherence, 如, **内存栅栏**. **JMM** 通过提供自己的内存模型, 从而在适当的时候由JVM使用内存栅栏来保证-不同平台之间同一语言 (JAVA) 的内存模型一致.

## 重排序 Reordering

没有正确的同步, 代码可能会重排序从而导致多线程之间没有按顺序进行操作, 同时也会导致缓存刷新到主内存的时序和时机不正确, 这也就是-导致操作对其他线程不可见. ***同步将限制编译器, 运行时和硬件对内存操作重排序的方式, 从而在实施重排序的时候不会破坏 JMM 提供的可见性保证***.

## Happens-Before

要想保证 B 操作看到 A 操作的结果 (对同一线程也适用), 那么 A 和 B 必须满足 **Happens-Before**. 反过来说, 只要确保 **Happens-Before**, 我们就可以达到我们希望的串行一致性.

**Happens-Before** 包括:

1. 程序顺序规则
    - (根据 JVM 书 这只包括控制流), 如果操作 A 在 B 之前, 那么 A 将在 B 之前执行.
2. 监视器锁规则
    - 在监视器锁上的解锁操作必须在同一个监视器锁上的加锁之前执行.
3. volatile 变量规则 
    - 对 volatile 变量的写必须在对其读之前执行
4. 线程启动规则
    - 在线程上对 `Thread.start` 的调用必须在读线程中任何操作之前
5. 线程结束规则
    - 线程任何操作都必须在其结束前完成
6. 中断规则
    - 当一个线程对另一个线程调用 `Thread.interrupt()` 时, 必须在被中断线程检测到 interrupt 之前执行, 也就是, 先有 `Thread.interrupt()` 后有 `InterruptedException`.
7. 终结器规则
    - 对象的构造函数必须在启动该对象终结器之前执行完成
8. 传递性
    - 如果 A happens before B, B happens before C, 则有 A happens before C.