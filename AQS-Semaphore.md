# AbstractQueuedSynchronizer and Semaphore

`Semaphore` 基于共享锁的概念，内部包含实现 AQS 的类。AQS 的 `state` 用于记录目前 `Semaphore` 内还有多少个共享锁，如果 `state == 0` 则代表尝试拿 `Semaphore` 共享锁的线程需要被挂起，放入 sync queue 中。

# 1. Semaphore 内部

`Semaphore` 内部非常简单，只保存了一个 AQS 的子类对象。而 AQS 的 `state` 在初始化时就是 `Semaphore` 构造时传入的 permit 总共数量。

```java
public class Semaphore implements java.io.Serializable {

    private final Sync sync;

}
```

同样的 `Semaphore` 的操作也是完全基于 `Sync` / AQS，具体逻辑看 AQS 代码。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public void release() {
    sync.releaseShared(1);
}
```

# 2. Semaphore.FairSync 和 NonfairSync 对 AQS 的继承

与 `ReentrantLock` 相似，`Semaphore` 也支持公平锁和非公平锁的实现，他们之间的不同也仍然是在于，是否有优先 sync queue 内等待线程。

非公平锁:

```java
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

公平锁:

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

这里可以看出，公平锁与非公平锁核心的区别仍然是 `hasQueuedPredecessors()` 方法。这里要回忆的是，对于共享锁，`tryAcquireShared` 方法返回 `< 0` 代表获取锁失败，`== 0` 代表成功但是没有锁了，`> 0` 代表成功并且仍然有锁。

释放共享锁:

```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

对于共享锁来说，并没有重入的概念，所以只要对 `state -= 1` 就是返回 `true`。

# 2. AQS 相关代码

释放共享锁:

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
``` 

获取共享锁:

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

# 3. 关于 `PROPAGATE` 状态存在的原因

- [更多说明1](https://www.cnblogs.com/micrari/p/6937995.html)
- [更多说明2](https://cloud.tencent.com/developer/article/1113761)

我们可以留意到在 `doReleaseShared()` 方法中，当 `head.waitStatus == 0`, 我们会尝试将该状态改为 `PROPAGATE`，具体原因如下。

1. 首先在旧版本中 `Node.PROPAGATE` 是不存在的, 当某一线程被唤醒，拿到锁且进入 `setHeadAndPropagate` 方法中时，是否调用 `doReleaseShared()` 继续唤醒后续节点主要看 `propagate > 0` 这个判断，这里要理解 `propagate` 到底代表什么。

2. 首先 `propagate` 的值就是 `tryAcquireShared` 方法的返回值，只要是 `propagate >= 0`，该线程都会进入方法 `setHeadAndPropagate`, 因为该线程成功拿到了锁。但是如果 `propagate == 0`，这代表当前线程**认为**自己拿到了最后一个锁 (这里不理解就看 `tryAcquireShared` 方法实现)，那么在旧版本代码中 (只靠 `propagate > 0` 这个判断)，这个线程并没有充足理由尝试唤醒下一个线程，毕竟目前也没有剩余的 permit。但是根据下面的例子就可看出，这样的判断并不准确，所以在后续版本加入了 `Node.PROPAGATE` 状态，使得判断 `state < 0` 也可以通过，这里就算我们出现不必要的唤醒也没有关系，因为被唤醒的线程仍然需要尝试拿共享锁。

3. 由于下面解释到的原因，不增加该状态会因为 race condition 导致有线程在 sync queue 中挂起且未被唤醒。下面例子就是 `doReleaseShared()` 方法与 `setHeadAndPropagate(...)` 方法之间的 race condition 问题。

假设我们有 `Semaphore(2)` (两个 permits), `L-1` 和 `L-2` 线程都已经拿了共享锁, 而 `W-3` 和 `W-4` 正在等待 `Semaphore` 共享锁的释放。我们假设 sync queue 中排队的节点情况如下:

```
head -> W-3 -> W-4
```

信号量释放的顺序为 `L-1` 先释放，`L-2` 后释放:

1. `L-1` 调用 releaseShared 释放共享锁，调用了 `unparkSuccessor(h)` 唤醒了 `W-3`，且 head 的等待状态从 `-1` 变为 `0`

2. `W-3` 由于被 `L-1` 唤醒，调用 `Semaphore.NonfairSync` 的 `tryAcquireShared` 尝试拿共享锁, 拿到后返回值为 `0` (因为 `L-1` 释放的 permit 被 `W-3` 拿了, 目前 `Semaphore` 中仍然没有 permit), 但此时还没有调用 `setHeadAndPropagate` 方法

3. `L-2` 调用 `releaseShared(1)` 释放共享锁，此时 `h.waitStatus` 为 0 (因为 `L-1` 将 `head` 的 `waitStatus` 改为了 `0`), 不满足条件 `h.waitStatus == Node.SIGNAL`, 因此不会调用 `unparkSuccessor(h)`, 但是会将 `head` 的 `waitStatus` 状态更新为 `Node.PROPAGATE` (如果这里没有更新, 那么 `head.waitStatus` 还是 `0`, 那么下一步就会出现 bug)

4. `W-3` 调用 `setHeadAndPropagate` 方法时，因为不满足 `propagate > 0` （`propagate = tryAcquireShared()` 也就是 `0`）, 如果没有 `Node.PROPAGATE` 状态的设置就没有 `waitStatus < 0` 判断的匹配，那么也不会调用 `doReleaseShared()` 唤醒后继节点 (也就是 `W-4`), 这个时候 `W-4` 节点就会在 sync queue 中一直挂起

Food of Thought:

- 如果在上面第2步, `W-3` 在 `L-2` 调用 `doReleaseShared` 之前调用 `setHeadAndPropagate`, 那么 `head` 就会变成 `W-3`, 而 `W-3` 不是最后一个节点, 那么 `W-3.waitStatus` 应该是 `Node.SIGNAL`。所以当 `L-2` 调用 `doReleaseShared` 方法时，该节点就会发现 `ws == Node.SIGNAL`，也就会调用 `unparkSuccessor(h)` 唤醒 `W-4` 了。
- 所以使用 `Node.PROPAGATE` 的主要原因是避免 `doReleaseShared` 和 `setHeadAndPropagate` 方法之间对 `head` 节点的竞争关系。
- 这个竞争关系，换句话来解释就是, 如果没有 `PROPAGATE` 状态: `L-2` 在 `W-3` 用 `tryAcquireShared` 拿锁之后释放了锁 (使得 `state` 实际上 > 0, 但是 `W-3` 未能感知), 但是 `L-2` 又在 `W-3` 更新 `head` 之前检查 `head.waitStatus`， 使得 `L-2` 没有尝试唤醒后续节点 (`head.waitStatus != SIGNAL`) 同时 `W-3` 也没有尝试唤醒后续节点 (`tryAcquireShared` 返回 0, 误以为没有共享锁剩下)

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {

    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 2) W-3 waked up by L-1, acquire shared lock, r == 0 because W-3 took the last one
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 4) W-3 set itself as head, and try to wake up following nodes
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

    // 4)-cont, propagate == 0 but head.waitStatus == PROPAGATE (L-2 updated it)
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;

            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 1) L-1 release shared lock, wake up W-1
                unparkSuccessor(h);
            }

            // 3) L-2 release shared lock, head.waitStatus == 0, update it to PROPAGATE
            else if (ws == 0 &&
                        !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```