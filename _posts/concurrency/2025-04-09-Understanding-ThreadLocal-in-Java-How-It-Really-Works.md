---
layout: post
title: "Understanding `ThreadLocal` in Java: How It Really Works"
date: 2025-04-09
categories: concurrency
published: true
---

# Understanding `ThreadLocal` in Java: How It Really Works
`ThreadLocal` provides thread-local variables. Each thread accessing a `ThreadLocal` variable maintains an independent copy of the variable, preventing issues caused by shared mutable state across threads.

### From the official definition:
> ThreadLocal provides get and set accessor methods that maintain a separate copy of the value for each thread that uses it, so a get returns the most recent value passed to set from the currently executing thread.

This design helps isolate state per thread, commonly used in scenarios such as database connection handling.

## 1. How ThreadLocal Works
### 1.1 `set()` Method
```java
public void set(T value) {
    // Get the current thread
    Thread t = Thread.currentThread();
    // Retrieve the ThreadLocalMap associated with the current thread
    ThreadLocalMap map = getMap(t);
    // If the map exists, store the value using the current ThreadLocal as the key
    if (map != null)
        map.set(this, value);
    else
        // Otherwise, create a new map
        createMap(t, value);
}
```

### 1.2 `get()` Method
```java
public T get() {
    // Get the current thread
    Thread t = Thread.currentThread();
    // Retrieve the ThreadLocalMap associated with the current thread
    ThreadLocalMap map = getMap(t);
    // If the map is not null, 
    // look up the Entry using the current ThreadLocal as the key and return the value
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // If not found, initialize the value (returns null by default)
    return setInitialValue();
}
```
From the above `set` and `get` method implementations,
- The data structure used to store values—`ThreadLocalMap`—belongs to the `Thread` object itself.
- The key in the `ThreadLocalMap` is the current `ThreadLocal` instance.

The ThreadLocal instance, when calling its own `set()` or `get()` method, uses itself as the key, and stores the corresponding value in the `ThreadLocalMap` of the current thread. Furthermore, the `ThreadLocalMap` stores both the key and the value in an `Entry` structure.

## 2. `Thread`'s `ThreadLocalMap` structure
### `set()` Method of `ThreadLocalMap`
The `set()` method stores the `key-value` pair in an `Entry` structure and places it into the internal array `table`. It uses the hash value of the `ThreadLocal` object to determine the appropriate bucket index.
```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // Calculate the hash to find the appropriate bucket
    int i = key.threadLocalHashCode & (len - 1);
    // Traverse the array using linear probing
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // If the key already exists, replace the value
        if (k == key) {
            e.value = value;
            return;
        }
        // If the key has been garbage collected (null), replace the stale entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // Otherwise, insert the new Entry at the empty slot
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // Resize if necessary
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

#### Explanation
- The `key` and `value` are wrapped in an Entry structure.
- `ThreadLocalMap` is implemented as an array of `Entry` objects.
- It uses open addressing with linear probing to resolve hash collisions.
    - If the same key already exists, it updates the value.
    - If it finds a stale entry (whose key is null), it reuses the slot.
    - Otherwise, it continues probing to find the next available slot.
- When the number of entries exceeds the threshold, the table is resized.

## 3. WeakReference in `ThreadLocalMap.Entry`
The `Entry` class is a static inner class within `ThreadLocalMap`, and it **extends `WeakReference<ThreadLocal<?>>`**. In the constructor, the `ThreadLocal` key is stored as a weak reference.

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
    ... 
}
```

#### Explanation
- A **weak reference** does not prevent its referent (in this case, the `ThreadLocal` object) from being garbage collected.
- This design ensures that if a `ThreadLocal` key becomes unreachable (i.e., has no strong references pointing to it), it **can be garbage collected**, even though the associated value is still stored in the map.
- The use of a weak reference helps prevent **memory leaks** caused by threads holding on to `ThreadLocal` values longer than necessary.
- When the `ThreadLocal` key is reclaimed, the `ThreadLocalMap` will eventually detect that the key is `null` and clean up the corresponding entry using the `replaceStaleEntry()` method.
- However, if the `ThreadLocal` instance is still strongly referenced elsewhere, it will not be collected, and the map remains accessible.

This weak reference mechanism is a key part of the memory management strategy in `ThreadLocalMap`, balancing usability with garbage collection safety.
