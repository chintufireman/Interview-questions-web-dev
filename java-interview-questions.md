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

#### Q11: what is classpath.

**Answer:** 

1. the classpath is a parameter that tells the Java Virtual Machine (JVM) and Java compiler where to find the compiled class files (.class) and other resources needed to run the program.

2. path to the directories and JAR files that contain these files.

3. There are three common ways to set the classpath:

    1. Using the -cp or -classpath option:
    `javac -cp .;lib/* MyProgram.java` and `java -cp .;lib/* MyProgram`

    2. Using the CLASSPATH environment variable:
    for windows `set CLASSPATH=.;C:\libs\someLibrary.jar` for linux `export CLASSPATH=.:/usr/local/libs/someLibrary.jar`

    3. Using IDE settings.

4. Classpath Formats
    
    1. Directories: Where .class files are stored (e.g., . for the current directory).
    2. AR Files: Java Archive files that package multiple .class files (e.g., lib/myLibrary.jar).
    3. Wildcard (\*): Includes all .jar files in a specified directory (e.g., lib/*).


#### Q12: why do we prefer character array to store passwords instead of string?

**Answer:** Key Differences in Security

1. Immutability of Strings

    - String is immutable, meaning once a String is created, its contents cannot be changed. Even if you assign a new value to the String, the old value remains in memory until garbage collection occurs. This creates a security risk because
        
        - The password might linger in memory for an unpredictable amount of time.

        - Malicious code or memory dumps could potentially expose sensitive information
2. Mutable Character Arrays

    - char[] is mutable, meaning you can overwrite its contents once the password is no longer needed. 
    - By setting each character in the array to '\0' (null character) or another dummy value, you minimize the risk of the password being exposed in memory.

3. Memory Visibility

    - Since String objects are stored in the String Pool, they may persist longer than expected.
    - char[] values are stored in the heap, making them easier to clear immediately after use.

4. Garbage Collection Timing

    - Since you can't force garbage collection, an unused String containing a password may sit in memory longer than desired. On the other hand, a char[] can be explicitly modified and cleared when no longer needed.


5. Example

```
public class PasswordDemo {
    public static void main(String[] args) {
        String password = "Secret123"; // Stays in memory until GC
        System.out.println("Password stored in String: " + password);
        // Even if 'password' is reassigned, "Secret123" may still reside in memory
    }
}
```

```
import java.util.Arrays;

public class SecurePasswordDemo {
    public static void main(String[] args) {
        char[] password = {'S', 'e', 'c', 'r', 'e', 't', '1', '2', '3'};
        System.out.println("Password stored in char[]: " + new String(password));

        // Immediately overwrite the array for security
        Arrays.fill(password, '\0');  
    }
}
```

#### Q13: what is autoboxing and boxing/unboxing?

**Answer:** autoboxing and unboxing refer to the automatic conversion between primitive data types (e.g., int, double) and their corresponding wrapper classes (e.g., Integer, Double).

1. Autoboxing (Primitive → Wrapper): Autoboxing is the automatic conversion of a primitive type to its corresponding wrapper class when required. <br> Happens automatically when: 
    - Assigning a primitive value to a wrapper object.
    - Passing a primitive value to a method that expects an object.
```
public class AutoboxingExample {
    public static void main(String[] args) {
        int num = 10;       // Primitive int
        Integer obj = num;  // Autoboxing: int → Integer
        System.out.println("Autoboxed value: " + obj);
    }
}
```

2. Unboxing (Wrapper → Primitive): Unboxing is the automatic conversion of a wrapper object back into its corresponding primitive type.<br> Happens automatically when:

    - Assigning a wrapper object to a primitive variable.
    - Passing a wrapper object to a method that expects a primitive type.

```
public class UnboxingExample {
    public static void main(String[] args) {
        Integer obj = 20;  // Wrapper class
        int num = obj;     // Unboxing: Integer → int
        System.out.println("Unboxed value: " + num);
    }
}
```

3. Boxing (Manual Conversion): Before Java 5 (when autoboxing was introduced), developers had to manually convert primitives to wrapper objects using methods like .valueOf().

```
int num = 30;
Integer obj = Integer.valueOf(num);  // Explicit boxing
```

|Concept|Description|Example|
|---|---|---|
|Autoboxing|Primitive → Wrapper (automatic)|Integer obj = 10;|
|Unboxing|Wrapper → Primitive (automatic)|int num = obj;|
|Boxing|Primitive → Wrapper (manual)|Integer obj = Integer.valueOf(5);|
|Unboxing|Wrapper → Primitive (manual)|int num = obj.intValue();|

#### Q14: what is aggregation in java?

**Answer:** 
1. Aggregation is a key concept in Object-Oriented Programming (OOP) that represents a "Has-a" relationship between two classes.

2. It is a weaker form of association where one class contains a reference to another class, but both can exist independently.

3. In simple terms:
    - Aggregation is used when one object owns or contains another object.
    - The contained object can exist outside the parent object’s lifecycle.
    - It is a one-way relationship where one class depends on another.

4. Key Characteristics of Aggregation:
    - Represents a "Has-a" relationship.
    - Both objects can exist independently.
    - Implemented using class references (not inheritance).
    - Aggregation is a weak association — no strict dependency.

5. Example
    -  Department has multiple Students.
    - Even if the Department is deleted, the Students can still exist independently.

6. When to Use Aggregation?
    - When one object "Has-a" relationship with another but they are not tightly coupled.
    - When objects can exist independently in your application design.
    - Common in real-world modeling like Departments & Students, Library & Books, etc.

#### Q15: what is composition in java?

**Answer:**
1. Composition is a design principle in Java that represents a "Has-a" relationship where one object is strongly dependent on another.

2. It’s a type of association that implies a strong lifecycle dependency — if the parent object is deleted, the child object is also destroyed. 

3. In simple terms:
    - Composition = Strong "Has-a" relationship
    - The child object cannot exist independently without the parent. 
    - Typically implemented by having one class create and manage the instance of another class.

4. Example of Composition
    - A Human cannot live without a Heart.
    - If the Human object is deleted, the Heart object should be deleted too.

