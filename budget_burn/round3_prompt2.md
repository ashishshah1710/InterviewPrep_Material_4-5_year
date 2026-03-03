# Ultimate Guide to Java Concurrency

Java concurrency is essential for building high-performance, responsive, and scalable applications. This guide covers in detail:

- `CompletableFuture`
- `ExecutorService`
- `synchronized` vs `ReentrantLock`
- `volatile`
- `ConcurrentHashMap`
- Thread pools
- `ForkJoinPool`
- Atomic variables (`AtomicInteger` and family)
- `StampedLock`

Each section contains explanations and numerous code examples.

---

## 1. **`CompletableFuture`**

`CompletableFuture` introduced in Java 8 represents a future result and offers API for asynchronous computation and chaining.

### Basic Creation

```java
CompletableFuture<String> cf = new CompletableFuture<>();
// Complete manually
cf.complete("Hello World!");

String result = cf.get(); // Results: "Hello World!"
```

### Supply an Asynchronous Value

```java
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
    return "Result";
});
```

### Runnables & Consumers

```java
CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> {
    System.out.println("Running async task.");
});
```
### Chaining: thenApply, thenAccept

```java
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> 10)
    .thenApply(x -> x * 2); // 20

cf.thenAccept(result -> System.out.println("Result: " + result));
```

### Combine Futures

```java
CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> 20);

CompletableFuture<Integer> combined = f1.thenCombine(f2, (a, b) -> a + b); // 30
```

### Compose Futures

```java
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> 10)
    .thenCompose(num -> CompletableFuture.supplyAsync(() -> num * 2)); // 20
```

### Exception Handling

```java
CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("Fail!");
    return 10;
}).exceptionally(e -> {
    return -1;
});
```

### AllOf & AnyOf

#### `allOf`

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "a");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "b");

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
all.join(); // Wait for both

String res1 = f1.get();
String res2 = f2.get();
```

#### `anyOf`

```java
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<Object> winner = CompletableFuture.anyOf(cf1, cf2);
winner.join(); // Returns fastest
```

---

## 2. **ExecutorService**

`ExecutorService` manages thread lifecycle. It offers fixed, cached, and custom thread pools.

### Creating Executor

```java
ExecutorService exec = Executors.newFixedThreadPool(4);
// Or:
ExecutorService exec2 = Executors.newCachedThreadPool();
ExecutorService exec3 = Executors.newSingleThreadExecutor();
```

### Submit Tasks (`Runnable` and `Callable`)

```java
exec.submit(() -> System.out.println("Runnable Task"));

Future<Integer> f = exec.submit(() -> {
    return 42;
});

Integer num = f.get(); // 42
```

### Shutting Down Executor

```java
exec.shutdown();
exec.awaitTermination(1, TimeUnit.MINUTES);
```

### Scheduled Executor

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

scheduler.schedule(() -> System.out.println("Delayed"), 1, TimeUnit.SECONDS);

scheduler.scheduleAtFixedRate(() -> System.out.println("Repeat"), 1, 2, TimeUnit.SECONDS);
```

---

## 3. **`synchronized` vs `ReentrantLock`**

Both are for mutual exclusion, but `ReentrantLock` gives more control.

### `synchronized` block

```java
public class Counter {
    private int value = 0;

    public synchronized void increment() {
        value++;
    }

    public int get() {
        return value;
    }
}
```

Or sync on an object:

```java
private final Object lock = new Object();

void safeUpdate() {
    synchronized (lock) {
        // critical section
    }
}
```

### ReentrantLock

```java
Lock lock = new ReentrantLock();

public void safeOp() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

### Features of ReentrantLock

- Timed lock:

    ```java
    if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
        try {
            // critical section
        } finally {
            lock.unlock();
        }
    }
    ```

- Fairness:

    ```java
    Lock fairLock = new ReentrantLock(true);
    ```

- Multiple lock acquisition (reentrant):

### Condition Variables

```java
Lock lock = new ReentrantLock();
Condition cond = lock.newCondition();

void awaitCondition() throws InterruptedException {
    lock.lock();
    try {
        cond.await(); // Wait for signal
    } finally {
        lock.unlock();
    }
}

void signalCondition() {
    lock.lock();
    try {
        cond.signal();
    } finally {
        lock.unlock();
    }
}
```

---

## 4. **`volatile`**

`volatile` ensures visibility between threads for primitives/references.

```java
private volatile boolean running = true;

public void stop() {
    running = false;
}

public void run() {
    while (running) {
        // do work
    }
}
```

### When to Use

- For flags, status fields, not for compound actions (count++, etc.)

### Not atomic for composite operations

```java
private volatile int count;

void increment() {
    count++; // NOT atomic!
}
```

---

## 5. **ConcurrentHashMap**

Thread-safe hash map for concurrent usage.

### Basic Usage

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.put("b", 2);
```

### Concurrent Operations

```java
map.computeIfAbsent("c", k -> 3);
int value = map.getOrDefault("a", 0);
```

### Atomic operations

```java
map.putIfAbsent("x", 10);
map.compute("b", (key, val) -> val == null ? 0 : val + 1);
map.remove("a", 2); // Remove only if mapping matches
```

### For Each

```java
map.forEach((key, val) -> System.out.println(key + " = " + val));
```

### Parallelism

```java
map.forEach(4, (key, val) -> System.out.println(key));
map.reduce(2,
    (k, v) -> v,
    (a, b) -> a + b);
```

---

## 6. **Thread Pools**

Java thread pools handle worker threads for task execution.

### Fixed Pool

```java
ExecutorService pool = Executors.newFixedThreadPool(10);

for (int i = 0; i < 100; i++) {
    pool.submit(() -> {
        System.out.println(Thread.currentThread().getName());
    });
}
```

### Cached Pool

```java
ExecutorService cachePool = Executors.newCachedThreadPool();
```

### Custom ThreadPoolExecutor

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    5, 10, 1, TimeUnit.MINUTES,
    new LinkedBlockingQueue<Runnable>(),
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### Pool Configuration

- Core size, max size
- KeepAlive time
- Queue types: `ArrayBlockingQueue`, `LinkedBlockingQueue`
- Rejection policies: `AbortPolicy`, `CallerRunsPolicy`, `DiscardPolicy`

---

## 7. **ForkJoinPool**

ForkJoinPool is for parallel recursive tasks (`ForkJoinTask`, `RecursiveTask`, `RecursiveAction`).

### Basic Usage

```java
ForkJoinPool fjPool = new ForkJoinPool();

int sum = fjPool.invoke(new SumTask(arr, 0, arr.length));
```

### RecursiveTask Example

```java
class SumTask extends RecursiveTask<Integer> {
    private int[] arr;
    private int start, end;

    SumTask(int[] arr, int start, int end) {
        this.arr = arr;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        if (end - start <= 100) {
            int tot = 0;
            for (int i = start; i < end; i++) tot += arr[i];
            return tot;
        } else {
            int mid = (start + end) / 2;
            SumTask left = new SumTask(arr, start, mid);
            SumTask right = new SumTask(arr, mid, end);
            left.fork();
            int rightResult = right.compute();
            int leftResult = left.join();
            return leftResult + rightResult;
        }
    }
}
```

### RecursiveAction Example

```java
class PrintTask extends RecursiveAction {
    int[] arr;
    int start, end;

    PrintTask(int[] arr, int start, int end) {
        this.arr = arr;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if (end - start <= 10) {
            for (int i = start; i < end; i++)
                System.out.println(arr[i]);
        } else {
            int mid = (start + end) / 2;
            invokeAll(
                new PrintTask(arr, start, mid),
                new PrintTask(arr, mid, end)
            );
        }
    }
}
```

---

## 8. **Atomic Variables**

Atomic variables (`AtomicInteger`, `AtomicLong`, `AtomicReference`, etc.) provide atomic operations.

### AtomicInteger

```java
AtomicInteger ai = new AtomicInteger(0);

ai.incrementAndGet(); // +1
ai.getAndAdd(5); // +5
ai.compareAndSet(5, 10); // CAS
int cur = ai.get();
```

### AtomicReference

```java
AtomicReference<String> ref = new AtomicReference<>("foo");

ref.set("bar");
String old = ref.getAndSet("baz");
boolean updated = ref.compareAndSet("baz", "qux");
```

### AtomicLong, AtomicBoolean, etc.

```java
AtomicBoolean ab = new AtomicBoolean(true);
ab.set(false);
```

### Array atomics

```java
AtomicIntegerArray arr = new AtomicIntegerArray(10);
arr.incrementAndGet(2);
```

---

## 9. **`StampedLock`**

`StampedLock` offers three locking modes: **write**, **read (optimistic and regular)**.

### Basic Usage

```java
StampedLock lock = new StampedLock();

long stamp = lock.writeLock();
try {
    // Exclusive access
} finally {
    lock.unlockWrite(stamp);
}
```

### Read Lock

```java
long stamp = lock.readLock();
try {
    // Shared access
} finally {
    lock.unlockRead(stamp);
}
```

### Optimistic Read

```java
long stamp = lock.tryOptimisticRead();
// Read state
if (!lock.validate(stamp)) {
    // Retry with read lock
    stamp = lock.readLock();
    try {
        // Read state
    } finally {
        lock.unlockRead(stamp);
    }
}
// else: successful optimistic read
```

### Example: StampedLock Counter

```java
class CounterStamped {
    private final StampedLock lock = new StampedLock();
    private int count = 0;

    public void increment() {
        long stamp = lock.writeLock();
        try {
            count++;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public int getCount() {
        long stamp = lock.tryOptimisticRead();
        int current = count;
        if (!lock.validate(stamp)) {
            stamp = lock.readLock();
            try {
                current = count;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return current;
    }
}
```

---

# Practical Examples

## 1. Producer-Consumer with BlockingQueue

```java
BlockingQueue<Integer> queue = new LinkedBlockingQueue<>();

// Producer
Runnable producer = () -> {
    for (int i = 0; i < 100; i++) {
        queue.put(i);
        System.out.println("Produced: " + i);
    }
};

// Consumer
Runnable consumer = () -> {
    for (int i = 0; i < 100; i++) {
        int val = queue.take();
        System.out.println("Consumed: " + val);
    }
};

Thread p = new Thread(producer);
Thread c = new Thread(consumer);
p.start();
c.start();
```

## 2. Thread-safe Counter (AtomicInteger)

```java
class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int get() {
        return count.get();
    }
}
```

## 3. Deadlock Example

```java
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {
        // ...
        synchronized (lockB) {
            // ...
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {
        // ...
        synchronized (lockA) {
            // deadlock!
        }
    }
});
```

---

# Hundreds of Short Code Examples

Here are dozens more concise examples.

```java
// Basic Thread Start
new Thread(() -> System.out.println("Hello")).start();

// Synchronized Block
synchronized (this) { }

// ReentrantLock Timeout
lock.tryLock(1, TimeUnit.SECONDS);

// AtomicDouble from Guava
AtomicDouble dbl = new AtomicDouble(1.23);

// ConcurrentHashMap forEach
map.forEach((k, v) -> System.out.println(k + ":" + v));

// ExecutorService Lifetime
exec.shutdown();
exec.awaitTermination(1, TimeUnit.MINUTES);

// ForkJoinTask Submission
forkJoinPool.submit(new Task());

// CompletableFuture Exception Handling
cf.handle((result, ex) -> ex == null ? result : "Fallback");

// CompletableFuture Timeout
cf.orTimeout(5, TimeUnit.SECONDS);

// ThreadPoolExecutor Core/max size
ThreadPoolExecutor tp = new ThreadPoolExecutor(2, 4, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<>());

// CountDownLatch for synchronization
CountDownLatch latch = new CountDownLatch(2);
latch.countDown();
latch.await();

// CyclicBarrier for group sync
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All reached!"));

// ReadWriteLock Example
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock(); rw.readLock().unlock();
rw.writeLock().lock(); rw.writeLock().unlock();

// Semaphore for permitting
Semaphore sem = new Semaphore(2);
sem.acquire(); sem.release();

// ThreadLocal usage
ThreadLocal<Integer> local = ThreadLocal.withInitial(() -> 0);
local.set(10); int val = local.get();

// Phaser for phase synchronization
Phaser phaser = new Phaser(3);
phaser.arriveAndAwaitAdvance();
```

---

# Best Practices

- Use thread-safe collections for concurrency
- Avoid shared mutable state when possible
- Favor higher-level abstractions (ExecutorService, CompletableFuture) over manual thread creation
- Always release locks in `finally`
- Monitor thread pools and configure bounds carefully
- Prefer atomic classes for counters/flags
- Use `StampedLock`/`ReadWriteLock` for frequent read, rare write scenarios
- Beware of deadlock, race conditions
- Test multithreaded code!

---

# Summary Table

| Aspect | Use Case | Example API/Class |
|---|---|---|
| Async tasks | Background tasks | CompletableFuture, ExecutorService |
| Locks | Critical section | synchronized, ReentrantLock, StampedLock |
| Read-heavy data | Fast reads, rare writes | StampedLock, ReadWriteLock |
| Atomic updates | Counters, flags | AtomicInteger, AtomicReference |
| Collections | Map/list/set | ConcurrentHashMap, CopyOnWriteArrayList |
| Thread pools | Task queueing | ThreadPoolExecutor, ForkJoinPool |
| Coordination | Group sync | CountDownLatch, CyclicBarrier, Phaser |

---

# Further Reading

- [Java Concurrency in Practice](https://jcip.net/)
- [Official Java Concurrency Tutorial](https://docs.oracle.com/javase/tutorial/essential/concurrency/)

---

# Conclusion

This guide has covered Java concurrency comprehensively. Use the code examples and frameworks above to build robust, scalable, and maintainable concurrent applications. If you need specific code samples for any scenario, please ask!
