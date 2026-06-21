# Java Multithreading & Concurrency — Complete Guide

## 1. What is Multithreading?

A **thread** is the smallest unit of execution within a process. Java supports multithreading natively, allowing multiple threads to run concurrently within a single program. This enables:
- Better CPU utilization
- Faster execution of parallelizable tasks
- Responsive UIs (e.g., background tasks without freezing the screen)

A Java application always starts with one thread — the **main thread** — and you can spawn additional threads from it.

---

## 2. Creating Threads

### Method 1: Extending `Thread`
```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}

MyThread t = new MyThread();
t.start(); // Do NOT call run() directly — that won't create a new thread
```

### Method 2: Implementing `Runnable`
```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Runnable running: " + Thread.currentThread().getName());
    }
}

Thread t = new Thread(new MyRunnable());
t.start();
```

### Method 3: Lambda (Java 8+)
```java
Thread t = new Thread(() -> System.out.println("Lambda thread!"));
t.start();
```

> ✅ Prefer `Runnable` or lambdas over extending `Thread` — it keeps your class hierarchy clean and allows the task to be submitted to thread pools.

---

## 3. Thread Lifecycle

```
NEW → RUNNABLE → RUNNING → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

| State | Description |
|---|---|
| `NEW` | Thread created but `start()` not called yet |
| `RUNNABLE` | Ready to run, waiting for CPU |
| `RUNNING` | Currently executing |
| `BLOCKED` | Waiting to acquire a monitor lock |
| `WAITING` | Waiting indefinitely (e.g., `wait()`) |
| `TIMED_WAITING` | Waiting for a specified time (e.g., `sleep(ms)`) |
| `TERMINATED` | Finished execution |

---

## 4. Thread Methods

```java
t.start();          // Start the thread
t.join();           // Wait for thread t to finish before continuing
t.sleep(1000);      // Pause current thread for 1 second
t.interrupt();      // Request thread to stop
t.isAlive();        // Check if thread is still running
t.setName("name");  // Set thread name for debugging
t.setPriority(5);   // Set priority (1–10, default 5)
```

---

## 5. Concurrency Problems

When multiple threads share data, three classic problems arise:

### 🔴 Race Condition
Two threads read and write shared data simultaneously, leading to unpredictable results.
```java
// Dangerous — counter++ is NOT atomic
int counter = 0;
counter++; // read → increment → write (3 steps, can be interrupted)
```

### 🔴 Deadlock
Two threads each hold a lock the other needs — both wait forever.
```java
// Thread 1 holds lockA, waits for lockB
// Thread 2 holds lockB, waits for lockA → Deadlock!
```

### 🔴 Thread Starvation
A low-priority thread never gets CPU time because higher-priority threads always run first.

---

## 6. Synchronization

### `synchronized` keyword
Ensures only one thread executes a block at a time by acquiring a **monitor lock**.

```java
// Synchronized method
public synchronized void increment() {
    counter++;
}

// Synchronized block (more fine-grained)
public void increment() {
    synchronized (this) {
        counter++;
    }
}
```

### `volatile` keyword
Ensures a variable is always read from main memory, not a thread's local cache. Use for simple flags — NOT for compound operations like `counter++`.

```java
private volatile boolean running = true;
```

---

## 7. `java.util.concurrent` — The Modern Way

Java 5+ introduced the `java.util.concurrent` package, which provides high-level concurrency utilities.

### Atomic Classes
Thread-safe operations without `synchronized`:
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // atomic, no lock needed
```

### Locks (`ReentrantLock`)
More flexible than `synchronized` — supports try-lock, timed lock, fairness:
```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // always unlock in finally!
}
```

### `ReadWriteLock`
Allows multiple readers OR one writer at a time — great for read-heavy workloads:
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();   // multiple threads can hold this simultaneously
rwLock.writeLock().lock();  // exclusive
```

---

## 8. Thread Pools & `ExecutorService`

Creating a new thread for every task is expensive. Thread pools reuse threads.

```java
// Fixed pool of 4 threads
ExecutorService executor = Executors.newFixedThreadPool(4);

executor.submit(() -> {
    System.out.println("Task running in pool");
});

executor.shutdown(); // gracefully stop after tasks complete
```

### Types of Thread Pools

| Pool | Method | Use Case |
|---|---|---|
| Fixed | `newFixedThreadPool(n)` | Known, bounded concurrency |
| Cached | `newCachedThreadPool()` | Many short-lived tasks |
| Single | `newSingleThreadExecutor()` | Sequential background tasks |
| Scheduled | `newScheduledThreadPool(n)` | Delayed / periodic tasks |

---

## 9. `Future` and `Callable`

`Callable` is like `Runnable` but returns a result and can throw exceptions.

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

Future<Integer> future = executor.submit(task);

// Do other work...

Integer result = future.get(); // blocks until result is ready
System.out.println("Result: " + result);
```

---

## 10. `CompletableFuture` (Java 8+)

The most powerful way to write async, non-blocking code in Java.

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchDataFromDB())       // runs async
    .thenApply(data -> process(data))           // transform result
    .thenApply(result -> "Final: " + result)    // chain transformations
    .exceptionally(ex -> "Error: " + ex.getMessage()); // handle errors

String result = future.get();
```

### Combining Futures
```java
// Run two tasks in parallel, combine results
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
System.out.println(combined.get()); // "Hello World"
```

---

## 11. Concurrent Collections

Regular collections like `ArrayList` and `HashMap` are NOT thread-safe. Use these instead:

| Collection | Thread-Safe Alternative |
|---|---|
| `ArrayList` | `CopyOnWriteArrayList` |
| `HashMap` | `ConcurrentHashMap` |
| `LinkedList` (queue) | `ConcurrentLinkedQueue` |
| `PriorityQueue` | `PriorityBlockingQueue` |
| `Deque` | `LinkedBlockingDeque` |

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("key", 1);
map.computeIfAbsent("key2", k -> expensiveCompute(k));
```

---

## 12. Synchronization Utilities

### `CountDownLatch`
Wait for N threads to complete before proceeding:
```java
CountDownLatch latch = new CountDownLatch(3);

// In each of 3 threads:
latch.countDown(); // signal done

// Main thread:
latch.await(); // blocks until count reaches 0
```

### `CyclicBarrier`
Make N threads wait for each other at a common point:
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All threads ready!"));
barrier.await(); // each thread calls this; all proceed together
```

### `Semaphore`
Limit the number of threads accessing a resource:
```java
Semaphore semaphore = new Semaphore(3); // max 3 concurrent threads
semaphore.acquire();
try {
    // access limited resource
} finally {
    semaphore.release();
}
```

---

## 13. Best Practices

- ✅ Prefer `ExecutorService` over raw `Thread` creation
- ✅ Use `AtomicInteger`, `AtomicLong` for simple counters
- ✅ Use `ConcurrentHashMap` instead of `synchronized HashMap`
- ✅ Always release locks in a `finally` block
- ✅ Keep synchronized blocks as short as possible
- ✅ Avoid nested locks to prevent deadlocks
- ✅ Use `volatile` for simple shared flags only
- ✅ Prefer `CompletableFuture` for async workflows
- ❌ Never call `run()` directly — always use `start()`
- ❌ Don't use `Thread.stop()` — it's deprecated and unsafe

---

## 14. Quick Reference Cheat Sheet

```
Simple async task        → Runnable + ExecutorService
Task with return value   → Callable + Future
Async chaining/pipeline  → CompletableFuture
Shared counter           → AtomicInteger
Shared map               → ConcurrentHashMap
Limit concurrent access  → Semaphore
Wait for N threads       → CountDownLatch
Mutual exclusion         → synchronized / ReentrantLock
Simple shared flag       → volatile boolean
```

---

*Created with Claude — part of the Java World vault*
# Java Multithreading & Concurrency — Complete Guide

## 1. What is Multithreading?

A **thread** is the smallest unit of execution within a process. Java supports multithreading natively, allowing multiple threads to run concurrently within a single program. This enables:
- Better CPU utilization
- Faster execution of parallelizable tasks
- Responsive UIs (e.g., background tasks without freezing the screen)

A Java application always starts with one thread — the **main thread** — and you can spawn additional threads from it.

---

## 2. Creating Threads

### Method 1: Extending `Thread`
```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Thread running: " + Thread.currentThread().getName());
    }
}

MyThread t = new MyThread();
t.start(); // Do NOT call run() directly — that won't create a new thread
```

### Method 2: Implementing `Runnable`
```java
class MyRunnable implements Runnable {
    public void run() {
        System.out.println("Runnable running: " + Thread.currentThread().getName());
    }
}

Thread t = new Thread(new MyRunnable());
t.start();
```

### Method 3: Lambda (Java 8+)
```java
Thread t = new Thread(() -> System.out.println("Lambda thread!"));
t.start();
```

> Prefer `Runnable` or lambdas over extending `Thread` — it keeps your class hierarchy clean and allows the task to be submitted to thread pools.

---

## 3. Thread Lifecycle

```
NEW → RUNNABLE → RUNNING → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
```

| State | Description |
|---|---|
| `NEW` | Thread created but `start()` not called yet |
| `RUNNABLE` | Ready to run, waiting for CPU |
| `RUNNING` | Currently executing |
| `BLOCKED` | Waiting to acquire a monitor lock |
| `WAITING` | Waiting indefinitely (e.g., `wait()`) |
| `TIMED_WAITING` | Waiting for a specified time (e.g., `sleep(ms)`) |
| `TERMINATED` | Finished execution |

---

## 4. Thread Methods

```java
t.start();          // Start the thread
t.join();           // Wait for thread t to finish before continuing
t.sleep(1000);      // Pause current thread for 1 second
t.interrupt();      // Request thread to stop
t.isAlive();        // Check if thread is still running
t.setName("name");  // Set thread name for debugging
t.setPriority(5);   // Set priority (1–10, default 5)
```

---

## 5. Concurrency Problems

When multiple threads share data, three classic problems arise:

### Race Condition
Two threads read and write shared data simultaneously, leading to unpredictable results.
```java
// Dangerous — counter++ is NOT atomic
int counter = 0;
counter++; // read → increment → write (3 steps, can be interrupted)
```

### Deadlock
Two threads each hold a lock the other needs — both wait forever.
```java
// Thread 1 holds lockA, waits for lockB
// Thread 2 holds lockB, waits for lockA → Deadlock!
```

### Thread Starvation
A low-priority thread never gets CPU time because higher-priority threads always run first.

---

## 6. Synchronization

### `synchronized` keyword
Ensures only one thread executes a block at a time by acquiring a **monitor lock**.

```java
// Synchronized method
public synchronized void increment() {
    counter++;
}

// Synchronized block (more fine-grained)
public void increment() {
    synchronized (this) {
        counter++;
    }
}
```

### `volatile` keyword
Ensures a variable is always read from main memory, not a thread's local cache. Use for simple flags — NOT for compound operations like `counter++`.

```java
private volatile boolean running = true;
```

---

## 7. `java.util.concurrent` — The Modern Way

Java 5+ introduced the `java.util.concurrent` package, which provides high-level concurrency utilities.

### Atomic Classes
Thread-safe operations without `synchronized`:
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // atomic, no lock needed
```

### Locks (`ReentrantLock`)
More flexible than `synchronized` — supports try-lock, timed lock, fairness:
```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // always unlock in finally!
}
```

### `ReadWriteLock`
Allows multiple readers OR one writer at a time — great for read-heavy workloads:
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();   // multiple threads can hold this simultaneously
rwLock.writeLock().lock();  // exclusive
```

---

## 8. Thread Pools & `ExecutorService`

Creating a new thread for every task is expensive. Thread pools reuse threads.

```java
// Fixed pool of 4 threads
ExecutorService executor = Executors.newFixedThreadPool(4);

executor.submit(() -> {
    System.out.println("Task running in pool");
});

executor.shutdown(); // gracefully stop after tasks complete
```

### Types of Thread Pools

| Pool | Method | Use Case |
|---|---|---|
| Fixed | `newFixedThreadPool(n)` | Known, bounded concurrency |
| Cached | `newCachedThreadPool()` | Many short-lived tasks |
| Single | `newSingleThreadExecutor()` | Sequential background tasks |
| Scheduled | `newScheduledThreadPool(n)` | Delayed / periodic tasks |

---

## 9. `Future` and `Callable`

`Callable` is like `Runnable` but returns a result and can throw exceptions.

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

Future<Integer> future = executor.submit(task);

// Do other work...

Integer result = future.get(); // blocks until result is ready
System.out.println("Result: " + result);
```

---

## 10. `CompletableFuture` (Java 8+)

The most powerful way to write async, non-blocking code in Java.

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchDataFromDB())       // runs async
    .thenApply(data -> process(data))           // transform result
    .thenApply(result -> "Final: " + result)    // chain transformations
    .exceptionally(ex -> "Error: " + ex.getMessage()); // handle errors

String result = future.get();
```

### Combining Futures
```java
// Run two tasks in parallel, combine results
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
System.out.println(combined.get()); // "Hello World"
```

---

## 11. Concurrent Collections

Regular collections like `ArrayList` and `HashMap` are NOT thread-safe. Use these instead:

| Collection | Thread-Safe Alternative |
|---|---|
| `ArrayList` | `CopyOnWriteArrayList` |
| `HashMap` | `ConcurrentHashMap` |
| `LinkedList` (queue) | `ConcurrentLinkedQueue` |
| `PriorityQueue` | `PriorityBlockingQueue` |
| `Deque` | `LinkedBlockingDeque` |

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("key", 1);
map.computeIfAbsent("key2", k -> expensiveCompute(k));
```

---

## 12. Synchronization Utilities

### `CountDownLatch`
Wait for N threads to complete before proceeding:
```java
CountDownLatch latch = new CountDownLatch(3);

// In each of 3 threads:
latch.countDown(); // signal done

// Main thread:
latch.await(); // blocks until count reaches 0
```

### `CyclicBarrier`
Make N threads wait for each other at a common point:
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All threads ready!"));
barrier.await(); // each thread calls this; all proceed together
```

### `Semaphore`
Limit the number of threads accessing a resource:
```java
Semaphore semaphore = new Semaphore(3); // max 3 concurrent threads
semaphore.acquire();
try {
    // access limited resource
} finally {
    semaphore.release();
}
```

---

## 13. Best Practices

- Prefer `ExecutorService` over raw `Thread` creation
- Use `AtomicInteger`, `AtomicLong` for simple counters
- Use `ConcurrentHashMap` instead of synchronized `HashMap`
- Always release locks in a `finally` block
- Keep synchronized blocks as short as possible
- Avoid nested locks to prevent deadlocks
- Use `volatile` for simple shared flags only
- Prefer `CompletableFuture` for async workflows
- Never call `run()` directly — always use `start()`
- Don't use `Thread.stop()` — it's deprecated and unsafe

---

## 14. Quick Reference Cheat Sheet

```
Simple async task        → Runnable + ExecutorService
Task with return value   → Callable + Future
Async chaining/pipeline  → CompletableFuture
Shared counter           → AtomicInteger
Shared map               → ConcurrentHashMap
Limit concurrent access  → Semaphore
Wait for N threads       → CountDownLatch
Mutual exclusion         → synchronized / ReentrantLock
Simple shared flag       → volatile boolean
```

---

*Created with Claude — part of the Java World vault*
