# Java & Java 8 Interview Recall Sheet

> Quick, clear recall notes with simple demonstrations. Focus on concepts, how they work, and when to use them.

---

## 1) Java Basics

### 1.1 JVM, JRE, JDK

- **JVM**: Runs bytecode (.class). Platform-specific runtime.
- **JRE**: JVM + standard libraries to run apps.
- **JDK**: JRE + compilers/tools to build apps.

### 1.2 Compilation Flow

- `javac` compiles `.java` → `.class` (bytecode).
- JVM interprets / JIT-compiles bytecode to native code.

### 1.3 Primitive Types

| Type    | Bits          | Example            |
| ------- | ------------- | ------------------ |
| byte    | 8             | `byte b=10;`       |
| short   | 16            | `short s=100;`     |
| int     | 32            | `int i=1000;`      |
| long    | 64            | `long l=1000L;`    |
| float   | 32            | `float f=1.5f;`    |
| double  | 64            | `double d=1.5;`    |
| char    | 16            | `char c='A';`      |
| boolean | 1-bit logical | `boolean ok=true;` |

### 1.4 Operators (Quick)

- Arithmetic: `+ - * / %`
- Relational: `> < >= <= == !=`
- Logical: `&& || !`
- Bitwise: `& | ^ ~ << >> >>>`
- Ternary: `cond ? a : b`

---

## 2) OOP Concepts (Core)

### 2.1 Class & Object

```java
class Car {
    String model;
    void drive() { System.out.println(model + " driving"); }
}

Car c = new Car();
c.model = "Tesla";
c.drive(); // Tesla driving
```

### 2.2 Encapsulation

- Keep fields private, expose via getters/setters.

```java
class Account {
    private double balance;
    public void deposit(double amt){ balance += amt; }
    public double getBalance(){ return balance; }
}
```

### 2.3 Inheritance

```java
class Animal { void sound(){ System.out.println("..."); } }
class Dog extends Animal { void sound(){ System.out.println("bark"); } }
```

### 2.4 Polymorphism

- **Overriding** (runtime):

```java
Animal a = new Dog();
a.sound(); // bark
```

- **Overloading** (compile-time):

```java
void sum(int a, int b){}
void sum(double a, double b){}
```

### 2.5 Abstraction

- **Abstract class**:

```java
abstract class Shape { abstract double area(); }
class Circle extends Shape {
    double r; Circle(double r){this.r=r;}
    double area(){ return Math.PI*r*r; }
}
```

- **Interface**:

```java
interface Flyable { void fly(); }
class Bird implements Flyable { public void fly(){ } }
```

---

## 3) Interfaces (Key Points)

- Multiple inheritance of type.
- Java 8: **default** and **static** methods.

```java
interface Logger {
    default void log(String msg){ System.out.println("LOG:" + msg); }
    static Logger noop(){ return msg -> {}; }
    void write(String msg);
}
```

---

## 4) Collections Framework

### 4.1 Core Interfaces

- **Collection** → List, Set, Queue
- **Map** (separate hierarchy)

### 4.2 List

- Ordered, duplicates allowed.
- Implementations: ArrayList, LinkedList, Vector.

```java
List<String> list = new ArrayList<>();
list.add("A");
list.add("A");
```

### 4.3 Set

- No duplicates.
- HashSet, LinkedHashSet, TreeSet.

```java
Set<Integer> set = new HashSet<>();
set.add(1); set.add(1); // one element
```

### 4.4 Map

- Key-Value pairs. Keys unique.
- HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap.

```java
Map<String,Integer> map = new HashMap<>();
map.put("A", 1);
```

### 4.5 Queue / Deque

- FIFO: Queue
- Double-ended: Deque (ArrayDeque)

### 4.6 Comparable vs Comparator

```java
class Person implements Comparable<Person> {
    int age;
    public int compareTo(Person o){ return this.age - o.age; }
}
Comparator<Person> byName = (a,b)-> a.name.compareTo(b.name);
```

---

## 5) Exception Handling

### 5.1 Types

- **Checked**: must handle (IOException)
- **Unchecked**: RuntimeException

### 5.2 try-catch-finally

```java
try { int x = 10/0; }
catch (ArithmeticException e) { System.out.println("Divide by zero"); }
finally { System.out.println("Always"); }
```

### 5.3 try-with-resources (Java 7+)

```java
try (BufferedReader br = new BufferedReader(new FileReader("a.txt"))) {
    System.out.println(br.readLine());
}
```

---

## 6) Multithreading (Basics)

### 6.1 Thread vs Runnable

```java
class Task implements Runnable {
    public void run(){ System.out.println("run"); }
}
new Thread(new Task()).start();
```

### 6.2 Synchronization

```java
synchronized void increment(){ count++; }
```

### 6.3 volatile

- Ensures visibility across threads.

### 6.4 Executor Framework

```java
ExecutorService es = Executors.newFixedThreadPool(2);
es.submit(()-> System.out.println("job"));
es.shutdown();
```

---

## 7) Java 8 Features (Important)

### 7.1 Lambda Expressions

```java
Runnable r = () -> System.out.println("Hello");
```

### 7.2 Functional Interfaces

- Single abstract method.
- Examples: `Runnable`, `Callable`, `Comparator`.

```java
@FunctionalInterface
interface Adder { int add(int a,int b); }
Adder ad = (a,b) -> a+b;
```

### 7.3 Built-in Functional Interfaces

- `Predicate<T>` → boolean
- `Function<T,R>` → R
- `Consumer<T>` → void
- `Supplier<T>` → T

```java
Predicate<Integer> even = x -> x%2==0;
Function<String,Integer> len = s -> s.length();
Consumer<String> print = s -> System.out.println(s);
Supplier<Double> rnd = () -> Math.random();
```

### 7.4 Method References

```java
List<String> names = Arrays.asList("a","b");
names.forEach(System.out::println);
```

### 7.5 Stream API

```java
List<Integer> nums = Arrays.asList(1,2,3,4,5);
int sum = nums.stream()
    .filter(n -> n%2==0)
    .map(n -> n*n)
    .reduce(0, Integer::sum); // sum of squares of evens
```

Common operations:

- Intermediate: `filter`, `map`, `sorted`, `distinct`, `limit`, `skip`
- Terminal: `collect`, `forEach`, `reduce`, `count`, `anyMatch`

### 7.6 Optional

```java
Optional<String> opt = Optional.ofNullable(name);
String val = opt.orElse("default");
```

### 7.7 Default & Static Methods in Interfaces

```java
interface Vehicle {
    default void start(){ System.out.println("start"); }
    static void info(){ System.out.println("info"); }
}
```

### 7.8 Date/Time API (java.time)

```java
LocalDate d = LocalDate.now();
LocalDate next = d.plusDays(1);
Duration dur = Duration.between(Instant.now(), Instant.now().plusSeconds(10));
```

### 7.9 Parallel Streams

```java
int sum = nums.parallelStream().reduce(0, Integer::sum);
```

### 7.10 Collectors

```java
Map<Integer, List<String>> grouped = names.stream()
    .collect(Collectors.groupingBy(String::length));
```

---

## 8) Strings (Important)

- `String` immutable.
- `StringBuilder` mutable, faster for concat.
- `StringBuffer` mutable, thread-safe.

```java
String s = "a" + "b";
StringBuilder sb = new StringBuilder("a").append("b");
```

---

## 9) Equality & Hashing

- `==` compares references for objects.
- `equals()` compares content.
- Must override `hashCode()` when overriding `equals()`.

```java
@Override
public boolean equals(Object o){ ... }
@Override
public int hashCode(){ ... }
```

---

## 10) SOLID (Quick)

- **S**ingle Responsibility
- **O**pen/Closed
- **L**iskov Substitution
- **I**nterface Segregation
- **D**ependency Inversion

---

## 11) Common Interview Q&A (Quick)

1. **ArrayList vs LinkedList**: random access vs insert/delete.
2. **HashMap vs TreeMap**: unordered vs sorted.
3. **HashSet vs TreeSet**: hash vs ordered set.
4. **synchronized vs volatile**: mutual exclusion vs visibility.
5. **checked vs unchecked**: compile-time vs runtime.
6. **Comparator vs Comparable**: external vs internal ordering.

---

## 12) Mini Demo: Java 8 Collection Processing

```java
List<String> names = Arrays.asList("bob","alex","bob","zara");
List<String> result = names.stream()
    .distinct()
    .sorted()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// result: [ALEX, BOB, ZARA]
```

---

## 13) Mini Demo: Optional + Streams

```java
Optional<String> maybe = Optional.of("java");
maybe.filter(s -> s.length() > 3)
     .ifPresent(System.out::println); // prints java
```

---

## 14) Mini Demo: Functional Interface

```java
@FunctionalInterface
interface Printer { void print(String s); }
Printer p = System.out::println;
p.print("Hello");
```

---

## 15) OOP Recap (1-Minute Revision)

- **Encapsulation**: data hiding via `private`.
- **Inheritance**: reuse via `extends`.
- **Polymorphism**: same method name, different behavior.
- **Abstraction**: expose only what’s necessary.

---

## 16) Quick Tips

- Prefer interfaces in APIs.
- Use `final` for constants.
- Use `try-with-resources` for I/O.
- Use `Optional` to avoid null checks.
- Prefer `ArrayList` unless many middle inserts.

---

## 17) Full Java 8 Feature Demo (Single File)

```java
import java.time.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.util.stream.*;

public class Java8Demo {
    public static void main(String[] args) throws Exception {
        // 1) Lambdas + Functional Interfaces
        Predicate<Integer> even = n -> n % 2 == 0;
        Function<String, Integer> len = String::length;
        Consumer<String> printer = System.out::println;
        Supplier<Double> rnd = Math::random;

        printer.accept("len(hello)=" + len.apply("hello"));
        printer.accept("even(4)=" + even.test(4));
        printer.accept("rand=" + rnd.get());

        // 2) Method reference
        List<String> names = Arrays.asList("bob", "alex", "bob", "zara");
        names.forEach(System.out::println);

        // 3) Streams + Collectors
        List<String> result = names.stream()
                .distinct()
                .sorted()
                .map(String::toUpperCase)
                .collect(Collectors.toList());
        printer.accept("result=" + result);

        Map<Integer, List<String>> grouped = names.stream()
                .collect(Collectors.groupingBy(String::length));
        printer.accept("grouped=" + grouped);

        int sumSquares = Stream.of(1,2,3,4,5)
                .filter(even)
                .map(n -> n * n)
                .reduce(0, Integer::sum);
        printer.accept("sumSquares=" + sumSquares);

        // 4) Optional
        Optional<String> maybe = Optional.of("java");
        printer.accept("maybe=" + maybe.orElse("default"));

        // 5) Default & static interface methods
        Vehicle v = new Car();
        v.start();
        Vehicle.info();

        // 6) Date/Time API
        LocalDate today = LocalDate.now();
        LocalDate tomorrow = today.plusDays(1);
        Duration d = Duration.between(Instant.now(), Instant.now().plusSeconds(5));
        printer.accept("today=" + today + ", tomorrow=" + tomorrow + ", duration=" + d.getSeconds());

        // 7) Parallel stream
        int parallelSum = IntStream.rangeClosed(1, 1000).parallel().sum();
        printer.accept("parallelSum=" + parallelSum);
    }

    interface Vehicle {
        default void start(){ System.out.println("start"); }
        static void info(){ System.out.println("info"); }
    }

    static class Car implements Vehicle {}
}
```

---

## 18) Threading & Multithreading (Detailed Demos)

### 18.1 Create Thread (extends Thread)

```java
class MyThread extends Thread {
    public void run(){ System.out.println("Running in " + Thread.currentThread().getName()); }
}
new MyThread().start();
```

### 18.2 Create Thread (implements Runnable)

```java
Runnable task = () -> System.out.println("Runnable " + Thread.currentThread().getName());
new Thread(task, "T-1").start();
```

### 18.3 Callable + Future

```java
ExecutorService es = Executors.newSingleThreadExecutor();
Future<Integer> f = es.submit(() -> 10 + 20);
System.out.println("Future result=" + f.get());
es.shutdown();
```

### 18.4 Synchronization (critical section)

```java
class Counter {
    private int count = 0;
    public synchronized void inc(){ count++; }
    public int get(){ return count; }
}
```

### 18.5 Lock (ReentrantLock)

```java
Lock lock = new ReentrantLock();
lock.lock();
try { /* critical section */ }
finally { lock.unlock(); }
```

### 18.6 AtomicInteger (lock-free)

```java
AtomicInteger ai = new AtomicInteger(0);
ai.incrementAndGet();
```

### 18.7 CountDownLatch (coordination)

```java
CountDownLatch latch = new CountDownLatch(2);
new Thread(() -> { latch.countDown(); }).start();
new Thread(() -> { latch.countDown(); }).start();
latch.await(); // waits for 2 counts
```

### 18.8 wait/notify (low-level coordination)

```java
class MessageBox {
    private String msg;
    synchronized void put(String m) throws InterruptedException {
        while (msg != null) wait();
        msg = m;
        notifyAll();
    }
    synchronized String take() throws InterruptedException {
        while (msg == null) wait();
        String m = msg;
        msg = null;
        notifyAll();
        return m;
    }
}
```

---

## 19) Collections Brainstorm (Fast Recall Map)

### 19.1 Interfaces

- Collection → List, Set, Queue, Deque
- Map → Map, SortedMap, NavigableMap

### 19.2 List Implementations

- ArrayList (fast random access)
- LinkedList (fast inserts/removes in middle)
- Vector (legacy, synchronized)
- Stack (legacy; prefer Deque)

### 19.3 Set Implementations

- HashSet (unordered)
- LinkedHashSet (insertion order)
- TreeSet (sorted, NavigableSet)
- EnumSet (for enums)

### 19.4 Map Implementations

- HashMap (unordered)
- LinkedHashMap (insertion order)
- TreeMap (sorted, NavigableMap)
- Hashtable (legacy, synchronized)
- ConcurrentHashMap (thread-safe, scalable)
- WeakHashMap (weak keys)
- IdentityHashMap (== on keys)
- EnumMap (enum keys)

### 19.5 Queue/Deque Implementations

- PriorityQueue (heap, not FIFO)
- ArrayDeque (fast stack/queue)
- LinkedList (queue + deque)

### 19.6 Concurrent Collections

- ConcurrentHashMap
- CopyOnWriteArrayList
- CopyOnWriteArraySet
- ConcurrentLinkedQueue
- BlockingQueue: ArrayBlockingQueue, LinkedBlockingQueue, PriorityBlockingQueue

---

## 20) Interview Q/A (Last-Minute Revision)

1. **Q:** What is a functional interface?  
   **A:** An interface with exactly one abstract method; enables lambdas.

2. **Q:** Difference between `map()` and `flatMap()` in streams?  
   **A:** `map()` transforms each element; `flatMap()` flattens nested streams.

3. **Q:** Why `Optional`?  
   **A:** Avoid `null` checks; model presence/absence explicitly.

4. **Q:** `HashMap` vs `ConcurrentHashMap`?  
   **A:** `HashMap` is not thread-safe; `ConcurrentHashMap` supports concurrent access.

5. **Q:** `synchronized` vs `Lock`?  
   **A:** `Lock` is more flexible (tryLock, fairness, conditions).

6. **Q:** `volatile` does what?  
   **A:** Ensures visibility of changes across threads, not atomicity.

7. **Q:** `ArrayList` vs `LinkedList`?  
   **A:** ArrayList: fast random access; LinkedList: fast insert/remove in middle.

8. **Q:** `HashSet` vs `TreeSet`?  
   **A:** HashSet: unordered; TreeSet: sorted.

9. **Q:** Why override `equals()` and `hashCode()` together?  
   **A:** To maintain contract for hashing collections.

10. **Q:** What is a stream terminal operation?  
    **A:** Ends the pipeline and produces result (e.g., `collect`, `reduce`).
