# Java Production Debugging Guide

## How to Use This Guide

Each question below is the kind of problem you will face in real production systems. I will explain:

1. **What is happening** 
2. **Why it happens** (root cause)
3. **How to debug it**
4. **How to fix it** 
5. **How to prevent it**
---

## Table of Contents

1. [OutOfMemoryError - Sudden Memory Crash](#1-outofmemoryerror---sudden-memory-crash)
2. [High CPU Usage with Low Traffic](#2-high-cpu-usage-with-low-traffic)
3. [Thread Stuck in BLOCKED State](#3-thread-stuck-in-blocked-state)
4. [Application Slows Down After Few Hours](#4-application-slows-down-after-few-hours)
5. [Frequent Garbage Collection Pauses](#5-frequent-garbage-collection-pauses)
6. [HashMap Performance Issues Under Load](#6-hashmap-performance-issues-under-load)
7. [Multiple Threads Updating Shared Data Incorrectly](#7-multiple-threads-updating-shared-data-incorrectly)
8. [API Works Locally but Fails in Production](#8-api-works-locally-but-fails-in-production)
9. [Confirming a Memory Leak](#9-confirming-a-memory-leak)
10. [Service Becomes Unresponsive Randomly](#10-service-becomes-unresponsive-randomly)
11. [Detecting and Resolving Deadlocks](#11-detecting-and-resolving-deadlocks)
12. [Inconsistent Behavior Across Requests](#12-inconsistent-behavior-across-requests)
13. [Application Crashes Without Clear Error](#13-application-crashes-without-clear-error)
14. [Slow Database Calls](#14-slow-database-calls)
15. [Thread Pool Exhausted Under Load](#15-thread-pool-exhausted-under-load)
16. [Handling High Concurrency Safely](#16-handling-high-concurrency-safely)
17. [Handling Duplicate Requests](#17-handling-duplicate-requests)
18. [Cache Giving Stale Data](#18-cache-giving-stale-data)
19. [Application Not Scaling After Adding Instances](#19-application-not-scaling-after-adding-instances)
20. [Tracing a Request Across Multiple Layers](#20-tracing-a-request-across-multiple-layers)
21. [Connection Pool Exhausted](#21-bonus---connection-pool-exhausted)
22. [SSL Handshake Failures](#22-bonus---ssl-handshake-failures)
23. [Sudden Latency Spikes](#23-bonus---sudden-latency-spikes)
24. [Slow Application Startup](#24-bonus---slow-application-startup)
25. [Race Condition Under Load](#25-bonus---race-condition-under-load)
26. [Intermittent 500 Errors](#26-bonus---intermittent-500-errors)
27. [Logs Filling Up Disk](#27-bonus---logs-filling-up-disk)
28. [Different Behavior in Different Environments](#28-bonus---different-behavior-in-different-environments)
29. [Building Observability Into a Service](#29-bonus---building-observability-into-a-service)
30. [StackOverflowError Mystery](#30-bonus---stackoverflowerror-mystery)
31. [File Descriptor Leak](#31-bonus---file-descriptor-leak)
32. [Kafka Consumer Lag Building Up](#32-bonus---kafka-consumer-lag-building-up)
33. [Random Pod Restarts in Kubernetes](#33-bonus---random-pod-restarts-in-kubernetes)
34. [Application Hangs During Shutdown](#34-bonus---application-hangs-during-shutdown)
35. [High Memory Usage but No OutOfMemoryError](#35-bonus---high-memory-usage-but-no-outofmemoryerror)

---

## 1. OutOfMemoryError - Sudden Memory Crash

### What is Happening?

Your Java application is running fine. Suddenly it throws `OutOfMemoryError` and crashes. This means the application asked for memory, but the Java Virtual Machine (the engine running your code) had no more memory to give.

### Types of OutOfMemoryError

There are different types. Each one means something different:

| Type | What It Means |
|------|---------------|
| Java heap space | The main memory area is full |
| GC overhead limit exceeded | Garbage collector is working too hard but freeing very little |
| Metaspace | Memory holding class definitions is full |
| Direct buffer memory | Off-heap memory is full |
| Unable to create new native thread | The operating system cannot create more threads |

### How to Debug Step by Step

**Step 1: Always enable heap dump on crash.**

Add these flags when starting your Java application:

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdumps/
```

This means: "When you crash, save a snapshot of memory to a file." Without this, you are debugging blind.

**Step 2: Open the heap dump file in a tool.**

Use tools like:
- Eclipse Memory Analyzer Tool (free)
- VisualVM (comes with Java)
- JProfiler (paid but powerful)

These tools show you which objects are using the most memory.

**Step 3: Look for the "leak suspect" report.**

The tool will show you which classes are holding the most memory. For example:
- "ArrayList holding 5 million User objects"
- "HashMap with 10 million entries"

That is your culprit.

**Step 4: Check garbage collection logs.**

Enable garbage collection logs:

```bash
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=10,filesize=10M
```

If you see memory keep growing and never coming down, you have a memory leak.

### How to Fix It

1. **If it is a real memory leak:** Find the code holding objects forever. Common causes:
   - Static collections that keep growing
   - Cache without size limit
   - Forgotten listeners not removed
   - ThreadLocal variables not cleaned up

2. **If memory is not enough:** Increase heap size:
   ```bash
   -Xms2g -Xmx4g
   ```
   This means start with 2 Gigabytes, grow up to 4 Gigabytes.

3. **If creating too many threads:** Use thread pools instead of creating new threads.

### How to Prevent It

- Always set memory limits in production
- Always enable heap dump on crash
- Monitor memory usage over time (not just at one moment)
- Use bounded caches (Caffeine, Guava with maxSize)
- Set up alerts when memory usage crosses 80%

---

## 2. High CPU Usage with Low Traffic

### What is Happening?

Your servers show 90% CPU usage. But you only have 10 users online. Something is burning CPU without doing useful work.

### Common Reasons

1. **Infinite loops** - Code stuck spinning forever
2. **Busy waiting** - Code checking a condition millions of times per second
3. **Inefficient algorithms** - Like sorting one million items every second
4. **Garbage collection thrashing** - Garbage collector running constantly
5. **Regular expression backtracking** - Bad regex patterns can take hours
6. **Hash collisions** - HashMap turning slow due to bad keys
7. **Logging too much** - Writing millions of log lines per second
8. **Background jobs gone wild** - Scheduled tasks running too often

### How to Debug Step by Step

**Step 1: Find the hot threads using `top` command (Linux).**

```bash
top -H -p <java_process_id>
```

This shows threads inside the Java process and their CPU usage. Note down the thread IDs using high CPU.

**Step 2: Convert thread ID to hexadecimal.**

```bash
printf "%x\n" <thread_id>
```

For example, thread ID 12345 becomes 3039 in hexadecimal.

**Step 3: Take a thread dump.**

```bash
jstack <java_process_id> > thread_dump.txt
```

**Step 4: Search the thread dump for the hexadecimal ID.**

In the file, find lines like `nid=0x3039`. The stack trace below shows what that thread was doing.

**Step 5: Use a profiler for deeper analysis.**

Tools like:
- async-profiler (very low overhead, can run in production)
- Java Flight Recorder
- VisualVM

Run a 30-second profile and check the flame graph. The widest functions in the graph are eating your CPU.

### How to Fix It

Once you find the culprit:

- **Infinite loop:** Add an exit condition
- **Busy waiting:** Use proper synchronization (wait/notify, BlockingQueue)
- **Bad algorithm:** Replace with better one (use HashMap instead of nested loop)
- **Garbage collection:** Tune memory settings or fix object creation patterns
- **Bad regex:** Rewrite the regex to avoid backtracking
- **Too much logging:** Set log level to WARN or ERROR in production

### How to Prevent It

- Always profile before going to production
- Set up CPU usage alerts (alert above 70% for 5 minutes)
- Code reviews should look for tight loops without sleep
- Test with realistic load before deploying

---

## 3. Thread Stuck in BLOCKED State

### What is Happening?

A thread in your application is in BLOCKED state. This means the thread is waiting for a lock that another thread is holding. The thread cannot do any work.

If many threads are blocked, your application slows down or stops responding.

### Visual Picture

```
   Thread A (holding lock)        Thread B (BLOCKED)
   ┌─────────────────┐            ┌─────────────────┐
   │  Working...     │            │ Waiting for     │
   │  Has the lock   │ ◄──────────│ lock from A     │
   │  Not releasing  │            │ Cannot proceed  │
   └─────────────────┘            └─────────────────┘
```

### How to Debug

**Step 1: Take a thread dump.**

```bash
jstack <process_id> > threads.txt
```

Or use:
```bash
kill -3 <process_id>
```
This sends a signal that prints thread dump to logs.

**Step 2: Look for BLOCKED threads.**

Search for `BLOCKED` in the dump file. You will see something like:

```
"Thread-5" #25 prio=5 BLOCKED
   waiting to lock <0x000000076b2f3c40>
   at com.example.Service.process(Service.java:42)
```

**Step 3: Find who is holding the lock.**

Search for the lock address `0x000000076b2f3c40`. You will find:

```
"Thread-3" #20 prio=5 RUNNABLE
   - locked <0x000000076b2f3c40>
   at com.example.Service.compute(Service.java:78)
```

Thread-3 is the holder. Now investigate why Thread-3 is not releasing.

**Step 4: Check what Thread-3 is doing.**

Common patterns:
- It is doing slow input/output (database call, network call)
- It is also blocked on another lock (potential deadlock)
- It is doing very heavy computation while holding the lock

### How to Fix

1. **Reduce lock scope.** Don't hold a lock while doing slow operations:

   **Bad:**
   ```java
   synchronized(lock) {
       data = database.fetch();  // Slow! Lock held the whole time
       process(data);
   }
   ```

   **Good:**
   ```java
   data = database.fetch();  // Fetch outside the lock
   synchronized(lock) {
       process(data);  // Lock only for processing
   }
   ```

2. **Use ReadWriteLock** when many threads only read but few threads write.

3. **Use lock-free data structures** like `ConcurrentHashMap`, `AtomicInteger`.

4. **Use `tryLock` with timeout** so threads don't wait forever:
   ```java
   if (lock.tryLock(5, TimeUnit.SECONDS)) {
       try { /* work */ } finally { lock.unlock(); }
   }
   ```

### How to Prevent

- Keep critical sections (code inside locks) very short
- Never call slow operations while holding a lock
- Use modern concurrent collections instead of synchronized blocks
- Test with high concurrency before production

---

## 4. Application Slows Down After Few Hours

### What is Happening?

You start the application. It runs fast. After 4 hours, requests take 5 seconds instead of 100 milliseconds. After 8 hours, requests time out completely.

This is a classic sign of a **resource leak** or **growing data structure**.

### Common Reasons

1. **Memory leak** - Objects accumulating, garbage collector struggling
2. **Connection leak** - Database connections not returned to pool
3. **File handle leak** - Files opened but never closed
4. **Thread leak** - Threads created but never finished
5. **Cache growing without limit** - Cache eats all memory
6. **Logs filling disk** - No log rotation
7. **Background jobs piling up** - Queue growing because consumers cannot keep up

### How to Debug

**Step 1: Check memory trends over time.**

Look at memory graphs from monitoring tools (Grafana, Datadog, CloudWatch). If memory keeps going up and never comes down, that's a leak.

```
Healthy:    ▁▂▃▂▁▂▃▂▁▂▃▂▁  (saw-tooth pattern)
Leaking:    ▁▂▃▄▅▆▇█████▶  (constantly growing)
```

**Step 2: Check thread count over time.**

```bash
jstack <pid> | grep "^\"" | wc -l
```

Run this every hour. If thread count keeps growing, you have a thread leak.

**Step 3: Check open file descriptors.**

```bash
ls /proc/<pid>/fd | wc -l
```

If this number grows over time, you have file or connection leaks.

**Step 4: Check database connection pool stats.**

Most pools (HikariCP, Tomcat) expose metrics. Look for:
- Active connections (should not stay at maximum)
- Pending requests (should not grow)
- Connection leaks logged in warnings

**Step 5: Take heap dumps at different times.**

Take a heap dump after 1 hour. Take another after 4 hours. Compare them. The classes that grew the most are your suspects.

### How to Fix

- **Memory leak:** Find the growing collection. Add size limits or cleanup.
- **Connection leak:** Always use try-with-resources:
  ```java
  try (Connection conn = dataSource.getConnection()) {
      // use connection
  } // automatically closed
  ```
- **Thread leak:** Use thread pools, never `new Thread().start()` repeatedly.
- **Cache growing:** Use bounded cache with eviction (Caffeine, Guava).
- **Log rotation:** Configure log rotation in logback or log4j.

### How to Prevent

- Run long-duration tests before production (24+ hours)
- Use try-with-resources for everything closeable
- Always set bounds on collections, caches, and pools
- Monitor everything that can grow

---

## 5. Frequent Garbage Collection Pauses

### What is Happening?

Garbage collection is when the Java Virtual Machine cleans up unused objects. During cleanup, your application can pause briefly. If pauses are too frequent or too long, requests slow down.

### Visual Picture

```
   Healthy garbage collection:
   ──work────GC─────work──────GC─────work─────
              (fast)            (fast)

   Bad garbage collection:
   ──work──GC GC GC GC GC GC GC GC GC GC GC──
           (constant pauses, app barely runs)
```

### How to Debug

**Step 1: Enable garbage collection logs.**

```bash
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=10,filesize=10M
```

**Step 2: Analyze the logs.**

Use a tool like GCEasy or GCViewer (both free). Upload your log. Look for:

- **Pause time** - How long does each pause last?
- **Pause frequency** - How often do pauses happen?
- **Throughput** - What percent of time is spent on real work vs garbage collection?

**Healthy numbers:**
- Pause time: under 200 milliseconds
- Throughput: above 95%

**Bad numbers:**
- Pause time: over 1 second
- Throughput: below 90%

**Step 3: Identify the type of pause.**

- **Young generation pause (Minor GC):** Should be fast (under 50 ms)
- **Old generation pause (Major GC or Full GC):** Often slow (seconds)

If you see frequent Full GC, your old generation is full or getting full quickly.

### How to Fix

**1. Choose the right garbage collector:**

| Collector | When to Use |
|-----------|-------------|
| G1GC | Default for most apps (Java 9+) |
| ZGC | Low pause time needed (under 10 ms) |
| Shenandoah | Low pause time, alternative to ZGC |
| Parallel GC | Throughput more important than pause time |

For modern services, use:
```bash
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
```

**2. Increase heap size** if memory is too tight:
```bash
-Xms4g -Xmx8g
```

**3. Reduce object creation.** Reuse objects when possible. Use object pools for expensive objects.

**4. Don't create huge short-lived objects.** They go directly to old generation and trigger Full GC.

**5. Tune young generation size:**
```bash
-Xmn2g  # 2 GB young generation
```

### How to Prevent

- Profile your application's allocation rate
- Pre-size collections (`new ArrayList<>(1000)` instead of default)
- Avoid creating objects in hot loops
- Use primitives instead of wrapper classes when possible

---

## 6. HashMap Performance Issues Under Load

### What is Happening?

Your code uses HashMap. Under low load, it works fast. Under heavy load, operations that should be instant take seconds. The application slows down dramatically.

### Why It Happens

HashMap has two main problems under load:

**Problem 1: It is not thread-safe.**

If multiple threads modify a HashMap at the same time, bad things happen:
- Lost updates (you put 100 items, only 80 stay)
- Infinite loops (Java 7 had a famous bug where HashMap operations went into infinite loop)
- Corrupted internal structure
- ConcurrentModificationException when iterating

**Problem 2: Hash collisions.**

Each item in HashMap goes into a "bucket" based on its hash code. If many items go into the same bucket, lookups become slow.

```
Good HashMap:                  Bad HashMap (collisions):
Bucket 0: [item1]              Bucket 0: [item1, item2, item3, item4, item5...]
Bucket 1: [item2]              Bucket 1: []
Bucket 2: [item3]              Bucket 2: []
...                            ...

Lookup is O(1) - constant      Lookup is O(N) - slow
```

### How to Debug

**Step 1: Take thread dumps during slowdown.**

Look for many threads stuck in HashMap methods like `HashMap.put` or `HashMap.get`.

**Step 2: Check for `ConcurrentModificationException` in logs.**

If you see this, multiple threads are using the same HashMap unsafely.

**Step 3: Profile the application.**

Look at CPU usage per method. If `HashMap.get` shows high CPU, you have a problem.

**Step 4: Check your custom hashCode and equals methods.**

If you use custom objects as HashMap keys, bad hashCode causes collisions.

### How to Fix

**1. For multi-threaded access, use ConcurrentHashMap:**

```java
// Bad - not thread safe
Map<String, User> users = new HashMap<>();

// Good - thread safe and fast
Map<String, User> users = new ConcurrentHashMap<>();
```

ConcurrentHashMap allows multiple threads to read and write at the same time without locking the whole map.

**2. Pre-size your HashMap if you know the size:**

```java
// If you expect 10,000 items
Map<String, User> users = new HashMap<>(16384);
```

This avoids expensive resizing as the map grows.

**3. Fix bad hashCode methods:**

```java
// Bad - all objects hash to same bucket
@Override
public int hashCode() { return 1; }

// Good - distributes objects across buckets
@Override
public int hashCode() {
    return Objects.hash(id, name, email);
}
```

**4. Always implement equals when you implement hashCode** and vice versa.

### How to Prevent

- Default to ConcurrentHashMap in multi-threaded code
- Always pre-size collections when you know the expected size
- Test custom objects with proper unit tests for hashCode and equals
- Run load tests with concurrent access

---

## 7. Multiple Threads Updating Shared Data Incorrectly

### What is Happening?

You have a counter. Two threads both increment it. You expect 200, but you get 197. Numbers are missing. This is called a **race condition**.

### Why It Happens

The operation `counter++` looks simple but is actually three steps:
1. Read counter value
2. Add 1 to the value
3. Write the new value back

If two threads run these steps at the same time, they can read the same old value and overwrite each other.

### Visual Picture

```
   Thread A             counter = 100         Thread B
   ─────────                                  ─────────
   Read: 100                                  Read: 100
   Add 1: 101                                 Add 1: 101
   Write: 101         ─────►  100             Write: 101
                                              (B overwrites A)

   Result: counter = 101 (should be 102!)
```

### How to Fix

**Solution 1: Use synchronized**

```java
private int counter = 0;

public synchronized void increment() {
    counter++;
}
```

This makes only one thread enter the method at a time. Simple but can be slow.

**Solution 2: Use AtomicInteger (better for simple cases)**

```java
private AtomicInteger counter = new AtomicInteger(0);

public void increment() {
    counter.incrementAndGet();
}
```

This uses special CPU instructions for atomic operations. Faster than synchronized.

**Solution 3: Use ReentrantLock for complex cases**

```java
private final ReentrantLock lock = new ReentrantLock();
private int counter = 0;

public void increment() {
    lock.lock();
    try {
        counter++;
    } finally {
        lock.unlock();
    }
}
```

More flexible than synchronized. You can try to lock with timeout.

**Solution 4: Use immutable objects**

If the data never changes after creation, no thread can corrupt it:

```java
public final class User {
    private final String name;
    private final int age;
    // Only getters, no setters
}
```

**Solution 5: Use thread-safe collections**

```java
// Instead of:
List<String> list = new ArrayList<>();

// Use:
List<String> list = new CopyOnWriteArrayList<>();
// or
List<String> list = Collections.synchronizedList(new ArrayList<>());
```

### Quick Reference - Which Tool to Use

| Situation | Use This |
|-----------|----------|
| Simple counter | AtomicInteger / AtomicLong |
| Single boolean flag | AtomicBoolean / volatile |
| Map with concurrent access | ConcurrentHashMap |
| List with mostly reads | CopyOnWriteArrayList |
| Complex updates | ReentrantLock or synchronized |
| Read-heavy data | ReadWriteLock |
| Producer-consumer | BlockingQueue |

### How to Prevent

- Default to immutable objects
- Use thread-safe collections from `java.util.concurrent`
- Avoid sharing state between threads when possible
- Use proper synchronization tools, not your own
- Test with multiple threads (use stress tests)

---

## 8. API Works Locally but Fails in Production

### What is Happening?

On your laptop, everything is perfect. You deploy to production. The API returns errors, times out, or behaves strangely. This is the most common interview question for a reason.

### Common Reasons

1. **Different configuration** - URLs, credentials, feature flags differ
2. **Different data** - Production has 10 million records, you tested with 10
3. **Different network** - Production has firewalls, latency, DNS issues
4. **Different scale** - Production has 1000 concurrent users, you tested with 1
5. **Different versions** - Library or Java version mismatch
6. **Missing dependencies** - External services in production are different
7. **Resource limits** - Production containers have memory/CPU limits
8. **Time zone issues** - Servers in different time zones
9. **Character encoding** - Default encoding differs
10. **File path differences** - Linux vs Windows paths

### How to Debug

**Step 1: Compare configurations.**

Check both environments for:
- Database URLs and credentials
- External service URLs
- Feature flags
- Environment variables
- Java version
- Library versions

Use a config diff tool. Don't trust your memory.

**Step 2: Check the logs in production.**

Read carefully:
- The exact error message
- The full stack trace
- Logs just before the error
- Correlate with metrics (CPU, memory, network at that moment)

**Step 3: Try to reproduce locally with production-like setup.**

- Use production-size data
- Use production configuration (but with safe credentials)
- Use Docker to match container limits
- Add network latency to simulate production

**Step 4: Check production-specific things:**

- Network policies (firewalls blocking calls?)
- DNS resolution (can production resolve the host?)
- Certificates (SSL issues?)
- Permissions (can the user read the file?)
- Disk space (is disk full?)
- Memory limits (is container OOMKilled?)

**Step 5: Add more logging temporarily.**

Sometimes you need to add detailed logs to understand. Add them, deploy, gather data, then remove.

### Common Fixes

- **Configuration management:** Use a single source of truth (Vault, Config Service)
- **Externalize all configs:** No hardcoded values
- **Match environments:** Use containers to make them identical
- **Production data sample:** Test with real-size data in staging
- **Health checks:** Add detailed health endpoints showing dependencies

### How to Prevent

- Use a staging environment that mirrors production
- Run integration tests against production-like data
- Use feature flags to test in production safely
- Use canary deployments (release to 1% of traffic first)
- Monitor everything from day one

---

## 9. Confirming a Memory Leak

### What is Happening?

You suspect memory leak. But "suspect" is not enough. You need proof. A memory leak is when objects are kept in memory even though they are no longer needed.

### How to Confirm

**Step 1: Look at memory pattern over time.**

Healthy application memory looks like a saw-tooth pattern:
```
Memory Used
   │
   │      ╱╲      ╱╲      ╱╲
   │    ╱   ╲   ╱   ╲   ╱   ╲
   │  ╱      ╲╱     ╲╱     ╲
   │╱
   └────────────────────────► Time
   
   (Goes up, then GC cleans up, repeats)
```

Leaking application memory looks like a staircase:
```
Memory Used
   │              ___________
   │         ____╱
   │    ____╱
   │___╱
   │
   └────────────────────────► Time
   
   (Always going up, never comes down fully)
```

**Step 2: Force a garbage collection and check.**

If you trigger garbage collection and memory does not drop significantly, the objects are referenced somewhere.

```bash
jcmd <pid> GC.run
```

**Step 3: Take heap dumps at intervals.**

Take three heap dumps:
1. After application starts (baseline)
2. After 1 hour
3. After 4 hours

Open them in Eclipse Memory Analyzer. Compare the three dumps.

**Step 4: Look at the "Histogram" view.**

This shows you how many of each type of object exist. Compare the counts:
- If `User` objects went from 100 to 10,000 to 100,000, you have a leak.

**Step 5: Use "Path to GC Roots".**

This is the most powerful feature. It tells you what is keeping an object alive.

For example, it might show:
```
User object → held by → ArrayList → held by → Static field → kept alive forever
```

Now you know the static field is the leak source.

**Step 6: Use leak detection tools.**

- Eclipse Memory Analyzer has "Leak Suspects Report"
- JProfiler has built-in leak detection
- IntelliJ has memory profiler

### Common Memory Leak Causes

1. **Static collections that grow**
   ```java
   private static List<User> ALL_USERS = new ArrayList<>();
   // Never cleaned up!
   ```

2. **Listeners not removed**
   ```java
   button.addListener(myListener);
   // myListener is never removed
   ```

3. **ThreadLocal not cleaned**
   ```java
   threadLocal.set(largeObject);
   // Forgot to call threadLocal.remove()
   ```

4. **Cache without size limit**
   ```java
   Map<String, Data> cache = new HashMap<>();
   // Grows forever
   ```

5. **Inner classes holding outer reference**
   ```java
   public class Outer {
       class Inner { /* holds reference to Outer */ }
   }
   ```

### How to Fix

- Remove static collection growth
- Always remove listeners when done
- Always call `threadLocal.remove()` in finally block
- Use bounded cache (Caffeine with maxSize)
- Use static inner classes when possible
- Use weak references for caches

### How to Prevent

- Code reviews should look for static collections
- Use bounded data structures
- Profile during development
- Run long-duration tests

---

## 10. Service Becomes Unresponsive Randomly

### What is Happening?

Your service runs fine. Suddenly it does not respond. After some time, it works again. Or you have to restart it. This is one of the trickiest problems.

### Common Causes

1. **All threads blocked** - Thread pool exhausted
2. **Garbage collection thrashing** - Long GC pauses
3. **Database connection pool exhausted** - All connections used
4. **External service slow** - Waiting for downstream
5. **Memory pressure** - Almost out of memory, slow operations
6. **CPU throttling** - Container hitting CPU limit
7. **Network issues** - Packet loss, DNS issues
8. **Disk full or slow** - Cannot write logs or data
9. **Lock contention** - Many threads waiting for same lock
10. **Long-running query** - One query blocks the whole service

### How to Debug

**Step 1: Set up continuous monitoring.**

You need data from when the freeze happens. Set up:
- Continuous heap dump capability
- Thread dump every minute during issues
- Detailed metrics (CPU, memory, threads, GC)

**Step 2: Trigger a thread dump during freeze.**

```bash
jstack -l <pid> > freeze_threads.txt
```

Look at:
- How many threads are RUNNABLE? Should be similar to CPU count
- How many are BLOCKED? Why?
- How many are WAITING? On what?
- Are many threads waiting for the same resource?

**Step 3: Check garbage collection.**

If garbage collection is taking 30 seconds, the service appears frozen.

```bash
jstat -gcutil <pid> 1000
```

This shows GC stats every second.

**Step 4: Check external dependencies.**

```bash
# Check database
SELECT * FROM pg_stat_activity WHERE state = 'active';

# Check network
netstat -an | grep ESTABLISHED | wc -l

# Check disk
df -h
iostat -x 1
```

**Step 5: Check container/system limits.**

```bash
# CPU throttling
cat /sys/fs/cgroup/cpu/cpu.stat

# Memory pressure
dmesg | grep -i "memory"

# Open files
ulimit -n
lsof -p <pid> | wc -l
```

### How to Fix

**For thread pool exhaustion:**
- Increase pool size carefully
- Add timeouts so threads don't wait forever
- Use circuit breakers (Resilience4j) for external calls

**For GC thrashing:**
- Increase heap size
- Tune garbage collector
- Find and fix object creation patterns

**For database issues:**
- Add connection timeout
- Use connection pool monitoring
- Kill long-running queries

**For external service issues:**
- Add timeouts everywhere (default is often forever!)
- Implement circuit breakers
- Use bulkheads (separate pools for different services)

### How to Prevent

- Set timeouts on EVERYTHING (HTTP calls, database calls, locks)
- Use circuit breakers for all external calls
- Monitor everything continuously
- Have automated thread dump capture on slowness

---

## 11. Detecting and Resolving Deadlocks

### What is Happening?

Two or more threads are waiting for each other forever. Nothing happens. The threads are alive but cannot make progress.

### Visual Picture

```
   Thread A               Thread B
   ┌────────┐             ┌────────┐
   │Has Lock│             │Has Lock│
   │   1    │             │   2    │
   │        │             │        │
   │Wants   │ ◄────────── │Wants   │
   │Lock 2  │ ──────────► │Lock 1  │
   └────────┘             └────────┘

   Both wait forever - DEADLOCK!
```

### How to Detect

**Method 1: Thread dump shows it directly.**

```bash
jstack <pid>
```

If there is a deadlock, you will see:
```
Found one Java-level deadlock:
==============================
"Thread-1":
  waiting to lock monitor 0x...,
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x...,
  which is held by "Thread-1"
```

The Java Virtual Machine actually detects deadlocks for you!

**Method 2: Use jconsole.**

Connect jconsole to your Java process. Click "Threads" tab. Click "Detect Deadlock" button.

**Method 3: Programmatically.**

```java
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] threadIds = bean.findDeadlockedThreads();
if (threadIds != null) {
    // Deadlock detected!
    ThreadInfo[] threads = bean.getThreadInfo(threadIds);
    // Log details
}
```

You can run this check periodically.

### How to Resolve a Live Deadlock

In production, when a deadlock is happening right now:
- You usually have to **restart the service**
- There is no clean way to break a deadlock at runtime
- Java cannot interrupt threads holding monitor locks

This is why prevention is everything.

### How to Prevent

**Rule 1: Always acquire locks in the same order.**

```java
// Bad - different order in different threads
// Thread A: lock1 then lock2
// Thread B: lock2 then lock1

// Good - always same order
// Both threads: lock1 then lock2
```

**Rule 2: Use tryLock with timeout.**

```java
if (lock1.tryLock(5, TimeUnit.SECONDS)) {
    try {
        if (lock2.tryLock(5, TimeUnit.SECONDS)) {
            try {
                // do work
            } finally {
                lock2.unlock();
            }
        }
    } finally {
        lock1.unlock();
    }
}
```

If you can't get the lock, you don't deadlock - you just fail.

**Rule 3: Hold locks for the shortest time possible.**

Don't do slow operations while holding a lock.

**Rule 4: Use higher-level concurrency tools.**

Often you don't need explicit locks. Use:
- ConcurrentHashMap
- BlockingQueue
- ExecutorService
- CompletableFuture

These are designed to avoid deadlocks.

**Rule 5: Avoid nested locks.**

If you must use multiple locks, document the order clearly.

### How to Detect During Development

- Code reviews specifically looking at locking
- Use stress tests with random ordering
- Use static analysis tools (SpotBugs, ErrorProne)
- Run with `-XX:+PrintConcurrentLocks` flag

---

## 12. Inconsistent Behavior Across Requests

### What is Happening?

Same request gives different results. Sometimes it works, sometimes it fails. Sometimes returns A, sometimes returns B. This is a debugging nightmare.

### Common Causes

1. **Shared mutable state** - Multiple threads see different values
2. **Caching issues** - Some requests hit cache, some don't
3. **Different servers** - Load balancer routes to different instances with different state
4. **Race conditions** - Timing-dependent behavior
5. **Time-dependent logic** - Behaves differently based on time of day
6. **External service flakiness** - Downstream service inconsistent
7. **Database replication lag** - Reads from replica with stale data
8. **Random elements in code** - Math.random, UUID
9. **Order-dependent operations** - Order of HashMap iteration
10. **Different user data** - Edge cases in some user data

### How to Debug

**Step 1: Add request tracing.**

Give every request a unique ID. Log this ID at every step.

```java
String requestId = UUID.randomUUID().toString();
MDC.put("requestId", requestId);
log.info("Processing request");
```

Now you can trace one request across all logs.

**Step 2: Identify the pattern.**

Is it:
- Random (race condition?)
- Time-based (caching, scheduled job?)
- Server-based (specific instance has problem?)
- User-based (specific data triggers bug?)

**Step 3: Compare working vs failing requests.**

Find one that worked. Find one that failed. Compare:
- Input data
- Server they hit
- Time of day
- User account
- All logs from start to end

**Step 4: Check for shared state.**

Look for:
- Static variables that change
- Singleton objects with mutable state
- Thread-local variables not cleaned
- Caches with bad invalidation

**Step 5: Reproduce in controlled environment.**

If it happens 1 in 100 times, run 1000 requests. Capture failures with full context.

### Common Fixes

**Race condition fix:**
- Use proper synchronization
- Make objects immutable
- Use thread-safe collections

**Caching fix:**
- Use consistent cache key strategy
- Set proper time-to-live
- Implement cache invalidation correctly

**Replication lag fix:**
- Use primary database for writes and immediate reads
- Use sticky sessions
- Read your own writes pattern

**Multiple instance fix:**
- Make services stateless
- Store all state in shared database/cache
- Use distributed cache (Redis) instead of local cache

### How to Prevent

- Make services stateless when possible
- Use distributed tracing from day one
- Add correlation IDs to every request
- Test with realistic concurrent load
- Use chaos engineering (random failures)

---

## 13. Application Crashes Without Clear Error

### What is Happening?

Your application just disappears. No exception in logs. No clear error. The Java process is gone.

### Common Causes

1. **Out of memory killer (OOM Killer)** - Operating system killed the process
2. **Native memory crash** - Bug in native code (JNI, native libraries)
3. **JVM bug** - Rare but happens
4. **System.exit called** - Some code called exit
5. **Container OOMKilled** - Kubernetes killed the pod
6. **Disk full** - Cannot write log file with the error
7. **Stack overflow** - Infinite recursion
8. **Signal received** - SIGKILL, SIGTERM from outside

### How to Debug

**Step 1: Check operating system logs.**

```bash
# Linux
dmesg | grep -i "killed\|out of memory"
journalctl -u your-service

# Look for OOM killer:
# "Out of memory: Kill process 1234 (java)"
```

**Step 2: Check container/orchestrator logs.**

```bash
# Kubernetes
kubectl describe pod <pod-name>
# Look for "OOMKilled" or "Error" in events

kubectl logs <pod-name> --previous
# Logs from before the crash
```

**Step 3: Look for hs_err_pid files.**

When the JVM crashes, it writes `hs_err_pid<pid>.log`:

```bash
find / -name "hs_err_pid*.log" 2>/dev/null
```

This file contains:
- The signal that caused the crash
- The Java thread that was running
- The native stack trace
- All loaded libraries

**Step 4: Check disk space.**

```bash
df -h
```

If disk is full, logs cannot be written, application may crash.

**Step 5: Enable core dumps.**

```bash
ulimit -c unlimited
echo "/tmp/core.%p" > /proc/sys/kernel/core_pattern
```

After crash, analyze with `gdb`.

**Step 6: Add JVM crash flags.**

```bash
-XX:+CrashOnOutOfMemoryError
-XX:ErrorFile=/var/log/jvm_error_%p.log
-XX:OnError="kill -3 %p"
-XX:OnOutOfMemoryError="echo Out of memory at %d"
```

### Common Fixes

- **OOM killer:** Reduce memory usage or increase container memory
- **Native crash:** Update native libraries, check JNI code
- **Disk full:** Set up log rotation, monitor disk space
- **Stack overflow:** Find recursive code with no exit condition
- **Signals:** Check what process is sending signals

### How to Prevent

- Always run with crash detection flags enabled
- Set proper memory limits
- Monitor disk space
- Set up alerts for process restart
- Use systemd or container restart policies (with limits to prevent crash loops)

---

## 14. Slow Database Calls

### What is Happening?

A page that should load in 200 milliseconds takes 5 seconds. The slow part is database queries. Users complain about slowness.

### Common Causes

1. **Missing index** - Query scans entire table
2. **N+1 query problem** - 1 query becomes 1000 queries
3. **Bad query plan** - Database chose a slow path
4. **Lock contention** - Queries waiting for locks
5. **Network latency** - Database is far from application
6. **Connection pool issues** - Waiting for connection
7. **Large result set** - Returning too much data
8. **No query result caching** - Same query runs many times
9. **Bad data model** - Too many joins, denormalization needed
10. **Statistics outdated** - Database optimizer making bad choices

### How to Debug

**Step 1: Find the slow queries.**

Most databases have slow query logs:

```sql
-- PostgreSQL
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- MySQL
SELECT * FROM mysql.slow_log;
```

**Step 2: Use EXPLAIN to analyze the query.**

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

Look for:
- "Seq Scan" - bad, means full table scan
- "Index Scan" - good, using an index
- High "rows" estimates - reading too much data

**Step 3: Check for N+1 problem.**

Look at logs. Do you see the same query pattern repeated many times?

```
SELECT * FROM users WHERE id = 1
SELECT * FROM orders WHERE user_id = 1
SELECT * FROM users WHERE id = 2
SELECT * FROM orders WHERE user_id = 2
... (1000 more times)
```

This is N+1. Fix with JOIN or batch queries.

**Step 4: Check connection pool stats.**

If connections are exhausted, queries wait in queue:

```
HikariPool stats:
- Active: 50/50 (FULL!)
- Pending: 23 (waiting)
```

**Step 5: Check database server health.**

- CPU usage
- Memory usage
- Disk I/O
- Active connections
- Lock waits

### How to Fix

**Add indexes:**
```sql
CREATE INDEX idx_users_email ON users(email);
```

**Fix N+1 with JOIN:**
```java
// Bad - N+1
List<User> users = userRepo.findAll();
for (User u : users) {
    u.getOrders(); // Triggers query for each user
}

// Good - one query
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

**Use batch operations:**
```java
// Bad
for (User u : users) {
    userRepo.save(u);  // 1000 queries
}

// Good
userRepo.saveAll(users);  // 1 batch query
```

**Add caching for read-heavy data:**
```java
@Cacheable("users")
public User findById(Long id) {
    return userRepo.findById(id);
}
```

**Limit result size:**
```java
// Bad
List<Order> orders = orderRepo.findAll();  // 1 million rows

// Good
Page<Order> orders = orderRepo.findAll(PageRequest.of(0, 100));
```

**Tune connection pool:**
```java
hikari.maximum-pool-size=20
hikari.connection-timeout=5000
hikari.idle-timeout=300000
```

### How to Prevent

- Always EXPLAIN new queries
- Set up slow query monitoring
- Use database performance tools (pgAdmin, MySQL Workbench)
- Load test before production
- Profile in staging with production-size data

---

## 15. Thread Pool Exhausted Under Load

### What is Happening?

Your application uses a thread pool. Under load, all threads are busy. New requests wait in a queue. Eventually, queue is full, requests are rejected.

### Visual Picture

```
   Thread Pool (size 10)             Wait Queue (size 100)
   ┌─────────────────────┐           ┌──────────────────┐
   │ T1 [Slow API call]  │           │ R11, R12, R13... │
   │ T2 [Slow API call]  │  ◄───────  │ R20, R21... R110 │ ◄─ New request rejected!
   │ T3 [Slow API call]  │           │     FULL         │
   │ ...                 │           │                  │
   │ T10[Slow API call]  │           │                  │
   └─────────────────────┘           └──────────────────┘

   All threads busy, queue full → System overloaded
```

### Common Causes

1. **Slow downstream calls** - Threads stuck waiting
2. **Too small pool size** - Pool cannot handle load
3. **Long-running tasks** - Tasks taking forever
4. **Deadlocks in pool** - Tasks waiting for each other
5. **Memory pressure causing slowdowns**

### How to Debug

**Step 1: Check thread pool metrics.**

Most thread pools expose metrics. Check:
- Active thread count
- Queue size
- Tasks completed
- Tasks rejected

**Step 2: Take thread dump.**

What are the threads doing? Are they:
- All in network read (slow downstream)?
- All in database call (database slow)?
- All in synchronization (lock contention)?
- All in computation (need more threads)?

**Step 3: Identify slow operations.**

Profile to find which operations are slow.

### How to Fix

**Solution 1: Set proper pool size.**

For CPU-bound tasks:
```
poolSize = numberOfCores
```

For I/O-bound tasks (waiting for network/disk):
```
poolSize = numberOfCores × (1 + waitTime/computeTime)
```

For example, if a task waits 90ms and computes 10ms on a 4-core machine:
```
poolSize = 4 × (1 + 90/10) = 40 threads
```

**Solution 2: Add timeouts to all blocking calls.**

```java
// Bad - waits forever
restTemplate.getForObject(url, String.class);

// Good - times out
RestTemplate restTemplate = new RestTemplate();
ClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
((SimpleClientHttpRequestFactory) factory).setConnectTimeout(2000);
((SimpleClientHttpRequestFactory) factory).setReadTimeout(5000);
```

**Solution 3: Use bulkheads.**

Have separate pools for different downstream services. If one service is slow, others are not affected.

```java
// Pool for database calls
ExecutorService dbPool = Executors.newFixedThreadPool(20);

// Pool for external API calls  
ExecutorService apiPool = Executors.newFixedThreadPool(10);

// Pool for heavy computation
ExecutorService computePool = Executors.newFixedThreadPool(4);
```

**Solution 4: Use async / non-blocking I/O.**

With reactive frameworks (Project Reactor, WebFlux), one thread can handle thousands of requests because it doesn't wait.

**Solution 5: Implement circuit breakers.**

If downstream is slow, fail fast instead of waiting:

```java
@CircuitBreaker(name = "myService", fallbackMethod = "fallback")
public String callService() {
    return restTemplate.getForObject(url, String.class);
}

public String fallback(Exception e) {
    return "Default response";
}
```

**Solution 6: Use proper rejection policy.**

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                              // core size
    20,                              // max size
    60L, TimeUnit.SECONDS,           // keep alive
    new LinkedBlockingQueue<>(100),  // queue
    new ThreadPoolExecutor.CallerRunsPolicy()  // when full, caller does the work
);
```

### How to Prevent

- Always set timeouts on all blocking operations
- Use bulkheads to isolate failures
- Monitor thread pool metrics
- Load test to find the right pool sizes
- Use circuit breakers for external services

---

## 16. Handling High Concurrency Safely

### What is Happening?

Your application needs to handle many requests at the same time. You need to share data between requests. You need to do this safely without bugs.

### The Tools You Have

**1. Synchronized keyword (oldest, simplest)**

```java
public synchronized void update() {
    counter++;
}
```

Use when: simple cases, low contention.

**2. ReentrantLock (more flexible)**

```java
private final ReentrantLock lock = new ReentrantLock();

public void update() {
    lock.lock();
    try {
        counter++;
    } finally {
        lock.unlock();
    }
}
```

Use when: need timeout, need to interrupt, need fairness.

**3. ReadWriteLock (many readers, few writers)**

```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public String read() {
    rwLock.readLock().lock();
    try {
        return data;
    } finally {
        rwLock.readLock().unlock();
    }
}

public void write(String value) {
    rwLock.writeLock().lock();
    try {
        data = value;
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

Use when: read-heavy workload (many threads read, few write).

**4. Atomic classes (lock-free)**

```java
private final AtomicInteger counter = new AtomicInteger(0);
public void increment() {
    counter.incrementAndGet();
}

private final AtomicReference<User> currentUser = new AtomicReference<>();
public void update(User newUser) {
    currentUser.set(newUser);
}
```

Use when: simple updates on single variables.

**5. Concurrent Collections**

```java
// Thread-safe map - allows concurrent reads and writes
ConcurrentHashMap<String, User> users = new ConcurrentHashMap<>();

// Thread-safe queue - producer-consumer
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

// Thread-safe list - good for read-heavy workloads
CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();
```

**6. Immutable objects (best when possible)**

```java
public final class User {
    private final String name;
    private final int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    
    // To "modify", create a new object
    public User withAge(int newAge) {
        return new User(this.name, newAge);
    }
}
```

Use whenever possible. No locks needed!

**7. ThreadLocal (per-thread storage)**

```java
private static final ThreadLocal<DateFormat> formatter = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String format(Date date) {
    return formatter.get().format(date);
}
```

Use when: each thread needs its own copy of an object (like SimpleDateFormat which is not thread-safe).

**Important:** Always call `formatter.remove()` when done to prevent memory leaks!

**8. CompletableFuture (async programming)**

```java
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> 
    userService.fetchUser(id)
);

CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(() -> 
    orderService.fetchOrders(id)
);

// Combine both when ready
userFuture.thenCombine(ordersFuture, (user, orders) -> {
    return new UserDto(user, orders);
}).thenAccept(dto -> {
    sendResponse(dto);
});
```

Use when: chaining async operations.

### Decision Guide

| Need | Use |
|------|-----|
| Increment a counter | AtomicInteger |
| Boolean flag | AtomicBoolean / volatile |
| Replace an object | AtomicReference |
| Map with concurrent access | ConcurrentHashMap |
| Producer-consumer pattern | BlockingQueue |
| Read-heavy data | CopyOnWriteArrayList or ReadWriteLock |
| Complex updates | ReentrantLock |
| Simple critical section | synchronized |
| Async operations | CompletableFuture |
| Per-thread data | ThreadLocal |
| Don't share if possible | Immutable objects |

### Best Practices

1. **Prefer immutable objects** - No bugs possible
2. **Prefer concurrent collections over manual synchronization**
3. **Keep critical sections small** - Hold locks briefly
4. **Don't lock during slow operations** - Especially I/O
5. **Use proper tool for the job** - Don't use synchronized for everything
6. **Test under high concurrency** - Use stress tests

---

## 17. Handling Duplicate Requests

### What is Happening?

The same request comes to your service multiple times. The user clicked twice. The network retried. The mobile app sent twice. You should not process it twice (don't charge the user twice!).

This concept is called **idempotency**. An operation is idempotent if doing it multiple times has the same effect as doing it once.

### Examples

**Not idempotent (bad):**
- "Increase counter by 1" - 5 calls = +5
- "Charge $100" - 5 calls = $500 charged

**Idempotent (good):**
- "Set counter to 5" - 5 calls = counter is 5
- "Charge order with ID 12345" - 5 calls = $100 charged once

### How to Implement Idempotency

**Solution 1: Idempotency Key (most common)**

The client sends a unique key with each request. The server tracks processed keys.

```java
@PostMapping("/orders")
public Order createOrder(
    @RequestHeader("Idempotency-Key") String key,
    @RequestBody OrderRequest request) {
    
    // Check if already processed
    Order existing = orderRepo.findByIdempotencyKey(key);
    if (existing != null) {
        return existing;  // Return previous result
    }
    
    // Process and save with key
    Order order = orderService.process(request);
    order.setIdempotencyKey(key);
    return orderRepo.save(order);
}
```

The client sends:
```
POST /orders
Idempotency-Key: a1b2c3d4-...
Body: {...}
```

If they retry with the same key, they get the same response without double processing.

**Solution 2: Use natural unique IDs.**

If your business has a natural unique ID (order number, transaction ID), use it:

```java
public Order createOrder(OrderRequest request) {
    if (orderRepo.existsByOrderNumber(request.getOrderNumber())) {
        return orderRepo.findByOrderNumber(request.getOrderNumber());
    }
    return orderService.create(request);
}
```

**Solution 3: Database unique constraints.**

Make the database reject duplicates:

```sql
CREATE UNIQUE INDEX idx_idempotency 
ON orders(idempotency_key);
```

If you try to insert a duplicate, the database fails. Catch this and return the existing record.

**Solution 4: Distributed lock.**

For complex cases, use Redis or similar to prevent two simultaneous duplicates:

```java
public Order createOrder(String key, OrderRequest request) {
    String lockKey = "lock:order:" + key;
    
    // Try to acquire lock
    Boolean acquired = redis.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(30));
    
    if (Boolean.FALSE.equals(acquired)) {
        // Someone else is processing, return existing or wait
        return orderRepo.findByIdempotencyKey(key);
    }
    
    try {
        return process(request);
    } finally {
        redis.delete(lockKey);
    }
}
```

### How Long to Keep Idempotency Keys

- 24 hours for most cases
- 7 days for important transactions
- Never delete for critical financial data

### How to Prevent Duplicates from the Source

- **Disable submit button** after first click on UI
- **Use exponential backoff** for retries (don't retry instantly)
- **Use unique transaction IDs** in your business logic
- **Database constraints** as a final safety net

---

## 18. Cache Giving Stale Data

### What is Happening?

Your application uses a cache (Redis, local cache). The cache has old data. Users see information that is no longer correct.

### Common Reasons

1. **Cache not invalidated when data changes**
2. **Time to live (TTL) too long**
3. **Multiple instances with different local caches**
4. **Race condition between cache and database write**
5. **Cache populated with stale data on miss**
6. **Manual cache updates not synchronized**

### Common Caching Strategies

**Strategy 1: Cache-Aside (Lazy Loading)**

```
1. App checks cache for data
2. If hit, return data
3. If miss, fetch from database
4. Save to cache
5. Return data
```

```java
public User getUser(Long id) {
    User user = cache.get(id);
    if (user != null) {
        return user;
    }
    
    user = database.findById(id);
    cache.put(id, user, Duration.ofMinutes(10));
    return user;
}
```

Problem: If data changes in database, cache has old value until TTL expires.

**Strategy 2: Write-Through**

Every write goes to both cache and database:

```java
public void updateUser(User user) {
    database.save(user);
    cache.put(user.getId(), user);
}
```

Problem: If cache update fails after database update, inconsistency.

**Strategy 3: Write-Behind (or Write-Back)**

Write to cache first, then database asynchronously. Fast but risky.

**Strategy 4: Cache Invalidation**

When data changes, remove from cache:

```java
public void updateUser(User user) {
    database.save(user);
    cache.evict(user.getId());  // Will be reloaded on next read
}
```

This is the most common and recommended approach.

### How to Fix Stale Cache Issues

**Fix 1: Add cache invalidation to all write paths.**

Make sure every place that updates data also invalidates cache:

```java
@CacheEvict(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepo.save(user);
}

@CacheEvict(value = "users", key = "#id")
public void deleteUser(Long id) {
    userRepo.deleteById(id);
}
```

**Fix 2: Use shorter time to live.**

If perfectly fresh data is not required:
```java
cache.put(key, value, Duration.ofMinutes(5));  // 5 minutes max staleness
```

**Fix 3: Use distributed cache for multiple instances.**

If you have 5 servers each with local cache, invalidating one doesn't invalidate others.

Solution: use Redis as central cache. All instances share it.

**Fix 4: Use cache versioning.**

Append version to keys. When data changes globally, change version:

```java
String key = "user:" + version + ":" + userId;
```

When you change version, all old cache becomes invisible (effectively invalidated).

**Fix 5: Use pub/sub for invalidation.**

When data changes, publish to a topic. All instances listen and invalidate their local caches.

```java
public void updateUser(User user) {
    database.save(user);
    redis.convertAndSend("cache-invalidation", "user:" + user.getId());
}

@EventListener
public void onCacheInvalidation(String key) {
    localCache.evict(key);
}
```

### Cache Stampede Problem

What if cache expires and 1000 requests come at once? They all miss cache and hit database!

**Solution: Use a lock or "single flight":**

```java
public User getUser(Long id) {
    User user = cache.get(id);
    if (user != null) return user;
    
    String lockKey = "lock:user:" + id;
    if (redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(30))) {
        try {
            user = database.findById(id);
            cache.put(id, user);
            return user;
        } finally {
            redis.delete(lockKey);
        }
    } else {
        // Wait briefly and retry
        Thread.sleep(50);
        return cache.get(id);  // Should be there now
    }
}
```

### How to Prevent

- Always invalidate cache on writes
- Use shorter time to live for changing data
- Use distributed cache for multi-instance setups
- Monitor cache hit rate (should be above 80% usually)
- Have a way to manually clear cache in emergency

---

## 19. Application Not Scaling After Adding Instances

### What is Happening?

You added more servers expecting better performance. But performance did not improve much. Or it got worse. Why?

### Common Reasons

1. **Database is the bottleneck** - Not the application
2. **Shared resource contention** - All servers fight for same lock
3. **Stateful application** - Requests must go to specific server
4. **Singleton bottleneck** - Single component everyone uses
5. **External service limit** - Downstream cannot handle more requests
6. **Network bandwidth limit** - Hit network capacity
7. **Database connection limit** - Adding servers, but each has 50 connections, database has 100 limit
8. **Load balancer issue** - Not distributing traffic well
9. **Cache misses** - More servers, less cache hit rate per server
10. **Synchronous calls** - Adding servers doesn't help if calls are blocking

### How to Debug

**Step 1: Identify what is the bottleneck.**

```
Is CPU saturated on app servers?
   YES → You need more app servers (good - scaling helped)
   NO → CPU is not the issue, look elsewhere

Is database CPU saturated?
   YES → Database is the bottleneck, scaling app doesn't help
   
Is network saturated?
   YES → Network is the bottleneck

Are there many threads waiting?
   YES → Shared resource contention
```

**Step 2: Check Amdahl's Law.**

If 10% of your processing is sequential (cannot be parallelized), maximum speedup is 10x, no matter how many servers you add.

**Step 3: Find the shared resource.**

Look at:
- Database queries (especially writes)
- Cache (single Redis instance?)
- Message queue
- External APIs
- File system
- Network

**Step 4: Check load balancer.**

Is traffic distributed evenly? Sometimes one server gets 90% of traffic due to bad load balancing.

### How to Fix

**Fix 1: Make the application stateless.**

If a request can be handled by any server, you can scale horizontally.

```java
// Bad - state on server
public class UserController {
    private Map<String, Session> sessions = new HashMap<>();
    
    public Response handle(Request r) {
        Session s = sessions.get(r.getSessionId());
        // ...
    }
}

// Good - state in shared cache
public class UserController {
    @Autowired private RedisTemplate redis;
    
    public Response handle(Request r) {
        Session s = redis.opsForValue().get(r.getSessionId());
        // ...
    }
}
```

**Fix 2: Scale the database.**

- Add read replicas (for read-heavy)
- Use connection pooling effectively
- Shard the database (split by user ID, region, etc.)
- Use CQRS (separate read and write databases)

**Fix 3: Use distributed cache.**

Move cache from local to distributed (Redis):
- All servers benefit from cached data
- Higher hit rate

**Fix 4: Identify and fix the shared bottleneck.**

If everyone is fighting for one lock or one resource, that resource is the limit.

**Fix 5: Use async / message queues.**

Instead of synchronous calls between services, use queues:
```
Service A → Queue → Service B
```

A doesn't wait for B. Both scale independently.

**Fix 6: Use Content Delivery Network for static content.**

Don't serve images from your app servers. Use CDN.

**Fix 7: Implement caching at multiple levels.**

```
Browser cache → CDN → Load balancer cache → App cache → Database cache
```

Each level reduces load on the next.

### How to Test Scalability

Before adding instances in production:
1. Run load test on 1 instance, find breaking point
2. Run load test on 2 instances, see if it doubles
3. Run on 4 instances, see if it scales

If 1 instance handles 1000 requests/sec, 4 instances should handle close to 4000. If only 1500, you have a scalability problem.

### How to Prevent

- Design stateless services from day one
- Always use distributed cache, not local
- Database is usually the bottleneck - plan for scaling
- Load test with multiple instances
- Avoid shared mutable state
- Use async patterns where possible

---

## 20. Tracing a Request Across Multiple Layers

### What is Happening?

Modern systems have many services. A user request might touch:
- Mobile app
- API gateway
- Authentication service
- User service
- Order service
- Payment service
- Database
- External payment provider

When something is slow, where is the slow part? You need to trace the request across all these layers.

### Visual Picture

```
User Click
   │
   ▼
[API Gateway]   100ms ──┐
   │                    │
   ▼                    │
[Auth Service]   50ms ──┤── Total: 100 + 50 + 200 + 500 + 30 = 880ms
   │                    │   But where is the slow part?
   ▼                    │   Without tracing, you don't know!
[User Service]  200ms ──┤
   │                    │
   ▼                    │
[Order Service] 500ms ──┤  ← This is slow!
   │                    │
   ▼                    │
[Database]       30ms ──┘
```

### The Solution: Distributed Tracing

Each request gets a unique ID. Every service logs this ID with timing. You can then assemble the full picture.

### How to Implement

**Step 1: Use a distributed tracing tool.**

Popular options:
- Jaeger (open source)
- Zipkin (open source)
- AWS X-Ray
- Google Cloud Trace
- Datadog APM

These follow the OpenTelemetry standard.

**Step 2: Add tracing library to all services.**

For Spring Boot:

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

This automatically:
- Generates trace IDs
- Propagates them across HTTP calls
- Logs timing for each operation

**Step 3: Add custom spans for important operations.**

```java
@Autowired
private Tracer tracer;

public Order processOrder(OrderRequest request) {
    Span span = tracer.spanBuilder("processOrder")
        .setAttribute("orderId", request.getOrderId())
        .startSpan();
    
    try (Scope scope = span.makeCurrent()) {
        // Your business logic
        validate(request);
        Order order = save(request);
        notifyUser(order);
        return order;
    } finally {
        span.end();
    }
}
```

**Step 4: View traces in dashboard.**

In Jaeger or similar tool, you see:
```
Request ID: abc-123
Total time: 880ms

├── api-gateway (100ms)
│   └── auth-service (50ms)
│       └── user-service (200ms)
│           ├── database query (30ms)
│           └── order-service (500ms)  ← Slow!
│               ├── payment-service (450ms)  ← Actually here
│               └── database (40ms)
```

Now you know exactly where time is spent.

### Manual Tracing for Logs

Even without fancy tools, you can do basic tracing:

```java
public class TracingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }
        
        MDC.put("traceId", traceId);
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

Configure logger to include traceId:
```
%d{HH:mm:ss} [%thread] [%X{traceId}] %-5level %logger{36} - %msg%n
```

When making downstream calls, pass the trace ID:
```java
restTemplate.getForObject(url, String.class, 
    (req, exec) -> req.getHeaders().add("X-Trace-Id", MDC.get("traceId")));
```

Now you can grep all logs with the same trace ID.

### Best Practices

1. **Always use a standard like OpenTelemetry** - Don't roll your own
2. **Trace at every service boundary** - Including database calls
3. **Add business context** - User ID, order ID, transaction ID
4. **Don't log sensitive data** - No passwords, no credit cards in traces
5. **Use sampling for high-volume** - Trace 1% of requests, not all
6. **Combine traces with logs and metrics** - The three pillars of observability

### Tools That Help

| Tool | Type | Use For |
|------|------|---------|
| Jaeger / Zipkin | Tracing | See request flow |
| Prometheus / Grafana | Metrics | Numerical trends |
| ELK Stack (Elasticsearch, Logstash, Kibana) | Logs | Searchable logs |
| Datadog / New Relic | All-in-one | Production monitoring |
| OpenTelemetry | Standard | Generate traces and metrics |

---

## 21.  Connection Pool Exhausted

### What is Happening?

Your application uses a database connection pool (HikariCP, Tomcat). Under load, you see errors like "connection timeout" or "no connection available". The pool is exhausted.

### Common Causes

1. **Connection leak** - Connections borrowed but not returned
2. **Slow queries holding connections** - Each query takes too long
3. **Pool size too small** - Need more connections
4. **Long-running transactions** - Transactions held open
5. **Database itself is slow** - Returning results slowly

### How to Debug

**Step 1: Enable pool logging.**

For HikariCP:
```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 30000  # warn after 30 sec
```

This logs warnings about connections held too long.

**Step 2: Monitor pool metrics.**

Look at:
- Active connections (currently in use)
- Idle connections (available)
- Waiting threads (waiting for connection)
- Total connections used over time

**Step 3: Take thread dump during exhaustion.**

Find threads waiting for database connection. They look like:
```
"http-nio-8080-exec-15" WAITING
   at HikariPool.getConnection
   at MyService.fetchData
```

What is each thread doing? If they're all waiting on the same query, that query is the issue.

### How to Fix

**Fix 1: Always close connections (or use try-with-resources).**

```java
// Good - always closes
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql);
     ResultSet rs = stmt.executeQuery()) {
    // process results
}
```

**Fix 2: Tune pool size carefully.**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 5000  # max wait 5 seconds
      idle-timeout: 600000       # close idle after 10 min
      max-lifetime: 1800000      # max 30 min
```

**Don't make pool too big!** A common mistake is setting pool to 200. The database might not handle 200 concurrent connections from each app server.

Formula: `connections = ((core_count × 2) + effective_spindle_count)`

For most cases, 10-20 is enough.

**Fix 3: Reduce query time.**

Faster queries = connections returned faster = better throughput.

**Fix 4: Don't hold transactions across calls.**

```java
// Bad - holds connection during external call
@Transactional
public void process() {
    User user = userRepo.findById(id);
    externalApi.call(user);  // Holds DB connection during HTTP!
    userRepo.save(user);
}

// Good - separate transactions
public void process() {
    User user = fetchUser(id);
    externalApi.call(user);
    saveUser(user);
}

@Transactional
private User fetchUser(Long id) { return userRepo.findById(id); }

@Transactional
private void saveUser(User user) { userRepo.save(user); }
```

---

## 22.  SSL Handshake Failures

### What is Happening?

Your application cannot connect to another service over HTTPS. You see errors about SSL/TLS, certificate, or handshake.

### Common Errors

- `PKIX path building failed` - Cannot verify certificate
- `Certificate has expired`
- `Hostname does not match certificate`
- `Handshake failure`
- `SSL protocol mismatch`

### How to Debug

**Step 1: Test the connection manually.**

```bash
openssl s_client -connect example.com:443 -showcerts
```

This shows:
- Certificate details
- Certificate chain
- Connection success or failure

**Step 2: Check certificate expiry.**

```bash
echo | openssl s_client -servername example.com -connect example.com:443 2>/dev/null | openssl x509 -noout -dates
```

**Step 3: Check Java's truststore.**

```bash
keytool -list -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit
```

If the certificate authority is missing, you need to add it.

**Step 4: Enable SSL debugging.**

```bash
-Djavax.net.debug=ssl:handshake
```

This shows all SSL details in logs (use only in debug, not production).

### How to Fix

**Expired certificate:** Renew it.

**Missing root certificate:** Add to truststore:
```bash
keytool -import -alias myroot -file root.crt -keystore truststore.jks
```

**Hostname mismatch:** Use the correct hostname or get a certificate with proper names (Subject Alternative Names).

**Protocol mismatch:** Update Java or the server to support common protocols (TLS 1.2 or TLS 1.3).

**Self-signed certificate (development only):**
```java
// DO NOT USE IN PRODUCTION
TrustManager[] trustAll = new TrustManager[]{
    new X509TrustManager() {
        public X509Certificate[] getAcceptedIssuers() { return null; }
        public void checkClientTrusted(X509Certificate[] certs, String type) {}
        public void checkServerTrusted(X509Certificate[] certs, String type) {}
    }
};
```

### How to Prevent

- Monitor certificate expiry (alert 30 days before)
- Use automated certificate renewal (Let's Encrypt with cert-manager)
- Keep Java updated for latest TLS support
- Document required certificates in deployment guide

---

## 23.  Sudden Latency Spikes

### What is Happening?

Your service usually responds in 100 milliseconds. Sometimes responses take 5 seconds. These spikes happen randomly.

### Common Causes

1. **Garbage collection pauses**
2. **Network blips**
3. **Database lock contention**
4. **Cold cache** (after restart)
5. **Disk I/O slowdowns**
6. **Other tenants in shared infrastructure** (noisy neighbor)
7. **Background jobs running**
8. **Lazy initialization** (first request after deploy is slow)
9. **JIT compilation** (just-in-time, JVM optimizing code)
10. **DNS resolution failures**

### How to Debug

**Step 1: Look at p99 and p99.9 latency.**

Don't just look at average. Average hides spikes.

```
Average: 150ms (looks fine)
p50 (median): 100ms
p95: 200ms
p99: 2000ms (some users see 2 seconds!)
p99.9: 5000ms (some users see 5 seconds!)
```

**Step 2: Correlate with system events.**

When did spikes happen? Check if at same time:
- Garbage collection happened?
- Backup job ran?
- Network maintenance?
- Other application restart?

**Step 3: Check if specific endpoints are slow.**

Maybe one specific endpoint causes spikes, not all.

**Step 4: Check downstream dependencies.**

Maybe your service is fast, but a downstream call is slow occasionally.

### How to Fix

**For garbage collection spikes:** Use low-pause garbage collector (ZGC, Shenandoah).

**For cold cache:** Pre-warm cache after deploy.

**For lazy init:** Initialize at startup with a warm-up request.

**For just-in-time compilation:** Use ahead-of-time compilation (GraalVM) or accept the first few requests will be slower.

**For network blips:** Add retries with exponential backoff.

**For background jobs:** Schedule them at low-traffic times.

**For database locks:** Use shorter transactions, optimize queries.

### How to Prevent

- Monitor p99 latency, not just average
- Set SLO (service level objective) on p99
- Run load tests including warm-up phases
- Use chaos engineering to find weak points

---

## 24.  Slow Application Startup

Modern Java applications can take minutes to start. In Kubernetes with rolling deployments and auto-scaling, slow startup hurts a lot.

### Why startups are slow

Several things happen when a Java application starts:

1. **Class loading** - Loading thousands of classes from JAR files
2. **Spring context initialization** - Scanning packages, creating beans, wiring dependencies
3. **Database connection setup** - Opening connection pool, validating
4. **Cache warming** - Pre-loading frequently used data
5. **Just-In-Time compilation** - First few times code runs, it is interpreted (slow). After that, JIT compiles it (fast).

### How I diagnose slow startup

**Step one: Add timing logs**

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        SpringApplication.run(Application.class, args);
        long duration = System.currentTimeMillis() - start;
        System.out.println("Started in " + duration + " ms");
    }
}
```

**Step two: Enable Spring startup tracking**

```yaml
spring:
  main:
    lazy-initialization: false
  application:
    name: my-app
debug: true
```

This logs every bean creation. Find the slow ones.

**Step three: Use Spring Boot Actuator startup endpoint**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: startup
```

Then call `/actuator/startup` to see the slowest beans.

### Optimization techniques

**Technique one: Lazy initialization**

```yaml
spring:
  main:
    lazy-initialization: true
```

Beans are created only when first used. Faster startup, slower first request.

**Technique two: Reduce component scanning scope**

```java
// Bad - scans the world
@SpringBootApplication
@ComponentScan("com")

// Good - scans only your packages
@SpringBootApplication
@ComponentScan(basePackages = "com.mycompany.myapp")
```

**Technique three: Disable unused auto-configurations**

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class,
    KafkaAutoConfiguration.class
})
```

If you do not use Kafka, do not load Kafka auto-config.

**Technique four: Class Data Sharing**

```bash
# Create archive on first run
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar

# Use archive on next runs
java -XX:SharedArchiveFile=app.jsa -jar app.jar
```

Pre-loaded class metadata saves several seconds.

**Technique five: Native image with GraalVM**

```bash
mvn -Pnative spring-boot:build-image
```

Compiles your Java app to native code. Startup time drops from seconds to milliseconds.

**Trade-offs:**
- No reflection (or limited)
- No dynamic class loading
- Larger build time
- But: 100 ms startup, lower memory

**Technique six: JVM tuning for fast startup**

```
-XX:TieredStopAtLevel=1     # less aggressive JIT initially
-Xshareclasses              # share classes across JVMs
-XX:+UseG1GC                # good GC for startup
```

### Real example

A monolith took 4 minutes to start. We optimized in stages:

1. Removed unused auto-configurations: 4 min → 3 min
2. Enabled lazy initialization for non-critical beans: 3 min → 2 min  
3. Migrated to Spring Boot 3 with Ahead-Of-Time compilation: 2 min → 30 sec
4. Switched to GraalVM native image: 30 sec → 100 ms

100 ms startup time enabled true auto-scaling. We could spin up new instances faster than load was increasing. Beautiful to watch.

### When startup time matters most

- **Kubernetes with horizontal pod autoscaling** - need to scale fast
- **Serverless / Function as a Service** - cold start cost
- **Continuous deployment** - rolling updates faster
- **Local development** - developer happiness
- **Testing** - thousands of test runs faster

If your app starts once and runs for months, do not worry about it.

---

## 25.  Race Condition Under Load

A race condition is when behavior depends on timing. Code works fine with one thread but fails with many threads.

### A real example

```java
public class BankAccount {
    private int balance = 100;
    
    public void withdraw(int amount) {
        if (balance >= amount) {        // STEP 1: check
            balance = balance - amount;  // STEP 2: subtract
        }
    }
}
```

Looks fine. But under concurrent load:

```
    Time   Thread A                    Thread B
    ────   ────────                    ────────
    T1     reads balance = 100
    T2                                 reads balance = 100
    T3     check: 100 >= 80 ✓
    T4                                 check: 100 >= 80 ✓
    T5     balance = 100 - 80 = 20
    T6                                 balance = 100 - 80 = 20
    
    Result: balance = 20
    Should be: -60 (or rejected)
    The bank just lost 80 dollars!
```

### How to find race conditions

**Method one: Stress testing**

```java
@Test
void stressTest() throws Exception {
    BankAccount account = new BankAccount(); // starts with 1000
    ExecutorService pool = Executors.newFixedThreadPool(100);
    CountDownLatch latch = new CountDownLatch(1000);
    
    // 1000 threads each try to withdraw 1
    for (int i = 0; i < 1000; i++) {
        pool.submit(() -> {
            try {
                account.withdraw(1);
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await();
    pool.shutdown();
    
    // 1000 - 1000 should equal 0
    assertEquals(0, account.getBalance());
}
```

If this fails sometimes, you have a race condition.

**Method two: Run tests many times**

Race conditions are intermittent. Run the test 1000 times in a loop. If it fails even once, it is a bug.

**Method three: Use specialized tools**

- **Java Concurrency Stress test** - Designed for testing concurrent code
- **SpotBugs** - Static analysis catches some races
- **Code review** - Look for read-then-write patterns

### Common race condition patterns

**Pattern one: Check-then-act**

```java
if (map.containsKey(key)) {  // check
    value = map.get(key);    // act
    // race window in between!
}
```

Fix: Use `getOrDefault` or `computeIfAbsent`.

**Pattern two: Read-modify-write**

```java
counter = counter + 1;  // not atomic!
```

Fix: Use `AtomicInteger.incrementAndGet()`.

**Pattern three: Lazy initialization**

```java
if (instance == null) {     // check
    instance = new Foo();    // create
}
```

Fix: Double-checked locking with `volatile`, or use `enum` for singleton.

### How to fix race conditions

**Fix one: Synchronization**

```java
public synchronized void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
    }
}
```

Now check and update happen atomically. No race.

**Fix two: Atomic compare-and-set**

```java
private AtomicInteger balance = new AtomicInteger(100);

public boolean withdraw(int amount) {
    while (true) {
        int current = balance.get();
        if (current < amount) return false;
        int newBalance = current - amount;
        if (balance.compareAndSet(current, newBalance)) {
            return true;
        }
        // someone else changed it, retry
    }
}
```

Lock-free. Very fast under high contention.

**Fix three: Database optimistic locking**

```sql
UPDATE accounts 
SET balance = balance - 80, version = version + 1
WHERE id = 1 AND balance >= 80 AND version = 5;
```

If `version` changed (someone else updated), this affects 0 rows. Detect and retry.

**Fix four: Use higher-level abstractions**

```java
// Single-threaded queue processor
BlockingQueue<Withdrawal> queue = new LinkedBlockingQueue<>();
// One thread consumes, no races possible
```

### Real example

A coupon service had a "limit one per user" rule. Under load, some users got multiple coupons.

The buggy code:

```java
public void claimCoupon(Long userId) {
    if (!alreadyClaimed(userId)) {     // race condition here
        recordClaim(userId);
    }
}
```

Two simultaneous requests both saw "not claimed" and both recorded claims.

Fix one: Database unique constraint:

```sql
CREATE UNIQUE INDEX uk_user_coupon ON claims(user_id, coupon_id);
```

Insert, catch unique violation = already claimed. Atomic at database level.

Fix two: Distributed lock for hot coupons using Redis:

```java
public void claimCoupon(Long userId, Long couponId) {
    String lockKey = "coupon:" + couponId + ":user:" + userId;
    if (redisLock.tryAcquire(lockKey, 10, TimeUnit.SECONDS)) {
        try {
            if (!alreadyClaimed(userId, couponId)) {
                recordClaim(userId, couponId);
            }
        } finally {
            redisLock.release(lockKey);
        }
    }
}
```

Result: zero duplicate claims, even at 10,000 requests per second.

---

## 26.  Intermittent 500 Errors

Sometimes requests succeed, sometimes they fail with 500 Internal Server Error. Random failures are the trickiest bugs.

### My investigation order

**Step one: Group errors by type**

```bash
grep "ERROR" application.log | awk -F'Exception' '{print $1}' | sort | uniq -c | sort -rn
```

This shows top error types. Often you have multiple causes mixed together.

**Step two: Look for patterns**

- Same error every time? Real code bug.
- Random errors? Resource issue or external dependency.
- Errors at specific times? Cron job or scheduled load.
- Errors on specific servers? Bad node or config drift.
- Errors only for some users? Data-specific issue.

**Step three: Check downstream services**

```bash
grep "downstream-service" trace.log | grep -i "error\|timeout"
```

If your downstream is flaky, you will be flaky too.

**Step four: Check resource pressure**

- Memory near limit? Some allocations fail.
- File descriptors near limit? Sockets fail.
- Connection pool near full? Database calls fail.

**Step five: Look at health check status**

Maybe the load balancer is sending traffic to unhealthy instances during deployments.

### Common root causes

**Root cause one: Connection timeout to downstream**

```java
RestTemplate restTemplate = new RestTemplate();
// No timeout configured!
String result = restTemplate.getForObject(url, String.class);
// throws after 60 seconds = 500 to user
```

Fix: Set proper timeouts and use circuit breakers:

```java
RestTemplate restTemplate = new RestTemplateBuilder()
    .setConnectTimeout(Duration.ofSeconds(2))
    .setReadTimeout(Duration.ofSeconds(5))
    .build();
```

**Root cause two: Database deadlocks**

Some transactions deadlock under load. Database picks one to kill. That request fails.

Fix: Add retry logic for deadlock errors:

```java
@Retryable(
    value = {DeadlockLoserDataAccessException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100)
)
public void updateOrder(Order order) {
    // ...
}
```

**Root cause three: Optimistic lock conflicts**

```java
@Version
private Long version;
```

Concurrent updates fail with `OptimisticLockException`. User sees 500.

Fix: Catch and retry, or return clear "please try again" error.

**Root cause four: Cache stampede**

Cache expires. 1000 requests all try to rebuild it at the same time. Database overloaded. Some requests fail.

Fix:
- Use cache locks (only one rebuilds, others wait)
- Randomize expiry times (avoid simultaneous expiry)
- Use refresh-ahead pattern

**Root cause five: Memory pressure**

Some allocations fail with `OutOfMemoryError`. Service stays up but individual requests fail randomly.

Fix: Reduce memory pressure, fix leaks, increase heap if needed.

**Root cause six: Garbage collection pauses**

Long GC pauses cause request timeouts. Then those requests fail.

Fix: Tune GC. Use ZGC for low-latency services.

### Real example

A search service had 0.5% error rate. Investigation revealed multiple causes:

1. 0.2% - downstream API timeouts → added circuit breaker
2. 0.1% - database deadlocks → added retry logic
3. 0.1% - optimistic lock conflicts → proper retry with backoff
4. 0.1% - random errors from a buggy library → upgraded library

Fixed each separately. Got down to 0.01% (basically just real errors that should fail).

The lesson: 500 errors are usually multiple bugs mixed together. Investigate each one.

---

## 27.  Logs Filling Up Disk

Logging seems harmless. But it can take down production.

### Common logging problems

**Problem one: Synchronous logging**

```xml
<appender class="ch.qos.logback.core.FileAppender">
    <file>app.log</file>
</appender>
```

Each log line waits for disk write. Under load, all threads queue up on the logger.

**Problem two: Logging too much**

```java
log.info("Request received: {}", request);
log.info("User loaded: {}", user);
log.info("Order processed: {}", order);
log.info("Email sent");
```

Per request, four log lines. At 10,000 requests per second, 40,000 log lines per second. Disk cannot keep up.

**Problem three: Logging huge objects**

```java
log.info("Full state: {}", entireDatabase);  // logs 10 MB
```

Slow to serialize. Slow to write. Fills up disk fast.

**Problem four: Disk fills up**

Logs grow to 100 GB. Disk full = application cannot write anything = crashes silently.

### My logging best practices

**Practice one: Use async appenders**

```xml
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE"/>
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <neverBlock>true</neverBlock>
</appender>
```

Logs go to a queue. Background thread writes to disk. Application is not blocked.

**Practice two: Use proper log levels**

```
ERROR ──► Things that broke. Need to wake someone up.
WARN  ──► Suspicious. Investigate later.
INFO  ──► Important business events. Order placed, user registered.
DEBUG ──► Development details. Off in production.
TRACE ──► Very detailed. Only when debugging specific issue.
```

In production: ERROR, WARN, and selective INFO. DEBUG only for specific packages when debugging.

**Practice three: Use structured logging**

```java
// Bad - hard to search
log.info("User " + userId + " bought " + productId + " for " + amount);

// Good - structured
log.info("Purchase completed",
    kv("userId", userId),
    kv("productId", productId),
    kv("amount", amount));
```

Output as JSON. Tools like Elasticsearch can search and aggregate easily.

**Practice four: Sample high-volume logs**

```java
if (ThreadLocalRandom.current().nextInt(100) == 0) {
    log.info("Request: {}", request);  // log 1% of requests
}
```

For 1 million requests per second, sampling 1% gives 10,000 logs per second. Manageable.

**Practice five: Rotate logs**

```xml
<appender class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>app-%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>7</maxHistory>
        <totalSizeCap>10GB</totalSizeCap>
    </rollingPolicy>
</appender>
```

Auto-compress old logs. Auto-delete after 7 days. Cap total size at 10 GB.

**Practice six: Ship logs off the server**

Do not store logs on the server long-term. Ship to:
- Elasticsearch + Kibana
- Splunk
- DataDog
- CloudWatch Logs

The server only keeps recent logs for emergencies.

### Real example

A high-traffic API would freeze every few hours. Investigation showed:

- Synchronous logging at INFO level
- 50,000 requests per second × 5 logs per request = 250,000 logs per second
- Disk could not keep up
- Threads piled up waiting on the logger

Fixes applied:

1. Switched to async appenders → no more thread blocking
2. Reduced logging to ERROR + WARN + sampled INFO (1%) → 50x less volume
3. Shipped logs to Elasticsearch directly via Filebeat → no local disk pressure
4. Added log rotation with size cap → never fill disk

Result: no more freezes. Logging costs are real. Treat them like any other system resource.

---

## 28.  Different Behavior in Different Environments

Development works. Staging works. Production breaks. Or the opposite. Why?

### What is different across environments

```
                    Dev          Staging        Production
                    ───          ───────        ──────────
    Data volume     Tiny         Medium         Massive
    Load            None         Synthetic      Real users
    Configuration   Defaults     Mostly real    Real
    Secrets         Fake         Fake           Real
    Network         Localhost    Same VPC       Multi-region
    Services        Mocks        Real services  Real services
    Encryption      No           Yes            Yes
    Time/timezone   Local        UTC            UTC
    Java version    Latest       Production     Production
    Memory          Plenty       Limited        Limited
```

### My approach to environment-specific bugs

**Step one: Compare configurations**

```bash
diff dev.properties prod.properties
```

Look for:
- Different timeout values
- Different feature flags
- Different log levels
- Different connection strings
- Different pool sizes

**Step two: Check actual runtime values**

Add an admin endpoint that shows live configuration:

```java
@GetMapping("/admin/config")
public Map<String, String> showConfig() {
    return Map.of(
        "dbUrl", maskedDbUrl(),
        "cacheSize", String.valueOf(cacheSize),
        "javaVersion", System.getProperty("java.version"),
        "timezone", TimeZone.getDefault().getID(),
        "encoding", System.getProperty("file.encoding")
    );
}
```

Sometimes the config you set is not the config that actually loaded.

**Step three: Use feature flags**

Instead of different code per environment:

```java
if (featureFlags.isEnabled("new-checkout")) {
    return newCheckout();
} else {
    return oldCheckout();
}
```

Test the new feature in production with 1% of users before full rollout. This is called canary deployment.

**Step four: Reproduce production conditions locally**

Use Docker Compose mimicking production:

```yaml
version: '3'
services:
  app:
    image: myapp:latest
    environment:
      - JAVA_OPTS=-Xmx512m
    deploy:
      resources:
        limits:
          memory: 600M
          cpus: '1'
  database:
    image: postgres:14
```

**Step five: Use blue-green or canary deployments**

Deploy new version to a small percentage of traffic first. If it breaks, rollback affects only that small percentage.

### The classic environment bugs

**Bug one: Time zone confusion**

Developer machine in India Standard Time, production in Coordinated Universal Time. All times off by 5.5 hours.

Fix: Always use UTC in storage. Convert to user time zone only at display.

**Bug two: Default port conflicts**

Dev: only your app on port 8080. Production: shares server with other apps on different ports.

**Bug three: Resource limits**

Dev: 16 GB RAM available. Production container: 512 MB. Memory tuning that worked in dev fails in production.

**Bug four: Network policies**

Dev: open network. Production: strict firewalls. Code that works in dev cannot reach services in production.

**Bug five: Authentication**

Dev: no auth or fake auth. Production: real OAuth tokens. Token handling bugs only show in production.

**Bug six: Character encoding**

Dev on Mac/Windows: default encoding might be different. Production on Linux: UTF-8. Special characters break.

Fix: Always set `-Dfile.encoding=UTF-8`.

### Real example

A reporting feature worked in staging but generated wrong numbers in production. Investigation:

- Staging had 10,000 records → finished in 1 second
- Production had 10 million records → query timed out
- Code did `findAll()` and then filtered in Java
- With 10K records: fine
- With 10M records: ran out of memory before filtering completed

Fix: moved filtering to the database. Use `findByStatus()` instead of `findAll().filter()`. Always do filtering at the database level when possible.

---

## 29.  Building Observability Into a Service

You cannot fix what you cannot see. Observability is the practice of being able to ask any question about your running system.

### The three pillars

```
    ┌─────────────────────────────────────────┐
    │           OBSERVABILITY                  │
    ├─────────────────────────────────────────┤
    │                                         │
    │  Metrics    Logs       Traces           │
    │     │        │          │               │
    │     ▼        ▼          ▼               │
    │  How much  What         Where time      │
    │  How fast  When         went            │
    │  How many  What detail  Why slow        │
    │                                         │
    └─────────────────────────────────────────┘
```

### Setting up metrics

Use Micrometer (works with Prometheus, DataDog, CloudWatch, etc):

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Auto-collected metrics include:**
- Java Virtual Machine memory and garbage collection
- Thread counts and states
- HTTP request rates and latencies
- Database connection pool stats
- Cache hit rates

**Custom metrics:**

```java
@Autowired
MeterRegistry registry;

// Counter - things that go up
Counter orders = Counter.builder("orders.placed")
    .tag("type", "online")
    .register(registry);
orders.increment();

// Timer - measure duration
Timer paymentTime = Timer.builder("payment.duration")
    .register(registry);
paymentTime.record(() -> processPayment());

// Gauge - current value
Gauge.builder("queue.size", queue, Queue::size)
    .register(registry);
```

### Setting up structured logs

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
</dependency>
```

```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
```

Output is JSON. Easy to ship to Elasticsearch:

```json
{
  "timestamp": "2025-05-04T10:30:00.000Z",
  "level": "INFO",
  "logger": "com.app.OrderService",
  "thread": "http-thread-42",
  "message": "Order placed",
  "traceId": "abc123",
  "userId": "user_456",
  "orderId": "order_789"
}
```

### Setting up tracing

(Same as Section 20 - OpenTelemetry, Jaeger, or similar.)

### The four golden signals

Google's Site Reliability Engineering book recommends monitoring these four for every service:

1. **Latency** - How long requests take
2. **Traffic** - How many requests per second
3. **Errors** - Rate of failed requests
4. **Saturation** - How full the service is (CPU, memory, queue depth)

### My standard dashboard

Every service I build has this dashboard:

```
    ┌─────────────────────────────────────┐
    │           SERVICE NAME              │
    ├─────────────────────────────────────┤
    │                                     │
    │  Request Rate    │  Error Rate      │
    │     ╱╲╱╲         │   ─────────      │
    │                  │                  │
    ├──────────────────┼──────────────────┤
    │                  │                  │
    │  Latency p50/p95 │  CPU & Memory    │
    │  ────────╱╲──    │   ░░░░░░░░       │
    │                  │                  │
    ├──────────────────┼──────────────────┤
    │                  │                  │
    │  Database Pool   │  Garbage Collect │
    │   ████░░░░░░     │    █  █  █       │
    │                  │                  │
    └─────────────────────────────────────┘
```

### Setting up alerts

Alert on **symptoms**, not causes:

**Good alerts:**
- Error rate > 1% for 5 minutes
- Latency at 95th percentile > 500 ms for 10 minutes
- No requests for 5 minutes (service is dead)

**Bad alerts:**
- CPU > 80% (might be normal)
- Single request failed (noise)
- Disk 70% full (still has time)

Too many alerts equals alert fatigue equals real alerts get ignored.

### Real example

A team had no monitoring. Outages were discovered by users tweeting at the company. We added:

1. Micrometer for metrics → Prometheus → Grafana dashboards
2. JSON logs → Elasticsearch → Kibana
3. OpenTelemetry → Jaeger
4. PagerDuty alerts on the four golden signals

Mean Time To Detection went from 30 minutes (Twitter) to 2 minutes (PagerDuty). Mean Time To Resolution dropped 70% because we could quickly find root causes.

Observability is not optional in production. Build it from day one.

---

## 30.  StackOverflowError Mystery

A `StackOverflowError` happens when the call stack runs out of space. Each method call uses some stack memory. Too many calls = stack overflow.

### Why it happens

**Cause one: Infinite recursion**

```java
public int factorial(int n) {
    return n * factorial(n - 1);  // never stops!
    // missing base case: if (n == 0) return 1;
}
```

**Cause two: Very deep recursion**

```java
public int sum(List<Integer> list) {
    if (list.isEmpty()) return 0;
    return list.get(0) + sum(list.subList(1, list.size()));
}
// works for 100 elements, fails for 10,000
```

**Cause three: Mutual recursion**

```java
public boolean isEven(int n) {
    if (n == 0) return true;
    return isOdd(n - 1);
}

public boolean isOdd(int n) {
    if (n == 0) return false;
    return isEven(n - 1);
}
// works for small n, fails for large n
```

**Cause four: Cyclic toString or equals**

```java
public class Node {
    Node next;
    
    @Override
    public String toString() {
        return "Node" + next.toString();  // if cycle, infinite loop!
    }
}
```

**Cause five: Recursive serialization**

Two objects reference each other. Jackson or similar tries to serialize, gets stuck in infinite loop.

### How I debug it

**Step one: Read the stack trace**

```
java.lang.StackOverflowError
    at com.app.Service.method(Service.java:42)
    at com.app.Service.method(Service.java:42)
    at com.app.Service.method(Service.java:42)
    ... 5000 more
```

If you see the same method repeating, you found the recursion.

**Step two: Look at the recursion logic**

- Is there a base case?
- Does the input always get smaller toward base case?
- Could the input be unexpectedly large?

**Step three: Check for cycles in data**

If you have linked structures (graphs, trees), check for cycles.

### How to fix it

**Fix one: Add or fix the base case**

```java
public int factorial(int n) {
    if (n <= 1) return 1;  // base case!
    return n * factorial(n - 1);
}
```

**Fix two: Convert recursion to iteration**

Iterative code uses heap memory (much bigger) instead of stack:

```java
// Recursive (stack overflow risk)
public int sum(List<Integer> list) {
    if (list.isEmpty()) return 0;
    return list.get(0) + sum(list.subList(1, list.size()));
}

// Iterative (safe)
public int sum(List<Integer> list) {
    int total = 0;
    for (int n : list) {
        total += n;
    }
    return total;
}
```

**Fix three: Use explicit stack data structure**

For tree or graph traversal:

```java
// Recursive (might overflow)
public void traverse(Node node) {
    if (node == null) return;
    process(node);
    traverse(node.left);
    traverse(node.right);
}

// Using explicit stack (safe)
public void traverse(Node root) {
    Deque<Node> stack = new ArrayDeque<>();
    stack.push(root);
    while (!stack.isEmpty()) {
        Node node = stack.pop();
        if (node == null) continue;
        process(node);
        stack.push(node.right);
        stack.push(node.left);
    }
}
```

**Fix four: Detect cycles**

```java
public String safeToString(Node node, Set<Node> visited) {
    if (visited.contains(node)) return "[cycle]";
    visited.add(node);
    return "Node" + safeToString(node.next, visited);
}
```

**Fix five: Increase stack size (last resort)**

```
java -Xss2m -jar app.jar  # 2 MB stack instead of default 512 KB
```

This just delays the crash. Better to fix the recursion.

### Real example

An XML parsing service crashed on certain documents. Stack trace showed thousands of `parseElement` calls. Investigation: documents had 10,000 nested elements. Recursion depth exceeded.

Fix: rewrote parser as iterative using an explicit stack. Could now handle documents of any depth.

---

## 31.  File Descriptor Leak

In Linux, everything is a file: regular files, sockets, pipes. Each open one uses a file descriptor. Default limit is usually 1024 per process.

### Symptoms

- `Too many open files` error
- Cannot accept new connections
- Cannot open files
- Cannot write logs

### How to detect it

**Check current usage:**

```bash
# Total file descriptors used by process
ls /proc/<pid>/fd | wc -l

# What kind of file descriptors
ls -la /proc/<pid>/fd | awk '{print $11}' | sort | uniq -c

# Check the limit
cat /proc/<pid>/limits | grep "Max open files"
```

**Watch over time:**

```bash
while true; do
    echo "$(date) $(ls /proc/<pid>/fd | wc -l)"
    sleep 10
done
```

If the number keeps growing, you have a leak.

### Common causes

**Cause one: Not closing streams**

```java
// Bad - stream never closed if exception
FileInputStream fis = new FileInputStream("file.txt");
processFile(fis);  // if this throws, fis is leaked
fis.close();

// Good - try-with-resources
try (FileInputStream fis = new FileInputStream("file.txt")) {
    processFile(fis);
}  // automatically closed
```

**Cause two: Not closing HTTP connections**

```java
// Bad
HttpClient client = HttpClient.newHttpClient();
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
// connection might stay open

// Good - use connection pooling library
// Apache HttpClient with PoolingHttpClientConnectionManager
```

**Cause three: Not closing database connections**

```java
// Bad
Connection conn = dataSource.getConnection();
// ... use connection ...
return result;  // forgot to close!

// Good
try (Connection conn = dataSource.getConnection()) {
    // ... use connection ...
    return result;
}
```

**Cause four: Sockets in CLOSE_WAIT state**

```bash
netstat -an | grep CLOSE_WAIT | wc -l
```

If many `CLOSE_WAIT` connections, you are not closing client connections properly.

**Cause five: Watching too many files**

`WatchService` and similar APIs use file descriptors. If you create many watchers, you leak.

### How to fix

**Always use try-with-resources for closeable things:**

```java
try (
    FileInputStream fis = new FileInputStream("input.txt");
    FileOutputStream fos = new FileOutputStream("output.txt");
) {
    fis.transferTo(fos);
}  // both closed automatically
```

**Increase limits if legitimately needed:**

```bash
# Temporary
ulimit -n 65536

# Permanent in /etc/security/limits.conf
*  soft  nofile  65536
*  hard  nofile  65536
```

### Real example

A web service crashed every 3 days with `Too many open files`. Investigation:

- File descriptor count grew steadily over time
- `ls -la /proc/<pid>/fd` showed many sockets to a downstream service
- All in `CLOSE_WAIT` state

Root cause: HTTP client did not properly close response bodies. The pattern was:

```java
// Bad
HttpResponse response = client.execute(request);
return response.getEntity().getContent();
// connection not released back to pool!
```

Fix:

```java
// Good
try (CloseableHttpResponse response = client.execute(request)) {
    return EntityUtils.toString(response.getEntity());
}  // connection released
```

After fix, file descriptor count stayed flat. No more crashes.

---

## 32.  Kafka Consumer Lag Building Up

Kafka consumer lag means messages are being produced faster than consumed. Lag grows. Messages get older. Real-time becomes not-so-real-time.

### How to monitor lag

```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
    --describe --group my-group
```

Output:
```
TOPIC      PARTITION   CURRENT-OFFSET   LOG-END-OFFSET   LAG
orders     0           1000             5000             4000
orders     1           1500             5000             3500
```

`LAG` is messages waiting to be processed. Growing lag is bad.

### Common causes of lag

**Cause one: Consumer is too slow**

Each message takes too long to process. Adding more consumers does not help if there are only a few partitions.

**Cause two: Not enough consumers**

If topic has 10 partitions but you have 1 consumer, that consumer handles all 10. Add more.

**Cause three: Slow downstream calls**

Consumer processes message, then calls a slow database. Cannot keep up.

**Cause four: Large messages**

Big messages take longer to deserialize and process.

**Cause five: Rebalancing**

When consumers join or leave, Kafka rebalances. During rebalance, no consumption happens. Frequent rebalances = lag.

**Cause six: Skewed partitions**

One partition gets way more messages than others. One consumer is overwhelmed while others are idle.

### How to fix lag

**Fix one: Add more consumers (up to partition count)**

```yaml
# 10 partitions, run 10 consumer instances
spring.kafka.listener.concurrency: 10
```

You cannot have more consumers in a group than partitions. So if you need to scale beyond, increase partitions first.

**Fix two: Increase partition count**

```bash
kafka-topics.sh --alter --topic orders --partitions 30 \
    --bootstrap-server kafka:9092
```

Now you can scale to 30 consumers.

**Fix three: Process messages in parallel within a consumer**

```java
@KafkaListener(topics = "orders", concurrency = "10")
public void process(List<Order> orders) {
    orders.parallelStream().forEach(this::processOrder);
}
```

But be careful: parallel processing breaks ordering guarantees.

**Fix four: Batch processing**

```java
@KafkaListener(topics = "orders")
public void processBatch(List<ConsumerRecord<String, Order>> records) {
    // process all together
    List<Order> orders = records.stream()
        .map(ConsumerRecord::value)
        .collect(Collectors.toList());
    orderService.saveBatch(orders);  // one DB call instead of N
}
```

Configure:
```yaml
spring.kafka.consumer.max-poll-records: 500
spring.kafka.listener.type: batch
```

**Fix five: Async processing**

Consumer just queues messages, separate threads process:

```java
@KafkaListener(topics = "orders")
public void receive(Order order) {
    processingQueue.offer(order);  // fast
    // background workers pull from queue
}
```

**Fix six: Avoid slow downstream calls in consumer**

If you must call slow services:
- Use batch APIs
- Use connection pools properly
- Cache aggressively
- Use circuit breakers

**Fix seven: Tune consumer settings**

```yaml
spring.kafka.consumer:
  fetch-min-size: 50000          # batch up small messages
  fetch-max-wait: 500            # max wait for batch
  max-poll-records: 500          # records per poll
  session-timeout: 30000         # avoid premature rebalance
  heartbeat-interval: 3000
```

### Real example

A payment processing system had 1 million message lag growing daily. Investigation:

- Topic had 5 partitions, 5 consumers
- Each consumer made a synchronous database call (50 ms)
- Max throughput: 5 partitions × 20 messages per second = 100 per second
- Incoming rate: 500 messages per second
- Lag grew by 400 per second

Fixes:
1. Increased partitions to 50, consumers to 50 → 1000/sec capacity
2. Switched to batch processing of 100 records → less DB overhead
3. Added Redis cache for frequent lookups → less DB load

Result: cleared the backlog in 1 hour. Never lagged again.

---

## 33.  Random Pod Restarts in Kubernetes

Your Kubernetes pods restart on their own. No clear error. What is happening?

### Common causes

**Cause one: OutOfMemoryKilled (OOMKilled)**

Pod used more memory than its limit. Kubernetes killed it.

```bash
kubectl describe pod <pod>
# Look for "Last State: OOMKilled"
```

Fix: Increase memory limit, or fix memory leak.

**Cause two: Liveness probe failures**

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

If `/health` does not respond in time, Kubernetes restarts the pod.

Common reasons for probe failures:
- Health endpoint is slow (database check inside)
- Garbage collection pause during probe
- Application is busy and cannot respond

Fix: make health check fast. Do not check downstream services in liveness probe. Use readiness probe for that.

**Cause three: Readiness probe failures**

Pod cannot accept traffic but is not killed (unlike liveness). However, if pod stays not-ready too long, deployment may consider it failed.

**Cause four: Node issues**

Node ran out of resources. Or node was drained for maintenance. Pods get evicted and rescheduled.

**Cause five: Signal kill from outside**

Someone ran `kubectl delete pod`. Or HorizontalPodAutoscaler scaled down.

### How to investigate

**Step one: Get pod status**

```bash
kubectl describe pod <pod>
kubectl get events --field-selector involvedObject.name=<pod>
```

Look for:
- `Last State: OOMKilled` → memory issue
- `Last State: Error` with exit code → crashed
- `Liveness probe failed` → probe issue
- `Evicted` → node pressure

**Step two: Get logs from previous container**

```bash
kubectl logs <pod> --previous
```

This shows logs from before the restart. Often the most useful info.

**Step three: Check resource usage**

```bash
kubectl top pod <pod>
```

If memory is near limit before restart, that is your cause.

**Step four: Check node**

```bash
kubectl describe node <node>
# Look for memory pressure, disk pressure
```

### How to fix

**Fix one: Right-size memory**

```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"  # JVM should use less than this
```

For Java: heap should be about 70% of container limit:
```
container limit: 1 Gi
java heap: 700 Mi
```
The other 300 Mi is for Metaspace, threads, native memory.

**Fix two: Tune liveness probes**

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness  # fast endpoint
    port: 8080
  initialDelaySeconds: 60     # give time to start
  periodSeconds: 30           # check every 30s
  timeoutSeconds: 5           # 5s timeout
  failureThreshold: 3         # 3 failures = restart
```

Use Spring Boot's separate liveness and readiness endpoints. Liveness should be very simple - just "am I running?"

**Fix three: Set proper JVM flags**

```yaml
env:
- name: JAVA_OPTS
  value: "-XX:MaxRAMPercentage=70 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"
```

`-XX:MaxRAMPercentage=70` makes JVM aware of container limits.

**Fix four: Use pod disruption budgets**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

Prevents too many pods from being killed at once during maintenance.

### Real example

A pod was restarting every 30 minutes. `kubectl describe` showed `Liveness probe failed`. Investigation:

- Health endpoint checked database connection
- During garbage collection (10 seconds), endpoint did not respond
- Probe timed out
- Pod was killed

Fix:
1. Used Spring Boot's `/actuator/health/liveness` (does not check database)
2. Used `/actuator/health/readiness` for database check (no auto-kill)
3. Increased liveness probe timeout to 10 seconds

No more random restarts. Database issues now reflected in readiness only (pod stops getting traffic but is not killed).

---

## 34.  Application Hangs During Shutdown

You send a shutdown signal. Application does not exit. Container takes 30 seconds to die. Why?

### What is happening

When the application receives shutdown signal (SIGTERM):

1. Spring Boot starts graceful shutdown
2. Stops accepting new requests
3. Waits for in-flight requests to complete
4. Closes connections, executors, etc.
5. Exits

If any step hangs, shutdown stalls.

### Common causes

**Cause one: Long-running requests**

In-flight request taking too long. Spring waits for it to complete.

**Cause two: Non-daemon threads still running**

```java
ExecutorService executor = Executors.newFixedThreadPool(10);
// these threads are non-daemon by default
// JVM will not exit until they finish
```

If you do not call `executor.shutdown()`, the threads keep running.

**Cause three: Unclosed external connections**

Open Kafka producers, database connections, HTTP clients holding resources.

**Cause four: Thread waiting on something that never happens**

```java
public void waitForEvent() {
    while (!eventArrived) {
        Thread.sleep(1000);  // never wakes up if shutdown
    }
}
```

**Cause five: Kubernetes terminationGracePeriodSeconds too short**

If shutdown takes 60 seconds but terminationGracePeriodSeconds is 30, Kubernetes sends SIGKILL after 30 seconds. Hard kill.

### How to debug

**Step one: Take thread dump during shutdown**

When app is hanging, get thread dump:

```bash
jstack <pid> > shutdown-dump.txt
```

Look for threads still active. Their stack traces tell you what is hanging.

**Step two: Check Spring shutdown logs**

Spring Boot logs each step. If logs stop at "Shutting down ExecutorService", that is your hang.

### How to fix

**Fix one: Configure graceful shutdown timeout**

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

After 30 seconds, force shutdown.

**Fix two: Close executors properly**

```java
@PreDestroy
public void cleanup() {
    executor.shutdown();
    try {
        if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
            executor.shutdownNow();  // force
        }
    } catch (InterruptedException e) {
        executor.shutdownNow();
    }
}
```

**Fix three: Make threads daemon if appropriate**

```java
ThreadFactory factory = r -> {
    Thread t = new Thread(r);
    t.setDaemon(true);  // does not prevent JVM exit
    return t;
};
ExecutorService executor = Executors.newFixedThreadPool(10, factory);
```

**Fix four: Make blocking operations interruptible**

```java
// Bad - cannot be interrupted
while (!shutdownRequested) {
    Thread.sleep(1000);
}

// Good - responds to interrupt
while (!Thread.currentThread().isInterrupted() && !shutdownRequested) {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        break;
    }
}
```

**Fix five: Set proper Kubernetes grace period**

```yaml
spec:
  terminationGracePeriodSeconds: 60  # default 30
```

Match this to your application's max shutdown time.

### Real example

A service took 5 minutes to shut down. Thread dump showed a background thread polling Kafka with infinite retry. The retry loop did not check for shutdown.

Fix:

```java
private volatile boolean running = true;

public void run() {
    while (running && !Thread.currentThread().isInterrupted()) {
        try {
            consumer.poll(Duration.ofSeconds(1));
        } catch (WakeupException e) {
            running = false;
        }
    }
}

@PreDestroy
public void shutdown() {
    running = false;
    consumer.wakeup();  // breaks out of poll
}
```

Shutdown now takes 2 seconds. Kubernetes is happy.

---

## 35.  High Memory Usage but No OutOfMemoryError

Your monitoring shows memory at 90%. But the application is not crashing. Should you worry?

### What might be happening

**Possibility one: This is normal**

JVM allocates memory aggressively. If you set `-Xmx4g`, JVM may use 4 GB even if it only "needs" 2 GB. This is not a leak.

**Possibility two: Heap is healthy, off-heap is full**

Java memory has multiple parts:

```
    JVM Memory Layout
    
    ┌────────────────────────────────────┐
    │              HEAP                  │ ← managed by GC
    │  (where most objects live)         │
    └────────────────────────────────────┘
    ┌────────────────────────────────────┐
    │           METASPACE                │ ← class metadata
    └────────────────────────────────────┘
    ┌────────────────────────────────────┐
    │         CODE CACHE                 │ ← JIT compiled code
    └────────────────────────────────────┘
    ┌────────────────────────────────────┐
    │      DIRECT MEMORY                 │ ← off-heap buffers
    │  (Netty, NIO, etc.)               │
    └────────────────────────────────────┘
    ┌────────────────────────────────────┐
    │      THREAD STACKS                 │ ← one per thread
    │   (each ~512 KB by default)        │
    └────────────────────────────────────┘
```

If heap is fine but native memory is huge, off-heap buffers might be leaking.

**Possibility three: Memory mapped files**

If you mmap large files, they show as memory usage but are actually disk-backed.

**Possibility four: Glibc memory fragmentation**

Linux's default memory allocator can fragment. Java releases memory but Linux keeps holding it.

### How to investigate

**Step one: Native Memory Tracking**

Add JVM flag:
```
-XX:NativeMemoryTracking=detail
```

Then query:
```bash
jcmd <pid> VM.native_memory summary
```

Output shows breakdown:
```
Total: reserved=4500MB, committed=3000MB
- Java Heap (reserved=2048MB, committed=2048MB)
- Class (reserved=200MB, committed=200MB)
- Thread (reserved=300MB, committed=300MB)
- Code (reserved=200MB, committed=100MB)
- GC (reserved=100MB, committed=100MB)
- Internal (reserved=50MB, committed=50MB)
- Other (reserved=1602MB, committed=202MB)
```

If "Other" or "Internal" is huge, you have native memory issues.

**Step two: Check direct memory**

```java
// Add monitoring
BufferPoolMXBean direct = ManagementFactory
    .getPlatformMXBeans(BufferPoolMXBean.class)
    .stream()
    .filter(b -> b.getName().equals("direct"))
    .findFirst()
    .orElseThrow();

System.out.println("Direct memory: " + direct.getMemoryUsed());
```

Set a limit:
```
-XX:MaxDirectMemorySize=1g
```

**Step three: Check thread count**

```bash
jstack <pid> | grep "^\"" | wc -l
```

Each thread uses about 512 KB by default. 10,000 threads = 5 GB of stack memory.

**Step four: Check Metaspace**

```bash
jcmd <pid> GC.heap_info
```

If Metaspace keeps growing, you have a class loader leak (often from frequent class generation, like in dynamic proxies).

### How to fix

**Fix one: Set memory limits explicitly**

```
-Xmx2g                           # max heap
-XX:MaxMetaspaceSize=512m        # max metaspace
-XX:MaxDirectMemorySize=512m     # max direct memory
-Xss256k                         # stack size per thread (smaller)
```

This prevents memory from growing unbounded.

**Fix two: Tune for container memory**

```
-XX:MaxRAMPercentage=70
```

JVM uses 70% of container memory for heap, leaving room for other JVM memory.

**Fix three: Reduce thread count**

Use thread pools instead of creating threads.

**Fix four: For Netty / NIO heavy apps, monitor direct buffers**

Netty allows pooled vs unpooled buffers. Check pool stats.

**Fix five: Try alternative allocator (Linux)**

```bash
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 java -jar app.jar
```

jemalloc handles fragmentation better than glibc malloc for some workloads.

### Real example

A service used 8 GB out of 10 GB container limit. Heap was only 4 GB. Where was the other 4 GB?

Investigation with Native Memory Tracking:
- Heap: 4 GB (expected)
- Metaspace: 200 MB (fine)
- Threads: 2 GB (problem!)
- Direct memory: 1.5 GB (problem!)

Two issues:
1. 4000 threads from misconfigured executor (4000 × 512 KB = 2 GB)
2. Netty direct buffers not being released

Fixes:
1. Reduced thread pool from 4000 to 200
2. Properly released ByteBuf in Netty handlers
3. Set `-XX:MaxDirectMemorySize=512m` to catch leaks early

Memory dropped to 5 GB total. Plenty of headroom.

### Key lesson

`OutOfMemoryError: Java heap space` is just one type. Other memory areas can run out too. Monitor all of them.

---

After 10 years of debugging production systems, here is my advice:

### 1. Logs Are Your Best Friend

If you don't have good logs, you cannot debug production. Spend time on logging.

- Log enough but not too much
- Always log request IDs
- Log slow operations
- Don't log sensitive data
- Use structured logging (JSON)

### 2. Metrics Tell the Story

Set up dashboards for:
- CPU and memory
- Request rate and error rate
- Latency (especially p99)
- Database connections
- Thread pool stats
- Garbage collection

If you cannot see it, you cannot fix it.

### 3. Always Have a Way to Capture State

When things go wrong, you need:
- Heap dumps
- Thread dumps
- Profiling data

Have automation to capture these on errors.

### 4. Practice Production Debugging

Most engineers panic when production breaks. Practice in non-production:
- Inject failures intentionally (chaos engineering)
- Run game days where you simulate problems
- Build runbooks for common issues

### 5. Think in Systems, Not Code

In production, problems are rarely just code. They involve:
- Network
- Database
- Operating system
- External services
- Configuration
- Hardware

Learn each layer.

### 6. Start Simple

When debugging, check simple things first:
- Is the service running?
- Is the database reachable?
- Is the disk full?
- Are the credentials correct?

You'll be amazed how often it's something simple.

### 7. Make It Reproducible

If you cannot reproduce a bug, you usually cannot fix it. Spend time figuring out the exact conditions that cause the issue.

### 8. Document Your Findings

After every incident:
- Write a postmortem
- Document the root cause
- Create runbook for next time
- Add monitoring/alerts to detect early

This builds team knowledge over time.

### 9. Stay Calm

When production is down:
- Take a deep breath
- Communicate clearly
- Don't make rushed changes
- Roll back first, debug later
- Get help if needed

## Quick Tools Reference

| Tool | What It Does | When to Use |
|------|--------------|-------------|
| jstack | Thread dump | Debugging deadlocks, blocked threads |
| jmap | Heap dump | Memory analysis |
| jstat | Garbage collection stats | GC tuning |
| jcmd | Various JVM commands | All-purpose tool |
| VisualVM | Visual debugging | Interactive analysis |
| Eclipse Memory Analyzer | Heap analysis | Finding memory leaks |
| async-profiler | Low-overhead profiling | Production profiling |
| Java Flight Recorder | Built-in profiler | Long-running profiling |
| jconsole | Monitoring | Quick checks |
| top -H | Thread CPU usage | Finding hot threads |
| netstat | Network connections | Debugging network issues |
| iostat | Disk I/O | Disk performance |
| dmesg | System messages | Out of memory, crashes |
| lsof | Open files | File handle leaks |

---
