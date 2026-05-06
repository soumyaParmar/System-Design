# Deep Dive: Concurrency & Multithreading (Executors, Locks, CompletableFuture, Virtual Threads)

## 1. Concurrency vs. Parallelism
- **Concurrency:** Managing multiple tasks at the same time (interleaving). Think of one chef multitasking between two dishes.
- **Parallelism:** Executing multiple tasks simultaneously. Think of two chefs working on two dishes in parallel.
- **In Java:** We use threads to achieve both, but the underlying OS and CPU determine how they are physically executed.

---

## 2. Why do we need it?
1. **Throughput:** Handle multiple concurrent requests (e.g., a web server).
2. **Responsiveness:** Keep the UI/Main thread free while performing background I/O.
3. **Resource Utilization:** Use all cores of modern multi-core CPUs.

---

## 3. The Core Building Blocks (The "How")

### Visibility & Atomicity
- **`volatile`:** Ensures that a variable's value is always read from and written to main memory, preventing thread-local caching issues (Visibility).
- **`Atomic` Classes (`AtomicInteger`, etc.):** Use low-level CPU instructions (CAS - Compare and Swap) to perform thread-safe operations without heavy locking.

### Synchronization & Locking
- **`synchronized`:** The simplest way to lock a block or method. It uses the "Intrinsic Lock" of an object.
- **`ReentrantLock`:** More flexible than `synchronized`. Supports fairness, interruptible locks, and try-lock (timeout).
- **`ReadWriteLock`:** Allows multiple readers but only one writer. Highly efficient for read-heavy caches.

---

## 4. Modern Concurrency: Executors & Beyond

### The Executor Framework
Don't create threads manually (`new Thread().start()`). It's expensive and hard to manage. Use **Thread Pools**.
- **FixedThreadPool:** Best for CPU-bound tasks.
- **CachedThreadPool:** Good for short-lived asynchronous tasks.
- **ScheduledThreadPool:** For periodic tasks (cron-like).

### CompletableFuture (Java 8+)
Allows you to chain asynchronous tasks in a functional style without "Callback Hell."
```java
CompletableFuture.supplyAsync(() -> fetchOrder(id))
    .thenCompose(order -> fetchUser(order.userId))
    .thenAccept(user -> sendEmail(user))
    .exceptionally(ex -> { log.error(ex); return null; });
```

---

## 5. The Future: Virtual Threads (Project Loom / Java 21+)
Traditional Java threads (Platform Threads) are wrappers around OS threads. They are **heavy** (1MB stack) and expensive to context-switch.
- **Virtual Threads:** Extremely lightweight. You can run **millions** of them on a single machine.
- **Use Case:** Perfect for I/O-bound tasks (Waiting for DB, API calls).
- **Change:** `Executors.newVirtualThreadPerTaskExecutor()`.

---

## 6. Production Grade Example: The "Double Spend" Problem
**Scenario:** Two concurrent requests try to withdraw $100 from a $150 balance at the exact same time.

**Solution (Pessimistic Locking):**
```java
public void withdraw(Long userId, BigDecimal amount) {
    lock.lock(); // ReentrantLock
    try {
        Account acc = repo.findById(userId);
        if (acc.getBalance().compareTo(amount) >= 0) {
            acc.setBalance(acc.getBalance().subtract(amount));
            repo.save(acc);
        }
    } finally {
        lock.unlock();
    }
}
```
*Note: In distributed systems, you'd use Distributed Locks (Redis/Redlock) or Optimistic Locking (@Version in JPA).*

---

## 7. Common Traps & Pitfalls

### Deadlocks
Thread A holds Lock 1 and waits for Lock 2. Thread B holds Lock 2 and waits for Lock 1. Both wait forever.
- **Prevention:** Always acquire locks in a consistent order.

### ThreadLocal Memory Leaks
`ThreadLocal` variables are tied to the Thread. In a Thread Pool, threads are reused. If you don't call `.remove()`, the data from the previous request might leak into the next request, or prevent GC from reclaiming memory.

### Race Conditions
When the outcome depends on the timing of thread execution.
- **Detection:** Use tools like `jcstress` or look for non-atomic operations on shared state.

---

## 8. Summary Checklist
- [ ] Concurrency is managing tasks; Parallelism is executing tasks.
- [ ] Use `volatile` for visibility and `Atomic` for simple thread-safe counters.
- [ ] `ReentrantLock` offers more control than `synchronized`.
- [ ] Always use `ExecutorService` instead of raw threads.
- [ ] `CompletableFuture` is for async orchestration.
- [ ] Virtual Threads (Java 21) are the new standard for scaling I/O-bound apps.
- [ ] Clean up `ThreadLocal` variables to avoid memory leaks.
