```
/*
 * Segmented Non-Blocking LRU Cache (Lock-striping)
 *
 * Design goals:
 *  - Reduce contention by sharding keyspace into segments
 *  - Each segment holds its own LRU (LinkedHashMap with accessOrder=true)
 *  - Global capacity is divided evenly across segments
 *  - Provide strong LRU semantics within each segment (approximate global LRU)
 *  - Operations are thread-safe with a per-segment ReentrantLock
 *
 * Notes:
 *  - This is NOT a fully lock-free LRU (those are extremely complex).
 *    Instead it is a practical, production-friendly approach that gives
 *    near-linear throughput improvement with more segments.
 *  - For stricter low-latency workloads consider using Caffeine or a
 *    ConcurrentLinkedHash alternative.
 */

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.*;

public class SegmentedLRUCache<K, V> {
    private final int segmentMask;
    private final Segment<K,V>[] segments;
    private final int segmentCount;

    @SuppressWarnings("unchecked")
    public SegmentedLRUCache(int capacity, int requestedSegments) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity>0");
        int segmentsPow2 = 1;
        while (segmentsPow2 < requestedSegments) segmentsPow2 <<= 1; // power of two
        this.segmentCount = Math.min(segmentsPow2, capacity); // no more segments than capacity
        this.segmentMask = this.segmentCount - 1;

        // Divide capacity approximately equally
        int perSegment = Math.max(1, capacity / this.segmentCount);

        segments = (Segment<K,V>[]) new Segment[this.segmentCount];
        for (int i = 0; i < this.segmentCount; i++) {
            segments[i] = new Segment<>(perSegment);
        }
    }

    private final int hash(Object key) {
        int h = key.hashCode();
        // spread bits (similar to ConcurrentHashMap)
        h ^= (h >>> 16);
        return h;
    }

    private final Segment<K,V> segmentFor(Object key) {
        int h = hash(key);
        int idx = h & segmentMask; // because segmentCount is power of two
        return segments[idx];
    }

    /** Strong get: obtains lock and promotes to MRU on access */
    public V get(K key) {
        if (key == null) return null;
        return segmentFor(key).get(key);
    }

    /** Put or update value */
    public void put(K key, V value) {
        if (key == null) throw new NullPointerException("key");
        segmentFor(key).put(key, value);
    }

    /** Remove key */
    public V remove(K key) {
        if (key == null) return null;
        return segmentFor(key).remove(key);
    }

    /** Approximate size (sum of segments) */
    public int size() {
        int sum = 0;
        for (Segment<K,V> s : segments) sum += s.size();
        return sum;
    }

    /** Return snapshot keys in LRU->MRU order per segment */
    public List<K> keysInLRUOrder() {
        List<K> res = new ArrayList<>();
        for (Segment<K,V> s : segments) {
            res.addAll(s.keysInLRUOrder());
        }
        return res;
    }

    // Inner Segment with LinkedHashMap for LRU
    private static final class Segment<K,V> {
        private final ReentrantLock lock = new ReentrantLock();
        private final int capacity;
        // Using LinkedHashMap with accessOrder=true to maintain LRU within segment
        private final LinkedHashMap<K,V> map;

        Segment(int capacity) {
            this.capacity = capacity;
            // initial capacity slightly larger to avoid resize cost
            this.map = new LinkedHashMap<K,V>(Math.max(4, capacity), 0.75f, true) {
                @Override
                protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
                    return size() > Segment.this.capacity;
                }
            };
        }

        V get(K key) {
            lock.lock();
            try {
                return map.get(key); // accessOrder = true will promote
            } finally {
                lock.unlock();
            }
        }

        void put(K key, V value) {
            lock.lock();
            try {
                map.put(key, value);
            } finally {
                lock.unlock();
            }
        }

        V remove(K key) {
            lock.lock();
            try {
                return map.remove(key);
            } finally {
                lock.unlock();
            }
        }

        int size() {
            lock.lock();
            try {
                return map.size();
            } finally {
                lock.unlock();
            }
        }

        List<K> keysInLRUOrder() {
            lock.lock();
            try {
                // LinkedHashMap iterator is LRU->MRU when accessOrder=true
                return new ArrayList<>(map.keySet());
            } finally {
                lock.unlock();
            }
        }
    }

    // ---------------------
    // Demo + Simple concurrency test
    // ---------------------
    public static void main(String[] args) throws Exception {
        final SegmentedLRUCache<Integer, String> cache = new SegmentedLRUCache<>(100, 8);

        // Warm-up: insert
        for (int i = 0; i < 100; i++) cache.put(i, "v" + i);
        System.out.println("Initial size: " + cache.size());

        // Concurrent access test
        int threads = 16;
        ExecutorService ex = Executors.newFixedThreadPool(threads);
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done = new CountDownLatch(threads);
        for (int t = 0; t < threads; t++) {
            final int id = t;
            ex.submit(() -> {
                try {
                    start.await();
                    Random r = new Random();
                    for (int i = 0; i < 2000; i++) {
                        int key = r.nextInt(150);
                        if ((key + id) % 5 == 0) {
                            cache.put(key, "v" + key + "-t" + id);
                        } else {
                            cache.get(key);
                        }
                    }
                } catch (InterruptedException ignored) {
                } finally {
                    done.countDown();
                }
            });
        }
        long s = System.nanoTime();
        start.countDown();
        done.await();
        long e = System.nanoTime();
        ex.shutdown();
        System.out.printf("Concurrent ops done in %.3f ms, final size=%d\n", (e - s) / 1_000_000.0, cache.size());

        // Print a few keys snapshot
        List<Integer> keys = cache.keysInLRUOrder();
        System.out.println("Snapshot (per-segment LRU order, segment concatenation) count=" + keys.size());
        System.out.println(keys.subList(0, Math.min(30, keys.size())));
    }
}

```

1) File header & design summary

Purpose: implement an LRU cache that reduces contention by sharding the keyspace into segments.

Trade-off: strong LRU per-segment, approximate global LRU. This gives far better concurrency than a single global lock while keeping implementation straightforward.

Why this design is chosen:

Fully lock-free LRU is extremely hard and bug-prone (ABA, complex pointer CAS, hazard pointers).

Lock-striping (segmentation) is pragmatic: scale reads/writes with segments, each segment independently maintains LRU via LinkedHashMap.

2) Fields: segmentMask, segments[], segmentCount

segmentCount is power-of-two to allow fast mapping from a hashed key to a segment using a bitmask (h & segmentMask).

segmentMask is segmentCount - 1. This avoids modulo and is CPU-friendly.

segments is an array of segment objects, each with its own lock + LRU map.

Pitfall: If segmentCount is not power-of-two, the bitmask selection will break — the constructor ensures power-of-two.

3) Constructor (SegmentedLRUCache(int capacity, int requestedSegments))

Validates capacity > 0.

Rounds requestedSegments up to the next power-of-two. Also caps number of segments by capacity to avoid segments with zero capacity.

Divides the capacity approximately equally across segments (perSegment = capacity / segmentCount, floored to at least 1).

Initializes each Segment with its per-segment capacity.

Important notes:

Per-segment capacity is fixed. If capacity not divisible by segmentCount, some slack remains or some segments may be slightly smaller.

Hot-key skew: if one key-space is hot, that segment may see higher contention — you can mitigate by increasing segmentCount (subject to capacity) or by using a hashing scheme that spreads hot keys.

4) hash(Object key)

Computes h = key.hashCode() then mixes high bits into low bits with h ^= (h >>> 16).

This is a standard technique (same idea as ConcurrentHashMap) to reduce clustering for poorly distributed hashCodes.

Outcome: better spread across segments.

Pitfall: The code assumes non-null keys (public API enforces null checks). If null keys were allowed, you'd need a special bucket.

5) segmentFor(Object key)

Uses the mixed hash and h & segmentMask to pick a segment index.

Because segmentCount is a power-of-two, this is equivalent to h % segmentCount but faster.

Edge cases:

Negative hash codes are fine because bitmasking handles the bits; the bitwise operations yield a usable index.

6) Public API: get(K key)

Delegates to the segment's get, which acquires the segment lock, calls map.get(key), and returns the value.

Because LinkedHashMap was created with accessOrder=true, calling get moves the entry to MRU (most recently used) inside that segment.

Concurrency implications:

get is not lock-free: it locks the segment. This keeps implementation simple and guarantees correct promotion to MRU under the segment lock.

In read-heavy workloads, segment locks can still contend. Possible future optimization: a read-optimized path (e.g., ReadWriteLock or optimistic read) or an asynchronous promotion where get reads without lock and enqueues a promotion task.

7) Public API: put(K key, V value)

Delegates to segment's put, which locks and performs map.put. LinkedHashMap automatically evicts eldest if removeEldestEntry returns true, which here is when size > capacity.

Eviction happens under the segment lock, guaranteeing atomicity of the put+evict operation for that segment.

Important:

Eviction is per-segment only. There is no global eviction policy; thus, an item in one segment will not be evicted due to usage in another segment. This yields "approximate" global LRU.

Suggestion: add an eviction listener (callback) that gets called under lock to log or free resources.

8) Public API: remove(K key)

Locks the relevant segment and removes the key, returning the removed value.

O(1) under segment lock.

Suggestion: You could add removeIfPresent(K key, V expectedValue) for atomic compare-and-remove semantics to support conditional deletion patterns.

9) size() and keysInLRUOrder()

size() iterates all segments and sums sizes (acquires each lock in turn). This is an approximate snapshot because other threads may be mutating concurrently.

keysInLRUOrder() returns a concatenation of per-segment LRU orders. Important to document: this is not strict global LRU order; it's per-segment LRU concatenated.

Pitfall:

While summing sizes or collecting keys, locks are taken segment-by-segment. If you want a consistent snapshot across the whole cache, you'd need a global freeze (bad for throughput). Usually, approximate snapshots are acceptable.

10) Inner Segment class — fields & constructor

Each Segment has:

a ReentrantLock used to guard access to the LinkedHashMap.

a per-segment capacity.

a LinkedHashMap with accessOrder=true and a custom removeEldestEntry that keeps size <= capacity.

Why LinkedHashMap:

It provides O(1) get/put and maintains insertion/access order. With accessOrder=true, get causes the entry to move to MRU. It also allows a simple way to evict eldest entry by overriding removeEldestEntry.

Constructor details:

Initial capacity is chosen to reduce early resizing; Math.max(4, capacity) is used.

11) Segment methods: get, put, remove, size, keysInLRUOrder

Each method wraps a lock.lock() / lock.unlock() pair (including in finally) to ensure the lock is released even on exceptions.

get returns map.get(key) — this also promotes the key to MRU.

put performs map.put(key, value); eviction is handled by removeEldestEntry.

remove calls map.remove(key).

size returns map size under lock.

keysInLRUOrder iterates map.keySet() and returns a new ArrayList — because LinkedHashMap iterates in LRU->MRU order when using accessOrder=true.

Concurrency notes:

These per-segment locks make operations atomic at segment level. There’s no global lock hence multiple threads accessing different segments proceed concurrently.

12) Demo main method — warmup + concurrent test

Warm-up fills the cache.

Then uses an ExecutorService with multiple threads to run mixed get/put operations on random keys.

It uses CountDownLatch to start all threads simultaneously for a fair stress test and measure elapsed time.

After finishing, it prints runtime and a snapshot of keys (per-segment concatenated order).

Why useful:

Quick smoke test that the cache behaves under concurrency and remains consistent (no exceptions, size bounded).

Not a replacement for benchmarks; for performance profiling, use JMH.

13) Performance characteristics & complexity

get, put, remove: O(1) amortized within segment due to LinkedHashMap operations.

Contention: If many threads hit the same segment (hot keys), they'll contend on that segment's lock. With enough segments and a good hash distribution, contention reduces roughly proportional to segmentCount.

Memory: Each segment holds its own LinkedHashMap entries. Overhead compared to a single map includes per-segment structures.

14) Limitations, edge-cases, and mitigation strategies

Not strictly global LRU: Hot items from different segments may outrank cold items in other segments. Acceptable in many real-world caches; if you need strict global LRU, heavier synchronization or more complex algorithms are required.

Hot-key bottleneck: If one segment gets most traffic, it will still be a bottleneck. Mitigations:

Increase number of segments (if capacity allows)

Use hashed key transformation to better spread hot keys

Use a more advanced cache like Caffeine that handles hotspots and admission policies

Fixed per-segment capacity: Resizing or dynamic rebalancing across segments is not implemented — could be added later but is complex.

Eviction listener: Not implemented; useful to release external resources on eviction — add a callback executed under the segment lock, or enqueue to an executor to avoid blocking.

15) Improvements & production hardening ideas

Eviction listener: async or sync callback for removed entries. If async, you must be careful about ordering vs resource cleanup.

Per-key weights & size-based eviction: instead of count, track entry weights and evict by weight. More complex but useful for variable-sized objects.

Metrics: per-segment hit/miss counters, lock wait times, ops/sec. Expose via Micrometer/Prometheus.

Optimistic read path: use ReadWriteLock or try lock-free read + async promotion. Example: read under no lock using ConcurrentHashMap and then schedule a promotion into the LRU list via a background executor — gives eventual promotion and lower read latency.

Adaptive segments / rebalancer: detect hot segments and rehash keys or move capacity; complex but mitigates hotspots.

JMH benchmarking: measure throughput and latency across workloads (heavy reads, heavy writes, mixed) and vary segmentCount to find the sweet spot.

16) Interview points — what to say about this design

If asked in an interview, say:

“We chose segmentation to reduce lock contention while keeping implementation simple and maintainable. This gives per-segment strong LRU and an approximate global LRU. It’s what many caches do when you need a pragmatic, well-understood trade-off. For even higher performance and better hit rates, libraries like Caffeine can be used; they provide advanced eviction/admission policies and near-lock-free reads.”

Prepare to discuss:

Hot-key handling, eviction accuracy, and how to extend for TTL/weights.

Why not fully lock-free: complexity (ABA, hazard pointers), debugging difficulty.

How metrics and operational visibility are added.