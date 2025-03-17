#### Q1: Can we override a static method in Java?
**Answer:** ->: 
1. No, static methods cannot be overridden in Java.
2. Static methods are not overridden; they are redeclared in the subclass.
```
class Parent {
    static void display() {
        System.out.println("Parent static method");
    }
}

class Child extends Parent {
    static void display() {
        System.out.println("Child static method");
    }
}

public class Test {
    public static void main(String[] args) {
        Parent p = new Child();
        p.display();  // Outputs: Parent static method
    }
}
```
3. Key Differences Between Overriding and Redefining Static Methods.

| Aspect | Overriding (Instance Method) | Redefining (Static Method) |
|---|---|---|
|Binding Type|Runtime (Dynamic Binding) or (dynamic method dispatch)|Compile-time (Static Binding) or (static method dispatch)|
|Polymorphism|Supported|Not Supported|
|Access via Object|Calls method based on actual object type|Calls method based on reference type| 


#### Q2: what is superclass of exception
**Answer:** 
1. In Java, the Throwable class is the superclass of all errors and exceptions.

#### Q3: What are the methods in object class?
**Answer:**
1. Key Methods in Object Class.

|Method|Description|
|---|---|
|clone()|Creates and returns a copy of the object (requires the class to implement Cloneable).|
|equals(Object obj)|Compares this object to the specified object for equality.|
|finalize()|Called by the garbage collector before an object is destroyed.|
|getClass()|Returns the runtime class of the object.|
|hashCode()|Returns a hash code value for the object (used in hashing mechanisms like HashMap).|
|toString()|Returns a string representation of the object.|
|notify()|Wakes up a single thread waiting on this object's monitor.|
|notifyAll()|Wakes up all threads waiting on this object's monitor.|
|wait()|Causes the current thread to wait until another thread calls notify() or notifyAll().|

#### Q4: String literal?
**Answer**
1. `String str1 = "Hello, World!";` It's a direct value assigned to a String variable.

#### Q5: String pool?
**Answer**
1. The String Constant Pool (also called the String Pool) is a special area in the heap memory where string literals are stored.
2. In Java, string literals are automatically placed in the String Pool when they are created.

#### Q6: String constant pool reside in heap or somewhere else?
**Answer**
1. Before Java 7 the String Constant Pool was part of the Method Area (a.k.a. PermGen space).
2. From Java 7 The String Constant Pool was moved to the heap memory for improved scalability and to avoid OutOfMemoryError issues caused by a limited PermGen.

#### Q7: what is throwable?
**Answer**
1. Throwable is the superclass for all error and exception classes.

#### Q8: How to write multiple try/catch?
**Answer** Types of Multiple try-catch Patterns.
1. Multiple catch Blocks with a Single try Block
2. Nested try-catch Blocks
3. Multiple Independent try-catch Blocks
