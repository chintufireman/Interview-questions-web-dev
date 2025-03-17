#### Question 1: Can we override a static method in Java?
**Answer:** ->: 
1. No, static methods cannot be overridden in Java.
2. Method overriding is based on runtime polymorphism (dynamic method dispatch), which requires an object to determine which method to call.
3. Static methods are not overridden; they are redeclared in the subclass.
4. The method that gets called depends on the reference type, not the object type.
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
5. Key Differences Between Overriding and Redefining Static Methods.

| Aspect | Overriding (Instance Method) | Redefining (Static Method) |
|---|---|
|Binding Type|Runtime (Dynamic Binding)|Compile-time (Static Binding)|
|Polymorphism| Supported|Not Supported|
|Access via Object|Calls method based on actual object type|Calls method based on reference type|