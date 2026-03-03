**Java Multithreading Interview Guide (Intermediate: 4-5 Years Experience)**

---

## Contents

1. **Thread Lifecycle**
2. **Runnable vs Callable**
3. **Future vs CompletableFuture**
4. **Thread Pools**
   - Fixed
   - Cached
   - Scheduled
   - ForkJoinPool
5. **Synchronized Blocks**
6. **ReentrantLock**
7. **ReadWriteLock**
8. **CountDownLatch**
9. **CyclicBarrier**
10. **Semaphore**
11. **Phaser**
12. **ThreadLocal**
13. **Deadlock: Detection and Prevention**
14. **Producer-Consumer Problem**
15. **Race Conditions**
16. **Atomic Variables**
17. **Java Memory Model**
18. **Happens-Before Relationship**
19. **Tricky Output Questions**

---

## 1. Thread Lifecycle

**Thread States:**
- NEW
- RUNNABLE
- BLOCKED
- WAITING
- TIMED_WAITING
- TERMINATED

**Code Example:**

```java
public class ThreadLifecycleDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {}
        });
        System.out.println(t.getState()); // NEW

        t.start();
        System.out.println(t.getState()); // RUNNABLE (maybe)

        Thread.sleep(10); // Let thread start
        System.out.println(t.getState()); // RUNNABLE or TIMED_WAITING

        t.join();
        System.out.println(t.getState()); // TERMINATED
    }
}
```

**Interview Question:**  
Explain each Thread state and transitions.

**Answer:**  
- *NEW*: Thread created, not started.
- *RUNNABLE*: Started, ready/running.
- *BLOCKED*: Waiting for lock.
- *WAITING*: Waiting for another thread (e.g., via `wait()`, `join()`).
- *TIMED_WAITING*: Waiting for a specific time (e.g., `sleep()`, timed `wait()`).
- *TERMINATED*: Finished.

---

## 2. Runnable vs Callable

**Runnable:**
- No return value.
- Cannot throw checked exceptions.

**Callable:**
- Returns a value (`call()` method).
- Can throw checked exceptions.

**Code Comparison:**

```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Running...");
    }
}

class MyCallable implements Callable<String> {
    public String call() throws Exception {
        return "Callable executed";
    }
}
```

**Execution:**

```java
ExecutorService executor = Executors.newFixedThreadPool(1);

executor.submit(new MyRunnable()); // No return value
Future<String> future = executor.submit(new MyCallable());
String result = future.get(); // Returns "Callable executed"
```

**Interview Question:**  
Difference between Runnable and Callable?

---

## 3. Future vs CompletableFuture

**Future:**
- Represents result of an async computation.
- blocking get().
- No chaining.

**CompletableFuture:**
- Supports chaining, composing, async execution via methods like `thenApply`, `thenCompose`, etc.
- Non-blocking.

**Example:**

```java
Future<Integer> future = executor.submit(() -> 5);
int result = future.get(); // blocking

CompletableFuture<Integer> cf = CompletableFuture.supplyAsync(() -> 5);
cf.thenApply(x -> x * 2)
  .thenAccept(System.out::println); // prints 10
```

**Interview Question:**
What happens if you call `get()` multiple times on a Future or CompletableFuture?

**Answer:**
- `Future.get()` blocks until result available.
- `CompletableFuture.get()` also blocks, but can be combined non-blocking with `then*` methods.

---

## 4. Thread Pools

### Fixed Thread Pool

```java
ExecutorService fixedPool = Executors.newFixedThreadPool(3);
for (int i = 0; i < 5; i++) {
    fixedPool.submit(() -> {
        System.out.println(Thread.currentThread().getName());
    });
}
```

**Behavior:** Up to N simultaneous threads, others queued.

### Cached Thread Pool

```java
ExecutorService cachedPool = Executors.newCachedThreadPool();
for (int i = 0; i < 5; i++) {
    cachedPool.submit(() -> {
        System.out.println(Thread.currentThread().getName());
    });
}
```

**Behavior:** Creates new threads as needed, reuses idle.

### Scheduled Thread Pool

```java
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(2);
scheduledPool.schedule(() -> System.out.println("Task delayed"), 1, TimeUnit.SECONDS);
scheduledPool.scheduleAtFixedRate(() -> System.out.println("Fixed Rate Task"), 0, 1, TimeUnit.SECONDS);
```

### ForkJoinPool

```java
ForkJoinPool pool = new ForkJoinPool();
pool.submit(() -> {
    // parallel tasks
});
```

**Interview Questions:**
- How is ForkJoinPool different from ExecutorService?
- Explain work-stealing algorithm.

---

## 5. Synchronized Blocks

**Syntax:**

```java
synchronized (lockObject) {
    // critical section
}
```

**Code Example:**

```java
class Counter {
    private int count = 0;
    public void increment() {
        synchronized (this) {
            count++;
        }
    }
}
```

**Interview Question:**
What happens if you synchronize on a mutable object?

**Answer:** Always synchronize on a constant reference to avoid accidental release.

---

## 6. ReentrantLock

**Usage:**

```java
Lock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

**Advantages over synchronized:**  
- TryLock, fairness, interruptibility, condition variables.

**Example: TryLock**

```java
if (lock.tryLock()) {
    try {
        // critical section
    } finally {
        lock.unlock();
    }
}
```

---

## 7. ReadWriteLock

**Usage:**

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();
try {
    // reading
} finally {
    rwLock.readLock().unlock();
}

rwLock.writeLock().lock();
try {
    // writing
} finally {
    rwLock.writeLock().unlock();
}
```

**Interview Question:**
When does ReadWriteLock provide advantage?

**Answer:** High reads, low writes - multiple readers, one writer.

---

## 8. CountDownLatch

**Usage:**

```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        // task
        latch.countDown();
    }).start();
}
latch.await(); // main thread waits
```

**Interview Question:**
What happens if count never reaches zero?

**Answer:** Await will block indefinitely.

---

## 9. CyclicBarrier

**Usage:**

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All threads reached barrier!"));
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            barrier.await();
        } catch (Exception e) {}
    }).start();
}
```

**Interview Question:**
Difference between CountDownLatch and CyclicBarrier?

**Answer:** CyclicBarrier is reusable; CountDownLatch is not.

---

## 10. Semaphore

**Usage:**

```java
Semaphore semaphore = new Semaphore(2);
for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        try {
            semaphore.acquire();
            System.out.println(Thread.currentThread().getName() + " acquired");
            Thread.sleep(100);
        } catch (Exception e) {}
        finally {
            semaphore.release();
        }
    }).start();
}
```

**Interview Question:**
Use case for Semaphore?

**Answer:** Limiting concurrent access (e.g., connection pools).

---

## 11. Phaser

**Usage:**

```java
Phaser phaser = new Phaser(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Before phase: " + phaser.getPhase());
        phaser.arriveAndAwaitAdvance();
        System.out.println("After phase: " + phaser.getPhase());
    }).start();
}
```

**Interview Question:**
How is Phaser better than CyclicBarrier?

**Answer:** Phaser supports dynamic registration and multiple phases.

---

## 12. ThreadLocal

**Usage:**

```java
ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> 0);

new Thread(() -> {
    threadLocal.set(10);
    System.out.println("Thread 1: " + threadLocal.get());
}).start();

new Thread(() -> {
    threadLocal.set(20);
    System.out.println("Thread 2: " + threadLocal.get());
}).start();
```

**Interview Question:**
Why is ThreadLocal useful?

**Answer:** Each thread gets its own variable instance; useful for user session, database connection, etc.

---

## 13. Deadlock: Detection and Prevention

**Deadlock Scenario Example:**

```java
class A {
    synchronized void methodA(B b) {
        b.last();
    }
    synchronized void last() {}
}

class B {
    synchronized void methodB(A a) {
        a.last();
    }
    synchronized void last() {}
}

public class DeadlockDemo {
    public static void main(String[] args) {
        A a = new A();
        B b = new B();

        Thread t1 = new Thread(() -> {
            a.methodA(b);
        }, "Thread1");

        Thread t2 = new Thread(() -> {
            b.methodB(a);
        }, "Thread2");

        t1.start();
        t2.start();
    }
}
```

**Detection:**
- Thread dump analysis.
- Using tools like VisualVM.

**Prevention:**
- Lock ordering
- Timeout
- Using tryLock

---

## 14. Producer-Consumer Problem

**Using BlockingQueue:**

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

Thread producer = new Thread(() -> {
    try {
        for (int i = 0; i < 5; i++) {
            queue.put(i);
            System.out.println("Produced " + i);
        }
    } catch (InterruptedException e) {}
});

Thread consumer = new Thread(() -> {
    try {
        for (int i = 0; i < 5; i++) {
            Integer val = queue.take();
            System.out.println("Consumed " + val);
        }
    } catch (InterruptedException e) {}
});

producer.start();
consumer.start();
```

---

## 15. Race Conditions

**Problem Example:**

```java
class Counter {
    int count = 0;
    public void increment() {
        count++; // not thread-safe
    }
}

Counter c = new Counter();
Thread t1 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) c.increment();
});
Thread t2 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) c.increment();
});
t1.start();
t2.start();
t1.join();
t2.join();
System.out.println(c.count); // <20000 likely
```

---

## 16. Atomic Variables

**Usage:**

```java
AtomicInteger atomicCount = new AtomicInteger(0);
Thread t1 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) atomicCount.incrementAndGet();
});
Thread t2 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) atomicCount.incrementAndGet();
});
t1.start();
t2.start();
t1.join();
t2.join();
System.out.println(atomicCount.get()); // 20000 guaranteed
```

---

## 17. Java Memory Model (JMM)

**Key Points:**
- All threads have local variables and main memory.
- Writes in one thread may not be visible to others unless synchronized via locks/volatile.

**volatile example:**

```java
volatile boolean stop = false;

Thread t = new Thread(() -> {
    while (!stop) {}
});
stop = true;
```

**Interview Question:**
What is the significance of volatile keyword?

**Answer:** Ensures visibility of changes across threads.

---

## 18. Happens-Before Relationship

**Examples:**
- Synchronization
- Lock/unlock
- volatile write/read
- start/join

```java
// Thread A:
synchronized(lock) {
    x = 1;
}

// Thread B:
synchronized(lock) {
    // can see x==1
}
```

**Interview Question:**  
How is happens-before enforced in Java?

---

## 19. Tricky Output Questions

### Question 1: Mixed Synchronization

```java
class Foo {
    public void test() {
        synchronized(this) {
            System.out.println("A");
        }
    }
    public synchronized void test2() {
        System.out.println("B");
    }
}
```

If two threads call `test()` and `test2()` on the same Foo instance, will they block each other?

**Answer:**  
Yes, both are synchronized on `this`, so only one may enter.

---

### Question 2: Visibility with Non-Volatile

```java
boolean run = true;
Thread t = new Thread(() -> {
    while (run) {}
});
t.start();
Thread.sleep(100);
run = false;
```

**Can the thread exit?**

**Answer:**  
"run" is not volatile, so thread may loop forever due to JVM optimizations.

---

### Question 3: ForkJoinPool vs FixedThreadPool

Given a recursive task in ForkJoinPool, what happens if task count exceeds thread count?

**Answer:**  
Tasks are queued and âstolenâ by worker threads when available (work-stealing).

---

### Question 4: Synchronized static vs instance

```java
class Test {
    public static synchronized void staticMethod() {}
    public synchronized void instanceMethod() {}
}
```

If two threads call static and instance methods simultaneously, will they block each other?

**Answer:**  
No. Static locks on class object; instance locks on instance.

---

### Question 5: Deadlock Output

```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized(lock1) {
        try { Thread.sleep(100); } catch(Exception e){}
        synchronized(lock2) {}
    }
});
Thread t2 = new Thread(() -> {
    synchronized(lock2) {
        try { Thread.sleep(100); } catch(Exception e){}
        synchronized(lock1) {}
    }
});
t1.start(); t2.start();
```

**Output:**  
Both threads may freeze, deadlock.

---

# Conclusion

This guide gives detailed coverage with code examples, interview questions, and tricky pitfalls. For further reading:
- *Java Concurrency in Practice* (Book)
- Java SDK official docs (`java.util.concurrent`)

**(Expand each section for specifics in interviews. For code, verify multithreading output with multiple runs - due to nondeterminism!)**

---

*If you need a full-length 10,000-word PDF/Markdown, let me know!*
