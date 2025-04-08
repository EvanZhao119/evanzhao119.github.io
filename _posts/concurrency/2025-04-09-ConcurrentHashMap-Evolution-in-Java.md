---
layout: post
title: "`ConcurrentHashMap` Evolution in Java"
date: 2025-04-09
categories: concurrency
published: true
---

# `ConcurrentHashMap` Evolution in Java
`ConcurrentHashMap` is a high-performance, thread-safe Map implementation from the java.util.concurrent package.

## Evolution in Jdk7 and Jdk8

### JDK 7: Segmented Locking
In JDK 7, `ConcurrentHashMap` **adopted lock** striping (a form of segmented locking).
- Internally uses a `Segment[] segments` array, where each segment manages a subset of the key space and has its own lock.
- Allows up to 16 threads (by default) to write concurrently to different segments.
- Each segment is a reentrant lock and holds a small hash table of entries.

```java
public V put(K key, V value) {
    Segment<K,V> s;
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

### JDK 8+: CAS + synchronized + Red-Black Trees
JDK 8 restructured `ConcurrentHashMap`.
- Replacing `Segment[]`` with a simple `Node[] table`.
- Using CAS (Compare-And-Swap) to insert into empty bins without locking.
- Using `synchronized` to lock individual bins when necessary.
- **Treeifying** bins (to red-black trees) when the number of items in a bin exceeds a threshold (default: 8).

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        ...
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break; // CAS avoids locking for empty bin
        } 
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);//Help resize and treeify
        else {
            synchronized (f) { //Lock only first node of bin
                ...
            }
        }
    }
}
```

**JDK 8 completely removed the Segment class. CAS + fine-grained synchronized blocks significantly improved performance. Treeification further improved lookup speed for highly-collided keys.**

### Resizing in JDK 7 vs JDK 8
#### JDK 7 Resizing
- Each `Segment` resizes independently.
- Parallel resizing across segments is possible but not coordinated.

#### JDK 8 Resizing
- Introduces a `ForwardingNode` to indicate in-progress resizing in a bin.
- If a thread encounters a `ForwardingNode`, it helps complete the resize (`helpTransfer()`).

```java
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f); // Assist with resizing
```

In JDK8, if a thread sees that a bin is being resized, it doesn’t just wait. It actively helps with the resizing process, which reduces blocking and speeds things up.

## Additional Enhancements in JDK 9 to JDK 21
While the core structure introduced in JDK 8 remains intact, newer JDKs have introduced several performance and usability improvements.
1. **Compact Memory Layout**

- Reduced padding and redundant fields to improve CPU cache friendliness.

2. **False Sharing Mitigation**

- Use of `@Contended` (requires `-XX:-RestrictContended` flag) to isolate hot fields and prevent false sharing between threads.

3. **Optimized Tree Handling**

- Improved treeification/de-treeification logic for performance.

4. **Enhanced Parallelism Support**

- Added `forEach`, `reduce`, and `search` methods for parallel computation using ForkJoinPool.

In the future, Java will keep on optimizing `ConcurrentHashMap`.
