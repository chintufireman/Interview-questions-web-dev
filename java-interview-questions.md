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
```
try {
            int[] numbers = {1, 2, 3};
            System.out.println(numbers[5]); // ArrayIndexOutOfBoundsException
            
            int result = 10 / 0; // ArithmeticException
            System.out.println(result);
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Array index is out of bounds: " + e.getMessage());
        } catch (ArithmeticException e) {
            System.out.println("Cannot divide by zero: " + e.getMessage());
        } catch (Exception e) {
            System.out.println("An unexpected error occurred: " + e.getMessage());
        } finally {
            System.out.println("Execution completed.");
        }
```

2. Nested try-catch Blocks

```
try {
            System.out.println("Outer try block started");

            try {
                int result = 10 / 0; // ArithmeticException
                System.out.println(result);
            } catch (ArithmeticException e) {
                System.out.println("Inner catch: Division by zero.");
            }

            String str = null;
            System.out.println(str.length()); // NullPointerException

        } catch (NullPointerException e) {
            System.out.println("Outer catch: Null pointer encountered.");
        } finally {
            System.out.println("Finally block executed.");
        }
```
When handling an exception may itself throw another exception.
When different levels of code require different error-handling logic.

3. Multiple Independent try-catch Blocks

```
 try {
            int result = 10 / 0;  // ArithmeticException
            System.out.println("Result: " + result);
        } catch (ArithmeticException e) {
            System.out.println("Error in first block: " + e.getMessage());
        }

        // Second independent try-catch block
        try {
            int[] numbers = {1, 2, 3};
            System.out.println(numbers[5]);  // ArrayIndexOutOfBoundsException
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Error in second block: " + e.getMessage());
        }

        // Third independent try-catch block
        try {
            String text = null;
            System.out.println(text.length());  // NullPointerException
        } catch (NullPointerException e) {
            System.out.println("Error in third block: " + e.getMessage());
        }
```
Each try block runs independently — one failure doesn't affect the rest.
Best suited when errors in one logic block shouldn't disrupt others.


#### Q9: finally is invoked before or after return keyword?
**Answer:**
1. finally block is always executed — even if the try block contains a return statement. 
2. The finally block runs after the return statement is evaluated but before the method actually returns.

3. finally Altering the Return Value
```
public static int testMethod() {
        int value = 10;
        try {
            return value; // `value` is 10
        } finally {
            value = 20; // This modifies the value, but it doesn't change the returned result
            System.out.println("Inside finally block");
        }
    }
```
Output:
```
Inside finally block
Result: 10
```
4. Why? Because the return statement evaluates the value first, then the finally block runs.
5. Even if the finally block changes the variable, it won’t affect the previously evaluated return value

#### Q10: main difference between abstract class and interface.

**Answer:** 
1. Use an abstract class when You need to share common behavior across related classes.
2. You want to provide default functionality along with abstract methods.

3. Use an interface when You need to define a contract for unrelated classes to follow.

4. You require multiple inheritance (since Java allows implementing multiple interfaces).

|Feature|Abstract Class|Interface|
|---|---|---|
|Purpose | Used for shared behavior across related classes. Ideal for code reuse and hierarchy |Used for defining contracts (rules) that classes must follow. Focuses on polymorphism |
| Methods|Can have both abstract (unimplemented) and concrete (implemented) methods|Methods are abstract by default (before Java 8). Since Java 8, can have default and static methods.|
|Access Modifiers| Methods can have any access modifier (public, protected, private, etc.) | All methods are implicitly public and abstract unless specified otherwise |
|Variables|Can have instance variables (with any access modifier)|Variables are public, static, and final (constants) by default|
|Inheritance|Can extend one abstract class only (single inheritance).|Can implement multiple interfaces (multiple inheritance support).|
|Constructor|Can have constructors to initialize state|Cannot have constructors (no object instantiation directly).|
|Implementation|A subclass must implement all abstract methods or be declared abstract itself| A class must implement all interface methods or be declared abstract.|
|Performance|Slightly faster because it supports partial implementation|Slightly slower due to the need for method resolution at runtime|
|Use Case|Best for inheriting common behavior across related classes|Best for defining contracts or capabilities that unrelated classes can implement|




