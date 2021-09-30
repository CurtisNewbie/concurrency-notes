# AbstractQueuedSynchronizer & ReentrantLock

# 1. AbstractQueuedSynchronizer 类继承

```
AbstractOwnableSynchronizer
            ^
            |
AbstractQueuedSynchronizer
```

`AbstractQueuedSynchronizer` 继承了 `AbstractOwnableSynchronizer`，而 `AbstractOwnableSynchronizer` 本身只记录了哪个线程拿了 exclusive 的锁，具体代码如下：

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * The current owner of exclusive mode synchronization.
     */
    private transient Thread exclusiveOwnerThread;

}
```

# 2. AbstractQueuedSynchronizer 中的属性

`AbstractQueuedSynchronizer` 中的属性：

```java
/**
* Head of the wait queue, lazily initialized.  Except for
* initialization, it is modified only via method setHead.  Note:
* If head exists, its waitStatus is guaranteed not to be
* CANCELLED.
*/
private transient volatile Node head;

/**
* Tail of the wait queue, lazily initialized.  Modified only via
* method enq to add new wait node.
*/
private transient volatile Node tail;

/**
* The synchronization state.
*/
private volatile int state;

// inherited from AbstractOwnableSynchronizer
private transient Thread exclusiveOwnerThread;
```

核心的属性为以上四个：

- `Node head` 
    - 头节点，代表当前正在持有锁的节点
- `Node tail` 
    - 尾节点，代表阻塞队伍内最后的一个结点，新进入阻塞队伍的线程，都会作为新的节点，添加到队伍的尾部
- `int state` 
    - 同步的状态，0 代表没有被持有，大于 0 代表锁被持有，因为锁是可重入的，这个值可能会大于 1
- `Thread exclusiveOwnerThread` 
    - 记录持有锁的线程，继承于 `AbstractOwnableSynchronizer`
    - 例如我们重入了我们已经持有的锁，此时 `Thread.currentThread() == exclusiveOwnerThread`，我们就可以 `state++;`

# 3. AbstractQueuedSynchronizer.Node 中属性

```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled. */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking. */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition. */
    static final int CONDITION = -2;
    /**
    * waitStatus value to indicate the next acquireShared should
    * unconditionally propagate.
    */
    static final int PROPAGATE = -3;

    volatile int waitStatus;

    /**
    * Link to predecessor node that current node/thread relies on
    * for checking waitStatus. Assigned during enqueuing, and nulled
    * out (for sake of GC) only upon dequeuing.  Also, upon
    * cancellation of a predecessor, we short-circuit while
    * finding a non-cancelled one, which will always exist
    * because the head node is never cancelled: A node becomes
    * head only as a result of successful acquire. A
    * cancelled thread never succeeds in acquiring, and a thread only
    * cancels itself, not any other node.
    */
    volatile Node prev;

    /**
    * Link to the successor node that the current node/thread
    * unparks upon release. Assigned during enqueuing, adjusted
    * when bypassing cancelled predecessors, and nulled out (for
    * sake of GC) when dequeued.  The enq operation does not
    * assign next field of a predecessor until after attachment,
    * so seeing a null next field does not necessarily mean that
    * node is at end of queue. However, if a next field appears
    * to be null, we can scan prev's from the tail to
    * double-check.  The next field of cancelled nodes is set to
    * point to the node itself instead of null, to make life
    * easier for isOnSyncQueue.
    */
    volatile Node next;

    /**
    * The thread that enqueued this node.  Initialized on
    * construction and nulled out after use.
    */
    volatile Thread thread;

    /**
    * Link to next node waiting on condition, or the special
    * value SHARED.  Because condition queues are accessed only
    * when holding in exclusive mode, we just need a simple
    * linked queue to hold nodes while they are waiting on
    * conditions. They are then transferred to the queue to
    * re-acquire. And because conditions can only be exclusive,
    * we save a field by using special value to indicate shared
    * mode.
    */
    Node nextWaiter;
}
```

核心的属性为以上四个:

- `int waitStatus`
    - `CANCELLED = 1`
        - 代表当前节点已取消拿锁，可以跳过
    - `SIGNAL = -1`
        - 代表当前节点的 `next` 节点或 successor 节点需要被在当前节点释放锁的时候被唤醒
    - `CONDITION = -2`
        - 代表当前节点正在等待 condition
    - `PROPAGATE = -3`
        - 代表当前下一个 `acquireShared` 可以无条件的 propagate
    - `0`
        - 初始值，刚加入队列
- `Node prev`
    - 上一个节点 predecessor
- `Node next`
    - 下一个节点 successor
- `Node nextWaiter`
    - **还不知道**
- `Thread thread`
    - 该节点代表的线程, 可用于对线程唤醒 (unpark)

# 4. AbstractQueuedSynchronizer 的 CLH 锁概念

`AbstractQueuedSynchronizer` 基于 **CLH (Craig, Landin, and Hagersten) Lock Queue** 算法的变形。本质上就是，等待锁的线程使用 `LinkedList` 存储，每一个节点代表一个正在等待的线程。头节点 (`head`) 为已经获得锁的节点/线程。

当一个已经拥有锁的线程 `head` 释放了锁，它会尝试唤醒它的 `next` 或 `successor`, 而这个节点会成为新的 `head`，如果 `successor` 里有取消等待的, 那么我们可以根据节点本身的状态 `waitStatus` 来判断是否需要跳过该节点。还句话来说，CLH queue 是从头节点 dequeue， 从尾节点 enqueue。当我们尝试获取锁，但当前已经有其他线程拥有了锁，那么我们会把当前线程作为节点，enqueue 到 CLH queue 的尾部，我们可以使用 CAS 去换 `tail` 节点，这个操作是原子性的，我们并不需要为这个操作使用同步。

```
     +------+  prev +-----+       +-----+
head |      | <---- |     | <---- |     | tail
     |      | ----> |     | ----> |     |
     +------+  next +-----+       +-----+
```

# 5. AbstractQueuedSynchronizer 需要被子类实现的方法

`AbstractQueuedSynchronizer` 规定了五个需要子类实现的方法。本质上就是，由子类去制定在什么状态下当前线程拿到了锁和在什么状态下当前线程释放了锁等操作。关于 CLH 节点的插入，线程唤醒等，都由 AQS 本身去实现。

- `boolean tryAcquire(int arg)`
    - 该方法被 `AbstractQueuedSynchronizer.acquire` 方法调用，返回 `true` 则说明当前线程持有锁
- `boolean tryRelease(int arg)`
    - 该方法被 `AbstractQueuedSynchronizer.release` 方法调用, 返回 `true` 则说明当前线程释放了锁, 对于可重入锁，只有当所有重入的锁都被释放，该方法才会返回 `true`
- `int tryAcquireShared(int arg)`
    - **还不知道** 
- `boolean tryReleaseShared(int arg)`
    - **还不知道** 
- `boolean isHeldExclusively()`
    - 返回锁当前线程是否被当前线程独占，对于 mutex lock, 该方法实现方式可能为: `return getExclusiveOwnerThread() == Thread.currentThread();`

# 6. 公平锁与非公平锁 

在 `ReentrantLock` 中，公平锁和非公平锁的区别就是在于，是否会严格遵守 CLH 内等待线程的顺序，也就是所谓的先来后到。代码上的区别非常小，具体如下：

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

# 7. AQS 核心代码和 ReentrantLock.lock() 实现方式

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

## 7.1 AQS 的 acquire(int) 方法

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

大致逻辑如下：

1. 首先通过 `tryAcquire` 尝试拿锁
2. 如果没有拿到锁，我们尝试通过 `addWaiter` 方法将当前线程放入 CLH queue 中
3. 然后我们通过 `acquireQueued` 方法，检查当前节点的 predecessor / prev 节点是否是 `head` 节点，如果是 `head` 节点，这个 `head` 节点可能是 CLH queue 第一次初始化的空节点，那么这个时候并没有任何线程持有锁，当前线程可以尝试争抢锁，如果再次争抢失败，则把当前线程挂起，等待上一个节点结束唤醒当前线程
4. 当代码进入这一步，代表线程已经被唤醒了，如果在上一步发生了异常，`acquireQueued` 会返回 boolean 值为 `true`，代表我们需要使用 `Thread.interrupt()` 中断当前线程

## 7.2 ReentrantLock.FairSync 的 tryAcquire(int)

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
2. 因为这是一个公平锁的实现，我们先检查 CLH queue 有没有正在排队的线程，如果有正在排队的线程，当前线程放弃线程争抢。因为我们知道 CLH queue 中下一个节点会被唤醒
3. 如果已经有线程持有锁，我们检查是否当前线程持有锁，如果 `current == getExclusiveOwnerThread()`，我们则可以线程安全的修改 `state` 状态, 因为这是可重入锁，我们则更新 `state += 1` 并且 `return true`
4. 获取锁失败，当前线程进入 CLH queue

## 7.3 AQS 的 addWaiter(Node) 方法

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
5. 如果 `tail` 为 `null`，那么尝试初始化 CLH queue，本质上就是把 `head` 和 `tail` 都设为新节点，这里非常关键。要关注的是，我们完全可以先进入 5) 然后在下一个 loop 进入 4)

```java
private final void initializeSyncQueue() {
    Node h;
    if (HEAD.compareAndSet(this, null, (h = new Node())))
        tail = h;
}
```

## 7.4 AQS 的 acquireQueued(Node, int) 方法

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
3. 先拿当前节点的 predecessor / prev 节点，查看前一个节点是否是 `head`，如果是 `head` 那么可能这个 CLH queue 刚初始化，那么此时没有线程真的获得锁，我们可以尝试拿锁。如果拿了锁，拿当前节点就是 `head`，上一个节点会被抛弃 (所以有 `node.predecessor().next == null`)，返回 `false` 结束方法
4. 如果上一个节点不是 `head` 或者当前线程没有拿到锁，我们尝试挂起当前线程，具体代码如下。`pred` 是 predecessor / prev 节点，`node` 是当前节点。该方法本质上就是，如果前一个节点的状态 (`waitStatus`) 是 `SIGNAL`，那么代表当前节点会被唤醒，所以可以安全的挂起。如果 `ws > 0`，这代表上一个节点状态为 `CANCELLED`，跳过前面所有取消了的节点。除这些情况以外，上一节点可能为 0, `PROPAGATE` 或 `CONDITION`，尝试更新上一节点为 `SIGNAL`。如果这里更新了上一个节点状态为 `SIGNAL`，下一次进入这个方法则会 `return true`。

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

6. 这段代码用于节点取消排队，取消排队本质上就是将该节点的状态更新为 `waitStatus = Node.CANCELLED`

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary, although with
    // a possibility that a cancelled node may transiently remain
    // reachable.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        pred.compareAndSetNext(predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
                (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                pred.compareAndSetNext(predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```
