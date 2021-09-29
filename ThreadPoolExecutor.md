# ThreadPoolExecutor

## 继承关系

首先了解 `ThreadPoolExecutor` 与它继承的类的关系。 

```
       Executor
          ^
          |
    ExecutorService
          ^
          |
AbstractExecutorService
          ^
          |
  ThreadPoolExecutor
```

`Executor` 接口只定义了 `execute` 方法，且只关注 `Runnable` 的使用场景。`ThreadPoolExecutor` 对此接口提供了实现。


```java
public interface Executor {

    void execute(Runnable command);
}
```

`ExecutorService` 在 `Executor` 的基础上增加了关于生命周期的方法, 并且增加了对 `Callable` 和 `Future` 的使用场景。但是实际上，在 `AbstractExecutorService` 的实现中，`submit` 方法都是基于子类 `Executor.execute` 方法的实现的，本质上就是把 `Callable` 封装成一个实现了 `Runnable` 的类，并且创建对应 `Future`。然后在将 `Runnable` 提交到线程池的同时直接返回创建的 `Future`。

```java
public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

## ThreadPoolExecutor 参数

`ThreadPoolExecutor` 包含以下参数：

- `int corePoolSize`
    - 线程池中**线程**的数量
- `int maximumPoolSize`
    - 线程池中最大的**线程**数量
- `long keepAliveTime`
    - 当线程池中线程数量大于 `corePoolSize`， 这是 idle 的线程在被删除前可存活的最长时间
- `TimeUnit unit`
    - `keepAliveTime` 时间单位
- `BlockingQueue<Runnable> workQueue`
    - work queue 
- `ThreadFactory threadFactory`
    - 用于创建线程的 factory
- `RejectedExecutionHandler handler`
    - 对被拒绝的任务进行处理的 handler, 用于当 `maximumPoolSize` 已经到达并且 `BlockingQueue` 已经满了的场景

## ThreadPoolExecutor 内的 Worker 和 Executor.execute(Runnable)

本质意义上，线程池就是包含可复用的线程，`ThreadPoolExecutor` 中编写了内部类 `Worker` 用于代表这些可复用的线程。首先，`Worker` 实现了 `Runnable` 接口，只要 `Worker` 内无限循环或者阻塞，那么这个线程就是可被复用的。我们首先看如何创建 `Worker`。

以下为 `Executor.execute(Runnable)` 的实现代码：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();

    // 1)
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 2)
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 3)
    else if (!addWorker(command, false))
        reject(command);
}
```

1. 当调用 `Executor.execute(Runnable)` 方法时，`ThreadPoolExecutor` 首先会检查我们当前线程的数量是否超过 `corePoolSize`， 如果没有，我们直接创建新线程 `Worker`，而传入的 `Runnable` 则作为这个线程第一个被执行的任务 `firstTask`。注意 `addWorker` 方法返回 `boolean` 值，代表该操作是否成功，因为这里并没有使用锁，所以是可能失败的。例如，尝试创建 `Worker` 线程的时候发现超过 `maximumPoolSize` 时。
2. 如果创建新线程 `Worker` 失败，我们确保线程池仍然运行，然后我们将 `Runnable` 放入 `BlockingQueue<Runnable>` 里，等待其他 `Worker` 进行拉取。
3. 如果我们没有办法把 `Runnable` 放入 `BlockingQueue<Runnable>` 中，那么代表线程池正在关闭，或者，我们的 `BlockingQueue` 已经满了，我们拒绝这个任务。

## addWorker(Runnable, boolean) 方法和 Worker.run() 方法

除去一些对线程池状态和大小的判断，`addWorker` 方法核心的代码如下。本质上就是 `new Worker(Runnable).thread.start()`。因为 `Worker` 的构造器中自行使用了 `ThreadFactory` 创建了线程，所以我们并没有通过 `new Thread(new Worker).start()` 的方式运行。

```java
private boolean addWorker(Runnable firstTask, boolean core) {

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();

                if (isRunning(c) ||
                    (runStateLessThan(c, STOP) && firstTask == null)) {
                    if (t.getState() != Thread.State.NEW)
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    workerAdded = true;
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
}
```

下面的代码就是关于 `Worker` 构造器中，构建线程的方式。注意, `this` 指向当前 `Worker`，而 `Worker` 本身就是一个 `Runnable`。

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

那么当我们运行 `new Worker(firstTask).thread.start()` 方法的时候，我们实际上执行的是 `worker.run()`, 而这个方法的实现实际上是 delegate 到 `ThreadPoolExecutor` 的 `runWorker(Worker w)` 方法中的。

以下是 `Worker.run()`

```java
public void run() {
    runWorker(this);
}
```

以下是 `ThreadPoolExecutor.runWorker(Worker w)`

重点看 `1)` 的代码，首先，从 `Worker` 中拿内部的 `firstTask`， 如果这个 `Worker` 是新创建的，那么这个 `firstTask` 就不为 `null`。然后，在运行了第一个 `firstTask` 之后，这个方法会尝试通过 `task = getTask()` 去拿下一个 `task`，

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {

            // 1) 
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        task.run();
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

这个 `ThreadPoolExecutor.getTask()` 方法的核心是从 `BlockingQueue<Runnable> workQueue` 里拿任务。

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();

        // Check if queue empty only if necessary.
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

## ThreadPoolExecutor.reject(Runnable) 方法

这个方法的实现非常直接，核心就是选择 `RejectedExecutionHandler`，然后使用这个 handler 去处理这个 `Runnable`.

```java
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

这个 `RejectedExecutionHandler` 有以下几种处理方式:

- `AbortPolicy`
    - 直接抛 `RejectedExecutionException`

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    throw new RejectedExecutionException("Task " + r.toString() +
                                            " rejected from " +
                                            e.toString());
}
```

- `CallerRunsPolicy`
    - 直接使用 caller 的线程运行

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        r.run();
    }
}
```

- `DiscardOldestPolicy`
    - 删除 `workQueue` 中最旧的 `Runnable`，然后重试

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```

- `DiscardPolicy`
    - 直接安静的忽略掉

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
}
```

到这里，`ThreadPoolExecutor`, `Executor` 和 `Worker` 的核心代码已经简单过了一遍。

## ExecutorService, Future, 和 Callable

回顾 `ExecutorService` 在 `Executor` 的基础上，增加了对 `Future`, `Callable` 的支持。这些支持都来源于 `AbstractExecutorService`, 而最终对提交的任务的执行，仍然是由 `ThreadPoolExecutor.execute(Runnable)` 负责完成的。

例如，我们可以看 `AbstractExecutorService.submit(Callable)` 方法。

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

对这个任务的执行还是由子类 `ThreadPoolExecutor` 的 `execute(Runnable)` 方法执行。关键在于我们如何将 `Callable` 封装成 `Runnable` 并且能将结果传递到 `Future` 中。

首先看 `RunnableFuture<T>`, 本质上就是 `Runnable` + `Future<V>` 而已。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {

    void run();
}
```

而 `newTaskFor(Callable)` 方法则是将 `Runnable` 封装成 `RunnableFuture` 的关键。核心就是使用 `FutureTask`。

```java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

## FutureTask

`FutureTask` 本质上就是 `Runnable` 和 `Callable` 上包的一层新的 `Runnable`。它通过封装传入的任务，在任务结束的时候，将状态进行管理，同时保存好任务返回的结果。

我们首先看 `FutureTask.run()` 方法的实现。这段方法非常直接，我们记录这个 `FutureTask` 的状态，如果要尝试运行该 `FutureTask`, 它必须为 `NEW` 状态，然后我们直接调用内部的 `Callable.call()` 方法，并且保存好返回的结果。如果发生了异常，我们则保存好抛出的异常。

```java
public void run() {
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

接下来看 `FutureTask` 如何保存结果和异常。我们在两个方法中都会更新状态。不管是保存结果还是状态，我们都保存到变量 `outcome` 中。

```java
protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = v;
        STATE.setRelease(this, NORMAL); // final state
        finishCompletion();
    }
}

protected void setException(Throwable t) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = t;
        STATE.setRelease(this, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
```

然后当调用 `Future.get()` 方法的时候，我们根据状态来判断，我们是应该自旋等待结果 (未完成)，还是返回结果或者抛出异常。根据下方的-实现就可以看出：

- `NEW` 
    - 未运行
- `COMPLETING`
    - 我们还没有结束，自旋等待。
- `NORMAL`
    - 我们就返回结果，也就是 `outcome` 变量
- `CANCELLED` 
    - 我们抛出 `CancellationException`
- `EXCEPTIONAL`
    - 我们知道这个 `outcome` 变量实际上是一个 `Throwable`， 我们直接抛出异常。

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}

private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

