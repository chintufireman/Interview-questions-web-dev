### Core Java Mastery
#### OOP principles (SOLID, DRY, KISS)

**Answer:** OOP (Object-Oriented Programming) has four core principles:
1. Encapsulation – Bundle data and methods operating on it in one unit (class) and restrict direct access.
2. Abstraction – Hide complex implementation details and expose only what’s necessary.
3. Inheritance – Allow a class to inherit fields and methods from another class.
4. Polymorphism – One interface, many implementations. E.g., method overriding and overloading.

#### SOLID Principles

**Answer:**The SOLID principles are five design principles to write clean, scalable, and maintainable OOP code.

1. S – Single Responsibility Principle, A class should have only one reason to change
    - Means: Each class should do one thing (e.g., don't mix logic for data access and UI handling).
    -   ```
            class InvoicePrinter {
                public void print(Invoice invoice) { ... }
            }

            class InvoiceSaver {
                public void saveToDB(Invoice invoice) { ... }
            }
        ```
2. O – Open/Closed Principle, Software entities should be open for extension, but closed for modification.
    - Means: You should be able to add new features without modifying existing code.
    - 
        ```
            interface Payment {
                void pay();
            }

            class CreditCardPayment implements Payment {
                public void pay() { ... }
            }

            class PayPalPayment implements Payment {
                public void pay() { ... }
            }
        ```
3. L – Liskov Substitution Principle, Subclasses should be substitutable for their base classes.
    - Means: You should be able to use a subclass anywhere you use a parent class without breaking functionality.
    - this is bad
        ```
            class Bird {
                void fly() {}
            }

            class Ostrich extends Bird {
                void fly() { throw new UnsupportedOperationException(); }
            }
        ```
    - Better: Separate flying birds and non-flying birds
4. I – Interface Segregation Principle, No client should be forced to depend on methods it doesn't use.
    - Means: Use smaller, specific interfaces instead of one large one
    - Bad
        ```
            interface Animal {
                void walk();
                void fly();
                void swim();
            }
        ```
    - Good
        ```
            interface Walkable { void walk(); }
            interface Flyable { void fly(); }
            interface Swimmable { void swim(); }
        ```
5. D – Dependency Inversion Principle, High-level modules should not depend on low-level modules. Both should depend on abstractions.
    - Means: Use interfaces to decouple code
    - Good:
        ```
            interface Keyboard {
                void type();
            }

            class WiredKeyboard implements Keyboard {
                public void type() { ... }
            }

            class Computer {
                private Keyboard keyboard;
                public Computer(Keyboard keyboard) {
                    this.keyboard = keyboard;
                }
            }
        ```
#### DRY – Don't Repeat Yourself

**Answer:** Avoid code duplication. Logic should be defined once and reused
1. Bad:
    ```
        if (user.age > 18) {
            // some logic
        }
        if (user.age > 18) {
            // same logic again
        }
    ```
    Good:
        ```
            boolean isAdult(User user) {
                return user.age > 18;
            }
        ```

#### KISS – Keep It Simple, Stupid
**Answer:** Simplicity is better than complexity. Write simple, readable, and minimal code.

1. Bad:
    ```
        if (x == true) {
            return true;
        } else {
            return false;
        }
    ```

2. Good: `return x;`

#### Generics
**answer:**

|Feature|Description|
|---|---|
|<\T>|Type parameter|
|Generic Class|Reusable for different types|
|Generic Method|Type-safe utility methods|
|Bounded Types|Limit types (<\T extends Number>)|
|Wildcards|Flexible API methods (?, ? extends, ? super)|
|Type Erasure|Type info is removed at runtime|

#### Java Reflection API
**Answer:** Key Classes in java.lang.reflect
|Class/Interface|Description|
|---|---|
|Class<?>|Represents the metadata of a class|
|Field|Represents a class field (variable)|
|Method|Represents a method in a class|
|Constructor|Represents a constructor|
|Modifier|Helper class for checking access modifiers|

1. Example: Using Reflection
    - Get Class Metadata: `Class<?> clazz = Class.forName("com.example.Person");  // fully qualified name`
    - Or if you already have an object `Class<?> clazz = person.getClass();`
    - Access Constructors and Create Instance: 
        ```
            Constructor<?> constructor = clazz.getConstructor(String.class, int.class);
            Object obj = constructor.newInstance("Alice", 30);
        ```
    - Access Fields: 
        ```
            Field field = clazz.getDeclaredField("name");
            field.setAccessible(true); // if it's private
            field.set(obj, "Bob");     // set new value
            String value = (String) field.get(obj); // read value
        ```
    - Access Methods: 
        ```
            Method method = clazz.getMethod("getName");
            String name = (String) method.invoke(obj);  // calls getName()
        ```
2. Accessing Private Members
    - Reflection can override Java access controls:
        ```
            field.setAccessible(true);    // for private fields
            method.setAccessible(true);   // for private methods
        ```


#### Java Memory Management
#### Garbage Collection (G1, CMS, ZGC)
**Answer:**
1. What is Garbage Collection?
    - When objects are no longer reachable in Java (i.e., no references pointing to them), the JVM automatically reclaims that memory.
    - GC runs in the background and reduces memory leaks and manual memory management.

2. Types of Garbage Collectors?
    - G1 (Garbage First) Collector:
        - Introduced in Java 7u4, default since Java 9
        - Designed for applications with large heaps (multi-GB)
        - Divides the heap into regions (not young/old generation as rigidly).
        - Key Features:
            - Low pause time goal
            - Concurrent marking
            - Prioritizes regions with the most garbage ("Garbage First")
            - Compacts memory during GC, reducing fragmentation
        - Best For:
            - Applications needing consistent performance and large memory
    - CMS (Concurrent Mark Sweep) Collector
        - Deprecated in Java 9, removed in Java 14
        - Focused on low pause times
        - Key Features:
            - Runs concurrently with application threads
            - Stops the world only briefly
            - Divides the heap into Young and Old generations
        - Limitations:
            - Causes fragmentation.
            - Deprecated because G1 and ZGC are better
    - ZGC (Z Garbage Collector)
        - Introduced in Java 11, production-ready in Java 15
        - Designed for very low latency and very large heaps (100 GB+)
        - Key Features:
            - Pause times under 10 ms, regardless of heap size.
            - Does most of the work concurrently.
            - Uses colored pointers and load barriers to track object references.
        - Best For
            - Low-latency, high-throughput apps (e.g., real-time systems).

#### JVM MEMORY STRUCTURE (High-Level)
**Answer** When a Java program runs, the Java Virtual Machine (JVM) divides memory into several areas. The most important two for most developers are:
1. Heap Memory – stores objects (instances).
2. Stack Memory – stores method call frames, local variables, and return addresses.

3. Heap Memory:
    - Purpose: Stores objects and class-level variables (also called instance variables or member variables).
    - Characteristics:
        - Shared across all threads.
        - Managed by Garbage Collector.
        - Slower to access than the stack.
    - Subdivided into:
        - |Area|Description|
            |---|---|
            |Young Generation|Newly created objects go here|
            |Eden Space|New objects are first created here|
            |Survivor Spaces (S0, S1)|After surviving a few GC cycles, they’re moved here|
            |Old Generation|Long-lived objects are promoted here from the young gen|
            |(Optional) Metaspace|Stores class metadata (replaced PermGen from Java 8 onward)|
4. Stack Memory
    - Purpose: Stores method call frames, local variables, and references to objects
    - Characteristics: 
        - Each thread has its own stack.
        - Memory is allocated in LIFO (Last In, First Out) order.
        - Faster than heap memory.
        - Not affected by garbage collection (memory is automatically reclaimed after method ends).
    - Common Error:
        - StackOverflowError — caused by deep recursion or infinite method calls

    Key Differences Between Heap and Stack:
    |Feature|Stack|Heap|
    |---|---|---|
    |Stores|Method calls, local vars, references|Objects, instance vars|
    |Access Speed|Fast|Slower|
    |Thread scope|One stack per thread|Shared across all threads|
    |Memory Management|Auto when method ends|Managed by Garbage Collector|
    |Error Type|StackOverflowError|OutOfMemoryError|

#### Profiling tools (JProfiler, VisualVM)
**Answer:**
1. What is Profiling?
    - Profiling is the process of monitoring and analyzing how a Java application behaves at runtime — including:
    - Memory usage
    - CPU usage
    - Thread activity
    - Object allocation
    - Garbage collection
    - Method execution time
    - This helps in identifying bottlenecks, memory leaks, thread contention, and performance issues.
2. JProfiler (Commercial Tool):
    - Key Features:
        - Memory profiling – tracks object allocations, GC roots, leaks.
        - CPU profiling – shows method execution time and call trees.
        - Thread profiling – identifies deadlocks, waits, and blocking calls.
        - Heap walker – drill down into memory usage and object references.
        - Database and I/O monitoring – shows slow queries or blocking I/O.
    - UI Highlights:
        - Visual graphs for memory/CPU/thread usage
        - Real-time analysis with historical snapshots
        - Integrated with IDEs (IntelliJ, Eclipse, NetBeans)
    - How to Use:
        - Start JProfiler and attach it to a running JVM
        - You can start the JVM with an agent like.
        - `-agentpath:/path/to/jprofiler/libjprofilerti.so=port=8849`
        - View live data in the JProfiler UI
3. VisualVM (Free, Open Source)
    - Key Features:
        - Monitor heap, CPU, GC in real-time.
        - View and capture heap dumps.
        - Track thread activity and deadlocks.
        - Analyze memory leaks.
        - Profile method execution.
    - How to Use:
        - Comes with most JDKs under bin/visualvm
        - `jvisualvm`
        - Attach to a running Java process
        - You can start CPU/memory profiling, view GC activity, and explore heap dumps.
    - Pro Tip:
        - You can use jmap, jstack, and jconsole together with VisualVM for deeper analysis.

4. Comparison Table
    |Feature|JProfiler |VisualVM|
    |---|---|---|
    |License|Commercial|Free/Open Source|
    |UI|Advanced|Simple|
    |Heap dump analysis|Yes (Heap Walker)|Yes|
    |CPU sampling/profiling|Yes|Yes|
    |Thread analysis|Advanced|Basic|
    |IDE integration|Yes|No (standalone only)|
    |Database & I/O insights|Yes|Limited|    