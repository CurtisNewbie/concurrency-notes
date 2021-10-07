# AbstractQueuedSynchronizer and CyclicBarrier

`CyclicBarrier` 用于同步参与线程的一致, 假如有 N 个线程并行, 每个线程在结束以后都要调用 `CyclicBarrier.await()` 方法并且挂起，当这 N 个 线程都完成并且调用了 `await()` 方法，`CyclicBarrier` 在最后一个完成的线程上触发构造 `CyclicBarrier` 时传入的 `Runnable barrierCommand`，并且重置 `CyclicBarrier` 状态。

# 1. CyclicBarrier 内部

```java
public class CyclicBarrier {

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition trip = lock.newCondition();
    private final int parties;
    private final Runnable barrierCommand;
    private Generation generation = new Generation();
    private int count;
}
```

- `ReentrantLock lock`
    - `CyclicBarrier` 内部使用 `ReentrantLock` 和 `Condition` 进行线程之间的协同和控制
- `Condition trip`
    - 用于挂起线程，放入条件队列中，并且在合适的时机唤醒所有挂起的线程 (实际上就是把挂起线程放到 AQS sync queue 中等待唤醒并且 在唤醒后拿锁继续执行)
- `int parties`
    - 参与 `CyclicBarrier` 协同的线程的数量
- `Runnable barrierCommand`
    - 在所有 parties 都完成并且调用 `await()` 方法之后，`CyclicBarrier` 需要触发的 `Runnable`
- `Generation generation`
    - 每一个 trip 都是一个 `Generation`，用于记录当前 `Generation` 是否 `broken`，如果是，则需要唤醒所有 parties 
- `int count`
    - 剩余 parties 的数量，每个 `Generation` 都会重置为 `parties`, 并且逐步降到 0

# 2. CyclicBarrier 构造 

`barrierCommand` 是选择性配置的，在所有 parties 都完成的时候被在最后一个线程上调用。`parties` 则是参与协同的线程的数量。

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    this(parties, null);
}
```

# 3. CyclicBarrier 的 await() 方法

`await()` 方法内部调用 `dowait(boolean timed, long nanos)` 方法，因为 `CyclicBarrier` 有计时版本的 `await` 方法。

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

1. 首先拿锁, 因为 `Condition` 必须在拥有锁的情况下使用。
2. 如果 `generation` 已经是 `broken` 状态，抛异常
3. 如果当前线程被中断，更新 `generation` 为 `broken` 状态，并且唤醒所有正在等待的线程。
4. 对 `count` 减一，如果 `count == 0`，则代表所有 parties 都已经完成，并且当前线程是最后一个线程，直接执行 `barrierCommand.run()`
5. 创建新的 `Generation`，重置 `count` 为初始值 (`parties`), 并且唤醒所有线程
6. 如果 `barrierCommand` 运行的时候抛出异常， 使用 `breakBarrier()` 方法唤醒所有线程，并且标记 `generation.broken = true`

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
            TimeoutException {

    // 1)
    final ReentrantLock lock = this.lock;
    lock.lock();

    try {
        // 2)
        final Generation g = generation;
        if (g.broken)
            throw new BrokenBarrierException();

        // 3)
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        // 4)
        int index = --count;
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;

                // 5)
                nextGeneration();
                return 0;
            } finally {
                // 6)
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                // 7)
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // 8)
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // 9)
                    Thread.currentThread().interrupt();
                }
            }

            // 10)
            if (g.broken)
                throw new BrokenBarrierException();

            // 11)
            if (g != generation)
                return index;

            // 12)
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        // 13)
        lock.unlock();
    }
}
```

7. 如果 `--count > 0`，代表还有线程未完成，使用 `Condition.await` 或 `Condition.awaitNanos` 将当前线程挂起 (实际上就是放入条件队列中) 
8. 如果线程在 `trip.await` 方法被挂起后发生了线程中断, 打破栅栏。注意这里的判断是 `g == generation && ! g.broken`，这代表-我们仍然在当前 `generation` 并且 `!g.broken` 代表栅栏没有被破坏，我们仍应该处于被挂起的状态，这个时候被唤醒，我们就是单纯的在 `await()` 方法中被中断了。
9. 但如果 `g != generation || g.broken`, 我们使用 `Thread.currentThread().interrupt()` 将 `is_interrupted` 标志改回正常状态, 在后续代码决定处理方式，这里有两种情况:
    1. 如果 `g != generation`，那么代表属于当前线程的 `generation` 已经完成 (最后一个线程调用了 `nextGeneration()` 方法)，我们可以继续运行 (也就是在第11步正常退出)
    2. 如果 `g.broken == true`，那么我们应该抛出 `BrokenBarrierException` (也就是在第10步抛异常退出)
10. 如果当前 `generation` 已经被破坏，抛出 `BrokenBarrierException` (回忆, `breakBarrier` 方法中会将 `generation.broken` 设为 `true`)
11. 如果 `g != generation`，这代表新的 `Generation` 已经被创建 (最后一个线程调用了 `nextGeneration()` 方法)，也就是说栅-栏已经完成，正常返回即可
12. 如果超时，则使用 `breakBarrier()` 破坏栅栏，唤醒其他线程，并且抛出 `TimeoutException`
13. 释放锁

# 4. breakBarrier() 方法

本质上该方法就是更新 `generation.broken` 为 `true`，重置 `count` 为 `parties` 初始值，然后使用 `trip.signalAll()` 唤醒所有线程。 

```java
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

# 5. nextGeneration() 方法

本质上来说，该方法唤醒所有线程，重置 `count` 为 `parties` 初始值，然后创建新的 `Generation` 对象，如果最后一个线程使用了该方法, 其他被唤醒的线程会发现 `g != generation`，这些线程就会正常退出。

```java
private void nextGeneration() {
    trip.signalAll();
    count = parties;
    generation = new Generation();
}
```

# 6. Condition.signalAll() 方法

上面实现使用 `Condition.signalAll()` 方法唤醒所有挂起线程，本质上就是把所有条件队列中节点移到 AQS 同步队列中。当我们释放当前锁的时候, 同步队列中的节点就一个一个被唤醒并且尝试拿锁继续执行。

```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}

private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}

final boolean transferForSignal(Node node) {

    if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
        return false;

    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
