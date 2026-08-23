# Java Interfaces Deep Dive: Default Methods, Functional Interfaces, Marker Interfaces

This note builds on the basics of interfaces covered in the abstraction note. Here we go deeper into how interfaces evolved and the special categories of interfaces that come up constantly in real code and interviews.

## Why default methods were added

Before Java 8, every method in an interface had to be abstract, no body at all. This created a real practical problem: if a widely used interface like `List` needed a new method added to it, every single class in the world that already implemented `List` would suddenly fail to compile, since they had not implemented the new method.

Java 8 solved this with **default methods**, interface methods that provide a body, so implementing classes automatically get a working implementation for free, and are not forced to write their own unless they want to override it.

```java
interface Vehicle {
    void drive();

    default void honk() {
        System.out.println("Beep beep");
    }
}

class Car implements Vehicle {
    public void drive() {
        System.out.println("Car driving");
    }
    // honk() is not written here, but Car still has it, inherited from the interface
}

Car c = new Car();
c.honk(); // "Beep beep", works without Car implementing it
```

A class can still override a default method if it needs different behavior.

```java
class SportsCar implements Vehicle {
    public void drive() { System.out.println("Sports car driving fast"); }
    @Override
    public void honk() { System.out.println("VROOM HONK"); }
}
```

## Static methods in interfaces

Also added in Java 8, a static method in an interface belongs to the interface itself, called directly through the interface name, similar to static methods in classes. It is commonly used for utility or factory logic closely tied to the interface's purpose.

```java
interface MathOperation {
    int apply(int a, int b);

    static MathOperation addition() {
        return (a, b) -> a + b;
    }
}

MathOperation add = MathOperation.addition();
System.out.println(add.apply(2, 3)); // 5
```

## Private methods in interfaces

Added in Java 9, a private method in an interface can only be called from within other default or static methods of the same interface. It exists purely to let default methods share common logic between themselves without exposing that helper logic to implementing classes.

```java
interface Logger {
    default void logInfo(String msg) {
        log("INFO", msg);
    }
    default void logError(String msg) {
        log("ERROR", msg);
    }
    private void log(String level, String msg) { // shared helper, not visible outside the interface
        System.out.println("[" + level + "] " + msg);
    }
}
```

## The diamond problem with default methods

Since default methods can carry real implementation, a genuine version of the diamond problem can occur if a class implements two interfaces that both provide a default method with the identical signature.

```java
interface A {
    default void greet() { System.out.println("Hello from A"); }
}
interface B {
    default void greet() { System.out.println("Hello from B"); }
}
class C implements A, B {
    // must override greet(), or this will not compile
    @Override
    public void greet() {
        A.super.greet(); // you can still explicitly choose one, or write fresh logic
    }
}
```

Java does not guess which default method to use. It forces the implementing class to resolve the conflict explicitly by overriding the method itself. Inside that override, you can call a specific parent interface's version using `InterfaceName.super.methodName()`, as shown above, or write entirely new logic. This is the cleanest way Java avoids ambiguity while still allowing multiple interface inheritance.

## Functional interfaces

A functional interface is an interface with exactly one abstract method. It can have any number of default, static, or private methods, only the count of abstract methods matters, it must be exactly one. This single abstract method is often called the SAM (Single Abstract Method).

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b); // the one abstract method

    default void printInfo() { // does not count against the "one abstract method" rule
        System.out.println("A calculator interface");
    }
}
```

The `@FunctionalInterface` annotation is optional but strongly recommended. It does not change behavior, but it tells the compiler to raise an error if the interface accidentally ends up with more than one abstract method, protecting you from breaking the contract by mistake later.

Functional interfaces are important because they are exactly what lambda expressions can implement.

```java
Calculator add = (a, b) -> a + b;
Calculator multiply = (a, b) -> a * b;
System.out.println(add.calculate(2, 3));      // 5
System.out.println(multiply.calculate(2, 3)); // 6
```

### Common built in functional interfaces, extra useful info

Java's `java.util.function` package provides many ready-made functional interfaces so you rarely need to define your own for common cases.

| Interface | Abstract method | Purpose |
|---|---|---|
| `Runnable` | `void run()` | An action with no input and no output |
| `Supplier<T>` | `T get()` | Produces a value with no input |
| `Consumer<T>` | `void accept(T t)` | Consumes a value, produces no output |
| `Function<T,R>` | `R apply(T t)` | Transforms input of type T into output of type R |
| `Predicate<T>` | `boolean test(T t)` | Tests a condition, returns true or false |
| `Comparator<T>` | `int compare(T a, T b)` | Compares two objects for ordering |

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4)); // true

Function<Integer, Integer> square = n -> n * n;
System.out.println(square.apply(5)); // 25
```

### A functional interface can extend other interfaces, with a caveat

A functional interface can extend other interfaces as long as the total count of abstract methods across the whole hierarchy still comes out to exactly one. If the parent interface also declares an abstract method, and it is different from the child's, the total would be two, which breaks the functional interface rule. If the child interface re-declares the exact same abstract method (matching the parent's), it still counts as just one overall, since it is the same method being clarified, not a new one.

## Marker interfaces

A marker interface is an interface with no methods at all, empty. It exists purely to "tag" or "mark" a class as having a certain property, which other code can check for using `instanceof`, without requiring the class to implement any specific method.

```java
interface Serializable { } // no methods, this is the actual real one from java.io

class Employee implements Serializable {
    // no methods forced, but now the JVM and serialization frameworks know it opts into being serialized
}
```

Well known real examples in Java: `Serializable` (marks a class as safe to convert to a byte stream), `Cloneable` (marks a class as allowing `Object.clone()` to work correctly instead of throwing `CloneNotSupportedException`), and `Remote` (used in older RMI code).

### Marker interfaces vs annotations, worth knowing as extra context

Marker interfaces were the original way to tag classes in Java, but since Java 5 introduced annotations, most new tagging needs are done with a custom annotation instead, for example `@Deprecated` or a custom `@Entity` style annotation used by frameworks. Annotations are generally more flexible, they can carry extra data (`@interface MyTag { String value(); }`), can be applied to things other than classes (methods, fields, parameters), and do not affect the class's type hierarchy the way implementing an interface does. Marker interfaces still have one advantage annotations do not: since they affect the actual type, you can use `instanceof` to check for them at compile-checked, type-safe locations, and they show up directly in the `implements` clause making them visible from the class declaration itself. Both approaches are still used in modern Java, and it is worth knowing the tradeoff rather than assuming one has fully replaced the other.

## Common mistakes to correct

- Saying "interfaces can only declare one method total" confuses functional interfaces with interfaces in general. Only functional interfaces are restricted to one abstract method, and even they can have any number of default, static, or private methods alongside it.
- Saying marker interfaces are "outdated and replaced by annotations" is an overstatement. Both are still used in the standard library and in real code today, they serve overlapping but not identical purposes, and instanceof checks against a marker interface are genuinely useful in ways annotations are not.
- Not mentioning that default method conflicts must be resolved manually with `InterfaceName.super.method()` leaves out how the diamond problem is actually solved in practice, not just that "it does not happen".

## Interview questions

**Q: What is a functional interface? Give an example from the standard library.**
An interface with exactly one abstract method, usable as the target of a lambda expression. Examples include Runnable, Supplier, Consumer, Function, and Predicate from java.util.function.

**Q: Can a functional interface have more than one method in total?**
Yes, as long as only one of them is abstract. It can have any number of default, static, or private methods alongside that single abstract method.

**Q: How does Java resolve a conflict when a class implements two interfaces with the same default method signature?**
It does not resolve it automatically. The implementing class is forced to override the method itself, and can call a specific interface's version explicitly using InterfaceName.super.methodName() if needed.

**Q: What is a marker interface? Give two real examples.**
An interface with no methods, used to tag a class with a property that other code can check using instanceof. Serializable and Cloneable are two well known examples in the standard library.

**Q: Why were private methods added to interfaces in Java 9?**
To let default methods within the same interface share common logic through a helper method, without exposing that helper to implementing classes.

## Quick summary

- Default methods (Java 8) give interface methods a body, allowing interfaces to evolve without breaking existing implementers.
- Static methods (Java 8) belong to the interface itself, called through the interface name.
- Private methods (Java 9) let default methods share logic internally, invisible outside the interface.
- Conflicting default methods from multiple interfaces must be resolved explicitly by the implementing class using InterfaceName.super.method().
- A functional interface has exactly one abstract method (the SAM), can have other default/static/private methods, and is the target type for lambda expressions.
- A marker interface has no methods, used to tag a class for type-checking with instanceof, still used alongside annotations, not fully replaced by them.
