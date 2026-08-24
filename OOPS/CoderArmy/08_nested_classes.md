# Java Nested Classes: Static Nested, Inner, Local and Anonymous

## What is a nested class

A nested class is a class defined inside another class. The class that contains it is called the outer class. Nested classes exist to group classes that are only meaningfully used together, and to give the nested class controlled access to the outer class's private members. Java has four kinds, and they behave quite differently from each other, which is a common source of confusion.

## 1. Static nested class

A static nested class is declared with the `static` keyword inside the outer class. It behaves almost like a regular top level class, it just happens to be namespaced inside another class. It does NOT need an instance of the outer class to exist, and it cannot access the outer class's instance (non-static) members directly, only its static members.

```java
class Outer {
    static int outerStaticField = 10;
    int outerInstanceField = 20;

    static class StaticNested {
        void show() {
            System.out.println(outerStaticField); // fine, static member
            // System.out.println(outerInstanceField); // compile error, no outer instance available
        }
    }
}

Outer.StaticNested nested = new Outer.StaticNested(); // no Outer instance needed
nested.show();
```

Use case: when the nested class is logically related to the outer class, but does not need access to the outer object's specific state. `Map.Entry` in the Java standard library is a well known example of a static nested class.

## 2. Inner class (non-static nested class)

An inner class is declared without `static`. Unlike a static nested class, it DOES need an instance of the outer class to exist, because every inner class object implicitly holds a hidden reference to the specific outer object that created it. This means it CAN freely access the outer object's instance fields and methods.

```java
class Outer {
    int outerInstanceField = 20;

    class Inner {
        void show() {
            System.out.println(outerInstanceField); // fine, has implicit access to outer instance
        }
    }
}

Outer outer = new Outer();
Outer.Inner inner = outer.new Inner(); // notice the unusual syntax, needs an outer instance
inner.show();
```

The syntax `outer.new Inner()` looks unusual the first time you see it, but it makes sense once you know an inner class object cannot exist without being tied to a specific outer object.

### An important practical detail: memory implications

Because every inner class object holds a hidden reference to its outer object, keeping an inner class object alive (for example storing it in a long-lived collection) also keeps its outer object alive, even if nothing else references that outer object anymore. This can cause unexpected memory leaks in long running applications. A course explaining inner classes purely as "a way to organize code" without mentioning this hidden reference is leaving out a genuinely important practical detail. If you do not need access to the outer instance, prefer a static nested class specifically to avoid this hidden coupling.

## 3. Local class

A local class is defined inside a method body, and it is only visible within that method. It can access the fields of the outer class as usual (if the method is an instance method), and it can also access local variables and parameters of the enclosing method, but only if those variables are effectively final, meaning their value is never reassigned after being set.

```java
class Outer {
    void process() {
        int factor = 10; // effectively final, never reassigned after this line

        class Multiplier {
            int multiply(int x) {
                return x * factor; // allowed, factor is effectively final
            }
        }

        Multiplier m = new Multiplier();
        System.out.println(m.multiply(5)); // 50
    }
}
```

If you tried to reassign `factor` anywhere later in `process()`, the code inside `Multiplier` referencing `factor` would fail to compile. The reason is that the local class might outlive the method call in some cases (its object could be returned or stored elsewhere), while the method's local variables live on the stack and disappear once the method returns. To make this safe, Java internally captures a fixed copy of the variable's value at the time the class is created, which only makes sense if the variable's value never changes.

## 4. Anonymous class

An anonymous class is a local class without a name, defined and instantiated in a single expression. It is normally used to provide a one-time implementation of an interface or an abstract class, right where it is needed, without writing a full separate named class.

```java
interface Greeting {
    void greet();
}

Greeting g = new Greeting() {
    @Override
    public void greet() {
        System.out.println("Hello from an anonymous class");
    }
};
g.greet();
```

This was extremely common before Java 8 for things like event listeners and `Runnable` implementations.

```java
Runnable task = new Runnable() {
    @Override
    public void run() {
        System.out.println("Task running");
    }
};
new Thread(task).start();
```

### Anonymous classes vs lambda expressions, extra relevant context

Since Java 8, if the interface being implemented is a functional interface (exactly one abstract method, covered fully in the interfaces note), you can usually replace an anonymous class with a much shorter lambda expression.

```java
Runnable task = () -> System.out.println("Task running"); // same effect as the anonymous class above
```

It is worth knowing that lambdas and anonymous classes are not perfectly identical under the hood. A lambda does not create a genuinely new named class file at compile time the way an anonymous class does, and `this` inside a lambda refers to the enclosing outer instance, while `this` inside an anonymous class refers to the anonymous class instance itself. For most day to day interview purposes, knowing that lambdas are the modern, shorter replacement for anonymous classes implementing functional interfaces is the key takeaway. Anonymous classes are still necessary when you need to implement an abstract class, or an interface with more than one abstract method, since lambdas only work for functional interfaces.

## Comparing all four

| Type | Needs outer instance | Can access outer instance fields | Can access local variables of enclosing method | Has a name |
|---|---|---|---|---|
| Static nested | No | No (only static members) | No | Yes |
| Inner | Yes | Yes | No | Yes |
| Local | Depends on context | Yes, if non-static context | Yes, if effectively final | Yes, but scoped to the method |
| Anonymous | Depends on context | Yes, if non-static context | Yes, if effectively final | No |

## Common mistakes to correct

- Describing nested classes as purely a code organization tool undersells the real access control and memory implications, especially the outer instance reference held by inner classes.
- Saying inner classes and static nested classes are basically the same with a small syntax difference is inaccurate. Their relationship to the outer instance is fundamentally different, one holds a live reference to a specific outer object, the other does not.
- Saying lambdas are just shorthand syntax for anonymous classes with no other difference misses the `this` binding difference and the fact lambdas only apply to functional interfaces, not multi-method interfaces or abstract classes.

## Interview questions

**Q: What is the key difference between a static nested class and an inner class?**
A static nested class does not need an instance of the outer class and cannot access its instance members directly. An inner class needs an outer instance, holds an implicit reference to it, and can freely access its instance members.

**Q: Why can a local or anonymous class only access effectively final local variables of the enclosing method?**
Because the method's local variables live on the stack and may disappear after the method returns, while the local or anonymous class object might outlive that call. Java captures a fixed snapshot of the variable's value, which is only safe if the value never changes after initialization.

**Q: What memory issue can inner classes cause?**
Since every inner class instance holds a reference to its outer instance, keeping the inner instance alive keeps the outer instance alive too, which can cause memory leaks if inner instances are stored somewhere long lived unintentionally.

**Q: Can a lambda expression always replace an anonymous class?**
No, only when the anonymous class implements a functional interface with exactly one abstract method. Lambdas cannot implement an abstract class or an interface with multiple abstract methods.

## Quick summary

- Static nested class: no outer instance needed, no access to outer instance members, behaves like a namespaced top level class.
- Inner class: needs an outer instance, holds a hidden reference to it, can access outer instance members, can risk memory leaks if kept alive too long.
- Local class: defined inside a method, can access effectively final local variables of that method.
- Anonymous class: unnamed one-time class, commonly used for functional interface implementations, now often replaced by lambdas since Java 8.
