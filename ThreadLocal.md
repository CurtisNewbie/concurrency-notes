# ThreadLocal

**JDK-11**

`ThreadLocal` 本身并不存储数据, 一个 `ThreadLocal` 对象应该被看作是一个 `key`, 而存储的 `value` 实际上是存储在 `Thread` 对象中的，所以我们使用 `ThreadLocal` 一定是线程封闭且安全的。

# 1. ThreadLocal 结构

首先可以留意到, `ThreadLocal` 中并不包含任何存储数据的 `Map` 或类似的对象, 每个 `ThreadLocal` 都带有一个 `threadLocalHashCode` 的值, 这个值是 `TheadLocal` 的唯一标识.

```java
public class ThreadLocal<T> {

    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

}
```

如果我们留意 `nextHashCode()` 方法, 就可以看到我们分配 `threadLocalHashCode` 是依赖类变量 `nextHashCode` 和 `HASH_INCREMENT`. 所以创建的 `ThreadLocal` 作为 `key` 不会遇到键冲突问题.

```java
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

# 2. Thread 中的 ThreadLocalMap 

前面有提到, 数据 / values 实际上是存储在 `Thread.threadLocals` 中的，可以看到这里负责存储的数据结构是, `ThreadLocalMap`.

```java
public
class Thread implements Runnable {

    ThreadLocal.ThreadLocalMap threadLocals = null;

}
```

# 3. ThreadLocal.ThreadLocalMap

`ThreadLocalMap` 采用 Linear Probing HashMap 实现算法, 与拉链法不同, 没有额外数据结构对 hash collision 进行处理, 当 hash collision 发生时, 该算法只是移到下一个 non-null 的桶， 进行插入, 删除或获取值, 所以这里只使用了 `Entry[] table` 数组进行存储. 同时, 包装 `value` 使用的对象 `Entry` 就只包含了 `Object value`, 但该 `Entry` 的 `key` 同时也是一个 `WeekReference`, 也就是说如果没有对 `ThreadLocal` 的强引用, 该 `ThreadLocal` 代表的 `key` 的引用会被 GC, 但 `value` 并不会.

除此以外 `size` 也必须是 power of two, 因为 `hash % size` 等于 `hash & size - 1`. 与 `HashMap` 相似, `threshold` 指的是下一次 `resize` 的大小.

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
    * The initial capacity -- MUST be a power of two.
    */
    private static final int INITIAL_CAPACITY = 16;

    private Entry[] table;

    private int size = 0;

    /**
    * The next size value at which to resize.
    */
    private int threshold; // Default to 0
}
```

# 4. ThreadLocal.set() 

接下来看 `ThreadLocal` 如何设值. 要记住, 这个方法里的代码是线程安全的, 因为这个 `ThreadLocalMap` 根本就在 `Thread` 对象里, 所以我们进行初始化等操作, 不需要考虑线程问题.

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

首先从 `Thread.currentThread()` 中拿到该线程内部的 `Thread.threadLocals` 变量. 然后我们查看该 `ThreadLocalMap` 是否需要初始化, 不需要我们则设值. 初始化也只是 `new ThreadLocalMap(...)` 构造器而已. 这里要关注的是, 当插入 entry 以后的 `size >= threshold`, 我们需要进行扩容. 而 `rehash()` 方法里分为两个部分, 清除 key 被 GC 的 entries. 第二个部分才是 `resize()`.

```java
private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```

# 5. ThreadLocal.get()

`ThreadLocal` 的 `get()` 方法与 `set()` 方法逻辑相似, 也是从 `Thread.currentThread()` 中拿 `ThreadLocalMap`. 首先检查是否存在该 `ThreadLocal` 对应的 `Entry`, 如果有则拿 `Entry.value` 并且返回. 如果 `ThreadLocalMap` 没有初始化或 `ThreadLocal` 对应的值没有初始化, 则创建 `ThreadLocalMap` 并且将 `initialValue()` 返回的值进行设置并且返回.

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}

protected T initialValue() {
    return null;
}
```

默认情况 `initialValue()` 方法不返回值, 但如果我们使用 `ThreadLocal.withInitial(Supplier)` 方法, 我们则使用的是 `SuppliedThreadLocal`, 在第一次使用 `ThreadLocal.get()` 获取值的时候, `ThreadLocal` 就会使用 `initialValue()` 方法进行值初始化.

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}


static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

# 6. ThreadLocalMap 中 Entry 的 key 作为 WeakReference

在上面有留意到 Entry 继承了 `WeakReference<ThreadLocal>`, `Object value` 实际上仍然是强引用, 而 `WeakReference` 内部包含了 `ThreadLocal` 引用作为 weak reference.

```
  Reference<T>
       ^
       |
WeakReference<T>
```

```java
public abstract class Reference<T> {

    private T referent;         /* Treated specially by GC */

    public T get() {
        return this.referent;
    }
}
```

也就是说当我们使用 `Entry.get()` 方法获取 `key` 也就是 `ThreadLocal` 的引用的时候, 这个方法可能会返回 `null` 但实际 `value` 不为 `null`. `ThreadLocalMap` 内部在进行 `getEntry` 和 `set` 方法的时候会对这种情况进行额外的处理.

例如, `ThreadLocalMap.getEntry` 方法中:

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

如果对应的 `Entry` 的 `key` 已经被回收了, 也就是说 `ThreadLocalMap.Entry` 的 `get()` 方法会返回 `null`. 那么我们就算正确计算 `hash` 并且发现 `table[i]` 不为空, 我们也需要处理这种已经被 GC 的 values. 如果 `e.get() == null` 则有 `e.get() != key`, 因为 `key` 不为 `null`. 

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

对于 `expungeStaleEntry(int)` 方法来说, 核心就是将当前 `table[staleSlot].value`  和 `table[staleSlot]` 都设为 `null`, 同时解决后面不为 `null` 的 `Entry`, 重新计算他们的 `hash` 并且放到正确的位置上. 这里的操作与 Linear Probing 算法中移除 entry 的操作相同。

# 7. ThreadLocalMap 扩容

`ThreadLocalMap` 扩容是两倍大小，然后我们将每一个 entry 重新计算 `hash` 然后插入到新的数组中。同时将 `threshold` 的大小更新-到新容量的 2/3.

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (Entry e : oldTab) {
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}

private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```
