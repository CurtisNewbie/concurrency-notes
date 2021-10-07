# AbstractQueuedSynchronizer.ConditionObject and Condition

# 1. ConditionObject 属性

```java
public class ConditionObject implements Condition, java.io.Serializable {

    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;

}
```

`ConditionObject` 作为 `Condition` 的实现，只带有两个属性:

- `Node firstWaiter`
    - 条件队列中的第一个节点
- `Node lastWaiter`
    - 条件队列中的最后一个节点

要关注的是 `ConditionObject` 是 `private class` 而不是 `private static class`，代表这个类的对象是与外层对象相关联的，也就是 `AbstractQueuedSynchronizer`。条件队列本质上就是一个单向队列，用于保存被 `Condition` 挂起的线程，这与 AQS 非常相似。

# 2. Condition 的核心方法

以下是 `Condition` 中最常用的方法，这里也主要讲这两个方法。

```java
public interface Condition {

    void await() throws InterruptedException;

    void signal();

}
```

前提要了解的是，在我们使用 `Condition` 对象之前，我们必须要获得锁，在我们使用 `Condition` 挂起线程等待条件的时候，我们也必须释-放锁，这一点非常关键。这也表示，我们在 `Condition` 方法内，部分代码是线程安全的，在我们释放锁之前，我们不需要担心这个问题。

# 3. Condition.await() 方法

这个方法挂起当前线程，并且将当前线程作为节点放入单链条件队列中。

```java
public final void await() throws InterruptedException {
    // 1)
    if (Thread.interrupted())
        throw new InterruptedException();

    // 2)
    Node node = addConditionWaiter();

    // 3)
    int savedState = fullyRelease(node);

    // 4)
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        // 5)
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }

    // 6)
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;

    // 7)
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();

    // 8)
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

1. 因为这个方法是可中断的，所以我们对 `Thread.isInterrupted` 做判断，如果是被中断，我们抛 `InterruptedException`
2. 这一步，我们创建节点，将当前线程放入条件队列 (condition queue) 中，记住，这里我们仍然持有锁，所以 `addConditionWaiter()` 完全是线程安全的
3. 这一步，我们释放锁，返回的是释放锁之前的 `state` 值
4. 我们查看该结点是否被从条件队列中转移到同步队列中 (sync queue/ CLH queue), 只要还没有，那就继续挂起
5. 在这一步，线程被唤醒了，我们检查线程是否被中断，并且尝试将当前节点的状态从 `CONDITION` 改到 0 (也就是新加入 CLH 队列的初始状态)，并且把该节点放到 CLH 队列中 (这样它就能排队拿锁了)。如果我们成功将节点从条件队列放到了 CLH 队列中，我们退出这个 while loop
6. 这个位置，当前节点已经被唤醒，并且被放到了 CLH 队列中，我们尝试进行 CLH 队列中的锁争抢和线程挂起
7. 再次清除已取消的节点
8. 根据情况抛中断异常或调用 `Thread.currentThread().interrupt()`

## 3.1 AQS 的 addConditionWaiter() 方法 

```java
private Node addConditionWaiter() {
    // 1)
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();

    // 2)
    Node t = lastWaiter;
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }

    // 3)
    Node node = new Node(Node.CONDITION);

    // 4)
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;

    // 5)
    return node;
}
```

这一步，我们创建节点，将当前线程放入条件队列中，记住，这里我们仍然持有锁，所以 `addConditionWaiter()` 完全是线程安全的。

1. `isHeldExclusively` 方法判断当前线程是否持有锁，因为这个方法只有当线程持有锁的情况才可以进入，不持有锁就抛异常
2. 首先拿到 `lastWaiter`，也就是条件队列中的尾节点，如果 `lastWaiter` 不是 `CONDITION` 状态 (这是新加入条件队列节点的初始状态), 那么这代表 `lastWaiter` 已经是 `CANCELLED` 状态。那么此时我们调用 `unlinkCancelledWaiters()` 方法，从 `firstWaiter` 开始去除 `CANCELLED` 状态的节点，该方法会遍历整个条件队列

```java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

3. 为当前线程创建节点, 使用的 `waitStatus` 是 `CONDITION`
4. 这里 `t` 应该是 `lastWaiter`， 如果 `t == null`，那就代表条件队列中没有 waiter，直接把 `node` 放到 `firstWaiter` 即可，否则放到 `t` 的后面。不管何种情况, `node` 都是最后一个.
5. 返回放入的节点，也就是当前线程的节点

## 3.2 AQS 的 fullyRelease(Node node) 方法

这个方法负责将使用 `Condition.await()` 的线程完全释放锁，要记住这个锁时可以重入的，也就是说, 如果当前线程重入了两次，`state = 2`，那么我们释放锁，被唤醒，再次拿锁的时候，我们必须要保证 `state` 也为 2，否则我们释放重入锁的时候，`state` 就会不正确的降低到了 0 以下。

```java
final int fullyRelease(Node node) {
    try {
        // 1)
        int savedState = getState();

        // 2)
        if (release(savedState))
            return savedState;

        // 3)
        throw new IllegalMonitorStateException();
    } catch (Throwable t) {
        node.waitStatus = Node.CANCELLED;
        throw t;
    }
}
```

1. 首先我们现在仍然拿着锁，我们取当前 `state` 的值，以便于我们线程唤醒时，我们也拿相同数量的锁
2. 我们尝试释放锁，这个时候我们必须要成功，如果失败，代表我们根本没有拿到锁，那么我们也不应该走到这个位置
3. 再次强调，我们必须带着锁来到这个方法进行锁的释放, 所以我们也必须成功的释放锁，否则抛异常

## 3.3 AQS 的 isOnSyncQueue(Node) 方法

这个方法主要判断当前节点是否在同步队列 / CLH 队列中，而不是条件队列中。如果当前节点在 CLH 队列中，那么我们就可以尝试拿锁了。

```java
final boolean isOnSyncQueue(Node node) {
    // 1)
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;

    // 2)
    if (node.next != null) // If has successor, it must be on queue
        return true;

    // 3)
    return findNodeFromTail(node);
}

private boolean findNodeFromTail(Node node) {
    for (Node p = tail;;) {
        if (p == node)
            return true;
        if (p == null)
            return false;
        p = p.prev;
    }
}
```

1. 如果当前节点的 `waitStatus` 仍然是 `CONDITION`，这代表该节点仍然在条件队列中，因为在 CLH 队列中的节点默认都是 0。如果 `node.prev == null`，这也代表该结点不在队列中，因为即使新节点放入新 CLH 队列，也会初始化一个空节点
2. 如果当前 `node.next != null` 这保证该节点已经在队列中了，记住 `node.prev != null` 不代表当前节点已经成功放入队列，因为放入队列的 CAS 可能失败，而 `node.prev = pred` 是在 CAS 尝试之前就做了的 (不理解这里就看 AQS 的 `addWaiter` 方法)，也就是说，我们很可能在 CAS 失败的同时也设置了 `node.prev = pred`，导致 `node.prev != null`。
3. 从 CLH 队列尾节点往前走，确保当前节点的确在 CLH 队列内。

只要这个 `isOnSyncQueue(Node)` 方法返回 `true`，该节点就肯定在 CLH 队列内，我们就调用 `acquireQueued` 方法，尝试拿锁或者被挂起，等待-被唤醒。但是我们可以发现，在 `Condition.await()` 方法中，根本没有将节点从条件队列放到同步队列中的操作，当我们调用 `Condition.await()` 方法，当前线程一定在条件-队列中，并且被挂起，直到某一刻，我们把这个节点放到了 CLH 队列中，并且唤醒了它。只有这个时候它才会去争抢锁，或者-继续被挂起等待上一节点唤醒他。这个操作就是在 `Condition.signal()` 中发生的。 

## 4. Condition 的 signal() 方法 

该方法用于唤醒某一个被同一个 `Condition` 对象挂起的线程, 注意, 只有拥有锁的线程可以使用 `signal` 方法。记住，`signal` 方法并不会真的直接唤醒线程 (除特殊情况外), 只是把节点放到 CLH 队列中，由它在 CLH 队列中的 predecessor 唤醒，唤醒的线程是从 `Condition.await` 方法的第5步唤醒的，被唤醒的节点要做的事情就是调用 `acquireQueued` 方法，跟其他在 CLH 队列内的节点一样，排队拿锁, 拿不到就挂起等上一个节点唤醒。换句话来说，从 `Condition.await` 方法中被唤醒的线程，肯定拿着锁。

```java
public final void signal() {
    // 1)
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 2)
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

1. 首先检查该线程是否拥有锁，只有拥有锁的线程可以使用 `signal` 方法
2. 从条件队列中拿第一个 `waiter`，这里要关注，这个队列是条件队列，而不是 CLH 队列。然后尝试将这个 `waiter` 线程放到 CLH 队列中。

## 4.1 ConditionObject 的 doSignal(Node) 方法

```java
private void doSignal(Node first) {
    do {
        // 1)
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // 2)
        first.nextWaiter = null;
        // 3)
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}
```

1. 首先，我们检查第二个节点是否存在，也就是 `first.nextWaiter` 是否为 `null`，如果为 `null`，这代表条件队列为空，我们可以把 `lastWaiter` 也设为 `null`
2. 如果第二个节点存在，我们把第二个节点设为 `firstWaiter`，跳过当前节点，因为我们将要把节点 `first` 移到 CLH 队列中
3. 更新 `first` 节点的 `waitStatus` 为 0 (CLH 队列中新加入节点的默认 `waitStatus` 值)，并且尝试把 `first` 节点放到 
CLH 队列的尾部。如果转移失败 (因为使用 CAS 更新 `first.waitStatus` 从 `CONDITION` 到 0 时失败，也可以认为这个节点为 `CANCELLED` 所以我们可以条)，我们尝试下一个节点进行转移

## 4.2 ConditionObject 的 transferForSignal(Node) 方法

```java
final boolean transferForSignal(Node node) {

    // 1)
    if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
        return false;

    // 2)
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

1. 尝试使用 CAS 将 `node` 的节点状态从 `CONDITION` 更新为 0，如果失败，则我们认为该节点转移失败。如果这里失败，我们可以认为该节点不再是 `CONDITION`，或者说它已经取消排队了，状态为 `CANCELLED`
2. 如果状态更新成功，我们尝试把节点放入 CLH 中，本质上该方法与 AQS 的 `addWaiter` 方法并没有区别，新节点是放在 CLH 队列的尾部。注意，返回的节点是插入节点的 predecessor, 我们检查该 predecessor `waitStatus`，如果 predecessor 是 `cancelled` 状态，或者我们无法将该状态设置为 `signal`，我们唤醒该 `node` 节点

```java
private Node enq(Node node) {
    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return oldTail;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```

















