# Static and Final in Java

## The static keyword

`static` means "belongs to the class, not to any individual object". A static member is shared by all objects of the class, instead of every object having its own copy.

### Static fields

```java
class Counter {
    static int count = 0; // shared by all objects
    int id;               // unique per object

    Counter() {
        count++;
        id = count;
    }
}

Counter c1 = new Counter();
Counter c2 = new Counter();
Counter c3 = new Counter();
System.out.println(Counter.count); // 3, shared across all objects
System.out.println(c1.id); // 1
System.out.println(c2.id); // 2
```

`count` is not duplicated for every object. There is exactly one copy of it, stored with the class itself. Every object reads and writes the same shared value. `id` on the other hand is a normal instance field, each object has its own copy.

### Static methods

A static method also belongs to the class, so it can be called without creating an object, using `ClassName.methodName()`.

```java
class MathUtils {
    static int square(int x) {
        return x * x;
    }
}
int result = MathUtils.square(5); // no object needed
```

Restriction: a static method cannot directly access instance (non-static) fields or methods, because there is no specific object for it to work with. It also cannot use the `this` keyword, since `this` refers to a specific object and a static method is not tied to one.

```java
class Example {
    int x = 10;
    static void show() {
        System.out.println(x); // compile error, cannot access instance field from static context
    }
}
```

A static method CAN access other static fields and static methods directly, and it can access instance members if you give it an actual object to work through, for example `static void show(Example e) { System.out.println(e.x); }` is valid.

### Static blocks

A static block runs exactly once, when the class is first loaded into memory by the JVM, before any object is created and before the main method runs if the block is in the class containing main. It is commonly used to initialize static fields with logic more complex than a simple assignment.

```java
class Config {
    static int maxUsers;
    static {
        System.out.println("Static block running");
        maxUsers = 100;
    }
}
```

A class can have multiple static blocks, they run in the order they appear in the file, and each runs only once no matter how many objects you create afterward.

### Order of execution, a detail that is easy to get wrong

When a class is loaded and its first object is created, Java runs things in this order:
1. Static fields and static blocks run once, in the order they appear, the very first time the class is used (not repeated for later objects).
2. Instance fields and instance initializer blocks run, in the order they appear.
3. The constructor body runs.

Steps 2 and 3 repeat every time a new object is created. Step 1 never repeats.

```java
class Demo {
    static { System.out.println("A: static block"); }
    { System.out.println("B: instance block"); }
    Demo() { System.out.println("C: constructor"); }
}

new Demo();
new Demo();
// Output:
// A: static block      <- only once, ever
// B: instance block     <- for first object
// C: constructor
// B: instance block     <- for second object
// C: constructor
```

### The main method, why it must be static

```java
public static void main(String[] args) {
}
```

`main` must be static because the JVM calls it before any object of your class exists. There is no object yet to call a normal instance method on, so it has to be a class level (static) method that can be invoked directly through the class.

`String[] args` holds command line arguments passed when running the program, for example `java MyProgram hello world` makes `args[0]` equal to `"hello"` and `args[1]` equal to `"world"`. If no arguments are passed, `args` is an empty array, not null.

## The final keyword

`final` means "cannot be changed after being set". It applies differently depending on where you use it.

### final variable

A final variable can be assigned only once. After that its value is locked.

```java
final int MAX = 100;
MAX = 200; // compile error
```

Important correction on a point that is often stated too simply: **for final object references, only the reference itself is locked, not the internal state of the object it points to.** You cannot reassign a final reference to point to a different object, but you can still change the internal fields of the object it already points to, as long as those fields are not themselves final or otherwise protected.

```java
final StringBuilder sb = new StringBuilder("Hello");
sb.append(" World"); // perfectly legal, we are mutating the object, not reassigning the reference
System.out.println(sb); // "Hello World"

sb = new StringBuilder("New"); // compile error, cannot reassign a final reference
```

This is a very common point of confusion, so it is worth remembering precisely: `final` restricts reassignment of the variable, it does not make the object itself immutable.

### final method

A final method cannot be overridden by a subclass. This is used when a class wants to guarantee that a piece of behavior can never be changed by any subclass.

```java
class Vehicle {
    final void startEngine() {
        System.out.println("Engine starting");
    }
}
class Car extends Vehicle {
    // void startEngine() { } // compile error, cannot override a final method
}
```

### final class

A final class cannot be extended by any other class at all. `String`, `Integer` and other wrapper classes are final in Java, which is why you cannot create a subclass of `String`.

```java
final class Immutable { }
// class MyClass extends Immutable { } // compile error
```

## Can a static method be overridden

This is a genuinely important and often mistaught point. The short honest answer: **no, static methods are not overridden, they are hidden.** This is called method hiding, and it behaves differently from real overriding.

```java
class Parent {
    static void greet() { System.out.println("Parent"); }
}
class Child extends Parent {
    static void greet() { System.out.println("Child"); }
}

Parent p = new Child();
p.greet(); // prints "Parent", NOT "Child"
```

With true overriding (covered in the polymorphism note), the actual object's version always runs, decided at runtime. But static methods are resolved at compile time based on the reference type, not the actual object type. Here `p` is declared as type `Parent`, so `Parent.greet()` runs, even though the object is really a `Child`. If courses say static methods "can be overridden", that phrasing is misleading, the correct term is method hiding, and the runtime behavior is genuinely different from overriding.

## Common mistakes to correct

- "final variable means the value can never change" is only precisely true for primitives and for the reference itself. For a final reference to a mutable object, the object's internal state can still change.
- "static methods can be overridden" is incorrect terminology. They are hidden, and resolved by reference type at compile time, not by actual object type at runtime.
- "static blocks run every time an object is created" is wrong. They run exactly once, when the class is loaded.

## Interview questions

**Q: Can a static method access instance variables directly?**
No. It has no implicit object (no `this`) to work through. It can only access instance members if it is explicitly given an object reference as a parameter or through a variable.

**Q: What is the difference between method overriding and method hiding?**
Overriding applies to instance methods and is resolved at runtime based on the actual object type. Hiding applies to static methods and is resolved at compile time based on the reference type.

**Q: Can a final variable holding an object reference have that object's state changed?**
Yes. final only prevents the reference from being reassigned to a different object. The object's own mutable fields can still be changed through that reference.

**Q: Why is the main method static?**
Because the JVM must call it before any object of the class exists, so it needs to be callable at the class level without an instance.

**Q: When does a static block execute and how many times?**
It executes exactly once, when the class is first loaded by the JVM, before any object of the class is created.

## Quick summary

- static belongs to the class, shared across all objects, cannot use this or access instance members directly.
- static blocks run once at class loading time, before objects and before main.
- final variable locks the reference or primitive value, not necessarily the internal state of the object it points to.
- final method cannot be overridden. final class cannot be extended.
- static methods are hidden, not overridden, and are resolved at compile time by reference type.
