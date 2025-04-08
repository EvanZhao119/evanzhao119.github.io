---
layout: post
title: "The Secret Sauce Behind `ThreadPoolExecutor` in Java"
date: 2025-04-09
categories: concurrency
published: true
---

# The Secret Sauce Behind `ThreadPoolExecutor` in Java

## 1. Thread Pool Usage Example

```java
int nThreads = 10;
ExecutorService exec = Executors.newFixedThreadPool(nThreads);
Runnable task = new Runnable() {
    public void run() {
        // do something
    }
};
exec.execute(task);
```

### Explanation
- `Executors.newFixedThreadPool()` returns a `ThreadPoolExecutor` backed by a `LinkedBlockingQueue`.
- Similarly, `Executors.newSingleThreadExecutor()` also uses `LinkedBlockingQueue`.
- In contrast, `Executors.newCachedThreadPool()` uses a `SynchronousQueue` instead.

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

## 2. `ThreadPoolExecutor.execute()` Workflow

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();

    // Step ①: Create a new thread if below corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // Step ②: Try queuing the task
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }

    // Step ③: Reject or attempt to add thread again
    else if (!addWorker(command, false))
        reject(command);
}
```

## 3. Task Execution with `addWorker()` and `Worker.run()`
### Workflow Summary
1. The task is wrapped in a `Worker` object.
2. A thread is created from the `Worker`.
3. The thread is started and executes the task.
```java
Worker w = new Worker(firstTask); // Wrap the task
final Thread t = w.thread;
mainLock.lock(); // Ensure thread safety
...
t.start(); // Launch the thread
```

### Worker Constructor
```java
Worker(Runnable firstTask) {
    setState(-1); 
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```
Upon calling `t.start()`, the thread's `run()` method calls `runWorker()`.

## 4. Main Execution Loop: `runWorker()`
The `runWorker()` method is the core execution loop for a worker thread in `ThreadPoolExecutor`. 
- **Runs the first task** if present.
- **Continuously retrieves tasks** from the queue using `getTask()`
    - If a task is found, it locks the worker, runs the task, then unlocks.
    - If no task is available, the thread blocks or times out, depending on `keepAliveTime`.
- **Handles shutdowns** by checking pool state and interrupting the thread if needed.
- **Calls hooks** (`beforeExecute` and `afterExecute`) before and after each task for monitoring or customization.
- **Exits** when no tasks are available or the pool is shutting down, and calls `processWorkerExit()` for cleanup.

> This loop allows thread reuse by keeping workers alive and waiting for tasks, reducing thread creation overhead.

> Sometimes we can hook into the thread pool by overriding beforeExecute() and afterExecute() methods to log or record metrics about task execution — such as task start time, end time, duration, or exceptions.

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();
    boolean completedAbruptly = true;

    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            try {
                beforeExecute(wt, task);
                task.run();
                afterExecute(task, null);
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

## 5. Task Retrieval: `getTask()`
This mechanism ensures worker threads block when no task is available, enabling efficient thread reuse.
```java
Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();

boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
    if (compareAndDecrementWorkerCount(c))
        return null;
}
```

## 6. Key ThreadPoolExecutor Parameters

| Parameter               | Description                                                        |
|-------------------------|--------------------------------------------------------------------|
| `corePoolSize`          | Minimum number of threads to keep alive (even when idle)          |
| `maximumPoolSize`       | Maximum number of concurrently active threads                     |
| `keepAliveTime`         | Idle time before terminating excess threads                       |
| `allowCoreThreadTimeOut`| If `true`, allows core threads to time out as well                |

## 7. Rejection Policies (`RejectedExecutionHandler`)
When the thread pool and its queue are full, the `RejectedExecutionHandler` defines how new tasks are handled.

| Policy               | Behavior                                                                 |
|----------------------|--------------------------------------------------------------------------|
| `AbortPolicy`         | Rejects the task and throws `RejectedExecutionException` *(default)*    |
| `DiscardPolicy`       | Silently discards the task                                              |
| `DiscardOldestPolicy` | Discards the oldest task in the queue, then retries the new task        |
| `CallerRunsPolicy`    | Runs the task in the thread of the caller (the one that invoked `execute()`) |

### Example

```java
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

## 8. How to Set Thread Pool Size
Choosing the right thread pool size depends on the nature of tasks.

| Scenario        | Recommended Thread Count                              |
|------------------|-------------------------------------------------------|
| CPU-intensive    | `CPU core count + 1`                                  |
| IO-intensive     | `CPU core count * (1 + wait time / compute time)`     |

> CPU-bound tasks require fewer threads to keep all cores busy without overhead.  

> IO-bound tasks can use more threads to compensate for idle waiting time.

> Maybe use benchmarking tools like **[JMH (Java Microbenchmark Harness)](https://openjdk.org/projects/code-tools/jmh/)** to measure actual workload performance and fine-tune the thread pool size.

## 9. Latest
JDK 21 introduces **virtual threads** for lightweight concurrency. We can use the blow code instead.
```java
ExecutorService vExecutor = Executors.newVirtualThreadPerTaskExecutor();
vExecutor.submit(() -> {
    // Run logic in a virtual thread
});
