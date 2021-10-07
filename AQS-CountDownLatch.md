# AbstractQueuedSynchronizer and CountDownLatch

# 1. CountDownLatch 与 AQS 关系 

`CountDownLatch` 本质上就是将 W 线程阻塞, 直到 N 线程都完成，所以也被称为栅栏。对于被阻塞的 W 个线程，我们可以认为他们作为一个-整体，在尝试争抢同一把锁 L。对于 N 个正在运行的线程来说, 这 N 个线程作为一个整体, 在运行的时候就已经获得了 N 次重入锁, 而这个锁 L 只有当这 N 个线程都将他们手中的 N 个锁释放时, 这被阻塞的 W 个线程才能继续进行。可以看构造器中, AQS 的 `state` 初始化值就是 `count`。

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

我们可以把这些操作与 `CountDownLatch` 中方法进行匹配:

- 对于这 W 个线程，他们都调用了 `CountDownLatch.await()` 方法，也可以看出，这本质就是使用 AQS 获取共享锁. 
- 对于这 N 个线程，他们都调用了 `CountDownLatch.countDown()` 方法，也可以看出，这本质就是释放 N 个锁.

```java
public class CountDownLatch {

    public void countDown() {
        sync.releaseShared(1);
    }

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

}
```

因为实现基于 AQS, 有几个方法需要被实现:

```java
private static final class Sync extends AbstractQueuedSynchronizer {

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c - 1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

首先我们可以看 `tryAcquireShared(int)`, 这方法对应的是正在等共享锁的 W 个线程, 判断当前线程能否拿共享锁

- 该方法返回:
    - 大于0, 成功, 后续线程还**有机会**拿共享锁 
        - 可以思考 `Semaphore` 在这个 permit 被拿走后还有 permit, 返回的就是 remaining
        - 对于 `CountDownLatch`, 只要 `state == 0`, 我们认为 N 个线程已经完成, 共享锁无须等待, 所以会直接返回 `1`
    - 等于0, 成功, 但后续的共享锁获取不会成功 
        - 可以思考 `Semaphore` 最后一个 permit 被获取, 这种情况就会返回 0
    - 小于0, 失败
        - 对于 `CountDownLatch`, 只要 `state != 0`, 我们认为共享锁还要等待, 所以会直接返回 `-1`
- 逻辑基本上是：
    - 如果 AQS 的 `state` == 0, 代表 N 个线程都已经释放了他们的锁，我们可以准备争抢锁了
    - 如果 AQS 的 `state` != 0, 代表 N 个线程中仍有线程持有锁, 还没有结束, 则当前线程会被放入 sync queue 中，等待被唤醒

然后我们可以看 `tryReleaseShared(int)`，这方法对应的是正在运行的 N 个线程, 每一个完成的线程都会尝试释放手中的共享锁

- 该方法返回:
    - `true`, 共享锁完整释放, 可以唤醒 sync queue 中被挂起的线程 
    - `false`, 共享锁未能完整释放, 啥都不做
- 注意 N 个线程一共有 N 个锁，所以我们可以看到代码中使用 CAS 尝试 decrement `state`
- 当 N 个线程中的某一个线程释放锁后 `state == 0`，那么我们知道可以开始逐个唤醒 W 个线程了，而这个线程就负责这个事，所以这里-也是返回 `true`, 对于前面 N-1 线程在释放锁以后并不会阻塞, 直接在释放后返回 `nextc != 0` 也就是 `false`.

# 2. CountDownLatch.await() 和 AQS 的 acquireSharedInterruptibly(int) 方法

`CountDownLatch` 内部继承了 AQS, 并且 `CountDownLatch.await()` 方法调用了 AQS 的 `acquireSharedInterruptibly` 方法. 

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

这里的核心在于 `tryAcquireShared(int)` 方法, 我们可以回忆, 当 `state != 0` 时 `tryAcquireShared` 返回 -1, 否则返回 1. 也就是说, 只要 `state != 0` 也就是 N 个线程中仍有线程未完成, 对于这 W 个线程来说都会进入 `doAcquireSharedInterruptibly` 方法, 那么这个方法肯定是负责将这 W 个线程插入 sync queue 中，挂起并且等待被唤醒。

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

## 2.1. AQS 的 doAcquireSharedInterruptibly(int) 方法, W 线程挂起和进入队列

我们假设这 N 个线程没有全部完成, 或者说这 W 个线程应该阻塞. 

1. 我们在以下方法可以看到 `addWaiter(Node.SHARED)` 方法的调用, 当前线程作为节点插入到了 sync queue 的尾部. 
2. 我们首先检查该节点的 predecessor 是否为 `head`, 如果是 predecessor 为 `head`, 那么当前节点是 sync queue 中阻塞队列部分的第一个节点, 要考虑我们进入到这个位置完全可能因为我们被 `head` 唤醒了. 我们尝试调用 `tryAcquireShared(int)` 方法, 判断我们是否可以拿锁. 这里 `r < 0` 为失败, 也就是说 `state > 0`.
3. 如果当前线程未拿到锁，我们要检查是否要再次把当前锁挂起，我们调用 `shouldParkAfterFailedAcquire` 方法, 判断我们是否需要挂起, 挂起的原因是 predecessor 节点状态为 `SIGNAL`。如果需要挂起，我们通过 `parkAndCheckInterrupt()` 方法挂起当前线程. 


```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 1)
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            // 2)
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            // 3)
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}

private Node addWaiter(Node mode) {
    Node node = new Node(mode);

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}

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

        // 0 or PROPAGATE
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```

关键的点在于, 如果当这 N 个线程都完成 (`state == 0`)，sync queue 中第一个节点 (也就是 `head`) 的 successor 会被唤醒, 而唤醒的位置就在 `parkAndCheckInterrupt()` 的位置。被唤醒后, 该节点检查 predecessor 是否为 `head`, 如果是 `true`, 那么该节点调用 `tryAcquireShared(int)`。

因为这个时候 `state == 0`, 那么 `tryAcquireShared(int)` 方法返回 1。这个节点就通过 `setHeadAndPropagate` 方法来尝试将自-己设置为新的 `head`, 并且唤醒后面的节点。这里可以关注只有当 `propagate > 0` (`tryAcquireShared` 方法返回的值) 或者 `h.waitStatus < 0` (旧的 `head` 的 `waitStatus` 为 `0` 或者 `PROPAGATE`) 并且只有当下一个节点也是 `SHARED` 类型，我们才应该 propagate 这个唤醒的操作.

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

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
        // 1)
        if (h != null && h != tail) {
            int ws = h.waitStatus;

            // 2)
            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }

            // 3)
            else if (ws == 0 &&
                        !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

接下来在 `doReleaseShared()` 方法中，这个方法的本质就是释放等待共享锁的线程。对于 `CountDownLatch.countDown()` 方法, 也是调用这个方法进行线程唤醒。最后一个 `CountDownLatch.countDown()` 会第一次调用这个方法，然后后续被唤醒的节点会继续调用这个方法, 继续唤醒下一个节点。

1. 我们首先判断 `h != null` 并且 `h != tail`, 因为这两种情况都表示队列要么为空, 要么就只有一个头节点 (每个节点都是上一个节点唤醒的, 如果只有一个头节点, 代表没有需要唤醒的节点了)。

2. 然后我们判断 `h.waitStatus == Node.SIGNAL`, 如果是 `true`，则代表我们需要唤醒 `h.next`, 我们尝试将 `h` 节点的状态从 `SIGNAL` 更新为 `0`, 如果更新成功就代表我们拿到了唤醒 `h` 下一个节点的机会, 我们调用 `unparkSuccessor(h)` 方法唤醒 `h` 后面的节点。要记住, 我们已经在上面 `setHeadAndPropagate` 更新了 `head` 节点了, 这里的 `head` 已经更新了，不要搞混. 核心就是我们在尝试唤醒下一个节点。

3. 如果 `head.waitStatus == 0`, 我们则尝试将该节点的状态更新为 `PROPAGATE` (-3), 用于传递下一个节点的唤醒。
    - 这里的做法主要是为了前面 `setHeadAndPropagate` 方法的调用, 因为 `propagate` 实际上是 `tryAcquireShared(arg)` 返回的值, 在 `Semaphore` 实现中, 这个方法是会有返回 0 的情况 (当 `Semaphore` 中没有 permit 了)，并且由于这一系列代码都-没有使用锁而只是使用 CAS, 我们是有可能遇到线程 racing condition 导致虽然 `propagate` 值是 0, 但实际上我们应该唤醒下一个-节点。所以这里我们更新该节点为 `PROPAGATE` 状态,使得即使 `propagate == 0` 我们仍能满足 `h.waitStatus < 0`.

## 2.2 关于 `PROPAGATE` 状态的用法

首先在旧版本中 `Node.PROPAGATE` 是不存在的, 是否调用 `doReleaseShared()` 唤醒后续节点主要看 `propagate > 0` 这个判断。但-是由于下面解释到的原因，不增加该状态会因为 race condition 导致有线程在 sync queue 中挂起且未被唤醒。下面例子就是 `doReleaseShared()` 方法与 `setHeadAndPropagate(...)` 方法之间的 race condition 问题。

- [更多说明1](https://www.cnblogs.com/micrari/p/6937995.html)
- [更多说明2](https://cloud.tencent.com/developer/article/1113761)

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

# 3. CountDownLatch.countDown() 和 AQS 的 releaseShared(int) 方法

在上面我们看到 `CountDownLatch` 内部继承了 AQS, 并且 `CountDownLatch.countDown()` 方法调用了 AQS 的 `releaseShared(int)` 方法。我们进入 `releaseShared(int)` 方法后，我们可以看到 `CountDownLatch.Sync` 实现的 `tryReleaseShared(int)` 方法. 回忆 `tryReleaseShared(int)` 方法, 只要 `state != 0` 这个方法都只返回 `false`, 也就是说这个方法中 `doReleaseShared()` 的方法调用只有在全部 N 个线程都完成并且调用 `countDown()` 的情况下才会被调用，具体逻辑看前面解释。

```java
public void countDown() { 
    sync.releaseShared(1);
}
```
