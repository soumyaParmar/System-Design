# Deep Dive: Garbage Collection (Algorithms, Tuning, G1GC, ZGC)

## 1. What is Garbage Collection (GC)?
Garbage Collection is the process by which the JVM automatically identifies and deletes unused objects from the **Heap** to reclaim memory. Unlike languages like C++, Java developers don't need to manually `free()` or `delete` memory.

### The Core Philosophy:
"An object is eligible for garbage collection if it is no longer reachable from any 'GC Root'."
*   **GC Roots** include: Local variables on the stack, static variables, active threads, and JNI references.

---

## 2. Why do we need it? (The Value Proposition)
1.  **Dangling Pointers:** Prevents errors where memory is deleted while a pointer still refers to it.
2.  **Memory Leaks:** Reduces (but doesn't eliminate) memory leaks by reclaiming unreachable memory.
3.  **Developer Productivity:** Allows developers to focus on business logic rather than manual memory management.

---

## 3. How it Works: The Generational Hypothesis
Most objects die young. The JVM organizes the Heap into generations to optimize GC performance.

### Heap Structure:
1.  **Young Generation:**
    - **Eden Space:** New objects are created here.
    - **Survivor Spaces (S0 & S1):** Objects that survive a "Minor GC" move here.
    - *Mechanism:* Fast, frequent "Minor GCs" using the **Copying Algorithm**.
2.  **Old Generation (Tenured):**
    - Objects that survive multiple Minor GCs (reached a certain "age") are "promoted" here.
    - *Mechanism:* Less frequent "Major GCs" using **Mark-Sweep-Compact**.

---

## 4. Modern Garbage Collectors (The "When & Where")

| Collector | Strategy | Best For |
| :--- | :--- | :--- |
| **Serial GC** | Single-threaded | Small apps, client-side tools. |
| **Parallel GC** | Multi-threaded | Batch processing where throughput is prioritized over latency. |
| **G1GC (Garbage First)** | Region-based | Default since Java 9. Good balance for server-side apps with large heaps (4GB+). |
| **ZGC (Z Garbage Collector)** | Concurrent, Color Pointers | Ultra-low latency (<1ms pause). Best for massive heaps (GBs to TBs). |

### Deep Dive: G1GC
- **Regions:** Divides the heap into equal-sized regions instead of contiguous chunks.
- **Garbage First:** It identifies regions with the most "garbage" and collects them first, maximizing reclaimed space with minimal effort.
- **Pause Time Goal:** You can set a target pause time: `-XX:MaxGCPauseMillis=200`.

---

## 5. Production Grade Examples & Tuning

### Scenario: High Latency in an E-commerce Checkout
**Problem:** Users experience 2-second delays during checkout. GC logs show frequent "Full GC" events.
**Analysis:** The Heap is too small, causing objects to be promoted to the Old Gen too quickly (Premature Promotion).
**Solution:**
1.  Increase Heap size: `-Xms4g -Xmx4g`.
2.  Tune Young Gen: `-XX:NewRatio=2`.
3.  Switch to G1GC: `-XX:+UseG1GC`.

### Key Tuning Flags:
- `-Xms` / `-Xmx`: Initial and maximum heap size.
- `-XX:+PrintGCDetails`: Enables detailed GC logging.
- `-XX:MaxGCPauseMillis`: Sets a target for max pause time (G1GC).

---

## 6. Common Traps & Misconceptions

### "Memory Leaks don't exist in Java"
**TRAP:** Even with GC, you can have leaks. If you store an object in a **Static HashMap** and never remove it, it stays reachable from a GC Root (the class itself) and is never collected.

### "System.gc() forces a collection"
**MISCONCEPTION:** Calling `System.gc()` is just a *hint* to the JVM. The JVM can (and often does) ignore it. In production, this can actually trigger expensive Full GCs at the wrong time.

### "ZGC is always better than G1GC"
**TRAP:** ZGC prioritizes **Latency**. This comes at the cost of **Throughput**. If your app is a background batch processor, Parallel or G1GC might actually finish the job faster than ZGC.

---

## 7. Summary Checklist
- [x] GC identifies unreachable objects starting from GC Roots.
- [x] The Heap is divided into Young (Eden, S0, S1) and Old generations.
- [x] Minor GC = Young Gen; Major/Full GC = Old Gen.
- [x] G1GC is the modern standard; ZGC is the low-latency king.
- [x] Avoid static collections and unclosed resources to prevent leaks.
