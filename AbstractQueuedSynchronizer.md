# AbstractQueuedSynchronizer & ReentrantLock

# 1. AbstractQueuedSynchronizer 类继承

```
AbstractOwnableSynchronizer
            ^
            |
AbstractQueuedSynchronizer
```

`AbstractQueuedSynchronizer` 继承了 `AbstractOwnableSynchronizer`，而 `AbstractOwnableSynchronizer` 本身只记录了哪-个线程拿了 exclusive 的锁，具体代码如下：

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private transient Thread exclusiveOwnerThread;

}
```

# 2. AbstractQueuedSynchronizer 中的属性

`AbstractQueuedSynchronizer` 中的属性：

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private transient volatile Node head;
    private transient volatile Node tail;
    private volatile int state;

    // inherited from AbstractOwnableSynchronizer
    private transient Thread exclusiveOwnerThread;
}
```

核心的属性为以上四个：

- `Node head` 
    - 头节点 (没有特殊意义, 要记住 `head` 不是阻塞队列中的一员, 每一个节点都是被上一个节点唤醒的，如果是刚初始化的 sync queue, `head` 可能是一个空节点)
    - 如果 `head == null` 或者 `head == tail`, 那么可以认为当前没有线程在 sync queue 中阻塞
    - 并且 `head` 节点的 `thread` 是 `null`
- `Node tail` 
    - 尾节点，代表阻塞队伍内最后的一个结点，新进入阻塞队伍的线程，都会作为新的节点，添加到队伍的尾部
- `int state` 
    - 同步的状态，对于 AQS 来说没有特别意义, 主要是给 AQS 的子类使用
    - 对于 `ReentrantLock` 来说，0 代表没有线程持有锁，大于 0 代表锁被持有，因为锁是可重入的，这个值可能会大于 1
    - 对于 `CountDownLatch` 来说，大于 0 代表共享锁无法被持用，需要阻塞。只有当 `state == 0`，共享锁才可以被持有
    - 对于 `Semaphore` 来说，大于 0 代表还有共享锁可以获取, 而等于0 代表没有共享锁可以获取，需要阻塞
- `Thread exclusiveOwnerThread` 
    - 记录持有锁的线程，继承于 `AbstractOwnableSynchronizer`
    - 例如，在 `ReentrantLock` 中，我们重入了我们已经持有的锁, 那么此时当前线程就是 `exclusiveOwnerThread`, 我们也可以安全的 `state++;`

# 3. AbstractQueuedSynchronizer.Node 中属性

```java
static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;

    Node nextWaiter;
}
```

核心的属性为以上四个:

- `int waitStatus`
    -  注意这字段包含 successor 的状态，不全是当前节点的状态
    - `CANCELLED = 1`
        - 代表当前节点已取消拿锁，可以跳过
    - `SIGNAL = -1`
        - 代表当前节点的 `next` 或 successor 节点需要被在当前节点释放锁的时候被唤醒
    - `CONDITION = -2`
        - 代表当前节点正在等待 condition，只有在 `ConditionObject` 中维护的条件队列中使用
    - `PROPAGATE = -3`
        - 代表当前下一个 `acquireShared` 可以无条件的 propagate
    - `0`
        - 初始值，刚加入 sync queue, 新加入的节点会把它的 predecessor 更新为 `SIGNAL`, 这样 predecessor 就会在释放锁的时候唤醒当前节点
- `Node prev`
    - 上一个节点 predecessor
- `Node next`
    - 下一个节点 successor
- `Node nextWaiter`
    - 下一个条件队列中的 `waiter`，只用于 `ConditionObject` 维护的条件队列
- `Thread thread`
    - 该节点代表的线程, 可用于对线程唤醒 (unpark)

# 4. AbstractQueuedSynchronizer 的 CLH 锁概念

`AbstractQueuedSynchronizer` 基于 **CLH (Craig, Landin, and Hagersten) Lock Queue** 算法的变形。本质上就是，等待锁的线程使用 `LinkedList` 存储，每一个节点代表一个正在等待的线程。头节点 (`head`) 为已经获得锁的节点/线程。我们也叫这个队列 **Sync Queue** 同步队列。

当一个已经拥有锁的线程 `head` 释放了锁，它会尝试唤醒它的 `next` 或 `successor`, 而这个节点会成为新的 `head`，如果 `successor` 里有取消等待的, 那么我们可以根据节点本身的状态 `waitStatus` 来判断是否需要跳过该节点。换句话来说，CLH queue 是从头节点 dequeue， 从尾节点 enqueue。当我们尝试获取锁，但当前已经有其他线程拥有了锁，那么我们会把当前线程作为节点，enqueue 到 CLH queue 的尾部，我们可以使用 CAS 去换 `tail` 节点，这个操作是原子性的，我们并不需要为这个操作使用同步。

```
     +------+  prev +-----+       +-----+
head |      | <---- |     | <---- |     | tail
     |      | ----> |     | ----> |     |
     +------+       +-----+  next +-----+
```

# 5. AbstractQueuedSynchronizer 需要被子类实现的方法

`AbstractQueuedSynchronizer` 规定了五个需要子类实现的方法。本质上就是，由子类去制定在什么状态下当前线程拿到了锁和在什么状态下-当前线程释放了锁等操作。关于 CLH 节点的插入，线程唤醒等，都由 AQS 本身去实现。

- `boolean tryAcquire(int arg)`
    - 该方法被 `AbstractQueuedSynchronizer.acquire` 方法调用，返回 `true` 则说明当前线程持有锁
- `boolean tryRelease(int arg)`
    - 该方法被 `AbstractQueuedSynchronizer.release` 方法调用, 返回 `true` 则说明当前线程释放了锁, 对于可重入锁，只有当-所有重入的锁都被释放，该方法才会返回 `true`
- `int tryAcquireShared(int arg)`
    - 该方法被 `AbstractQueuedSynchronizer.acquireShared` 方法调用, 用于获取共享锁
- `boolean tryReleaseShared(int arg)`
    - 该方法被 `AbstractQueuedSynchronizer.releaseShared` 方法调用, 用于释放共享锁
- `boolean isHeldExclusively()`
    - 返回锁当前线程是否被当前线程独占，对于 mutex lock, 该方法实现方式可能为: `return getExclusiveOwnerThread() == Thread.currentThread();`

# 6. 公平锁与非公平锁 

在 `ReentrantLock` 中，公平锁和非公平锁的区别就是在于，是否会严格遵守 CLH 内等待线程的顺序，也就是所谓的先来后到。代码上的区别非常-小，具体如下：

`ReentrantLock.Sync` 中对非公平锁的实现:

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}


final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

`ReentrantLock.FairSync` 中对公平锁的实现

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

实现的主要区别是 `hasQueuedPredecessors()`，也就是我们是否优先 CLH queue 的等待线程。

# 7. 非公平锁逻辑

下面的例子解释了非公平锁运行的大致逻辑，或者说，为什么去除 `!hasQueuedPredecessors()` 方法就能达到非公平锁。假如 sync queue 中有以下节点且 `head` 线程仍在运行，未释放锁:

```
head -> t1 
```

1. `head` 对应的线程已完成，释放锁，将 `head` 此时状态从 `SIGNAL` 被更新为 0, `t1` 线程被唤醒, 但还没拿锁
2. 在 `t1` 被唤醒但还没拿锁的时候，线程 `t2` 进入尝试拿锁，并且拿到了锁
3. 此时 `t1` 拿锁失败，并且 `t1.prev` 仍然是 `head`, 调用 `shouldParkAfterFailedAcquire` 方法将 `head.waitStatus` 更新为 `SIGNAL`, 然后挂起
4. `t2` 完成操作，释放锁，并且唤醒了 `head` 的下一个节点, 而这个节点仍然是 `t1`。

如果是公平锁，那么 `t2` 一开始就不会成功，那就一定是 `t1` 拿到锁，并且 `t2` 作为 `t1` 节点的 successor，等待被唤醒。

# 8. AQS 核心代码和 ReentrantLock.lock() 实现方式

首先，当我们需要利用 AQS 实现获得锁和释放锁时，我们需要使用 AQS 的以下方法:

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

在 `ReentrantLock` 中 `arg` 是影响当前 AQS `state` 的状态的参数。对于 `ReentrantLock` 来说，在 AQS 中，`state == 0` 是没有锁，`state > 0` 是有锁，`state > 1` 是重入锁。相应的，在 `ReentrantLock.lock()` 和 `ReentrantLock.unLock()` 方法的实现中，代码调用了 `AbstractQueuedSynchronizer.acquire(int)` 和 `AbstractQueuedSynchronizer.release(int)` 方法，具体如下:

```java
public void lock() {
    sync.acquire(1);
}

public void unlock() {
    sync.release(1);
}
```

逻辑大致如下：

对于 `lock()` 方法，如果 AQS 当前没有线程持有锁，也就是 `state == 0`， 更新 `state = 1`。如果当前线程已经有锁, 则为 `state += 1`，否则，该线程由 AQS 放入 CLH queue 中，并且挂起。

对于 `unlock()` 方法，如果 AQS 当前有锁，但是当前线程不是持有锁的线程, 抛出异常。如果 `state > 1` 且当前线程持有锁，则更新 `state -= 1`。接下来看 AQS 内部的实现。

## 8.1 AQS 的 acquire(int) 方法

```java
public final void acquire(int arg) {
    // 1)
    if (!tryAcquire(arg) &&
        // 3)          2)
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 4)
        selfInterrupt();
}
```

大致逻辑如下：

1. 首先通过 `tryAcquire` 尝试拿锁
2. 如果没有拿到锁，我们尝试通过 `addWaiter` 方法将当前线程放入 CLH queue 中
3. 然后我们通过 `acquireQueued` 方法，检查当前节点的 predecessor / prev 节点是否是 `head` 节点 (`head` 节点的任务只是负责唤醒下一个节点, `head.thread` 肯定是 `null` 的, 我们可以认为 `head` 节点不包含在 sync queue 中). 如果是 `head` 节点，这个 `head` 节点可能是 sync queue 第一次初始化的空节点，那么这个时候并没有任何线程持有锁，当前线程可以尝试争抢锁，如果再次争抢失败，则把当前线程挂起, 等待上一个节点结束, 并且唤醒当前线程
4. 当代码进入这一步，代表线程已经被唤醒了，如果在上一步发生了异常，`acquireQueued` 会返回 boolean 值为 `true`，代表我们需要使用 `Thread.interrupt()` 中断当前线程

## 8.2 ReentrantLock.FairSync 的 tryAcquire(int)

该方法负责判断当前线程是否持有锁:

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();

    // 1)
    int c = getState();
    if (c == 0) {

        // 2)
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }

    // 3)
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }

    // 4)
    return false;
}
```

1. 首先我们查看 AQS `state` 的值是否为 0，如果为 0 代表没有线程持有锁。但是这个方法不是线程安全的，我们仍需要使用 CAS 来尝试争抢锁
2. 因为这是一个公平锁的实现，我们先检查 CLH queue 有没有正在排队的线程，如果有正在排队的线程，当前线程放弃线程争抢。因为我们知道 CLH queue 中下一个节点会被唤醒, 放弃争抢也代表当前线程会被插入到 sync queue 的尾部，挂起并且等待被唤醒
3. 如果已经有线程持有锁，我们检查是否当前线程持有锁，如果 `current == getExclusiveOwnerThread()`，我们则可以线程安全的修改 `state` 状态, 因为这是可重入锁，我们则更新 `state += 1` 并且 `return true`
4. 获取锁失败，当前线程进入 CLH queue

## 8.3 AQS 的 addWaiter(Node) 方法

该方法负责将当前线程作为节点放入 CLH queue，放入以后会返回代表当前线程的节点

```java
// 1)
private Node addWaiter(Node mode) {

    // 2)
    Node node = new Node(mode);

    // 3)
    for (;;) {

        // 4) 
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            // 5)
            initializeSyncQueue();
        }
    }
}
```

1. 首先要关注, `mode` 不是真的节点，`mode` 使用的以下两个引用:
    - `Node.EXCLUSIVE` 独占模式
    - `Node.SHARED` 共享模式
2. 创建新节点, 注意 `new Node(Node mode)` 构造器里使用了当前线程，具体如下:

```java
Node(Node nextWaiter) {
    this.nextWaiter = nextWaiter;
    THREAD.set(this, Thread.currentThread());
}
```
3. 要注意，这里是不停止的 for loop
4. 如果 `tail` 不为 `null`，把当前节点放到 `tail` 后面去，这里的关键就是对 `tail` 使用 CAS, 如果我们能成功把当前节点作为 `tail`，我们可以安全的做以下事情: `node.prev = tail` 和 `tail.next = node`
5. 如果 `tail` 为 `null`，那么尝试初始化 CLH queue，本质上就是把 `head` 和 `tail` 都设为新节点，这里非常关键。要关注的是，我们完全可以-先进入 5) 然后在下一个 loop 进入 4). 我们也可以理解，当 `head == tail` 时 sync queue 是空的，也可以认为这个 sync queue 可能刚被初始化。

```java
private final void initializeSyncQueue() {
    Node h;
    if (HEAD.compareAndSet(this, null, (h = new Node())))
        tail = h;
}
```

## 8.4 AQS 的 acquireQueued(Node, int) 方法

该方法负责线程挂起和线程唤醒后对锁的争抢。

```java
final boolean acquireQueued(final Node node, int arg) {
    // 1) 
    boolean interrupted = false;
    try {
        // 2)
        for (;;) {
            // 3)
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            // 4)
            if (shouldParkAfterFailedAcquire(p, node))
                // 5)
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        // 6)
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}
```

1. 该方法返回 boolean 值，代表当前线程是否需要被 interrupt
2. 注意，这里是无停止条件的 for loop
3. 先拿当前节点的 predecessor / prev 节点，查看前一个节点是否是 `head`，如果是 `head` 那么可能这个 CLH queue 刚初始化，那么此时没有线程-真的获得锁，我们可以尝试拿锁。如果拿了锁，拿当前节点就是 `head`，上一个节点会被抛弃 (所以有 `node.predecessor().next == null`)，返回 `false` 结束方法
4. 如果上一个节点不是 `head` (例如, 新插入到尾部的节点) 或者当前线程没有拿到锁 (例如，非公平锁的实现)，我们尝试挂起当前线程，`pred` 是 predecessor 节点，`node` 是当前节点。
    - 该方法本质上就是，如果 `pred` 的状态 (`waitStatus`) 是 `SIGNAL`，那么代表当前节点会被唤醒，所以可以安全的挂起。
    - 如果 `ws > 0`，这代表上一个节点状态为 `CANCELLED`，我们则需要跳过前面所有取消了的节点。
    - 除这些情况以外，我们可以确定 `pred` 要么为 0 要么为 `Node.PROPAGATE` (只用于共享锁, 所以对于 `ReentrantLock` 没有作用)，尝试更新 `pred` 为 `SIGNAL`，使得上一个节点在释放锁的时候会唤醒当前节点. 如果这里更新了上一个节点状态为 `SIGNAL`，下一次进入这个方法则会 `return true`，因为这一次 `ws == Node.SIGNAL`。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```

5. 进入这里的代码，代表当前线程会被挂起。要注意当返回当前方法时，这代表当前线程已经再次被唤醒了。

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

6. 这段代码用于节点取消排队，取消排队本质上就是将该节点从 CLH queue 中去除
    1. 我们首先跳过当前节点前面 `waitStatus == CANCELLED` 的节点，找到不为 `CANCELLED` 的 predecessor
    2. 更新当前节点 `waitStatus` 为 `CANCELLED`
    3. 对于我们当前节点是 `tail` 的情况，我们尝试去除当前节点，做法就是 CAS 使用 `pred` 替换 `tail`，如果替换成功，那么我们更新 `pred.next` 为 `null`，因为 `node` 和 `pred` 中间要么没节点，要么都是 `CANCELLED`。这样，我们也去除了中间取消的节点
    4. 如果当前节点既不是 `tail` 同时 predecessor 也不是 `head`，那就跳过当前节点，也就是 `pred.next = node.next`
    5. 如果当前节点不是 `tail` 但 predecessor 是 `head`，我们唤醒 `node.next` 或后续应该被唤醒的第一个节点
        - 要记住，这个时候 `node.waitStatus = Node.CANCELLED`，当 successor 节点被唤醒，successor 线程是在 `parkAndCheckInterrupt()` 方法被唤醒的，此时它的 predecessor 仍然是我们这里说的 `node`，`node` 应该出列但还没出，并且 `node.prev` 才是 `head`。
        - 那么唤醒以后第一个 for loop 并不会尝试拿锁 (因为 `p != head`)。也就是说，它会调用方法 `shouldParkAfterFailedAcquire`, 在该方法内, 该节点前面为 `CANCELLED` 的节点都被跳过，这个时候我们说的 `node` 被清出队列了。具体 `unparkSuccessor` 方法代码看后面的解释

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // 1)
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    // 2)
    node.waitStatus = Node.CANCELLED;

    // 3)
    if (node == tail && compareAndSetTail(node, pred)) {
        pred.compareAndSetNext(predNext, null);
    } else {
        // 4)
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                pred.compareAndSetNext(predNext, next);
        } else {
            // 5)
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

# 9. AQS 核心代码和 ReentrantLock.unlock() 实现方式

```java
public void unlock() {
    sync.release(1);
}
```

# 9.1 AQS 的 release(int) 方法

```java
public final boolean release(int arg) {
    // 1)
    if (tryRelease(arg)) {
        // 2) 
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    // 3)
    return false;
}
```

大致逻辑如下:

1. 先尝试使用方法 `tryRelease` 释放锁
2. 如果释放锁成功，我们尝试唤醒 `head` 的下一个节点，也就是 successor，返回 `true` 代表我们释放锁成功
3. 释放锁失败

## 9.2 ReentrantLock.Sync 的 tryRelease(int) 方法  

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;

    // 1)
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();

    // 2)
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }

    // 3)
    setState(c);
    return free;
}
```

释放锁的逻辑相对简单很多:

1. 检查当前线程是否持有锁，不持有锁的情况需要抛出异常
2. 检查 `state - releases` 是否等于 0, 这要考虑锁重入的情况， 如果 `c > 0` 那么代表我们还没释放所有重入的锁，`free` 仍然为 `false`
3. 更新 `state` 状态，`state == 0` 代表无锁，`state > 0` 代表有锁，`state > 1` 代表存在锁的重入

## 9.3 AbstractQueuedSynchronizer 的 unparkSuccessor(Node) 方法

该方法负责唤醒当前节点的 successor

```java
private void unparkSuccessor(Node node) {
    // 1)
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);

    // 2)
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    // 3)
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

大致逻辑如下:

1. 如果当前节点 `waitStatus < 0`，我们尝试清除该状态，将 `waitStatus` 改为 0，毕竟我们需要唤醒的是下一个节点
2. 从 `tail` 开始往前走，找到第一个非 `CANCELLED` 的 successor 节点，我们要做的就是唤醒这个 successor 节点
3. 唤醒找到的 successor 节点，这个节点代表的线程就在节点的 `thread` 字段里

```java
public static void unpark(Thread thread) {
    if (thread != null)
        U.unpark(thread);
}
```








