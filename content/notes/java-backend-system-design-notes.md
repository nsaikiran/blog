---
layout: post
title:  "Java · Concurrency & I/O Core · Caching"
description: "Notes on Java, concurrency & I/O Core, Caching"
categories: ["java,system-design"]
date: 2026-08-26 19:45:31 +0530
author: "Sai Kiran"
comments: false
---


# Java · Concurrency & I/O Core · Caching

**Java**

1. **Object identity** — `equals()` / `hashCode()` (the base contract every collection relies on)
2. **Collections** — what they are, how implementations differ, and `HashMap` internals
3. **Exceptions** — the error-handling model
4. **Functional Java** — functional interfaces, lambdas, and Streams

**Concurrency & I/O core**

5. **Concurrency fundamentals** — threads, the two core problems, and how to guard shared state
6. **Web request handling** — Servlets, Spring Boot, and the thread-per-request model
7. **(Advanced) Scaling I/O** — blocking I/O, the event-loop model, virtual threads, and reactive programming

**Caching**

8. **Caching** — why, strategies, cache stampede, and a full worked example
9. **External dependencies** — failure modes (backpressure, network failure) and resiliency patterns

---

## Part 1 — `equals()` and `hashCode()`

Both methods are inherited from `Object`. Together they define what it means for two
objects to be "the same," which is the foundation the entire Collections Framework
depends on.

### What each does

- **`equals(Object o)`** decides *logical equality*. The default implementation from
  `Object` compares references (`this == o`) — two objects are equal only if they are
  literally the same instance in memory. You override it to compare *contents*.
- **`hashCode()`** returns an `int` used as a fast "bucket" identifier. Hash-based
  collections use it to decide where to store an object so it can be found again quickly.

### The contract (why they are tied together)

> **If two objects are equal according to `equals()`, they must return the same `hashCode()`.**

The reverse is **not** required: two unequal objects *may* share a hash code — this is a
**collision**, and it is allowed.

This matters because `HashMap`, `HashSet`, and `Hashtable` locate objects in two steps:

1. Use `hashCode()` to find the right **bucket**.
2. Use `equals()` to find the **exact object** within that bucket.

If you override `equals()` but forget `hashCode()`, "equal" objects can land in different
buckets and the collection will fail to find them.

### Example

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                              // same reference
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age &&
               java.util.Objects.equals(name, person.name);
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(name, age);               // SAME fields as equals
    }
}
```

Both methods use the **same fields** (`name`, `age`). That consistency is what keeps the
contract intact.

### What breaks without it

```java
Set<Person> set = new HashSet<>();
set.add(new Person("Alice", 30));

// With equals + hashCode overridden: true
// Without them: false, because it is a different object reference
System.out.println(set.contains(new Person("Alice", 30)));
```

### The full contract

**`equals()`** must be:

- **Reflexive** — `x.equals(x)` is true
- **Symmetric** — `x.equals(y)` implies `y.equals(x)`
- **Transitive** — `x.equals(y)` and `y.equals(z)` implies `x.equals(z)`
- **Consistent** — repeated calls return the same result
- `x.equals(null)` must be **false**

**`hashCode()`** must:

- Return the same value on repeated calls (as long as the object does not change)
- Return equal values for equal objects

**Practical tip:** always override both together, use the same fields in each, and let
your IDE or `java.util.Objects` helpers generate them.

### Default implementation of `hashCode()`

- The `Object.hashCode()` method is declared **`native`** — implemented in the JVM (C++),
  not in Java.
- Classically described as returning a value **derived from the object's memory address**
  (an **identity hash code**), so distinct objects almost always differ.
- In practice, modern JVMs (HotSpot) do **not** use the raw address, because the garbage
  collector moves objects, which would break the "consistent across calls" rule. HotSpot
  typically computes the value **lazily on first call** and stores it in the object header
  (the mark word). The exact algorithm has varied (a pseudo-random generator historically;
  newer versions may use a per-thread seed) — it is an implementation detail you should not
  rely on.
- The default is an **identity** hash — unrelated to any field. That is exactly why you
  must override it when your `equals()` is field-based.
- `System.identityHashCode(obj)` returns the original identity hash for *any* object, even
  one that overrides `hashCode()` — useful for debugging.

```java
Object a = new Object();
System.out.println(a.hashCode());               // e.g. 1854778591
System.out.println(System.identityHashCode(a)); // same value

Person p = new Person("Alice", 30);
System.out.println(p.hashCode());               // field-based (overridden)
System.out.println(System.identityHashCode(p)); // identity hash (different)
```

### Records (Java 16+; previewed 14–15)

A record auto-generates `equals()`, `hashCode()`, `toString()`, and accessors based on
**all** components.

```java
public record Person(String name, int age) {}
```

```java
Person p1 = new Person("Alice", 30);
Person p2 = new Person("Alice", 30);
System.out.println(p1.equals(p2));                    // true
System.out.println(p1.hashCode() == p2.hashCode());   // true
```

- Generated `equals()` uses each component's own `equals()`, so it works for `String` and
  other objects, not just primitives.
- You *can* override the generated methods, but rarely should — if you want to, a record
  may be the wrong tool.
- Records are the recommended default for simple immutable data carriers precisely because
  they eliminate the risk of an inconsistent hand-written `equals`/`hashCode` pair.

---

## Part 2 — The Collections Framework

Interfaces describe *what* a collection does; concrete classes decide *how* (which
determines performance, ordering, null-handling, and thread-safety).

### The interface map

`Collection` is the root for groups of elements:

- **`List`** — ordered, allows duplicates, indexed access.
  - `ArrayList` — resizable array; default choice.
  - `LinkedList` — doubly-linked list.
- **`Set`** — no duplicates.
  - `HashSet` — unordered, backed by a `HashMap`.
  - `LinkedHashSet` — insertion order.
  - `TreeSet` — sorted (red-black tree, O(log n)).
- **`Queue` / `Deque`** — ordered processing.
  - `ArrayDeque` — stacks and queues.
  - `PriorityQueue` — heap-ordered.

**`Map`** is **not** a `Collection` (key–value pairs, not single elements) but is part of
the framework: `HashMap` (workhorse), `LinkedHashMap` (order), `TreeMap` (sorted keys).

### Quick decision guide

```
Need key→value?          → Map  (HashMap default)
Need uniqueness?         → Set  (HashSet default)
Need order + duplicates? → List (ArrayList default)
Need sorting?            → TreeSet / TreeMap
Need insertion order?    → LinkedHashSet / LinkedHashMap
```

Utility class **`Collections`** (with the *s*) provides static helpers: `sort()`,
`unmodifiableList()`, `emptyList()`, etc. Modern code often prefers factory methods
`List.of()`, `Set.of()`, `Map.of()` for compact immutable collections.

### Map implementations compared

| | HashMap | LinkedHashMap | TreeMap | ConcurrentHashMap | Hashtable |
|---|---|---|---|---|---|
| Structure | hash table | hash table + linked list | red-black tree | segmented / CAS hash table | synchronized hash table |
| Ordering | none | insertion (or access) | sorted by key | none | none |
| get/put | O(1) avg | O(1) avg | O(log n) | O(1) avg | O(1) avg |
| Null key | one | one | no | no | no |
| Thread-safe | no | no | no | yes | yes (coarse) |
| Pick when | default | need order / LRU | need sorting/ranges | concurrent access | never (legacy) |

**HashMap** — array of buckets indexed by `hashCode()`; no ordering guarantee; O(1)
average, degrading to O(log n) in a treeified bucket; allows one null key and any number of
null values; not synchronized. Default choice.

**LinkedHashMap** — a `HashMap` subclass that threads a doubly-linked list through entries
to record order. Preserves **insertion order** by default; keeps O(1) operations. Has an
**access-order mode** (constructor `accessOrder=true`) that moves the most-recently-used
entry to the end on every access — combined with `removeEldestEntry`, this gives an **LRU
cache** in a few lines:

```java
Map<K,V> cache = new LinkedHashMap<>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > MAX_ENTRIES;
    }
};
```

**TreeMap** — a self-balancing red-black tree; does not use `hashCode` at all. Orders keys
by natural ordering (`Comparable`) or a supplied `Comparator`. Keys kept **sorted**;
O(log n); **no null key**. Implements `NavigableMap` — the real reason to choose it:

```java
TreeMap<Integer,String> tm = new TreeMap<>();
tm.floorKey(50);   // largest key ≤ 50
tm.ceilingKey(50); // smallest key ≥ 50
tm.headMap(100);   // keys < 100
tm.subMap(10, 50); // range view
tm.firstKey();     // smallest key
```

**Hashtable** — legacy (Java 1.0). Every method `synchronized` (locks the whole table);
rejects null keys and values; slow under contention. Never choose for new code.

**ConcurrentHashMap** — the thread-safe choice. Locks/CAS-updates at fine granularity so
many threads read and write concurrently; provides atomic compound operations (`merge`,
`compute`, `computeIfAbsent`). **Forbids null keys and values** (null would be ambiguous
with "absent" in concurrent lookups).

### List implementations

- **ArrayList** — resizable array. O(1) random access, O(1) amortized append, O(n)
  middle insert/remove (shifting). Cache-friendly (contiguous memory). Default `List`.
- **LinkedList** — doubly-linked nodes. O(1) insert/remove at ends, O(n) random access.
  Rarely the right pick — even for queue/deque, `ArrayDeque` usually wins; per-node
  overhead and poor cache behavior hurt.
- **CopyOnWriteArrayList** — thread-safe; every mutation copies the whole array. Writes
  expensive, reads lock-free. Ideal only for read-heavy, rarely-written shared lists
  (e.g., event listeners).

**Rule of thumb:** default to `ArrayList`. `LinkedList`'s theoretical O(1)
middle-insertion almost never wins, because finding the insertion point is O(n) and array
copying is cache-efficient.

### Set implementations

These mirror the maps exactly — each is literally backed by the corresponding map:

- **HashSet** → backed by `HashMap`. No order, O(1), one null. Default `Set`.
- **LinkedHashSet** → backed by `LinkedHashMap`. Insertion order, O(1).
- **TreeSet** → backed by `TreeMap`. Sorted, O(log n), `NavigableSet` (`floor`,
  `ceiling`, `headSet`, `subSet`), no null.

### The unifying picture — three data structures underlie almost everything

- **Hash table** (HashMap/HashSet, and their Linked variants) — O(1) average, needs good
  `hashCode`/`equals`, no inherent order (unless a linked list adds insertion order).
- **Balanced tree** (TreeMap/TreeSet) — O(log n), needs `Comparable`/`Comparator`, keeps
  sorted order, enables range queries.
- **Array** (ArrayList, ArrayDeque) — O(1) indexed access and end operations, O(n) middle
  operations, cache-friendly.

Once you know the underlying structure, an implementation's entire performance and
behavior profile follows — no need to memorize each class.

### HashMap internals — collision resolution

`HashMap` stores entries in an array of buckets. On `put`:

1. Call the key's `hashCode()`.
2. Apply an internal **spreading function** mixing high bits into low bits
   (roughly `h ^ (h >>> 16)`) to reduce clustering.
3. Map to a bucket index with `(n - 1) & hash`, where `n` (array length) is a power of two.

A **collision** = two different keys in the same bucket. Lookups walk the bucket and call
`equals()` to find the exact match ("hashCode finds the bucket, equals finds the entry").

Bucket structure adapts to crowding:

- **Few entries per bucket** → a **linked list**; lookup is O(k) for k entries.
- **Many entries (≥ 8, since Java 8)** → converted to a **red-black tree**
  (**treeification**), making oversized-bucket lookup O(log k). Defends against
  performance degradation, including collision-based DoS attacks. Reverts to a list below 6.
  Treeification also requires ≥ 64 total buckets; a smaller table **resizes** instead.

Separately, when size exceeds `capacity × loadFactor` (default load factor **0.75**), the
map **resizes**: doubles the array and redistributes entries, keeping average
entries-per-bucket low.

**Why a bad `hashCode()` hurts:** if it returns a constant, every key collides into one
bucket and operations degrade from O(1) to O(log n) or worse.

---

## Part 3 — Exceptions

Java's mechanism for signaling that normal execution cannot continue, giving code a chance
to recover, clean up, or fail cleanly.

### The hierarchy

```
Throwable
├── Error                    (don't catch — JVM/system failures)
└── Exception
    ├── RuntimeException      (UNCHECKED — usually bugs)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   └── ArrayIndexOutOfBoundsException
    └── (checked)             (recoverable, compiler-enforced)
        ├── IOException
        └── SQLException
```

- **`Error`** — serious problems you can't recover from and shouldn't catch
  (`OutOfMemoryError`, `StackOverflowError`).
- **Checked exceptions** — everything under `Exception` except `RuntimeException`. The
  compiler **forces** you to catch or declare them. Represent recoverable conditions
  outside your control (missing file, dropped network).
- **Unchecked exceptions** — `RuntimeException` and subclasses. Compiler does **not** force
  handling. Usually indicate programming bugs.

### Checked vs unchecked in practice

```java
// Checked — must handle or declare, else won't compile
void readFile(String path) throws IOException {
    Files.readString(Path.of(path));
}

// Unchecked — no throws needed
int divide(int a, int b) {
    return a / b;   // may throw ArithmeticException at runtime
}
```

### Handling

```java
try {
    int result = 10 / divisor;
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero: " + e.getMessage());
} finally {
    // always runs — even if try returns
    System.out.println("Done");
}
```

**Multi-catch** (Java 7+):

```java
try {
    // ...
} catch (IOException | SQLException e) {
    log.error("Operation failed", e);
}
```

Catch order matters: a more specific exception must come **before** a more general one, or
the code won't compile.

**try-with-resources** — the modern way to close anything implementing `AutoCloseable`,
even on exception. Preferred over manual `finally` (handles the edge case where both body
and `close()` throw):

```java
try (BufferedReader reader = Files.newBufferedReader(path)) {
    return reader.readLine();
}   // reader.close() called automatically
```

### Throwing and custom exceptions

```java
public class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) { super(message); }
}

void withdraw(double amount) throws InsufficientFundsException {
    if (amount > balance) throw new InsufficientFundsException("Balance too low");
    balance -= amount;
}
```

Custom exceptions are worth it when they carry domain meaning a generic type doesn't.

### Exception chaining — preserve the cause

```java
try {
    loadConfig();
} catch (IOException e) {
    throw new ConfigurationException("Could not load config", e); // e is the cause
}
```

Losing the cause makes debugging painful — the trail to the real failure disappears.

### Best practices

- Catch **specific** exceptions, not bare `Exception`/`Throwable` (avoid swallowing bugs
  or `Error`s).
- **Never silently swallow** — an empty catch block hides failures; at minimum log.
- **Don't use exceptions for normal control flow** — throwing is expensive (stack-trace
  building) and obscures logic. Use `if` for expected conditions.
- **Throw early, catch late** — validate at boundaries; recover where you know what to do.
- **Include context** — `"Config file not found: /etc/app/config.yaml"`, not `"File not found"`.

### The checked-exception debate

Meant to force robust handling, but can cause boilerplate and the catch-and-ignore
anti-pattern. Spring leans on unchecked exceptions; Kotlin dropped checked exceptions
entirely. Java keeps both, so you'll encounter both — but know the design is debated, not
settled.

---

## Part 4 — Functional Interfaces, Lambdas, and Streams

These build on each other: functional interfaces let you pass *behavior* around; streams
use them to process collections.

### Functional interfaces and lambdas

A **functional interface** has exactly one abstract method, so it can be implemented with
a **lambda** instead of an anonymous class. `@FunctionalInterface` is optional but signals
intent and makes the compiler enforce the one-method rule.

Common interfaces in `java.util.function`:

| Interface | Method | Shape | Typical use |
|-----------|--------|-------|-------------|
| `Predicate<T>` | `test` | `T → boolean` | filtering |
| `Function<T,R>` | `apply` | `T → R` | transforming |
| `Consumer<T>` | `accept` | `T → void` | side effects (printing) |
| `Supplier<T>` | `get` | `() → T` | lazy creation |
| `UnaryOperator<T>` | `apply` | `T → T` | in-place transform |
| `BiFunction<T,U,R>` | `apply` | `(T,U) → R` | two-arg transform |

```java
Predicate<String> isLong = s -> s.length() > 5;
Function<String, Integer> len = String::length;   // method reference
Consumer<String> printer = System.out::println;

isLong.test("hello");   // false
len.apply("hello");     // 5
printer.accept("hi");   // prints "hi"
```

`::` is a **method reference** — shorthand for a lambda that calls one existing method.
`String::length` means `s -> s.length()`.

### Streams

A **stream** is a processing pipeline over a sequence of elements. It is **not** a data
structure — it stores nothing. It pulls from a source, runs operations, produces a result.

Three parts:

```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Dave");

List<String> result = names.stream()          // 1. source
    .filter(n -> n.length() > 3)              // 2. intermediate (lazy)
    .map(String::toUpperCase)                //    intermediate (lazy)
    .sorted()                                //    intermediate (lazy)
    .collect(Collectors.toList());           // 3. terminal (triggers work)
// → [ALICE, CHARLIE]
```

- **Intermediate operations** (`filter`, `map`, `sorted`, `distinct`, `limit`, `peek`)
  return a new stream and are **lazy** — nothing runs until a terminal operation attaches.
  Laziness enables operation fusing and short-circuiting.
- **Terminal operations** produce a result/side effect and **consume** the stream (no
  reuse afterward): `collect`, `forEach`, `reduce`, `count`, `anyMatch`, `findFirst`,
  `toList()` (Java 16+).

Key patterns:

```java
// Reduction — combine into one value
int sum = List.of(1, 2, 3, 4).stream().reduce(0, Integer::sum); // 10

// Grouping with Collectors
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=[Bob], 4=[Dave], 5=[Alice], 7=[Charlie]}

// Primitive streams — avoid boxing, add numeric helpers
int total = IntStream.rangeClosed(1, 100).sum();               // 5050
double avg = IntStream.of(4, 8, 15, 16).average().orElse(0);

// Optional — "maybe no result" without null
Optional<String> first = names.stream()
    .filter(n -> n.startsWith("C"))
    .findFirst();
first.ifPresent(System.out::println);                          // Charlie
```

**Declarative vs imperative:** you describe *what* transformation you want, not the loops
and mutable accumulators.

```java
// Imperative
List<String> out = new ArrayList<>();
for (String n : names) {
    if (n.length() > 3) out.add(n.toUpperCase());
}

// Declarative
List<String> out = names.stream()
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .toList();
```

### When streams are worth it — and when they are not

**Worth it:**

- **Multi-step transformations** (filter → map → sort → group) — clarity wins over nested
  loops and temporary collections.
- **Grouping, partitioning, aggregation** — the `Collectors` toolkit (`groupingBy`,
  `partitioningBy`, `counting`, `summingInt`, `averagingDouble`) in one line. Streams at
  their strongest.
- **Lazy pipelines with short-circuiting** — `findFirst()`, `anyMatch()`, `limit(n)` stop
  early; hard to replicate cleanly with loops.
- **Infinite/generated sequences** — `Stream.iterate`, `Stream.generate`, `IntStream.range`.
- **Declarative, self-documenting code** — real maintainability benefit.

**Not worth it:**

- **Tiny collections in hot loops** — stream overhead (pipeline allocation, lambda
  invocation, boxing) beats a plain `for`. Unwind streams flagged hot by a profiler.
- **Simple iteration with a side effect** — an enhanced `for` is clearer than `.forEach()`.
- **When you need the index** — streams don't expose position naturally.
- **Mutating external state / complex early break** — lambdas capture only *effectively
  final* variables; `break`/`continue`/`return` give control flow streams only approximate.
- **Checked exceptions** — lambdas can't throw them without ugly wrapping; loops handle
  them naturally.
- **Debugging / team fluency** — stream stack traces show pipeline internals, not your
  logic; maintainability is a legitimate criterion.

**On performance:** choose streams for **readability, not speed**. Sequential streams and
loops usually perform in the same ballpark. Measure with **JMH** when it matters. Use
primitive streams (`IntStream`, etc.) for numeric work to avoid boxing.

**Parallel streams** — `parallelStream()` looks like free speed but rarely is. It pays off
only when *all* hold:

- Dataset is **large** (tens of thousands+).
- Per-element work is **substantial**.
- Operations are **stateless and independent** (no shared mutable state, no ordering deps).
- Source **splits cheaply** (`ArrayList`, arrays yes; `LinkedList`, `Stream.iterate` no).

**Hazard:** parallel streams use the shared common `ForkJoinPool` by default; a slow or
blocking operation can starve *every other* parallel stream in the JVM. Avoid for I/O.
Default to sequential; reach for parallel only after measuring.

---

## Part 5 — Concurrency Fundamentals

Java's concurrency has two layers: a **low-level foundation** (threads, the memory model,
`synchronized`, `volatile`) and a **high-level library** (`java.util.concurrent`) you
should use in almost all real code. Understanding the foundation lets you use the library
correctly.

### Threads

A **thread** is an independent path of execution. Every program starts with `main`. The JVM
maps Java threads onto OS threads — historically one-to-one, making threads a
relatively heavyweight resource (which is why thread *pools* exist).

```java
Thread t = new Thread(() -> System.out.println("Running in "
        + Thread.currentThread().getName()));
t.start();   // start() runs on a new thread; run() would just call it inline
```

You *can* create threads directly, but you rarely should — submit tasks to an executor
instead (below).

### The two core problems (both from shared mutable state)

**1. Race conditions (atomicity).** Read-modify-write operations interleave and lose
updates. `count++` is three steps (read, increment, write); two threads can both read `5`,
both write `6`, losing an increment.

```java
class Counter {
    private int count = 0;
    void increment() { count++; }   // NOT atomic — broken under concurrency
}
```

**2. Visibility.** Without synchronization, there is **no guarantee** one thread's write is
ever *seen* by another. The JVM and CPU may cache values in registers, reorder
instructions, and defer writes to main memory. A thread can loop forever on a flag another
thread already changed.

### The Java Memory Model (JMM)

Formalizes when writes become visible. Its central concept is **happens-before**: if action
A happens-before B, A's effects are guaranteed visible to B. Synchronization actions
establish these edges — releasing a lock happens-before acquiring it; a `volatile` write
happens-before a subsequent `volatile` read of the same variable; `Thread.start()`
happens-before the started thread's first action; and so on. It's the bedrock that makes
higher-level tools' guarantees precise.

### Low-level primitives

**`synchronized`** — provides **mutual exclusion AND visibility**. Only one thread holds an
object's monitor at a time; entering/exiting establishes happens-before edges.

```java
class Counter {
    private int count = 0;
    synchronized void increment() { count++; }   // now atomic + visible
}
```

Synchronize a whole method (locks on `this`, or the `Class` for static methods) or a block
(`synchronized (lockObject) { ... }` to control exactly what you lock on).

**`volatile`** — guarantees **visibility but not atomicity**. A `volatile` field is always
read from/written to main memory, so a flag change is seen immediately. Perfect for a
one-way flag, useless for `count++` (doesn't make the compound op atomic).

```java
private volatile boolean running = true;
void stop() { running = false; }            // seen promptly by other threads
void loop() { while (running) { /* work */ } }
```

**Rule of thumb:** `volatile` for one-way flags / single-writer; `synchronized` (or a
lock) when you need a compound action atomic.

### How to guard shared state

Every access — **reads and writes** — must go through the **same** coordination mechanism.
The most common bug is guarding writes but leaving reads unprotected, reintroducing the
visibility problem. Techniques, simplest first:

**1. Don't share mutable state at all (best).** Immutable objects (only `final` fields,
fully constructed before sharing) are safe for any number of readers. Records make this
easy. `ThreadLocal` gives each thread its own private instance.

```java
public record Point(int x, int y) {}   // inherently thread-safe
```

**2. Confine state to one thread.** If only one thread touches it, no guarding needed.
Hand data *between* threads via a `BlockingQueue` rather than sharing simultaneously.

**3. `synchronized` — general-purpose lock.** For compound operations (check-then-act,
read-modify-write), wrap *all* access with the *same* lock. Reads must lock too.

```java
class BankAccount {
    private double balance;
    private final Object lock = new Object();   // private, dedicated lock

    void withdraw(double amount) {
        synchronized (lock) {
            if (balance >= amount) balance -= amount;  // check + act, atomic together
        }
    }

    double getBalance() {
        synchronized (lock) { return balance; }        // reads MUST lock too
    }
}
```

A **private dedicated lock** (not `this`) prevents outside code from accidentally locking
your object and causing deadlocks.

**4. `volatile` — single field, independent read/write.** Only "write a value / read a
value" with no compound logic (e.g., a status flag). Not for `count++`.

**5. Atomic classes — lock-free single variables.** `AtomicInteger`, `AtomicLong`,
`AtomicReference` use CPU compare-and-swap (CAS); safer and faster than locks for counters.

```java
private final AtomicInteger requestCount = new AtomicInteger();
requestCount.incrementAndGet();                        // atomic
requestCount.updateAndGet(n -> n > 100 ? 0 : n + 1);   // atomic compound
```

**6. Concurrent collections — shared data structures.** Use `ConcurrentHashMap` (not a
guarded `HashMap`); its atomic compound methods close check-then-act races:

```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();
counts.merge(key, 1, Integer::sum);                    // atomic increment-or-insert
counts.computeIfAbsent(key, k -> expensiveInit(k));    // atomic check-then-create
```

`if (!map.containsKey(k)) map.put(k, ...)` on a concurrent map is a **race** — two threads
can both pass the check. The combined methods close that gap.

**7. Explicit locks — more than `synchronized`.** `ReentrantLock` (timeouts,
interruptibility, fairness); `ReadWriteLock` (many concurrent readers, exclusive writers).
Always release in `finally`:

```java
private final ReentrantLock lock = new ReentrantLock();
void update() {
    lock.lock();
    try { /* critical section */ }
    finally { lock.unlock(); }   // must always run
}
```

**Unifying rules:** guard *all* accesses with the *same* mechanism; keep critical sections
small; never call unknown/external code while holding a lock (deadlock risk); acquire
multiple locks in a consistent global order (deadlock prevention).

### The high-level library: `java.util.concurrent` (Java 5+)

Where you should live — it replaces almost every reason to touch raw threads.

**Executors and thread pools** — submit tasks to an `ExecutorService`, which manages
reusable threads and caps resource use.

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> future = pool.submit(() -> expensiveComputation());
Integer result = future.get();   // blocks until complete
pool.shutdown();
```

A `Future` is a handle to a not-yet-ready result; `get()` blocks.

**`CompletableFuture`** — composable evolution of `Future`. Chain callbacks instead of
blocking:

```java
CompletableFuture.supplyAsync(() -> fetchUser(id))
    .thenApply(User::getName)
    .thenAccept(System.out::println)
    .exceptionally(ex -> { log.error("failed", ex); return null; });
```

The tool for orchestrating multiple async operations, running in parallel, combining
results, timeouts, failure handling.

**Atomic classes** — see technique #5 above.

**Concurrent collections** — `ConcurrentHashMap` (concurrent reads, segmented writes);
`CopyOnWriteArrayList` (many reads, rare writes); `BlockingQueue`
(`LinkedBlockingQueue`, `ArrayBlockingQueue` — backbone of producer-consumer; consumers
block until an element is available).

**Explicit locks** — `ReentrantLock`, `ReadWriteLock` (see #7). Worth the verbosity only
for their extra features; otherwise `synchronized` is simpler and just as fast on modern
JVMs.

**Coordination tools** — `CountDownLatch` (wait for N events), `CyclicBarrier` (wait for N
threads to reach a point), `Semaphore` (limit concurrent access to N permits).

### Guiding principles

- The best concurrency is often **no shared mutable state** — favor immutability.
- Prefer the **high-level library** to primitives (`ExecutorService`, `ConcurrentHashMap`,
  `AtomicInteger`, `CompletableFuture` before `synchronized`, `volatile`, raw `Thread`).
- Concurrency bugs are **nondeterministic and timing-dependent** — prevention through good
  design beats debugging.

---

## Part 6 — Web Request Handling: Servlets & Spring Boot

### What is a servlet?

The foundational Java standard for handling HTTP on the server, predating Spring and
sitting under everything. A servlet is a Java object with a method taking a request and
producing a response:

```java
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.getWriter().write("Hello");
    }
}
```

`HttpServletRequest` wraps the incoming request (URL, headers, params, body);
`HttpServletResponse` is what you write the reply into. `doGet`/`doPost`/`doPut` map to
HTTP methods.

A servlet runs inside a **servlet container** (web server / servlet engine): **Tomcat**,
**Jetty**, or **Undertow**. The container is the long-running process that listens on a TCP
port, parses raw HTTP into `HttpServletRequest`, routes to the right servlet, calls it, and
writes the response back. **Your code is the servlet; the container is the machinery.**

The spec is now **Jakarta Servlet** (renamed from Java Servlet after Java EE moved to the
Eclipse Foundation) — hence `jakarta.servlet.*` imports in current code (was `javax.servlet.*`).

**Key architectural point:** the container manages a **pool of worker threads** and
dedicates **one thread per request for that request's entire lifetime** — the
**thread-per-request** model.

### How Spring Boot processes a request

Spring Boot builds a large convenient structure on top of a **single** servlet. The journey
of one request:

1. **Embedded container.** Spring Boot bundles the container *inside* your app (default:
   embedded Tomcat), so a Spring Boot app is a runnable JAR with a `main()`, listening on
   port 8080 by default.
2. **DispatcherServlet.** Spring MVC registers exactly **one** servlet, the
   `DispatcherServlet`, mapped to all URLs. Every request funnels through it — a **front
   controller**. The servlet layer stays minimal; all routing happens after this in Java.
3. **Handler mapping.** The `DispatcherServlet` matches URL + HTTP method to a controller
   method (your `@GetMapping`, `@PostMapping`, …).

   ```java
   @RestController
   public class UserController {
       @GetMapping("/users/{id}")
       public User getUser(@PathVariable Long id) {
           return userService.findById(id);
       }
   }
   ```

4. **Argument resolution.** Spring resolves method parameters — path variables, JSON body
   deserialization, query params, headers (`HandlerMethodArgumentResolver`s).
5. **Your controller runs** — calling services → repositories → database, all on the
   **same single thread**.
6. **Response conversion.** The return value is serialized (Jackson for JSON) into the
   response body with content-type headers (message converters).
7. **Back down the stack** — through the `DispatcherServlet` and any filters; Tomcat writes
   to the socket; the thread returns to the pool.

Also: **filters** (servlet-level, wrap everything — logging, CORS, security) and
**interceptors** (Spring-level, wrap handler dispatch). Spring Security installs a filter
chain before the `DispatcherServlet`.

### The threading model

**Yes — in traditional Spring MVC, each request is handled on its own thread, dedicated to
that request for its entire duration.**

Tomcat keeps a thread pool (default max **200**, `server.tomcat.threads.max`). Per request:

- A thread is taken from the pool and assigned.
- It runs the *entire* chain synchronously: filters → `DispatcherServlet` → controller →
  service → repository → DB call → serialization.
- Every step, including blocking waits on the DB or downstream APIs, runs on that thread.
- When the response is written, the thread returns to the pool.

**Virtue:** simplicity — ordinary sequential blocking code, easy to read and debug; the
stack trace shows the whole logical flow.

**Limitation:** when a request thread **blocks on I/O**, it sits idle holding a pool thread
doing nothing. Under load, all 200 threads end up blocked on I/O and request 201 must
**queue** even though the CPU is nearly idle — throttled by **thread-pool exhaustion**, not
compute or network. This is the blocking-I/O inefficiency (Part 7).

### `@EnableAsync` in Spring Boot

`@EnableAsync` (on a `@Configuration` or the main class) turns on Spring's asynchronous
method execution — the machinery behind `@Async`.

**Mechanics:** any Spring bean method annotated `@Async` runs on a *different* thread from a
`TaskExecutor`, via a **proxy** — calling an `@Async` method really calls a proxy that
submits the actual method to an executor and returns immediately.

```java
@SpringBootApplication
@EnableAsync
public class MyApplication { /* ... */ }

@Service
public class NotificationService {

    @Async
    public void sendEmail(String to) {
        // runs on a background thread; caller doesn't wait
    }

    @Async
    public CompletableFuture<Report> generateReport(Long id) {
        Report r = slowReportBuild(id);
        return CompletableFuture.completedFuture(r);   // caller can compose on this
    }
}
```

**How it helps request handling:**

- **Fire-and-forget** — offload work whose result the caller doesn't need (audit logging,
  confirmation emails); the request thread returns to the pool immediately for a fast
  response.
- **Parallel fan-out** — for a request needing several independent slow calls, make each an
  `@Async` method returning `CompletableFuture`, then combine. Total latency becomes the
  slowest single call, not the sum:

  ```java
  CompletableFuture<A> a = service.fetchA();   // each @Async, runs in parallel
  CompletableFuture<B> b = service.fetchB();
  CompletableFuture.allOf(a, b).join();
  ```

**Critical caveats:**

- **Self-invocation fails.** Because `@Async` works via a proxy, calling an `@Async` method
  from *within the same class* bypasses the proxy and runs synchronously. The call must come
  from a *different* bean.
- **Always configure your own executor.** Without one, older Spring defaulted to a
  `SimpleAsyncTaskExecutor` that creates a new thread per call (no pooling) — dangerous
  under load. Define a bounded pool:

  ```java
  @Bean
  public Executor taskExecutor() {
      ThreadPoolTaskExecutor ex = new ThreadPoolTaskExecutor();
      ex.setCorePoolSize(8);
      ex.setMaxPoolSize(16);
      ex.setQueueCapacity(100);
      ex.setThreadNamePrefix("async-");
      ex.initialize();
      return ex;
  }
  ```

- **Exceptions from `void` `@Async` methods vanish** unless you register an
  `AsyncUncaughtExceptionHandler` (there's no caller to catch them). Methods returning
  `CompletableFuture` carry the exception in the future.

**Forward note:** much of the pressure to use `@Async` for I/O came from wanting to free
scarce pool threads. With virtual threads (Spring Boot 3.2+, `spring.threads.virtual.enabled=true`),
blocking a request thread is cheap, reducing — though not eliminating — the need for
`@Async`. It still earns its place for genuine fire-and-forget and parallel fan-out.

---

## Part 7 (Advanced) — Scaling I/O: Blocking, Event Loops, Virtual Threads, Reactive

### The core problem: concurrency primitives don't solve blocking I/O

These are **two orthogonal axes**:

- **Concurrency primitives** (`synchronized`, `volatile`, atomics, locks,
  `ExecutorService`) → **correctness of shared state** under contention.
- **Blocking vs non-blocking I/O** → **efficiency of thread usage** under I/O wait.

A thread calling `socket.read()` or a blocking JDBC query is **parked by the OS**, holding
its (expensive) stack and OS-thread resource while doing zero work, just waiting for bytes.
A thread pool doesn't fix this — it just bounds how many threads you can waste this way.
When all pool threads block on I/O, new requests queue even though the CPU is nearly idle:
resource-starved on threads while sitting on unused CPU and network capacity.

Reactive programming and virtual threads both attack the **second** axis; neither is about
the first.

### The event-loop model (browser, nginx, Netty)

The shared principle behind non-blocking I/O:

A **small, fixed number of threads never block**. Each runs an **event loop** — grab a
ready event, run its handler until it would block, register a callback for the I/O, and
immediately move to the next ready event. Because a handler never waits, one thread keeps
thousands of connections in flight. The OS I/O-readiness mechanism (**`epoll`** on Linux,
**`kqueue`** on BSD/macOS) tells the loop which operations now have data ready.

- **Browser JavaScript** — the pure single-threaded event loop.
- **nginx** — multiple worker processes, each an event loop.
- **Netty** (the server under Spring WebFlux) — a small pool of event-loop threads
  (sized to CPU cores).

Server implementations are **not literally single-threaded** — the accurate statement is
"a small, fixed number of event-loop threads, each never blocking." The essential property:
**threads are decoupled from connections.** In the blocking model one thread is bound to
one connection for its lifetime; in the event-loop model threads are a small shared pool
that any connection borrows only for the brief bursts when it has real work.

**Golden rule: never block the event loop.** One blocking call (a synchronous DB driver,
`Thread.sleep`, a CPU-heavy computation) freezes that loop thread and every connection it
served — the same discipline as "don't block the main thread" in browsers. This is why
reactive stacks demand non-blocking drivers all the way down.

### Reactive programming

Best understood as **asynchronous, non-blocking data streams with backpressure**, in a
declarative, functional style. Three ideas:

1. **Non-blocking I/O** — the engine (the event loop above). Very few threads, none
   blocked; that's what delivers efficiency.
2. **Streams and composition** — model the program as a pipeline over items arriving over
   time. Same declarative style as the Streams API, but **asynchronous** — values arrive
   whenever I/O completes.

   ```java
   // Project Reactor (behind Spring WebFlux)
   Flux.fromIterable(userIds)
       .flatMap(id -> userService.fetchUser(id))   // each fetch async, non-blocking
       .filter(user -> user.isActive())
       .map(User::getName)
       .subscribe(System.out::println);
   ```

   `Mono<T>` = 0-or-1 async value; `Flux<T>` = 0-to-many. Async cousins of `Optional` and
   `Stream`.
3. **Backpressure** — the distinctive feature (the Reactive Streams spec exists for it). If
   a fast producer feeds a slow consumer, an unbounded async system piles up items in memory
   until it fails. Backpressure lets the consumer signal how much it can handle so the
   producer slows down — flow control for async streams, a problem simple async I/O doesn't
   address.

**Not a "next level" of concurrency** — it's a *different execution model* replacing
thread-per-request with the event-loop model. It doesn't build on `synchronized`/
`ExecutorService`.

**Its cost:**

- Everything must be **non-blocking end to end** — one blocking call anywhere stalls a
  loop thread (needs R2DBC instead of JDBC, reactive clients, etc.).
- **Debugging is hard** — stack traces show reactor internals, not your logical flow.
- **Demanding mental model** — operator composition, lazy subscription/execution, in-stream
  error handling; subtle mistakes.

### Virtual threads (Java 21, Project Loom)

The biggest recent shift. A **virtual thread** is a lightweight thread managed by the JVM,
not the OS. Thousands-to-millions can run on a small number of OS threads. When a virtual
thread blocks (on I/O), the JVM **unmounts** it from its underlying OS thread (the
**carrier** thread) and lets that OS thread run other virtual threads; when the I/O
completes, the virtual thread is **remounted** and continues — all transparent.

```java
// Each task gets its own virtual thread — cheap even at massive scale
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> { handleRequest(i); return null; }));
}
```

**The elegant part:** virtual threads are essentially an **event loop hidden inside the JVM
runtime**. Underneath, the JVM does the same `epoll`-style non-blocking I/O and
continuation-scheduling that nginx and Netty do by hand — parking the logical task, reusing
the OS thread — but automates it and gives you a familiar **blocking API on top**. You get
event-loop throughput while writing straightforward sequential, thread-per-request code, and
you're freed from the "never block" discipline because "blocking" a virtual thread no
longer blocks an OS thread.

```java
// Virtual-thread style: reads like ordinary blocking code, scales like async
var user = userService.fetchUser(id);          // "blocks" the virtual thread, cheaply
var orders = orderService.fetchOrders(user);   // JVM parks/unparks under the hood
return buildResponse(user, orders);
```

**Impact:** the old workarounds — pooling threads, complex async callbacks (much of what
`CompletableFuture` was used for) — become less necessary for I/O-bound work. Especially
transformative for servers handling many simultaneous connections. In Spring Boot 3.2+,
`spring.threads.virtual.enabled=true` serves each request on a virtual thread, keeping the
simple model while removing the 200-thread wall.

*(Note: virtual threads help **I/O-bound** work; they don't speed up **CPU-bound** work.
There is also a **pinning** caveat where a virtual thread can't unmount from its carrier —
historically inside `synchronized` blocks holding a lock across a blocking call — behaving
like the old blocking model.)*

### Are virtual threads and reactive the same? No.

**Same problem, same underlying mechanism, opposite programming models.**

**What's the same:** both attack blocking-I/O inefficiency, and both rely on the same
low-level machinery — non-blocking OS I/O (`epoll`/`kqueue`) plus suspend-and-reuse. Both
achieve high throughput with few OS threads. For I/O-bound request handling they serve
largely the same purpose.

**What's fundamentally different — what you write, and who suspends:**

- **Virtual threads** keep the **blocking programming model and hide the machinery.** You
  write ordinary sequential code that *looks* like it blocks; the JVM does suspend/resume
  invisibly beneath your code.
- **Reactive** makes the asynchrony **explicit and puts it in your code.** You compose a
  pipeline of non-blocking operations with `Mono`/`Flux`; the suspension *is* your program's
  structure.

```java
// Virtual thread — sequential; JVM handles suspension
var user = userService.fetchUser(id);
var orders = orderService.fetchOrders(user);
return buildResponse(user, orders);

// Reactive — the async composition IS the code
return userService.fetchUser(id)                 // Mono<User>
    .flatMap(user -> orderService.fetchOrders(user)
        .map(orders -> buildResponse(user, orders)));
```

Both scale similarly; one reads like sequential code with async buried in the runtime, the
other like a data-flow pipeline with async on the surface. Consequences that cascade:

| Aspect | Virtual threads | Reactive |
|---|---|---|
| Mental model | Ordinary threads (familiar) | Streams, operators, subscription, laziness (learning curve) |
| Debugging | Normal linear stack traces | Reactor-internal stack traces (harder) |
| Backpressure | **No** — runs many independent tasks cheaply | **Yes** — built-in flow control |
| Streaming | No stream abstraction | Native (`Flux`) — SSE, live feeds |
| "Never block" rule | Inverted — blocking is cheap and expected | Absolute — one blocking call poisons the loop |
| Ecosystem | Works with ordinary blocking libraries (JDBC) | Requires non-blocking stack (R2DBC, reactive clients) |

### Where reactive still earns its place

Not obsolete — retain it for what only its stream model provides:

- **Streaming / real-time data** — server-sent events, live feeds, WebSocket pipelines
  where data arrives continuously (`Flux` fits natively; virtual threads don't give this).
- **Backpressure across async boundaries** — coordinated flow control between fast producers
  and slow consumers (built into reactive, absent from virtual threads).
- **Rich stream composition** — merging, throttling, windowing, retry-with-backoff,
  combining multiple async sources — expressed cleanly by operator libraries.
- **Existing reactive codebases** — an established fully non-blocking WebFlux stack.

### The whole arc

1. Concurrency primitives ensure **correctness of shared state** — they do nothing for
   I/O efficiency.
2. **Blocking I/O wastes threads** — a parked thread holds an expensive OS resource doing
   nothing; pools just bound the waste and exhaust under load.
3. Two escapes, both really implementing the **event-loop principle**:
   - **Reactive** exposes the event loop to you as a *programming model* (explicit async
     streams + backpressure).
   - **Virtual threads** bury it under the runtime and give you back *ordinary sequential
     code* (blocking made cheap).

**Practical takeaway:** for most I/O-bound applications, **virtual threads** now give
reactive-level scalability with the simplicity of ordinary code — the better default for
new projects. Reach for **reactive** when you specifically need what only its stream model
provides — **backpressure** across async boundaries or **rich composition of continuous
data streams** — not merely to stop wasting threads on blocking I/O. Virtual threads turned
reactive from "*the* way to do scalable JVM I/O" into "a specialized tool for streaming and
flow-control-heavy problems." Match the tool to the problem rather than reaching for the
most sophisticated one.

---

## Part 8 — Caching

Caching stores expensive-to-obtain data (responses from external dependencies, database
query results, computed values) close to where it's used, so repeat requests are served
cheaply instead of re-fetching or re-computing.

### Why cache dependency/database responses

- **Latency** — a cached value returns in microseconds vs. a network round-trip or a
  disk-bound query.
- **Cost** — third-party APIs are often metered/charged per call; a database has finite
  throughput. Caching cuts both.
- **Resilience** — a cache can serve data even when the underlying dependency is slow or
  down (see serve-stale, Part 9).
- **Load shedding** — protecting a downstream service or DB from being overwhelmed by
  repeated identical reads.

The universal tension: **freshness vs. cost/latency**. Every cache entry must eventually
expire (data changes), but expiring too aggressively defeats the purpose.

### Caching strategies (read/write patterns)

| Strategy | How it works | When to use |
|---|---|---|
| **Cache-aside (lazy loading)** | App checks cache; on miss, loads from source, then populates cache. | Most common; read-heavy, tolerant of a slightly stale first read. |
| **Read-through** | App always asks the cache; the cache itself loads from source on miss. | Cleaner app code; cache library owns loading (e.g., Caffeine `LoadingCache`). |
| **Write-through** | Writes go to cache and source synchronously. | Strong consistency between cache and source; writes slower. |
| **Write-behind (write-back)** | Writes go to cache immediately; source updated asynchronously. | Write-heavy, can tolerate small durability window; risk of loss if cache dies before flush. |
| **Refresh-ahead** | Cache proactively refreshes hot entries *before* they expire. | Predictable hot keys; removes API/DB latency from the request path. |

### Eviction and expiry policies

- **TTL (time-to-live)** — expire after a fixed duration (`expireAfterWrite`).
- **TTI (time-to-idle)** — expire after a period of no access (`expireAfterAccess`).
- **Size-based** — bound entry count / memory (`maximumSize`), evicting by policy:
  - **LRU** (least recently used) — evict the entry unused longest.
  - **LFU** (least frequently used) — evict the least-accessed.
  - **FIFO** — evict the oldest inserted.

**LRU implementation insight:** an LRU cache combines a **HashMap + doubly linked list** —
the HashMap gives O(1) lookup (values are pointers to list nodes), the doubly linked list
gives O(1) reordering and eviction. Neither structure alone achieves both.
(`LinkedHashMap` in access-order mode is the built-in shortcut, per Part 2.)

### Cache invalidation — the hard part

Beyond expiry, you sometimes must actively remove/replace entries when the source changes
(TTL alone leaves a staleness window). Options: explicit eviction on write, versioned keys,
or publish/subscribe invalidation across instances. "There are only two hard things in
computer science…" — invalidation is genuinely difficult; prefer short TTLs plus
serve-stale where exactness isn't required.

### Cache stampede (thundering herd) — and how to avoid it

A **cache stampede** happens when a popular entry expires and many concurrent requests all
find it missing and all trigger the expensive load simultaneously — you wanted 1 fetch, you
get N. It is the **check-then-act race** from concurrency, applied to cache population.

Avoidance techniques:

1. **Single-flight / locking (per-key).** Ensure the load runs **at most once per key**;
   other threads wait for and reuse the result. In-process: `ConcurrentHashMap.computeIfAbsent`
   or a per-key lock (see the worked example). Caffeine's `get(key, loader)` does this for
   you. Distributed: a **distributed lock** (e.g., Redis) so only one *instance* fetches.
2. **Serve-stale-while-revalidate.** Keep the old value past expiry; serve it immediately
   while a single background task refreshes. Users never wait on the source; the dependency
   is decoupled from your availability. (Caffeine `refreshAfterWrite`.)
3. **Early / probabilistic expiration.** Refresh an entry slightly *before* its hard expiry
   with a probability that rises as expiry approaches, so refreshes spread out over time
   instead of all firing at the same instant.
4. **Proactive (refresh-ahead) refresh.** A scheduled job warms hot keys before they
   expire, keeping user requests on a warm cache and smoothing cost into a steady,
   predictable call rate.
5. **Request coalescing.** Merge identical in-flight requests into one (the general form of
   single-flight).

### Worked example — a currency converter over a paid rate API

**Requirements in tension:** rates change (entries must **expire**), and API calls **cost
money** (never make N concurrent calls for one rate). This is a shared-state problem where
the shared state (the cache) is expensive to populate — the stampede is the central hazard.

**Stage 1 — naive (broken).**

```java
public class NaiveConverter {
    private final Map<String, Rate> cache = new HashMap<>();
    private final RateApiClient api;   // paid service

    public double convert(String from, String to, double amount) {
        String key = from + ":" + to;
        Rate rate = cache.get(key);                 // CHECK
        if (rate == null || rate.isExpired()) {
            rate = api.fetchRate(from, to);         // paid call
            cache.put(key, rate);                   // ACT
        }
        return amount * rate.value();
    }
}
```

Two bugs: `HashMap` is **not thread-safe** (concurrent `put` during resize can corrupt or
spin), and the check-then-act between `get` and `put` is a **race** — N threads all see
`null`, all call the API, all write. You get charged N times.

**Stage 2 — `ConcurrentHashMap` alone is not enough.** It stops corruption and makes each
*single* operation atomic, but the **check-then-act across two operations** remains a gap.
Ten threads can still each miss and each call the API. Thread-safe for integrity, still
wasting money. ("I used a concurrent collection so I'm safe" is true for data integrity,
false for compound logic.)

**Stage 3 — correct: `computeIfAbsent` + per-holder double-checked locking.**
`computeIfAbsent` makes "if absent, compute and store" a single atomic operation and runs
the mapping function **at most once per key** under concurrency. But it only computes when
*absent*, not when *expired* — so store a holder that guards its own refresh:

```java
public class CurrencyConverter {
    private final ConcurrentHashMap<String, RateHolder> cache = new ConcurrentHashMap<>();
    private final RateApiClient api;
    private final Duration ttl;

    public CurrencyConverter(RateApiClient api, Duration ttl) {
        this.api = api;
        this.ttl = ttl;
    }

    public double convert(String from, String to, double amount) {
        String key = from + ":" + to;
        // At most one holder is ever created per key, atomically.
        RateHolder holder = cache.computeIfAbsent(key, k -> new RateHolder());
        Rate rate = holder.get(from, to, api, ttl);   // holder guards its own refresh
        return amount * rate.value();
    }

    // One holder per currency pair; serializes only refreshes of THAT pair.
    private static class RateHolder {
        private volatile Rate rate;   // volatile: readers see fresh value without locking

        Rate get(String from, String to, RateApiClient api, Duration ttl) {
            Rate current = rate;                          // cheap volatile read
            if (current != null && !current.isExpired(ttl)) {
                return current;                           // fast path: no lock, no API call
            }
            synchronized (this) {                         // slow path: lock only this holder
                // Double-check inside the lock — another thread may have just refreshed.
                if (rate == null || rate.isExpired(ttl)) {
                    rate = api.fetchRate(from, to);       // exactly one thread calls the API
                }
                return rate;
            }
        }
    }
}
```

Why this is correct, combining earlier ideas:

- **Fast path is lock-free** — a `volatile` read (visibility without mutual exclusion) lets
  any number of threads read a valid unexpired rate concurrently. Reads vastly outnumber
  refreshes.
- **Slow path locks per-pair, not globally** — `synchronized (this)` locks only that
  holder, so refreshing USD→EUR doesn't block refreshing GBP→JPY. A single global lock
  would serialize all conversions.
- **Double-checked pattern** — re-test expiry inside the lock, because a thread that queued
  on the lock may find another already refreshed; without the recheck, the queued threads
  fire redundant paid calls. The `volatile` field is what makes double-checked locking
  correct (it was famously broken before the Java 5 memory model).

Net behavior: when a pair expires and 50 threads hit it, exactly one acquires the lock and
makes one paid call; the other 49 block briefly, then read the freshly-stored rate.

**Caching design decisions:**

- **TTL is a business trade-off** — shorter = fresher but more paid calls; longer =
  cheaper but staler. For currency, minutes-to-an-hour is often reasonable. Make it
  configurable.
- **Serve-stale-on-failure** — if the refresh call fails (blip, rate limit, outage),
  return the last known rate rather than failing the whole conversion; decouples your
  availability from the third party's.
- **Proactive vs reactive refresh** — the design above is *reactive* (one unlucky request
  pays the API latency on expiry). *Proactive* refresh (a `ScheduledExecutorService` or
  Spring `@Scheduled` job) warms hot pairs before expiry, removing API latency from the
  request path and smoothing cost — often better for high-traffic converters.

**Don't build it yourself if you don't have to.** Mature libraries implement all of this
correctly:

- **Caffeine** — the JVM standard. `expireAfterWrite` (TTL); `get(key, loader)` gives the
  same single-flight stampede protection; `refreshAfterWrite` (async proactive refresh that
  serves the stale value while refreshing — serve-stale built in); size-based eviction;
  hit/miss stats.

  ```java
  LoadingCache<String, Rate> cache = Caffeine.newBuilder()
      .expireAfterWrite(Duration.ofMinutes(10))
      .refreshAfterWrite(Duration.ofMinutes(8))   // async refresh before expiry
      .maximumSize(500)
      .build(key -> {
          String[] pair = key.split(":");
          return api.fetchRate(pair[0], pair[1]);  // at most once per key concurrently
      });

  Rate rate = cache.get(from + ":" + to);          // expiry + stampede handled for you
  ```

- **Spring `@Cacheable`** — one level higher; backed by a Caffeine `CacheManager`, removes
  the plumbing:

  ```java
  @Cacheable(value = "rates", key = "#from + ':' + #to")
  public Rate fetchRate(String from, String to) {
      return api.fetchRate(from, to);   // only called on cache miss
  }
  ```

- **Redis** — for a *distributed* system (many app instances), move the cache out of
  process so all instances share it and one instance's fetch benefits the rest. This
  reintroduces a *distributed* stampede (two instances racing), solved with a distributed
  lock or Redis atomic operations — same principle as the single-JVM case.

---

## Part 9 — Working with External Dependencies: Failure & Resilience

Calling another service or database introduces failure modes that don't exist in local
code. A robust system assumes dependencies *will* be slow, fail, or disappear, and
degrades gracefully instead of collapsing.

### Failure modes to design for

- **Network failure** — connection refused, DNS failure, dropped connections, partitions.
  The call never completes normally.
- **Latency / slow responses** — worse than an outright failure in some ways: a slow
  dependency ties up your threads (the blocking-I/O problem in Part 7), and slowness can
  **cascade** — one slow dependency exhausts your thread pool, making *your* service
  unavailable to everyone.
- **Partial / intermittent failure** — some calls succeed, some fail; flakiness.
- **Overload / backpressure** — the dependency (or you) can't keep up with the request
  rate. A fast producer overwhelming a slow consumer piles up unprocessed work in memory
  until something falls over. **Backpressure** is flow control: the consumer signals how
  much it can handle so the producer slows down (built into reactive streams; see Part 7).
- **Rate limits** — the dependency rejects calls above a quota (relevant to the paid API in
  Part 8).

### Resiliency patterns

- **Timeouts.** Never wait indefinitely. Every remote call needs a bounded timeout so a
  hung dependency can't hold your thread forever. The single most important resilience
  control — without it, every other pattern is undermined.
- **Retries (with backoff and jitter).** Retry transient failures — but with
  **exponential backoff** (increasing delays) and **jitter** (randomization) so a fleet of
  clients doesn't retry in lockstep and hammer a recovering service (a retry storm). Only
  retry **idempotent** operations, or you risk duplicate side effects; pair non-idempotent
  writes with an idempotency key.
- **Circuit breaker.** Track failure rate to a dependency; once it crosses a threshold,
  "open" the circuit and **fail fast** (reject calls immediately) instead of piling up
  doomed calls and cascading the failure. After a cooldown, allow a few trial calls
  ("half-open"); if they succeed, "close" and resume normal traffic. Protects both you (no
  thread exhaustion) and the struggling dependency (no pile-on).
- **Bulkhead.** Isolate resources per dependency (e.g., separate thread pools or connection
  pools), so one failing/slow dependency can't consume all resources and sink calls to
  healthy ones — named after a ship's watertight compartments.
- **Fallback / graceful degradation.** When a call fails or the circuit is open, return a
  sensible default: a cached (possibly stale) value, a degraded feature, or a clear partial
  response — rather than a hard error. The **serve-stale-on-failure** cache behavior from
  Part 8 is a fallback.
- **Rate limiting / throttling.** Cap the request rate you send to a dependency (and that
  clients send to you) to stay within quotas and protect capacity.
- **Backpressure.** For streaming/async pipelines, propagate slowness upstream so producers
  slow down rather than overflow buffers (Part 7).

These are typically applied together and are provided by libraries (e.g., Resilience4j on
the JVM) so you configure rather than hand-roll them. **Caching (Part 8) is itself a
resilience tool** — it reduces load on dependencies and can serve data when they are
unavailable.
